# Sprint Build Pipeline

A 7-phase sprint execution method I use with Claude Code subagents to ship a sprint of tickets through spec validation → parallel review → parallel build → quality gates → release.

> **Stack assumptions:** This template is written for a **Flutter + Firebase Cloud Functions** stack with **Jira** for tickets, **Confluence + FigJam** for test plans, and **TestFlight** for iOS distribution. The method itself is stack-agnostic — substitute your own paths, build commands, and distribution channels. Specific agent role names (`flutter-ui-builder`, etc.) assume you've defined matching Claude Code subagents in `.claude/agents/` for your project.
>
> **Placeholders:** `PROJ-XXX` = your Jira ticket prefix · `<PROJECT>` = your product/project name · `<core flow>` = a key user journey in your product.

## Input

The user provides one of:
- A list of Jira ticket IDs (e.g., `PROJ-500 through PROJ-510`)
- A sprint name (e.g., `Sprint X1-X3`)
- A feature scope description (e.g., `build the <core flow> end-to-end`)

## Pipeline

Follow this exact 7-phase sequence. Do not skip phases.

### Phase 0: Spec Validation (Sequential)

**Agent: product-spec-architect**

1. Read all Jira tickets in scope (e.g. via the Atlassian MCP `getJiraIssue` tool)
2. For each ticket, verify acceptance criteria are complete, unambiguous, and testable
3. Flag missing edge cases, incomplete API specs, vague user stories
4. Write missing specs directly into tickets (post comment or update description)
5. This phase prevents Phase 2 reviewers from guessing at intent

### Phase 1: Sprint Intake (Sequential)

**Agents: Orchestrator + cto-advisor + flutter-orchestrator**

1. **flutter-orchestrator** reads Jira tickets and maps them to files and layers automatically
2. Build a **Sprint File Map** table: ticket → files → layer (UI / logic / API / backend)
3. flutter-orchestrator identifies cross-ticket dependencies and proposes optimal build agent split
4. Flag any file conflicts (two tickets touching the same file)
5. Launch **cto-advisor** to validate architecture: no circular dependencies, correct layer boundaries, scalability, tech debt. CTO can veto or reshape the file map.
6. Propose the review groupings (Phase 2) and build agent splits (Phase 4)
7. Present to the user for approval before proceeding

### Phase 1.5: Design Gap Fill (Conditional)

**Agent: figma-design-agent**

Only runs if tickets introduce NEW screens that don't have existing Figma designs.

1. Scan ticket list for screens without Figma mockups
2. Create designs using your project's existing design system (color tokens, typography, components)
3. This gives the design reviewers in Phase 2 something concrete to evaluate against
4. Skip entirely if all screens already have designs

### Phase 2: Parallel Narrow Reviews

Launch **10 agents** in parallel, each with the ticket list but one review lens:

1. **code-maintainability-guardian** — naming conventions, interface-first pattern, sealed classes, no concrete dependencies
2. **mobile-perf-engineer** — provider rebuild scope, autoDispose correctness, Firestore read costs, lazy vs eager init
3. **premium-product-designer** — UI implementation vs Figma spec, design-token usage, dark mode, spacing
4. **flutter-api-architect** — request/response contract match between client and Cloud Functions, error codes, backward compatibility
5. **ux-flow-architect** — router paths, auth guards, back-button behavior, empty/loading states
6. **test-architect** — interfaces mockable, edge cases identified, acceptance criteria verifiable
7. **motion-delight-designer** — animation opportunities per screen, haptic feedback specs, transition choreography, RepaintBoundary placements, reduced-motion accessibility
8. **product-counsel** — GDPR/COPPA compliance, App Store Review Guidelines risks, data leaving device, content moderation, privacy policy alignment
9. **brand-identity-architect** — product persona/voice in user-facing copy, brand color palette usage, typography hierarchy, tone consistency across error messages/empty states/onboarding, no off-brand language
10. **flutter-mobile-architect** — iOS entitlements and Info.plist permissions, Android manifest and ProGuard, platform channel safety, release build config (obfuscation, tree shaking), code signing, keychain/keystore patterns, no debug-only code in release paths

Each reviewer returns findings tagged as **blocker** or **suggestion**.

