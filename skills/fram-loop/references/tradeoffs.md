# fram-loop trade-offs

Deliberate choices in the skill that came with known costs. Each entry documents *what we chose, why, and what to watch for* — so future maintainers and skill consumers can deviate intentionally rather than accidentally.

These are not bugs and not eval findings. They're decisions made under tension, where the alternative was costlier or less useful. Knowing the costs lets you judge whether a different deliverable shape, runtime, or run profile changes the calculus.

---

## TDD spot-check is a sample, not a full audit

**What we chose.** §Defaults #1 requires the Reviewer to spot-check Red on at least one task per phase — `git checkout <test-sha>`, run the test command, confirm it fails. The other 1–3 tasks per phase rely on Builder honesty + git log structure (test commit before impl commit, observed via `git log --oneline`).

**Why.** Spot-checking *every* task would double the Reviewer's checkout-and-run cost for marginal extra audit. One task per phase is enough to catch a Builder that's bundling tests-with-impls or committing trivially-passing tests as a pattern.

**What to watch for.** The audit is a *sample*. A Builder that's faithful on the spot-checked task but lazy on others slips through. If a phase consistently scores well on TDD compliance but the run still has bugs the Reviewer didn't catch, consider auditing all tasks for one phase as a calibration check.

---

## §Defaults #9 + #10 — coverage now broad, but not exhaustive

**What we chose.** §Defaults #9 gates push/PR creation behind machine-checkable destination signals (`/tmp` paths, orphan branches, missing remotes, explicit `mode: eval`). §Defaults #10 (added in v0.2.0) extends the boundary to a broader list of disallowed actions: harness-branch-only writes, no global config / credential surface mutation, no `npm publish` / destructive `gh`, no secrets reads, network restraint, intent-gated installs, with subagent dispatches required to re-assert the boundary in every `Agent` call.

**Why.** Push to public shared state is the irreversible action with the highest blast radius and the clearest signal of "this is not a real ship" — hence the dedicated #9 carve-out. §10 codifies the broader runtime-safety boundary that the loop self-enforces during the run, with subagent-dispatch re-assertion to prevent the boundary leaking through `Agent` calls (subagents don't inherit §10 by transitivity).

**What to watch for.** Together they cover most of the surface, but two slivers remain:

- **Migration target verification.** §10 governs network egress and file-write targets, but a migration tool talking to a database via `DATABASE_URL` doesn't read as "egress" the way a `curl` does — there's no machine check that the target is non-prod. The "non-prod copy" rule in §Defaults #2 (verification driver) stays correctness-by-convention.
- **External service writes via project tooling.** §10's network rule covers obvious shapes (curl, drive-by URLs, exfil), but project-internal tooling that writes to external services (a Slack-poster CLI, a webhook-firing test script, a license-publish step in CI) is governed by intent + the host allowlist, not by a hard gate.

For backend, migration-heavy, or external-integration projects, treat the host allowlist (Run Configuration) and §10's intent-gated install rule as policy, not enforcement.

---

## Reviewer read-only enforced by runtime dispatch + probe, with a compliance fallback

**What we chose.** SKILL.md §Architecture > Roles names the runtime requirement (dispatch Reviewer without write tools). `references/templates.md` §"Dispatching with narrowed tools" gives the concrete recipes — a Claude Code subagent definition with narrowed `allowed-tools` front-matter, plus a Codex/other-harness fallback. A probe step (deliberately invoke an omitted tool, confirm permission denial) verifies the narrowing is actually enforced.

**Why.** The skill is provider-agnostic (§Defaults #5). Hardcoding a single runtime's dispatch primitive into SKILL.md would couple the skill to that runtime. The probe step makes the gap honest: if the harness doesn't enforce, you know up front rather than discovering it through a misbehaving Reviewer.

**What to watch for.** Two layers, depending on whether the probe passes:

- **Probe passes (narrowing enforced):** the boundary is real. A hallucinating Reviewer with edit access can't write because the runtime denies. This is the strong case.
- **Probe fails (narrowing prose-only):** the Reviewer prompt's "Allowed tools" block (in the templates.md Reviewer scaffold) is the load-bearing rule. The Reviewer is constrained by *compliance with its own prompt*, not by *capability*. A misbehaving Reviewer can violate the declared surface — and that's a §Defaults #10 finding the Orchestrator escalates, not silent damage.

Record the probe result in RUN.md "Operating notes" so reviewers (human and agent) know which regime applies. For runtimes without probe-tested narrowing, plan extra Orchestrator vigilance on the Reviewer's actual tool surface.

---

## No fix-round cap (§Defaults #6)

**What we chose.** The loop continues until one of `complete | partial | blocked`. There is no `MAX_ITERATIONS=N` halt. "Stuck" means truly stuck, not "I expected this to be quicker."

**Why.** Most "stuck" rounds are misframed — bad task split, ambiguous spec, wrong rubric threshold for the surface, mid-run dependency conflict. All cheap to fix once recognised. A fixed cap halts the loop *before* recognition happens. The bet is that good carry-forward + good rubric + adaptive replanning give the loop enough self-awareness to know when it's stuck.

**What to watch for.** There is no upper bound on time or tokens. A pathological case — an ambiguous spec the agents can't resolve, a recurring environment failure that adaptation can't fix — could spend uncapped resources. Run #1 used ~1.4M tokens across ~5 hours wall-clock for a benign run; a pathological run could be longer. The skill doesn't surface this cost to users in the trigger description; consider setting expectations explicitly when invoking on a real project.

---

## Fresh Builder per phase vs continuous + auto-compaction

**What we chose.** Each phase gets a fresh Builder with a clean context window. The new Builder reconstructs state from spec + plan + git log + carry-forward + file pointers — never from a prior Builder's report.

**Why.** Engineered handoff (structured files) is less lossy than auto-compaction. Compaction is a summary; the Builder may "wrap up early" as context fills (the context-anxiety failure mode Anthropic documented for Sonnet 4.5-era models). Per-phase resets sidestep all of this.

**What to watch for.** The cost is per-phase dispatch overhead — bootstrap, prompt authoring, agent startup. For shorter runs on capable models (Opus 4.6+, GPT-5.5+), continuous mode may produce equivalent quality with less overhead. Don't switch defaults until ablation eval data exists (EVALS.md §6 lists "single-agent freeform baseline" as a future eval). The current default is conservative, not optimal.

---

## Carry-forward integrity is paper-only

**What we chose.** The carry-forward state machine (`Open / Addressed-in-K / Deferred-to-K / Out-of-scope-v2`) flows phase→phase via written `carry-forward.md` files. The Builder claims a CF resolution; the Orchestrator transcribes it into the next prompt; the Reviewer audits the transcription.

**Why.** A technically-enforced state machine (e.g., a CF database, a JSON ledger with schema validation) would couple the skill to runtime tooling. Markdown is universal, inspectable, resumable from any session, and survives across agent runtimes.

**What to watch for.** A misbehaving Builder can lie about a CF resolution, and the Orchestrator may transcribe the lie without catching it. The Reviewer's CF audit step (§Reviewer Protocol) is the only structural defense, and the Reviewer reads the Orchestrator's transcript, not the Builder's verbatim claim. If a CF chain shows wrong resolutions, the gap is at the Reviewer's audit pass, not at the state machine itself.
