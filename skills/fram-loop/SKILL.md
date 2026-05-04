---
name: fram-loop
description: >
  Two-agent autonomous development pattern for shipping a complete user-facing feature
  end-to-end without human intervention mid-run. A Builder agent implements each phase
  with a fresh context; a Reviewer agent scores against a rubric and runs browser-level
  E2E. Phase-based context resets prevent the model trying to wrap up early as context
  fills. Output is a finished feature a real user can use, not an MVP for a developer
  to review. Use only when the user EXPLICITLY opts in ("use the fram-loop", "run the
  fram-loop", "spin up the fram-loop", "run this through the fram-loop"). NEVER
  auto-invoke.
license: MIT
---

# fram-loop

A two-agent autonomous development system. Kick it off with a spec, a plan, and a branch — get back a fully functioning feature, ready for the dev to review the final output (not the per-phase progress). The **Builder** implements code; the **Reviewer** evaluates it; the **Orchestrator** drives the loop and never pauses to ask the human for opinions mid-run.

First reference run: Invite Form V2 (2026-03-26) — 5 phases, 17 commits, 98 tests, no human intervention beyond initial setup.

## Goal

Output is a **fully functioning feature a USER (not a developer) can use end-to-end.** Not an MVP. Not a demo. Not "here's progress, want me to continue?" Freight-train to the goal — the dev gives a kickoff and sees a finished feature.

This means:

- The loop NEVER pauses for human opinion mid-run. No "before I proceed with phase N, please confirm…"
- The loop is RESILIENT: a failed phase triggers adaptation (re-interpret spec, narrow fix scope, insert a recovery phase), not a halt.
- The loop only stops in two states: `complete` (goal met, verified end-to-end as a real user) or `blocked` (genuinely unrecoverable — spec contradicts itself, environment broken, external dependency missing).
- Halting because "5 fix rounds elapsed" is wrong. The right question is **"did I complete the goal?"** Keep going until the answer is yes (or until truly stuck).

## When to use this skill

- The user explicitly opts in
- The work is a multi-phase feature build (3+ phases is the sweet spot, 5+ shows clear value)
- A spec and a harness-shaped plan exist (or you're prepared to author them as Phase 0)
- Each phase produces something independently testable end-to-end

## When NOT to use

- Single-phase or single-task work — orchestration overhead isn't worth it
- The user hasn't explicitly opted in
- The plan doesn't have clear phase boundaries — fix the plan first
- The user wants per-phase review checkpoints — that's a different mode; this one freight-trains

---

## Prerequisites: spec + plan

The loop assumes a **reviewed spec** and a **harness-shaped plan**. If either is missing, build it first as Phase 0 — do not start the loop without them.

### No spec yet

Run `superpowers:brainstorming` (or equivalent in your environment) to converge on intent, requirements, scope, and acceptance criteria. Output is the spec document the orchestrator hands to every Builder.

### No plan yet

Run `superpowers:writing-plans` (or equivalent) with these constraints handed in up front:

- **Phases of 2–4 tasks each.** 1-task phases waste overhead. 5+ risks Builder context pressure.
- **Each phase produces something independently testable end-to-end** (UI to interact with, API to hit, data to verify).
- **Each task is small enough to split into two commits** — a test commit (Red) followed by the impl commit.
- **3+ phases minimum**; 5+ is where the loop really shines.
- **Explicit phase boundaries** with clear entry/exit criteria.
- **Server-contract changes flagged in the task description** so the Builder prompt can include the call-site enumeration check (see Defaults §3).

A plan that lacks these properties degrades the run — fix the plan before starting.

---

## Run state: plan-dir layout (memory + progress on disk)

Every run lives in a single directory. **Two things go on disk:** a per-feature `README.md` (live progress + resume guide + state machine) and per-phase `phases/phase-N/` files (memory of every prompt sent and report received). Together they make the run **resumable from a fresh session** — the conversation history is not required.

### Standard plan-dir layout

```
plan/<feature-name>/
  README.md                              # live progress: Phase Status table, state machine, resume guide, run config
  spec.md                                # human-reviewed feature spec
  plan.md                                # phased implementation plan
  phases/
    phase-N/
      builder-prompt.md                  # verbatim prompt sent to the Builder
      builder-report.md                  # Builder's return, written by orchestrator
      reviewer-prompt.md                 # verbatim prompt sent to the Reviewer
      reviewer-report.md                 # Reviewer's return, written by orchestrator
      carry-forward.md                   # non-blocking warnings the next phase must address
      fix-rounds/                        # only created if rubric failed
        round-K-builder.md
        round-K-reviewer.md
```

All files persist as audit trail. Nothing is deleted at the end of a run — these are the institutional memory of how the feature got built.

### README.md sections (required)

The README.md is the load-bearing file. Required sections, in this order:

1. **Header** — branch, started date, current status (e.g. `🏗️ PHASE 3 BUILDER_RUN`), last updated
2. **What this folder is** — one paragraph + the layout block above
3. **Run Configuration** — rubric thresholds, model routing, project skills, known test failures (see schema below)
4. **TDD discipline (this run)** — the per-task test→impl rule restated
5. **Quick resume guide (for a fresh session)** — exact steps to pick up mid-run
6. **Status state machine** — PENDING → BUILDER_RUN → BUILDER_DONE → REVIEWER_RUN → REVIEWER_DONE → PASSED | FIX_ROUND_K | BLOCKED
7. **Dispatch templates** — copy-pasteable Agent-call snippets that reference `phases/phase-N/builder-prompt.md` etc.
8. **Phase Status table** — the live progress tracker (one row per phase, updated after every transition)
9. **Operating notes** — skills always loaded, pre-existing failures, browser verification preference, "when in doubt" pointer

### Run Configuration block (in README.md)

A short structured section near the top — keeps the per-run config human-readable in the same file the orchestrator reads to resume. Markdown, not JSON.

```markdown
## Run Configuration

- **Rubric thresholds:** Functionality ≥ 4, Design Quality ≥ 4, Accessibility ≥ 3, Code Quality ≥ 4
- **Model routing:**
  - Builder (capability work): claude-opus-4-7
  - Builder (mechanical): claude-sonnet-4-6
  - Reviewer: claude-opus-4-7
  - Commit-agent: claude-haiku-4-5
- **Skills always loaded in agent prompts:** /nuxt4, /goodgest-design (UI phases), /i18n-sync (Phase 2 only)
- **Known pre-existing failures (don't conflate with regressions):**
  - 82 baseline test failures (per security audit Round 2)
  - 405 lint warnings (`no-explicit-any` in test mocks)
  - Legacy specs: `DashboardSidebar.spec.ts`, `dashboard/index.spec.ts`, `customer.spec.ts`
```

### Phase Status table (in README.md)

Updated after every state transition. One row per phase.

```markdown
| # | Phase | Status | Builder | Reviewer | Fix rounds | Last commit | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Mechanical cleanup (4 tasks) | **PASSED** | done (a886dc134) | done (acdc0ece9) — PASS, 5/5/4/5 | 0 | `81eefdf` | TDD experiment WORKED |
| 2 | i18n of validation strings (3 tasks) | **BUILDER_RUN** | dispatched 2026-05-01 14:22 | — | — | — | — |
```

### Carry-forward per phase (`phases/phase-N/carry-forward.md`)

Written when a phase passes, before advancing. Every non-blocking warning the Reviewer raised goes here. **Always written, even if empty** — an empty `carry-forward.md` with the line "No carry-forward items." is the signal the phase is closed.

Format:

```markdown
# Phase N — Carry-forward

- **CF-N.1** — `app/components/Foo.vue:47` — `vue/no-v-html` warning. Fix in next phase by replacing innerHTML with a sanitized `<DOMPurify />` wrapper.
- **CF-N.2** — `app/composables/useBar.ts:120` — duplicate request-shape definition; consolidate with the typed client in Phase N+2.
```

The next phase's Builder prompt MUST include the prior phase's carry-forward verbatim under a "Carry-forward fixes from prior reviews" section.

### Why this layout (and why no state.json)

Three properties:

1. **Resumable from a fresh session.** Read `README.md` → find the first phase row not `PASSED` → read that phase's folder → act. Conversation history not required.
2. **Inspectable.** A human peeking at the run reads markdown — no JSON parsing, no schema knowledge.
3. **Loose ledger, not a controller.** README.md is a memory of where we are, not an instruction set for what to do next. The loop should still adapt — re-interpret spec, narrow fix scope, insert a recovery phase — based on what's actually happening.

An earlier draft of this skill proposed a `state.json` file. **It is not used** — README.md already does the progress role, and a JSON file in parallel would just diverge from the markdown. Resilience over process.

---

## Architecture

```
Spec + Plan + README.md (Run Config + Phase Status)
    ↓
┌──────────────────────────────────────────────┐
│  Phase N                                      │
│                                               │
│  Orchestrator writes phases/phase-N/          │
│    builder-prompt.md, dispatches Builder      │
│  Builder reads: spec, plan (phase N tasks),   │
│    git log, prior phases' carry-forward.md,   │
│    key files                                  │
│  Builder implements tasks, commits each one   │
│  Builder terminates → orchestrator writes     │
│    builder-report.md + updates Phase Status   │
│                                               │
│  Orchestrator writes reviewer-prompt.md       │
│    (with builder-report appended), dispatches │
│  Reviewer: code review + browser E2E +        │
│    rubric scoring (1–5 per criterion)         │
│  Reviewer terminates → orchestrator writes    │
│    reviewer-report.md + updates Phase Status  │
│                                               │
│  All criteria ≥ threshold  → write            │
│    carry-forward.md, advance to Phase N+1     │
│  Any criterion fails       → write            │
│    fix-rounds/round-K-builder.md, dispatch    │
│  Truly stuck                → BLOCKED         │
└──────────────────────────────────────────────┘
    ↓ (repeat per phase)
Final verification (spec compliance + E2E user flow)
    ↓
COMPLETE
```

### Roles

- **Builder** — implements code. Full edit access. Receives spec, plan section, git history, prior carry-forward, and (on fix rounds) reviewer feedback. **Never self-evaluates.**
- **Reviewer** — evaluates the running app + code quality. Read-only code access + browser automation. Scores against the rubric. **Never implements.**
- **Orchestrator** — the parent conversation. Writes per-phase prompts/reports/carry-forward to disk, updates the README.md Phase Status table, dispatches agents. Never restarts an agent's work itself; never pauses to ask the human for opinions.

### Why context resets

Each phase gets a **fresh Builder** with a clean context window. When a Builder completes its phase and commits, it terminates. A new Builder for the next phase (or fix round) gets only:

- The spec
- The relevant plan section
- Git log (what prior phases produced)
- Prior phases' `carry-forward.md` (verbatim)
- Key file pointers — specifically: files in `git diff <base>..HEAD --name-only` from prior phases + spec-referenced files

This prevents **context anxiety** — the model trying to wrap up early as context fills. Each Builder works focused and fresh. Context resets > context compaction.

---

## Scoring Rubric

| Criterion | What's evaluated | Default threshold |
|---|---|---|
| **Functionality** | Spec features work as described, tested via browser interaction | ≥ 4/5 |
| **Design Quality** | Matches the patterns, visual language, and tone of the existing project. For projects with no documented design conventions, defer to platform defaults | ≥ 4/5 |
| **Accessibility** | ARIA, keyboard navigation, focus management, screen reader support | ≥ 3/5 |
| **Code Quality** | Lint clean, types correct, tests pass, no duplicated logic across files, framework conventions followed | ≥ 4/5 |

**All criteria must meet their threshold for a phase to pass.** Numeric scores are mandatory — Reviewer must provide 1–5 per criterion with specific evidence (file:line, screenshots), not pass/fail.

Thresholds are **project-configurable** via the README.md "Run Configuration" block. Defaults above are reasonable for a typical UI feature; a backend-only project might drop Design Quality and Accessibility from the active rubric.

---

## Phase Structure

Phases each contain 2–4 closely related tasks, produce committed code, and end with a review checkpoint.

| Size | Verdict |
|---|---|
| 1 task | Too small — orchestration overhead wins |
| **2–4 tasks** | **Right size — coherent unit, single Builder session, no context pressure** |
| 5+ tasks | Too large — context pressure + harder to act on review feedback |

---

## Reviewer Protocol

The Reviewer must:

1. **Read actual code** — never trust the Builder's claims
2. **Run tests and linters** — verify they pass; treat lint warnings as failures (`--max-warnings=0` or equivalent strictness)
3. **Interact with the running app** via browser automation — starting from the phase where UI exists
4. **Score the rubric** — numeric 1–5 per criterion with evidence
5. **Load project skills** — scan the project's `CLAUDE.md` / `AGENTS.md` for `/<skill-name>` mentions and invoke them before reviewing
6. **Flag specific issues** — file paths, line numbers, what's wrong, what the fix should be

### Reviewer prompt must include

- Spec excerpt (what was supposed to be built)
- Builder's report (what the Builder claims it built — explicitly: do NOT trust it)
- Rubric criteria with thresholds (read from README.md "Run Configuration")
- Browser automation steps for end-to-end verification
- Lint + test commands to run
- Required output format with scores table

---

## Final verification (feature-complete gate)

After the last per-phase review passes, before the loop declares `complete`:

1. **Re-read the spec.** Walk the spec section by section, not the plan. Confirm every requirement, acceptance criterion, and user-facing behaviour is delivered. The plan is implementation; the spec is what the user actually wanted.
2. **End-to-end user-flow exercise.** Drive the feature in the browser as a real user would, from entry point to completion. No back-doors, no test fixtures that bypass the UI. If it can't be used by a real user, it is not done.
3. **Run a project-appropriate security check.** This may be a security-focused subagent (e.g., `security-nuxt` if the project has it), a static analyser, a CI security job, or a manual checklist depending on what the project supports. If nothing is available, run a basic OWASP-shaped self-check (auth, input validation, SSRF, XSS, secrets in committed files).
4. **Run the full test suite, full lint, full typecheck** across the entire feature surface (not just per-phase affected files). A regression in Phase 2 caused by a Phase 5 edit should be caught here.

All four must pass before `phaseStatus: complete`. If any fail, dispatch a fix-Builder scoped to the specific gap and re-verify.

---

## Defaults applied to every run

These apply unless explicitly overridden in the README.md "Run Configuration" block.

### 1. Strict TDD discipline per task

For every task in every phase:

1. Write the test file first.
2. Commit it on its own. **Match the project's commit-message style** — check `git log` for the convention (Conventional Commits, lowercase imperatives, ticket prefixes, whatever the project does). The test MUST demonstrate a real Red state by the time the impl commit lands.
3. Then commit the implementation in a separate commit.
4. Never bundle test+impl in one commit.

#### RED-fix-forward exception (first-class)

If your first test attempt failed for the wrong reason (framework artefact, JSON-import AST issue, environment quirk), you may commit a refinement before the impl commit that fixes the test to fail for the right reason. The Reviewer checks both Red states. Acceptable when transparent and well-documented; abuse drops Code Quality below threshold.

The Reviewer verifies via `git log` that test commits precede impl commits per task. For at least one task per phase, the Reviewer also checks out the test commit and confirms it actually fails. Bundling test+impl drops Code Quality below threshold and fails the phase.

### 2. Mandatory browser smoke for any UI-touching phase

Any phase that modifies user-visible behaviour requires Reviewer-level browser verification — NOT optional, NOT substitutable by unit tests. The Builder is also expected to do a developer-level smoke before claiming done on UI-touching tasks.

Use whatever browser automation the environment provides (Claude in Chrome MCP, Playwright MCP, or equivalent). If browser automation is unavailable, the Reviewer returns `BLOCKED — browser unavailable` rather than passing on tests alone.

Real example from prior run: Phase 4 of the participants run shipped a server-contract change (PATCH requires `expectedVersion`). The Builder updated the composable and one component but missed `SectionDialog.vue` which does its own `$fetch`. Unit tests passed because the SectionDialog spec didn't assert on request body shape. Browser smoke caught it on the first edit.

### 3. Call-site enumeration when changing a server contract

When a Builder prompt asks for a server-contract change (adding a required body field, changing a return shape, renaming an endpoint, switching to optimistic locking, etc.), the prompt MUST require the Builder to enumerate every CLIENT call site that hits the changed endpoint — not just the composable wrapper.

How to enumerate is project-specific (grep the API path, list usages of a typed client, search for the endpoint name). The Builder must do it; the Reviewer must rerun the same enumeration as part of verification. Each result must be either (a) updated in the same task or (b) explicitly noted as out-of-scope with justification.

### 4. Phase sizing & timeout protocol

Phases that touch >5 tasks risk Builder/Reviewer agent timeouts (stream idle in long sessions). Two defenses:

- **Sizing:** prefer 3–4 task phases. If a phase has >5 tasks, the Builder prompt must include an explicit context-pressure escape hatch ("STOP and report `BLOCKED — context pressure after Task X`; the Orchestrator will dispatch a fresh Builder for the remaining tasks").

#### Delegation discipline (overrides "minimal own work" generalisation)

**The orchestrator never restarts an agent's work itself.** Failed/timed-out agent → spawn a fresh agent with narrowed scope. The orchestrator's job is dispatch + arbitration, not implementation or review. Pickups by the orchestrator are bounded to *finalising work that is already substantially done* — never *redoing work that was incomplete*.

#### Builder recovery (timeout mid-phase)

1. Inspect the working tree.
2. **If WIP impl is complete** (the new tests pass against it): commit the recovered impl with a message documenting the recovery. TDD ordering preserved (test commit was already in; impl commit is the recovery). Orchestrator MAY do this commit itself if the diff is small; for large diffs, dispatch a narrow commit-agent (briefed only with the diff, the test results, and the commit-message convention) to keep the orchestrator's context clean.
3. **If WIP is incomplete:** dispatch a fix-Builder with the exact remaining work scoped narrowly. The orchestrator does NOT continue the impl itself.

#### Reviewer recovery (timeout mid-review)

Inspect what the Reviewer reported before timing out. Decide by what is cheaper:

- If most of the verification is done and reported (rubric mostly scored, commands run, only the write-up missing) → finish inline. Bootstrap cost of a fresh Reviewer outweighs the small residual.
- If most of the verification is NOT done → dispatch a fresh Reviewer. Bootstrap cost beats the orchestrator running a full review and polluting its context.

When unsure: dispatch a fresh Reviewer. Default is delegation; pickup is the exception.

### 5. Model routing defaults

The loop is provider-agnostic. Defaults below; override in the README.md "Run Configuration" block.

| Role | Anthropic default | OpenAI Codex default | Notes |
|---|---|---|---|
| Builder (capability work) | claude-opus-4-7 | gpt-5.5 (fallback gpt-5.4) | UI logic, server logic, complex changes |
| Builder (mechanical) | claude-sonnet-4-6 | gpt-5.4-mini | Renames, dead-code removal, mechanical sweeps |
| **Reviewer (always)** | **claude-opus-4-7** | **gpt-5.5 (fallback gpt-5.4)** | Bad review = wasted fix round; never cheap-out |
| Commit agent (recovery) | claude-haiku-4-5 | gpt-5.4-mini | Mechanical commit-only work |

The Reviewer always gets the best available model in the environment. Cheap-out on the Builder where the work is mechanical; never cheap-out on review.

### 6. Goal-completion vs fix-round-cap

There is no fixed fix-round cap. The loop continues until one of:

- `complete` — goal met, final verification passes
- `blocked` — genuinely unrecoverable (spec contradicts itself, environment broken, external dependency missing)

Replace "have I exceeded N rounds?" with **"did I complete the goal?"** If the answer is no and a path forward exists (even if tortuous), continue. If the answer is no and there is no path forward, return BLOCKED with a specific diagnosis.

The fix-round count is logged in the README.md Phase Status table for retrospective analysis but is not a halting condition.

### 7. Resilience over rigidity

The loop should ADAPT, not HALT, when something doesn't work cleanly:

- A phase fails repeatedly with the same root cause → re-interpret the spec section, consider whether the plan's task split is wrong, possibly insert a recovery phase
- A test consistently fails for an environmental reason → add it to README.md "Known pre-existing failures" and continue
- A new constraint surfaces mid-run (an API limit, a dependency conflict) → record in carry-forward, narrow the next Builder's scope, continue
- A Reviewer keeps failing the same criterion despite mechanical fixes → re-examine whether the rubric threshold is too tight for this specific surface, or whether the underlying expectation is unrealistic

Stuck means truly stuck — not "this is harder than I expected."

---

## Templates

### Orchestrator pre-flight checklist

- [ ] Spec exists and reviewed by the human (else: `superpowers:brainstorming` first)
- [ ] Plan exists, harness-shaped (2–4 tasks per phase, each phase testable, tasks splittable into test+impl) — else: `superpowers:writing-plans` with constraints
- [ ] Plan dir created at `plan/<feature>/` with `spec.md` + `plan.md`
- [ ] `README.md` scaffolded with all required sections (header, layout block, Run Configuration with rubric thresholds + model routing + project skills scanned from CLAUDE.md/AGENTS.md + known test failures, TDD discipline, resume guide, state machine, dispatch templates, empty Phase Status table, operating notes)
- [ ] Permissions configured (`Bash(*)`, browser MCP tools, etc.)
- [ ] Agent mode set to autonomous operation (`bypassPermissions` or equivalent)
- [ ] Dev server running (do not restart it)
- [ ] Authenticated browser session available if the feature needs auth
- [ ] **TDD discipline confirmed default-on for every task**
- [ ] **Phase sizing checked** — no phase >5 tasks without an escape hatch in the Builder prompt
- [ ] Reviewer prompt template will pull thresholds from README.md "Run Configuration" (not hard-coded into the skill template)

### Builder prompt structure

```
You are a Builder agent implementing Phase N (Tasks X–Y) of [Feature].

## Your Mission
[What to build, where to work, what branch]

## Before You Begin
1. Read the spec: [path]
2. Read the plan: [path] — focus on Phase N
3. Check git log for prior work
4. Read the prior phase's carry-forward fixes: [inline contents of phases/phase-(N-1)/carry-forward.md]
5. Read these files (modified by prior phases, auto-generated from `git diff <base>..HEAD --name-only`): [list]

## Critical Rules
- [Project-specific rules from CLAUDE.md / AGENTS.md]
- [Framework conventions from project skills]
- **Strict TDD per task: test commit (Red) → impl commit. Match project commit-message style. Never bundle.** RED-fix-forward refinement allowed once if the first test failed for the wrong reason.
- Dev server is running — do NOT restart
- **For server-contract changes:** enumerate every client call site of the changed endpoint and update or justify each.
- **Context-pressure escape hatch:** if you feel context limits hitting, STOP and report `BLOCKED — context pressure after Task X`.

## Tasks
[Reference each task by spec/plan section — do NOT duplicate code into the prompt. Each task entry: which file gets the test, which file gets the impl, acceptance criteria.]

## Browser smoke (UI-touching tasks only)
For any task that modifies user-visible behaviour, after the impl commit drive the live UI to confirm the integration end-to-end. Inspect network requests for any new request-shape changes. Read console for new errors.

## Report Format
Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
Per-task: test SHA → impl SHA, test results, files changed, browser-smoke notes, any concerns.
```

### Reviewer prompt structure

```
You are a Reviewer evaluating Phase N of [Feature].

## IMPORTANT: Invoke [project skills from README.md "Run Configuration"] before reviewing.

## What Was Built
[Summary from the plan's Phase N]

## What the Builder Claims
[Builder's report — DO NOT TRUST IT, verify everything]

## Rubric Thresholds (from README.md "Run Configuration")
[Functionality ≥ X, Design Quality ≥ Y, Accessibility ≥ Z, Code Quality ≥ W]

## Verification Steps
1. **TDD discipline check:** `git log --oneline -<N>` and verify test→impl ordering per task. Spot-check at least one task by checking out its test commit and confirming it fails (Red was real). RED-fix-forward refinements acceptable if documented.
2. Run tests: [project commands]
3. Run lint with `--max-warnings=0` (or equivalent strictness)
4. Run typecheck (filtered to touched surface if the project supports it)
5. Read code: [specific files and what to check]
6. **For server-contract changes:** rerun the call-site enumeration; confirm every result was updated or justified.
7. **Browser smoke (MANDATORY for UI-touching phases):** drive the live UI via browser automation. If browser is unreachable, return `BLOCKED — browser unavailable`.

## RUBRIC SCORING (MANDATORY)

| Criterion | Threshold | Score (1–5) | Evidence |
|---|---|---|---|
| Functionality | ≥ X | __ | __ |
| Design Quality | ≥ Y | __ | __ |
| Accessibility | ≥ Z | __ | __ |
| Code Quality | ≥ W | __ | __ |

## Report Format
- Scores table (above)
- TDD compliance per task with SHAs + Red verification
- Verification commands run
- Issues list with severity / file:line / what's wrong / what to fix
- Carry-forward fixes (any warnings/issues to feed into the next phase's Builder)
- Verdict: PASS | FAIL (which criterion blocked it)
```

---

## Reference: prior run cost profile

Invite Form V2 (2026-03-26) — 5 phases, 14 agent invocations total, 3 fix rounds. Phases 1 and 3 (mechanical tasks) passed first try. Phases 2, 4, 5 (UI-heavy) needed exactly 1 fix round each. Use this as a rough baseline when sizing your own run.