### Phase 3: Review Integration (Sequential)

**Agents: Orchestrator + cto-advisor (tiebreaker)**

1. Collect all 10 review outputs
2. Deduplicate findings
3. **cto-advisor resolves conflicts** when reviewers disagree. CTO's decision is final.
4. For each blocker: write the exact fix into the ticket as a build instruction
5. Motion specs from motion-delight-designer become Phase 4.5 instructions
6. Legal blockers from product-counsel are hard gates — cannot proceed until resolved
7. Present consolidated review to user for approval

### Phase 4: Parallel Build

Launch build agents split by **layer** (not feature), with disjoint file ownership:

1. **flutter-ui-builder** — screens and widgets (`presentation/screens/`, `presentation/widgets/`)
2. **flutter-logic-architect** — state management and models (`presentation/providers/`, `domain/`)
3. **flutter-api-architect** — data layer (`data/`, `domain/` interfaces only)
4. **general-purpose** (backend) — Cloud Functions, utilities, constants, Firestore rules

**Shared file protocol:** One agent owns each shared file (e.g., `app_router.dart`, backend `index.ts`). Other agents output deferred edit instructions. Apply deferred edits after all agents complete.

Each build agent prompt MUST include:
- The tickets assigned to them with review-integrated build instructions
- Current contents of every file they'll modify
- Explicit list of files they may create/modify
- Explicit list of files they must NOT touch

### Phase 4.5: Motion & Delight Polish

**Agent: motion-delight-designer**

Runs AFTER Phase 4 build completes — needs the real built widget tree.

1. Reviews all screens created in Phase 4
2. Adds micro-interactions, transitions, haptic feedback per Phase 2 motion specs
3. All additions must respect `MediaQuery.disableAnimations` / `reduceMotion`
4. Uses `RepaintBoundary` around animated subtrees
5. Returns list of files modified for Phase 5 verification

### Phase 5: Integration, Verification & Quality Gates (Sequential)

**Agents: Orchestrator + test-architect + flutter-mobile-architect**

#### 5.A — Static analysis gate

1. Apply deferred edits to shared files
2. Run `flutter analyze lib/` — must be 0 errors, 0 warnings
3. Run `npm run build` in `functions/` — must be clean `tsc`
4. Fix any cross-agent conflicts (import mismatches, duplicate providers)
5. **flutter-mobile-architect: production readiness audit** — verifies no debug-only code leaks into release, `kDebugMode` guards correct, signing config valid
6. Re-run analysis after fixes — analysis must remain clean

#### 5.B — Test coverage gate (HARD GATE — non-negotiable)

**This phase CANNOT be marked complete without actual tests written, passing, and meeting coverage thresholds. A "verification checklist" alone is NOT acceptable — it is a complement to, not a substitute for, real automated tests.**

7. **test-architect writes / verifies tests across three layers:**
   - **Unit tests** for every new pure function, utility, data model, and reducer-style state transition.
     - Backend: `functions/test/**/*.test.ts` (or `*.spec.ts`) — runnable via `npm test` from `functions/`. Mocks Firestore via `@firebase/rules-unit-testing` or `firebase-functions-test`.
     - Flutter: `test/**/*_test.dart` — runnable via `flutter test`. Repositories tested against fakes, notifiers tested against fake repositories.
   - **Integration tests** for every new Cloud Function endpoint (request → auth → handler → Firestore → response shape) and every new Repository (HTTP client → CF → response parsing).
     - Backend: spawn `firebase emulators:start --only functions,firestore` in CI; hit endpoints via `node-fetch`; assert envelope, status codes, side-effects in emulator Firestore.
     - Flutter: integration_test directory exercising the repo + notifier path against a stubbed HTTP client.
   - **End-to-end regression tests** for every user-visible flow modified or added in the sprint, plus the existing critical flows (sign-in, `<core flow 1>`, `<core flow 2>`, `<premium/paywall flow>`, feedback submission). At minimum a Flutter `integration_test` driver script per flow.
8. **Coverage thresholds (enforced in CI):**
   - Lines: **≥ 80% on net-new code** (the diff vs. main, not whole-codebase)
   - Branches: **≥ 70% on net-new code**
   - All new public functions / endpoints: **must have at least one direct test**
   - Coverage report (`lcov`, `coverage-summary.json`) must be committed alongside the test files
