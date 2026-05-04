---
name: fram-loop
description: >
  Two-agent autonomous loop (Builder + Reviewer) for shipping any long-running
  coding deliverable end-to-end across multiple phases without mid-run human input —
  features, refactors, migrations, library design, perf, large cleanups. Output is a
  finished deliverable on an isolated harness branch with a PR opened back to the
  source branch. Use only when the user EXPLICITLY opts in ("use the fram-loop",
  "run the fram-loop", "spin up the fram-loop", "run this through the fram-loop").
  NEVER auto-invoke.
license: MIT
compatibility: >
  Requires parallel sub-agent dispatch, git, gh CLI for PR creation, autonomous-
  execution mode (Claude Code bypassPermissions or Codex full filesystem/network
  access), and a verification driver appropriate to the deliverable (browser
  automation MCP for UI; HTTP client / CLI shell / test database / library-
  consumer harness for non-UI work). The runtime needed to exercise the
  deliverable end-to-end (dev server, test DB, fresh shell, etc.) must be
  available before dispatch.
allowed-tools: Bash(git:*) Bash(gh:*) Bash(npm:*) Bash(npx:*) Bash(pnpm:*) Bash(yarn:*) Bash(bun:*) Bash(make:*) Bash(ps:*) Bash(lsof:*) Bash(curl:*) Read Write Edit Glob Grep Agent TodoWrite
---

# fram-loop

A two-agent autonomous development system for any long-running coding work — features, refactors, migrations, performance projects, library design, large cleanups. Kick it off with a spec and a plan from the branch you want the work to land back on; the orchestrator creates an isolated harness branch off that tip and opens a PR back to it when done. You review the final output, not the per-phase progress. The **Builder** implements; the **Reviewer** evaluates; the **Orchestrator** drives the loop and never pauses to ask the human for opinions mid-run.

## Goal

Output is **the deliverable described in the spec, fully working end-to-end as the spec describes.** Not an MVP. Not a demo. Not "here's progress, want me to continue?" The shape of "fully working" depends on what's being built: a UI feature is one a real user can drive from entry-point to completion; an HTTP API is one whose endpoints return the right shape under real calls; a library is one whose public API works as documented in a fresh consumer; a migration is one that runs cleanly on a non-prod copy with rollback verified; a refactor is one that preserves behaviour and meets the spec's structural goal. Freight-train to that goal — the dev gives a kickoff and sees a finished deliverable on its own branch with a PR opened back to the source branch.

This means:

