# Run state — plan-dir layout, RUN.md, baselines, carry-forward

This file documents the on-disk run state for a fram-loop run. The Orchestrator reads it when scaffolding Phase 0 (the setup phase before Phase 1) and when resuming from a fresh session.

The skill's §Run state section in SKILL.md carries the load-bearing summary (every run lives in a single directory; baselines captured at Phase 0; all artifacts persist as audit trail). This file has the structural detail: file layouts, RUN.md schema, Phase 0 mechanics, the carry-forward state machine, and the rationale for the layout choices.

---

## Standard plan-dir layout

```
fram-loop/<feature-name>/
  RUN.md                              # live progress: Phase Status table, state machine, resume guide, run config
  RUN-Phase-0.snapshot.md             # immutable copy of Run Configuration at Phase 0 close (anchor for base/head check)
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

---

## RUN.md sections (required)

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

---

## Run Configuration block (in RUN.md)

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
- **Verification host allowlist (signal, not gate):** localhost:3000, https://api.example.com
  *Documented intent for Reviewer scrutiny. Claude Code's `Bash(curl:*)` permission does not filter by URL — pair with a runner-level firewall or HTTP egress proxy if hard enforcement is required.*
- **Reviewer subagent:** `.claude/agents/fram-loop-reviewer.md` (probe-tested at Phase 0 — see [templates.md](templates.md) §"Dispatching with narrowed tools")
- **Known pre-existing failures (don't conflate with regressions):**
  - 82 baseline test failures (per prior security audit)
  - 405 lint warnings (`no-explicit-any` in test mocks)
  - Legacy specs (don't fail the run if these are already broken at baseline)
```

---

## Setup Phase 0: branch + baseline

Before dispatching Phase 1, the Orchestrator creates an isolated harness branch from the current branch tip:

1. Resolve `source_branch=$(git rev-parse --abbrev-ref HEAD)` and `source_sha=$(git rev-parse --short HEAD)`.
2. Create `harness_branch=fram/<source-leaf>-<feature-slug>-<source-sha>` from that exact source tip. Sanitize slashes in the source leaf (`feature/foo` → `feature-foo`). If the branch exists locally or on origin for the same source SHA, reuse/resume it; if it exists for a different SHA, mint a new branch with the new SHA. Never force-push.
3. Set the PR target to `source_branch` (not `main` unless the user explicitly started from `main` and project rules allow it).
4. Capture baseline outputs in `fram-loop/<feature>/baselines/` before any feature edits. Record command, exit code, timestamp, source branch, source SHA, and output path in `baseline.md`.
5. **Snapshot the Run Configuration.** Once the RUN.md "Run Configuration" block is finalised (source/harness branches, PR target, rubric thresholds, model routing, host allowlist, frozen-install command), copy the entire block to `fram-loop/<feature>/RUN-Phase-0.snapshot.md`. This file is **immutable** for the rest of the run — it is the comparison anchor for the final-verification base/head check (§Final verification step 6). Builders never edit it; the Orchestrator never overwrites it. If the run genuinely needs a configuration change mid-flight, that's an Orchestrator decision recorded in RUN.md "Operating notes" with a rationale, and the snapshot stays as the original intent of record.

The branch is load-bearing: Red test commits, recovery commits, plan artifacts, and implementation commits all live on the harness branch. The user's source branch is left untouched. Final delivery is a PR from the harness branch back to the source branch.

Baseline comparison rule: Reviewers verify **no new debt**, not "the full repo became clean." For each phase:

- Touched files must pass scoped lint with strict warnings (`npx eslint <touched-files> --max-warnings=0` or project equivalent).
- Full lint/test/typecheck may fail only if the failure set is at-or-below the Phase 0 baseline and no failing diagnostic points at touched files unless it already existed at baseline.
- Any new full-suite failure, new lint diagnostic, or new type diagnostic introduced by the harness branch fails Code Quality, even if the command already failed at baseline.
- If a phase intentionally fixes baseline debt, update `baseline.md` with the new lower count and the commit that improved it. Never raise the baseline silently.

---

## Phase Status table (in RUN.md)

Updated after every state transition. One row per phase.

```markdown
| # | Phase | Status | Builder | Reviewer | Fix rounds | Last commit | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Mechanical cleanup (4 tasks) | **PASSED** | done (a886dc134) | done (acdc0ece9) — PASS, 5/5/4/5 | 0 | `81eefdf` | All criteria above threshold |
| 2 | i18n of validation strings (3 tasks) | **BUILDER_RUN** | dispatched 2026-05-01 14:22 | — | — | — | — |
```

---

## Carry-forward per phase (`phases/phase-N/carry-forward.md`)

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

---

## Why this layout

Three properties:

1. **Resumable from a fresh session.** Read `RUN.md` → find the first phase row not `PASSED` → read that phase's folder → act. Conversation history not required.
2. **Inspectable.** A human peeking at the run reads markdown — no JSON parsing, no schema knowledge.
3. **Loose ledger, not a controller.** RUN.md is a memory of where we are, not an instruction set for what to do next. The loop should still adapt — re-interpret spec, narrow fix scope, insert a recovery phase — based on what's actually happening.