9. **Run the suite end-to-end before proceeding:**
   - `cd functions && npm test` — exit code 0
   - `flutter test` — exit code 0, with `--coverage`
   - `flutter test integration_test` — exit code 0 (on simulator)
   - If any test fails, Phase 5 halts. No exceptions for "flaky" tests — fix or quarantine with a tracking ticket linked.
10. **Phase 5 cannot proceed past 5.B until all of 7-9 are green.** No "we'll add tests next sprint" — that's how technical debt accrues. If genuine scope pressure forces a deviation, it requires explicit user approval AND a follow-up ticket created BEFORE the deferral.

#### 5.C — Test plan refresh (HARD GATE — manual testers depend on this)

**The manual test plan lives in THREE synced surfaces. All three are refreshed every sprint that touches a journey. None can lag the others.**

11. **`docs/test-plan.md`** (in-repo authoritative source):
    - One section per user journey covered by the app (sign-in, `<core flow 1>`, `<core flow 2>`, admin-only flows, etc.).
    - For every ticket in this sprint that touched a journey, refresh that journey's section.
    - Each journey entry contains: **Preconditions** (account state, feature flags, data seeded), **Steps** (numbered, atomic), **Expected results** (per step), **Evidence to capture** (screenshot path / log line / DB query), **Confluence link**, **FigJam link**.
    - Add a **"Sprint X — what's new for testers"** subsection at the top with the diff from the previous sprint.
    - Plan must be runnable by a manual tester with zero engineering involvement.

12. **Confluence — one page per user journey** under your team's Confluence space:
    - Page title convention: `Manual Test — <Journey Name>` (e.g. `Manual Test — <core flow 1>`).
    - **Find the existing page first** via `searchConfluenceUsingCql` with the title. **Update it** — do NOT create a duplicate.
    - Only create a new Confluence page when this journey has genuinely never had one (i.e. brand-new feature this sprint).
    - Page mirrors the markdown plan section but adds: ticket links for the touching sprint, screenshot embeds, embedded FigJam panel.
    - Cross-link: the Confluence page links to the FigJam, the test plan markdown links to the Confluence page.

13. **FigJam — one diagram per user journey** in your team's Figma file:
    - Frame name convention: `Test Flow — <Journey Name>`.
    - **Find the existing frame first** by name search. **Update the existing frame in place** — do NOT create a new diagram each sprint.
    - Only create a new FigJam frame when this journey is genuinely new this sprint.
    - Diagram shows: entry preconditions → numbered step nodes → decision branches → success / failure terminal states → evidence-capture annotations.
    - Cross-link: the FigJam frame links to its Confluence page; the Confluence page embeds the FigJam frame URL.

14. **All three surfaces (markdown + Confluence + FigJam) are committed / saved in the same sprint as the code.** Surface drift is a sprint-blocker — Phase 5 cannot complete until all three are aligned.

#### 5.D — Slash-command quality gates (HARD GATES)

15. **Run `/review` against the cumulative diff** (`origin/main..HEAD` for the sprint):
    - Address every blocker before proceeding. Tracked tickets created for non-blocking suggestions.
    - The review output is committed to the sprint's PR / commit message.
16. **Run `/security-review` against the cumulative diff**:
    - **HIGH and MEDIUM findings are hard gates** — must be resolved or have an explicit waiver from the user before release upload.
    - LOW findings get tracked tickets but don't block.
    - The security review output is committed alongside the review output.

#### 5.E — Ship

17. Bump build number in `pubspec.yaml`
18. **Build guard check** — list all ongoing builds + their durations, confirm with the user before starting a new one. (Concurrent local release builds tend to corrupt each other; explicit confirmation prevents that class of failure.)
19. `flutter build ipa --dart-define=FLAVOR=production`
20. Upload to TestFlight via `xcrun iTMSTransporter`
21. Post: TestFlight build link + verification checklist link + test plan link + review/security-review summaries
22. Update your build changelog (a per-build file or wiki page summarizing "what shipped, what to test") with the sprint's commit references and "what to test for this build"

## Full Agent Map (16 specialized + 1 general-purpose = 17 total)

