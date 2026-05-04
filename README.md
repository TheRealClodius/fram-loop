# fram-loop

A two-agent autonomous development pattern for shipping a complete user-facing feature end-to-end without human intervention mid-run. A **Builder** agent implements each phase with a fresh context; a **Reviewer** agent scores against a rubric and runs browser-level E2E. Phase-based context resets prevent the model from trying to wrap up early as context fills.

**Output is a finished feature a real user can use, not an MVP for a developer to review.**

## Install

```bash
gh skill install TheRealClodius/fram-loop
```

The skill becomes discoverable in any tool that supports the [agentskills.io](https://agentskills.io) convention — Claude Code, Codex CLI, Cursor, etc.

## Use

The skill is **opt-in only**. It does not auto-invoke. Trigger phrases:

- "use the fram-loop"
- "run the fram-loop"
- "spin up the fram-loop"
- "run this through the fram-loop"

Prerequisites: a reviewed spec and a harness-shaped plan (2–4 tasks per phase, each phase independently testable end-to-end). The skill itself documents how to author both as a Phase 0 if they don't exist.

## How it works (in 30 seconds)

1. Per-feature directory `plan/<feature>/` holds `spec.md`, `plan.md`, a live `README.md` (state machine + Phase Status table), and `phases/phase-N/` files (every prompt sent + report received).
2. For each phase: orchestrator dispatches a fresh Builder, then a Reviewer. Reviewer scores Functionality / Design Quality / Accessibility / Code Quality on 1–5 with file:line evidence.
3. Pass → next phase. Fail → fix-Builder + re-review until done. The loop never halts on transient friction; it adapts (re-interpret spec, narrow scope, insert recovery phase) and freight-trains to `complete` or `blocked`.
4. Final verification gate: spec re-walk + end-to-end user-flow exercise + project-appropriate security check + full suite/lint/typecheck.

## Defaults baked in

- **Strict TDD** per task (test commit Red → impl commit; never bundled). RED-fix-forward refinement allowed once.
- **Mandatory browser smoke** for any UI-touching phase via Claude in Chrome MCP, Playwright MCP, or equivalent.
- **Call-site enumeration** when changing a server contract.
- **Model routing** (provider-agnostic): best model for Reviewer always; cheap-out OK on mechanical Builder work; commit-agent for recovery.
- **Resilience over rigidity** — adapt, don't halt.

Full skill spec: [skills/fram-loop/SKILL.md](skills/fram-loop/SKILL.md).

## License

MIT — see [LICENSE](LICENSE).
