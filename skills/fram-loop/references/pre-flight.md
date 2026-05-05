# Orchestrator pre-flight checklist

Read once before kickoff. The Setup steps below are *operator setup* (run them once before Phase 1 dispatches); the binary checklist that follows is the gate the Orchestrator must clear before dispatching Phase 1.

## Setup (do these once before the run starts)

- **Define narrowed-tool subagents.** Create `.claude/agents/fram-loop-reviewer.md` (and `fram-loop-builder.md` if narrowing the Builder per SKILL.md §Architecture > Roles) with narrowed `allowed-tools` front-matter. Probe-test the harness honours the narrowing — dispatch a subagent with a deliberately-omitted tool and confirm the call fails. Record the probe result in RUN.md "Operating notes." If the probe shows narrowing isn't enforced, the Reviewer's prompt-scaffold "Allowed tools" block (in [templates.md](templates.md)) is the load-bearing rule and the Reviewer is constrained by compliance, not capability. See [templates.md](templates.md) §"Dispatching with narrowed tools" for the recipe.
- **Configure a static analyser** for §Final verification step 3 (`gitleaks` for secrets, `semgrep` with project-appropriate rulesets for static analysis) — or document its absence in RUN.md "Operating notes" and accept the manual-fallback caveat.

## Checklist (binary gates before Phase 1 dispatch)

- [ ] Spec exists and reviewed by the human (else: `superpowers:brainstorming` first)
- [ ] Plan exists, harness-shaped (2–4 tasks per phase, each phase testable, tasks splittable into test+impl) — else: `superpowers:writing-plans` with constraints
- [ ] Running in autonomous mode (Claude Code bypass/auto permissions, or Codex full access/no approval interruptions)
- [ ] Source branch + source SHA resolved
- [ ] Harness branch created from source branch tip (`fram/<source-leaf>-<feature-slug>-<source-sha>`)
- [ ] PR target recorded as the source branch
- [ ] Plan dir created at `plan/<feature>/` with `spec.md` + `plan.md`
- [ ] `RUN.md` scaffolded with all required sections (header, layout block, Run Configuration with rubric thresholds + model routing + project skills scanned from CLAUDE.md/AGENTS.md + verification host allowlist + Reviewer subagent path + known test failures, TDD discipline, resume guide, state machine, dispatch templates, empty Phase Status table, operating notes)
- [ ] **`plan/<feature>/RUN-Phase-0.snapshot.md` written** — immutable copy of the Run Configuration block (anchor for the §Final verification step 6 base/head check)
- [ ] Phase 0 baselines captured in `plan/<feature>/baselines/` for lint, tests, and typecheck (or explicitly marked unavailable)
- [ ] Permissions configured (`Bash(*)`, browser MCP tools, etc.)
- [ ] Agent mode set to autonomous operation (`bypassPermissions` or equivalent)
- [ ] Dev server running (do not restart it)
- [ ] Verification driver available for the deliverable shape (authenticated browser for UI; HTTP client for API; fresh shell for CLI; test DB for migrations; library-consumer harness for SDK work)
- [ ] **TDD discipline confirmed default-on for every task**
- [ ] **Phase sizing checked** — no phase >5 tasks without an escape hatch in the Builder prompt
- [ ] **Frozen-install command + `--ignore-scripts` form recorded in RUN.md** (or postinstall-required exception documented in Operating notes — see SKILL.md §Defaults #11)
- [ ] Reviewer prompt template will pull thresholds from RUN.md "Run Configuration" (not hard-coded into the skill template)