| Agent | Phase(s) | Unique contribution |
|---|---|---|
| product-spec-architect | 0 | Spec completeness — prevents ambiguous requirements |
| flutter-orchestrator | 1 | Task decomposition — maps tickets to files/layers |
| cto-advisor | 1, 3 | Architecture validation + conflict tiebreaker |
| figma-design-agent | 1.5 | Design gap fill — creates missing Figma mockups |
| code-maintainability-guardian | 2 | Naming & structure review |
| mobile-perf-engineer | 2 | Performance review |
| premium-product-designer | 2 | UI vs Figma spec review |
| flutter-api-architect | 2, 4 | API contract review + data layer build |
| ux-flow-architect | 2 | Navigation & edge case review |
| test-architect | 2, 5 | Testability review + verification checklist |
| motion-delight-designer | 2, 4.5 | Motion/haptic review + post-build polish |
| product-counsel | 2 | Legal/compliance review |
| brand-identity-architect | 2 | Brand consistency — voice, colors, typography, tone |
| flutter-mobile-architect | 2, 5 | Platform/security review + production readiness audit |
| flutter-ui-builder | 4 | Screen & widget build |
| flutter-logic-architect | 4 | State management & model build |
| general-purpose (backend) | 4 | Cloud Functions & Firestore build |

## Important Rules

- NEVER skip Phase 0 or Phase 2 reviews — they prevent ~80% of rework
- ALWAYS get user approval after Phase 1 and Phase 3 before proceeding
- Phase 1.5 is conditional — skip if all screens have Figma designs
- Build agents must have **DISJOINT file ownership** — no two agents edit the same file
- Motion polish (Phase 4.5) runs AFTER build — it needs the real widget tree
- CTO is the tiebreaker on all conflicting review findings — decision is final
- Legal blockers from product-counsel are hard gates — cannot proceed until resolved
- Run a build guard check before starting a new release build — list ongoing builds first

### Phase 5 quality gates — non-negotiable

Phase 5 has FOUR mandatory quality gates that cannot be skipped without explicit user approval AND a tracked deferral ticket:

1. **Test coverage gate (5.B)** — actual unit, integration, and end-to-end regression tests written, passing, and meeting the 80% line / 70% branch coverage threshold on net-new code. A "manual verification checklist" is NOT a substitute for automated tests — it is a complement.
2. **Test plan refresh (5.C)** — three-surface sync: `docs/test-plan.md` + Confluence page per journey + FigJam frame per journey. Existing Confluence pages and FigJam frames are UPDATED in place — never duplicated. All three surfaces committed/saved in the same sprint.
3. **`/review` gate (5.D)** — cumulative diff reviewed; blockers fixed; suggestions ticketed.
4. **`/security-review` gate (5.D)** — HIGH and MEDIUM findings resolved or explicitly waived by the user before release upload.

**Operating principle:** the goal is not "code that works" but "code that demonstrably works AND is verifiable in the future without re-reading the diff." Tests + test plan + reviews are how that demonstration is preserved.

---

## How to adapt this to your project

1. **Define the agents.** Each role above (`product-spec-architect`, `cto-advisor`, `flutter-ui-builder`, etc.) corresponds to a Claude Code subagent file in `.claude/agents/`. Write a one-paragraph charter for each agent describing its scope, the files/folders it owns, and what it must NOT do.
2. **Wire the slash command.** Save this file as `.claude/commands/sprint-plan.md` in your project. Invoke it with `/sprint-plan PROJ-500 through PROJ-510` (or a sprint name / scope description).
3. **Substitute your stack.** If you're not on Flutter + Firebase, replace:
   - `flutter analyze` / `flutter test` → your linter + test command
   - `npm run build` in `functions/` → your backend build command
   - `pubspec.yaml` build number bump → your equivalent versioning step
   - `flutter build ipa` + TestFlight → your release build + distribution channel
4. **Substitute your tools.** If you don't use Jira / Confluence / FigJam, swap to your equivalent (Linear / Notion / Whimsical / etc.) and adjust the MCP tool names accordingly.
5. **Tune the thresholds.** The 80% line / 70% branch coverage targets are an opinionated default. Pick numbers that match your team's risk tolerance, but **enforce them in CI** — a target without enforcement is a wish.