- The loop NEVER pauses for human opinion mid-run. No "before I proceed with phase N, please confirm…" (Except as documented in §Defaults #9 for ephemeral/eval push targets, where final push is gated.)
- The loop is RESILIENT: a failed phase triggers adaptation (re-interpret spec, narrow fix scope, insert a recovery phase), not a halt.
- The loop only stops in three states: `complete` (goal met, verified end-to-end in production-shape, PR opened), `partial` (locally verified, but push/PR creation failed and a local PR handoff was written), or `blocked` (genuinely unrecoverable — spec contradicts itself, environment broken, external dependency missing after workaround attempts).
- Halting because "5 fix rounds elapsed" is wrong. The right question is **"did I complete the goal?"** Keep going until the answer is yes (or until truly stuck).

## When to use this skill

- The user explicitly opts in
- The work is a multi-phase deliverable (3+ phases is the sweet spot, 5+ shows clear value) — feature, refactor, migration, library design, perf project, large cleanup
- A spec exists, and a plan exists or you're prepared to author/reshape one before kickoff (see Prerequisites for the harness-shape constraints the plan must satisfy)
- Each phase produces something independently testable end-to-end

## When NOT to use

- Single-phase or single-task work — orchestration overhead isn't worth it
- The user hasn't explicitly opted in
- The plan doesn't have clear phase boundaries — fix the plan first
- The user wants per-phase review checkpoints — that's a different mode; this one freight-trains

---

## Prerequisites: spec + plan

The loop **executes** a reviewed spec and a harness-shaped plan; it does not author either. If the spec is missing, the plan is missing, or the plan isn't yet in harness shape, build or reshape it as preparation work before kickoff. Both are human-reviewed before the loop starts.

### No spec yet

Run `superpowers:brainstorming` (or equivalent in your environment) to converge on intent, requirements, scope, and acceptance criteria. Output is the spec document the orchestrator hands to every Builder.

### No plan yet

Run `superpowers:writing-plans` (or equivalent) with these constraints handed in up front:

- **Phases of 2–4 tasks each.** 1-task phases waste overhead. 5+ risks Builder context pressure.
- **Each phase produces something independently testable end-to-end** (UI to interact with, API to hit, data to verify).
- **Each task is small enough to split into two commits** — a test commit (Red) followed by the impl commit.
- **3+ phases minimum**; 5+ is where the loop really shines.
- **Explicit phase boundaries** with clear entry/exit criteria.
- **Interface-boundary changes flagged in the task description** so the Builder prompt can include the call-site enumeration check (see Defaults §3).

A plan that lacks these properties degrades the run — fix the plan before starting.

---

## Run state: plan-dir layout (memory + progress on disk)

Every run lives in a single directory. **Two things go on disk:** a per-feature `RUN.md` (live progress + resume guide + state machine) and per-phase `phases/phase-N/` files (memory of every prompt sent and report received). Together they make the run **resumable from a fresh session** — the conversation history is not required.

### Standard plan-dir layout

```
plan/<feature-name>/
  RUN.md                              # live progress: Phase Status table, state machine, resume guide, run config
  spec.md                                # human-reviewed feature spec
  plan.md                                # phased implementation plan
  baselines/
    baseline.md                          # Phase 0 command matrix + comparison rules
    lint.txt                             # full lint output at source branch tip
    test.txt                             # full test output at source branch tip
    typecheck.txt                        # full typecheck output at source branch tip
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

### RUN.md sections (required)

The RUN.md is the load-bearing file. Required sections, in this order:

1. **Header** — branch, started date, current status (e.g. `🏗️ PHASE 3 BUILDER_RUN`), last updated
2. **What this folder is** — one paragraph + the layout block above
3. **Run Configuration** — source branch, harness branch, PR target, rubric thresholds, model routing, project skills, baseline commands, known test failures (see schema below)
4. **TDD discipline (this run)** — the per-task test→impl rule restated
5. **Quick resume guide (for a fresh session)** — exact steps to pick up mid-run
6. **Status state machine** — PENDING → BUILDER_RUN → BUILDER_DONE → REVIEWER_RUN → REVIEWER_DONE → PASSED | FIX_ROUND_K | FINAL_VERIFY | PR_OPEN | PARTIAL | BLOCKED
7. **Dispatch templates** — copy-pasteable Agent-call snippets that reference `phases/phase-N/builder-prompt.md` etc.
8. **Phase Status table** — the live progress tracker (one row per phase, updated after every transition)
9. **Operating notes** — skills always loaded, pre-existing failures, verification driver preference (browser, HTTP client, CLI shell, test DB, etc.), "when in doubt" pointer

### Run Configuration block (in RUN.md)

A short structured section near the top — keeps the per-run config human-readable in the same file the orchestrator reads to resume. Markdown, not JSON. The example below is for a Nuxt UI feature; rubric, baseline commands, and skills swap for your stack and deliverable shape.

```markdown
## Run Configuration

- **Source branch:** feature/current-work
- **Source SHA:** 123abcd
- **Harness branch:** fram/feature-current-work-checkout-redesign-123abcd
- **PR target:** feature/current-work
- **Rubric (active criteria + thresholds):** Functionality ≥ 4, Code Quality ≥ 4, Design Quality ≥ 4, Accessibility ≥ 3
- **Verification driver:** browser automation (Claude in Chrome MCP)
- **Baseline commands:**
  - Lint: `npm run lint -- --max-warnings=0`
  - Tests: `npm run test -- --run`
  - Typecheck: `npm run typecheck`
- **Model routing:**
  - Builder (capability work): claude-opus-4-7
  - Builder (mechanical): claude-sonnet-4-6
  - Reviewer: claude-opus-4-7
  - Commit-agent: claude-haiku-4-5
- **Skills always loaded in agent prompts:** /nuxt4, /design-system (UI phases), /i18n-sync (Phase 2 only)
- **Known pre-existing failures (don't conflate with regressions):**
  - 82 baseline test failures (per prior security audit)
  - 405 lint warnings (`no-explicit-any` in test mocks)
  - Legacy specs (don't fail the run if these are already broken at baseline)
```

### Setup Phase 0: branch + baseline

Before dispatching Phase 1, the Orchestrator creates an isolated harness branch from the current branch tip:

1. Resolve `source_branch=$(git rev-parse --abbrev-ref HEAD)` and `source_sha=$(git rev-parse --short HEAD)`.
2. Create `harness_branch=fram/<source-leaf>-<feature-slug>-<source-sha>` from that exact source tip. Sanitize slashes in the source leaf (`feature/foo` → `feature-foo`). If the branch exists locally or on origin for the same source SHA, reuse/resume it; if it exists for a different SHA, mint a new branch with the new SHA. Never force-push.
3. Set the PR target to `source_branch` (not `main` unless the user explicitly started from `main` and project rules allow it).
4. Capture baseline outputs in `plan/<feature>/baselines/` before any feature edits. Record command, exit code, timestamp, source branch, source SHA, and output path in `baseline.md`.

The branch is load-bearing: Red test commits, recovery commits, plan artifacts, and implementation commits all live on the harness branch. The user's source branch is left untouched. Final delivery is a PR from the harness branch back to the source branch.

Baseline comparison rule: Reviewers verify **no new debt**, not "the full repo became clean." For each phase:

- Touched files must pass scoped lint with strict warnings (`npx eslint <touched-files> --max-warnings=0` or project equivalent).
- Full lint/test/typecheck may fail only if the failure set is at-or-below the Phase 0 baseline and no failing diagnostic points at touched files unless it already existed at baseline.
- Any new full-suite failure, new lint diagnostic, or new type diagnostic introduced by the harness branch fails Code Quality, even if the command already failed at baseline.
- If a phase intentionally fixes baseline debt, update `baseline.md` with the new lower count and the commit that improved it. Never raise the baseline silently.

### Phase Status table (in RUN.md)

Updated after every state transition. One row per phase.

```markdown
| # | Phase | Status | Builder | Reviewer | Fix rounds | Last commit | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Mechanical cleanup (4 tasks) | **PASSED** | done (a886dc134) | done (acdc0ece9) — PASS, 5/5/4/5 | 0 | `81eefdf` | All criteria above threshold |
| 2 | i18n of validation strings (3 tasks) | **BUILDER_RUN** | dispatched 2026-05-01 14:22 | — | — | — | — |
```

### Carry-forward per phase (`phases/phase-N/carry-forward.md`)

Written when a phase passes, before advancing. Every non-blocking warning the Reviewer raised goes here. **Always written, even if empty** — an empty `carry-forward.md` with the line "No carry-forward items." is the signal the phase is closed.

Each CF entry carries an explicit lifecycle state in square brackets. Format:

```markdown
# Phase N — Carry-forward

- **CF-N.1** [Open] — `app/components/Foo.vue:47` — `vue/no-v-html` warning. Fix in next phase by replacing innerHTML with a sanitized `<DOMPurify />` wrapper.
- **CF-N.2** [Deferred-to-Phase-N+2] — `services/payments/handlers.go:120` — duplicate request-shape definition; consolidate when the typed client lands. Reason: requires the typed client from Phase N+2 plan section 3.
- **CF-N.3** [Out-of-scope-v2] — `src/legacy/auth.ts:*` — legacy session-token shape; addressing requires the auth rewrite tracked in spec §11. Out of scope for this run.
```

State machine:

```
[Open]  →  picked up by next Builder
   │
   ├──→  [Addressed-in-Phase-K]   (CF closed; cited in Builder report's CF-resolution section)
   ├──→  [Deferred-to-Phase-K]    (carried forward unchanged; reason required)
   └──→  [Out-of-scope-v2]        (explicitly removed from this run; reason + spec/issue link required)
```

Rules:

- Every CF starts as `[Open]` when first emitted by a Reviewer.
- The next Builder's prompt requires explicit handling of every `[Open]` CF — Address, Defer, or move Out-of-scope (with reason).
- A `[Deferred-to-Phase-K]` CF reappears in Phase K's carry-forward unchanged until Addressed.
- An `[Out-of-scope-v2]` CF still exists in the run's history but does not propagate further.
- The Orchestrator audits at phase close: an `[Open]` CF still open after the next Builder finishes is a tracking bug, not a feature; flag in RUN.md.

The next phase's Builder prompt MUST include the prior phase's carry-forward verbatim under a "Carry-forward fixes from prior reviews" section, and the Builder's report MUST resolve each `[Open]` entry to one of the three terminal states.

### Why this layout

Three properties:

1. **Resumable from a fresh session.** Read `RUN.md` → find the first phase row not `PASSED` → read that phase's folder → act. Conversation history not required.
2. **Inspectable.** A human peeking at the run reads markdown — no JSON parsing, no schema knowledge.
3. **Loose ledger, not a controller.** RUN.md is a memory of where we are, not an instruction set for what to do next. The loop should still adapt — re-interpret spec, narrow fix scope, insert a recovery phase — based on what's actually happening.

---

## Architecture

Each phase is a Builder→Reviewer round-trip on disk:

1. **Orchestrator** writes `phases/phase-N/builder-prompt.md`, dispatches Builder.
2. **Builder** reads spec + plan (phase-N tasks) + git log + prior `carry-forward.md` + key files. Implements tasks (test commit → impl commit per task) and terminates.
3. **Orchestrator** writes `builder-report.md`, then `reviewer-prompt.md`, dispatches Reviewer.
4. **Reviewer** reads code, runs lint/test/typecheck against the Phase 0 baseline, exercises the deliverable end-to-end (browser MCP for UI; HTTP/CLI/DB/library invocation for non-UI — see Defaults §2), and scores the rubric (1–5 per criterion with file:line evidence). Terminates.
5. **Orchestrator** writes `reviewer-report.md`, updates the Phase Status table.
6. All criteria ≥ threshold → write `carry-forward.md`, advance to Phase N+1. Any fails → write `fix-rounds/round-K-builder.md`, dispatch fix-Builder. Truly stuck → BLOCKED.

After the last phase passes, the orchestrator runs final verification (spec re-walk, end-to-end exercise of the deliverable, security check, full suite/lint/typecheck against baseline, push, PR creation) before declaring `complete`.

### Roles

- **Builder** — implements code. Full edit access. Receives spec, plan section, git history, prior carry-forward, and (on fix rounds) reviewer feedback. **Never self-evaluates.**
- **Reviewer** — evaluates the running deliverable + code quality. Read-only code access + verification tools (browser automation for UI; HTTP / CLI / DB / library-consumer invocation for non-UI). Scores against the rubric. **Never implements.**
- **Orchestrator** — the parent conversation. Writes per-phase prompts/reports/carry-forward to disk, updates the RUN.md Phase Status table, dispatches agents. Never restarts an agent's work itself; never pauses to ask the human for opinions.

**Commit boundary.** Builders commit only source/test/config changes per the per-task TDD pattern (one test commit + one impl commit per task). The Orchestrator commits the run-state artefacts (`builder-report.md`, `reviewer-prompt.md`, `reviewer-report.md`, `carry-forward.md`, RUN.md state-table updates, baseline refreshes) as a single phase-closure commit after the Reviewer passes. This keeps the Builder's commit graph a clean record of feature work, and the Orchestrator's a clean record of process.

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

The rubric is **two universal criteria + one or more deliverable-shape criteria.** Numeric 1–5 scores are mandatory; Reviewer must give specific evidence (file:line, screenshots, command output, log excerpts) per criterion, not pass/fail. **All active criteria must meet their threshold for a phase to pass.** Don't score criteria that don't apply — scoring Accessibility on a CLI run dilutes the signal.

### Universal (every run)

| Criterion | What's evaluated | Default threshold |
|---|---|---|
| **Functionality** | The spec's acceptance criteria work end-to-end against the deliverable in production-shape | ≥ 4/5 |
| **Code Quality** | Touched files lint-clean, types correct, tests pass, no duplicated logic across files, language/framework conventions followed | ≥ 4/5 |

### Deliverable-shape criteria (pick what applies)

| Deliverable | Suggested additional criteria |
|---|---|
| UI / web app | Design Quality (matches existing visual language; defaults for greenfield) ≥ 4 ; Accessibility (ARIA, keyboard, focus, screen reader) ≥ 3 |
| HTTP API / service | Contract Compliance (request/response shape, status codes, error semantics match spec) ≥ 4 ; Observability (logging, metrics, traces match project conventions) ≥ 3 |
| Library / SDK | API Ergonomics (signatures, naming, docstrings, runnable examples) ≥ 4 ; Backward Compatibility (semver discipline; deprecations explicit) ≥ 4 |
| CLI tool | UX (flag naming, help output, error messages, exit codes) ≥ 4 ; Cross-platform behaviour (where promised in spec) ≥ 3 |
| DB / schema migration | Data Integrity (no row loss, invariants preserved, rollback verified) ≥ 5 ; Performance (lock window, replication impact) ≥ 4 |
| Refactor / cleanup | Behaviour Preservation (existing tests pass + hand-driven regression of touched flows) ≥ 5 ; Structural Improvement (the spec's structural goal is achieved measurably) ≥ 4 |
| Performance project | Performance Target (the spec's metric meets the spec's threshold under representative load) ≥ 5 ; Regression Safety (no measurable regression elsewhere) ≥ 4 |

The active criteria + thresholds are declared in the RUN.md "Run Configuration" block. The Reviewer scores only the active set.

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
2. **Run tests and linters** — verify scoped touched-file lint is clean; compare full lint/test/typecheck against Phase 0 baselines and fail only on new debt/regressions
3. **Exercise the deliverable end-to-end** in production-shape — browser for UI, real HTTP calls for APIs, fresh-shell invocation for CLIs, a tiny consumer for libraries, a non-prod copy for migrations (see Defaults §2). Starts from the phase where observable behaviour exists.
4. **Score the rubric** — numeric 1–5 per criterion with evidence
5. **Load project skills** — scan the project's `CLAUDE.md` / `AGENTS.md` for `/<skill-name>` mentions and invoke them before reviewing
6. **Flag specific issues** — file paths, line numbers, what's wrong, what the fix should be

### Reviewer prompt must include

- Spec excerpt (what was supposed to be built)
- Builder's report (what the Builder claims it built — explicitly: do NOT trust it)
- Rubric criteria with thresholds (read from RUN.md "Run Configuration")
- Baseline command matrix + Phase 0 baseline outputs from `baselines/baseline.md`
- End-to-end verification steps appropriate to the deliverable shape (browser automation for UI; real invocation for non-UI — see Defaults §2)
- Lint + test commands to run
- Required output format with scores table

---

## Final verification and PR handoff

After the last per-phase review passes, before the loop declares `complete`:

1. **Re-read the spec.** Walk the spec section by section, not the plan. Confirm every requirement, acceptance criterion, and spec-defined behaviour is delivered. The plan is implementation; the spec is what the user actually wanted.
2. **End-to-end exercise of the deliverable.** Drive it as it will actually be used: a UI feature in the browser as a real user, an API via real HTTP calls against the running service, a CLI from a fresh shell, a library via a tiny consumer importing the public API, a migration on a non-prod copy with rollback verified. No back-doors, no test fixtures that bypass the real surface. If it can't be used in production-shape, it is not done.
3. **Run a project-appropriate security check.** This may be a security-focused subagent (e.g., `security-nuxt` if the project has it), a static analyser, a CI security job, or a manual checklist depending on what the project supports. If nothing is available, run a basic OWASP-shaped self-check (auth, input validation, SSRF, XSS, secrets in committed files).
4. **Run the full test suite, full lint, full typecheck** across the entire feature surface (not just per-phase affected files). Compare to Phase 0 baselines; any new debt blocks completion. A regression in Phase 2 caused by a Phase 5 edit should be caught here.
5. **Commit final run artifacts** — `RUN.md` state, phase reports, carry-forward files, baseline notes, final verification notes.
6. **Push the harness branch and open a PR** back to the source branch. The PR body must include: spec path, plan path, final verification summary, end-to-end smoke walkthrough notes (driver-appropriate), baseline comparison summary, known residual risks, and exact commands a reviewer should run before merge.

All six must pass before `phaseStatus: complete`. If verification fails, dispatch a fix-Builder scoped to the specific gap and re-verify. If push or PR creation fails after the feature is locally verified, return `partial` with the exact branch name, target branch, local commit SHA, PR title/body draft, and the commands to retry (`git push -u origin <harness_branch>` and `gh pr create --base <source_branch> --head <harness_branch> ...`). Do not call the feature `complete` until the PR exists.

---

## Defaults applied to every run

These apply unless explicitly overridden in the RUN.md "Run Configuration" block.

### 1. Strict TDD discipline per task

For every task in every phase:

1. Write the test file first.
2. Commit it on its own. **Match the project's commit-message style** — check `git log` for the convention (Conventional Commits, lowercase imperatives, ticket prefixes, whatever the project does). The test MUST demonstrate a real Red state by the time the impl commit lands.
3. Then commit the implementation in a separate commit.
4. Never bundle test+impl in one commit.

#### RED-fix-forward exception (first-class)

If your first test attempt failed for the wrong reason (framework artefact, JSON-import AST issue, environment quirk), you may iterate on the test before the impl commit lands. RFF requires **either** form, not casual prose claims:

- **(a) Separate refinement commit.** The original failing-test commit stays in the log; a follow-up commit rewrites the test to fail for the right reason; then the impl commit. The Reviewer reads both Red states from `git log`.
- **(b) Proof file.** Write `phases/phase-N/red-proof-task-X.md` containing: the diff of the test rewrite, the command + output showing the original wrong-reason failure, the command + output showing the corrected right-reason failure, and a one-line reason. Then a single test commit (the corrected version) followed by the impl commit.

The Reviewer accepts either form. Absence of both — i.e. the Builder iterated locally and only committed the final test — fails Code Quality, even if the final Red was real. RFF abuse (frequent rewrites without proof, or rewrites that change what's being tested rather than how) drops Code Quality below threshold.

The Reviewer verifies via `git log` that test commits precede impl commits per task. For at least one task per phase, the Reviewer checks out the final test commit and confirms it actually fails. Bundling test+impl drops Code Quality below threshold and fails the phase.

Because all work happens on the harness branch, Red commits do not pollute the user's source branch. Do not bypass hooks with `--no-verify`. If a repo hook blocks a Red commit because the targeted test fails, record the Red proof in `phases/phase-N/red-proof-task-X.md` (same shape as form (b) above), then commit the test+impl together with a `TDD-HOOK-EXCEPTION:` footer naming the hook and proof file. The Reviewer treats this as acceptable only when the proof is complete and the final commit is green.

### 2. Mandatory end-to-end smoke against the deliverable

Any phase that changes observable behaviour requires Reviewer-level end-to-end verification of the deliverable as it'll actually be used — NOT optional, NOT substitutable by unit tests. The Builder also does a developer-level smoke before claiming done.

The smoke driver matches the deliverable:

| Deliverable | Smoke driver |
|---|---|
| UI / web app | Browser automation (Claude in Chrome MCP, Playwright MCP, or equivalent) |
| HTTP API / service | `curl`, `httpie`, or a typed client invoking real endpoints against the running service |
| CLI tool | Run the binary with real flags from a fresh shell |
| Library / SDK | A tiny consumer that imports the public API and exercises it |
| DB / schema migration | Apply against a non-prod snapshot; verify forward + rollback + invariants |
| Refactor / cleanup | Rerun the existing behaviour-asserting test suite plus a hand-driven regression of the touched flows |

If the smoke driver is unavailable for a phase that needs it, the Reviewer returns `BLOCKED — verification driver unavailable` rather than passing on unit tests alone.

Common failure mode this defends against: a change to any interface boundary (HTTP endpoint shape, exported function signature, CLI flag schema, library API) where the Builder updates the obvious caller but misses a less-frequent caller that depends on the old shape. Unit tests pass because that caller's spec didn't assert on the changed surface. End-to-end smoke catches it on the first edit.

### 3. Call-site enumeration when changing an interface boundary

When a Builder prompt asks for a change to any interface boundary that other code depends on — HTTP endpoint shape, exported function or method signature, CLI flag schema, database column shape, message format, public library API — the prompt MUST require the Builder to enumerate every dependent call site, not just the obvious wrapper.

How to enumerate is project-specific (grep the path/symbol, list usages of a typed client, search for the endpoint or function name, query the schema graph). The Builder must do it; the Reviewer must rerun the same enumeration as part of verification. Each result must be either (a) updated in the same task or (b) explicitly noted as out-of-scope with justification.

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

The loop is provider-agnostic. Defaults below; override in the RUN.md "Run Configuration" block.

| Role | Anthropic default | OpenAI Codex default | Notes |
|---|---|---|---|
| Builder (capability work) | claude-opus-4-7 | gpt-5.5 (fallback gpt-5.4) | Complex application logic, contract design, architecture decisions |
| Builder (mechanical) | claude-sonnet-4-6 | gpt-5.4-mini | Renames, dead-code removal, mechanical sweeps |
| **Reviewer (always)** | **claude-opus-4-7** | **gpt-5.5 (fallback gpt-5.4)** | Bad review = wasted fix round; never cheap-out |
| Commit agent (recovery) | claude-haiku-4-5 | gpt-5.4-mini | Mechanical commit-only work |

The Reviewer always gets the best available model in the environment. Cheap-out on the Builder where the work is mechanical; never cheap-out on review.

### 6. Goal-completion vs fix-round-cap

There is no fixed fix-round cap. The loop continues until one of:

- `complete` — goal met, final verification passes, harness branch pushed, PR opened
- `partial` — feature is locally verified, but network/auth/remote failure prevented push or PR creation; local PR handoff written
- `blocked` — genuinely unrecoverable (spec contradicts itself, environment broken, external dependency missing after workaround attempts)

Replace "have I exceeded N rounds?" with **"did I complete the goal?"** If the answer is no and a path forward exists (even if tortuous), continue. If the answer is no and there is no path forward, return BLOCKED with a specific diagnosis.

The fix-round count is logged in the RUN.md Phase Status table for retrospective analysis but is not a halting condition. When stuck, suspect misframing first: the plan's task split is wrong, the spec section is ambiguous, the rubric threshold doesn't fit this surface, a dependency conflict surfaced mid-run. All cheap to fix once recognized — see §7.

### 7. Resilience over rigidity

The loop should ADAPT, not HALT, when something doesn't work cleanly:

- A phase fails repeatedly with the same root cause → re-interpret the spec section, consider whether the plan's task split is wrong, possibly insert a recovery phase
- A test consistently fails for an environmental reason → add it to RUN.md "Known pre-existing failures" and continue
- A new constraint surfaces mid-run (an API limit, a dependency conflict) → record in carry-forward, narrow the next Builder's scope, continue
- A Reviewer keeps failing the same criterion despite mechanical fixes → re-examine whether the rubric threshold is too tight for this specific surface, or whether the underlying expectation is unrealistic

Stuck means truly stuck — not "this is harder than I expected."

### 8. Autonomous runtime + workaround-first blocking

This skill is designed for a high-autonomy environment. Run it in Claude Code with autonomous/bypass permissions or in Codex with full filesystem/network access and approval mode that will not stop mid-run for routine commands. Browser automation, git, package scripts, and PR creation must be available up front.

If a capability is missing, the Orchestrator tries workarounds before asking the user:

- Verification driver unavailable → for UI work, try alternate browser drivers (Chrome MCP, Playwright MCP, local Playwright); for non-UI work, fall back to the project's documented invocation path (curl + the dev server, the project's CLI binary, a test-DB instance, a small example consumer for libraries). Return `blocked` only if no path can exercise the deliverable end-to-end.
- Runtime environment unavailable → start the dev server / test DB / sandbox if the project documents a command and no conflicting instance is running; otherwise return `blocked` with the missing command/env.
- Auth / session / credentials unavailable → use seeded/dev credentials, fixture setup, or a documented test login. If none exists and the deliverable requires auth, return `blocked`.
- GitHub auth unavailable → complete local verification, commit artifacts, and return `partial` with exact push/PR retry commands and a draft PR body.
- Tooling command unavailable → use the nearest project-equivalent command from README/package scripts. If no equivalent exists, mark that check `unavailable` in baseline and reviewers must not treat it as pass/fail.

**Dev-server hygiene between framework-config phases.** If a phase touches build-pipeline or framework-config files (content schemas, plugin lists, build outputs, route generation), the runtime's HMR may not reload through the change. Symptom: Reviewer's E2E smoke shows stale or 500-ing pages while the production build is correct. Workaround: between such phases, the Orchestrator restarts the runtime cleanly (`pkill -f <runtime> && rm -rf .cache && <restart-command>` or equivalent for the project). Add the framework-config files the project considers HMR-fragile to RUN.md "Operating notes" so the Builder of any phase touching them gets a heads-up in the prompt.

Do not pause for preference questions. Only return `blocked` when the next action truly requires a human credential, policy decision, external service, or contradictory spec resolution.

### 9. Push/PR autonomy boundary

Final delivery via `git push -u origin <harness>` + `gh pr create` is the default. **Carve-out for ephemeral/eval runs.** Before pushing, the Orchestrator checks for any of:

- Working tree is in a `/tmp` or scratch path
- Harness branch is on an orphan branch with no upstream commit history shared with origin
- The source branch has no `origin` remote, or the remote points at a fork the user did not specify as the PR target
- RUN.md "Run Configuration" sets `mode: eval` (explicit override)

If any of these holds, return `partial` with the exact `git push -u origin <harness>` and `gh pr create --base <source> --head <harness> ...` commands and the PR body draft, instead of pushing. This is a documented exception to "no human pauses" — push to a public destination is irreversible (deleting a PR leaves audit trace) and warrants confirmation when the destination signals are weak. For real feature runs targeting a real source branch on origin, push and PR creation proceed autonomously as part of final verification (see §Final verification step 6).

---

## Templates

### Orchestrator pre-flight checklist

- [ ] Spec exists and reviewed by the human (else: `superpowers:brainstorming` first)
- [ ] Plan exists, harness-shaped (2–4 tasks per phase, each phase testable, tasks splittable into test+impl) — else: `superpowers:writing-plans` with constraints
- [ ] Running in autonomous mode (Claude Code bypass/auto permissions, or Codex full access/no approval interruptions)
- [ ] Source branch + source SHA resolved
- [ ] Harness branch created from source branch tip (`fram/<source-leaf>-<feature-slug>-<source-sha>`)
- [ ] PR target recorded as the source branch
- [ ] Plan dir created at `plan/<feature>/` with `spec.md` + `plan.md`
- [ ] `RUN.md` scaffolded with all required sections (header, layout block, Run Configuration with rubric thresholds + model routing + project skills scanned from CLAUDE.md/AGENTS.md + known test failures, TDD discipline, resume guide, state machine, dispatch templates, empty Phase Status table, operating notes)
- [ ] Phase 0 baselines captured in `plan/<feature>/baselines/` for lint, tests, and typecheck (or explicitly marked unavailable)
- [ ] Permissions configured (`Bash(*)`, browser MCP tools, etc.)
- [ ] Agent mode set to autonomous operation (`bypassPermissions` or equivalent)
- [ ] Dev server running (do not restart it)
- [ ] Verification driver available for the deliverable shape (authenticated browser for UI; HTTP client for API; fresh shell for CLI; test DB for migrations; library-consumer harness for SDK work)
- [ ] **TDD discipline confirmed default-on for every task**
- [ ] **Phase sizing checked** — no phase >5 tasks without an escape hatch in the Builder prompt
- [ ] Reviewer prompt template will pull thresholds from RUN.md "Run Configuration" (not hard-coded into the skill template)

### Builder + Reviewer prompt templates

See [references/templates.md](references/templates.md) for the verbatim Builder and Reviewer prompt scaffolds. Copy into `phases/phase-N/builder-prompt.md` and `phases/phase-N/reviewer-prompt.md`, substituting placeholders before dispatch.

