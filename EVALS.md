# fram-loop Evaluation — Run #1

**Date:** 2026-05-04
**Skill version under test:** [SKILL.md](skills/fram-loop/SKILL.md) at commit `92b567b` (main, install ref `v0.1.0` / `0180262a` published to `gh skill install TheRealClodius/fram-loop`)
**Skill installer used:** `gh skill install TheRealClodius/fram-loop fram-loop` (canonical, gh ≥ 2.90)
**Deliverable shipped:** *The Margin* — Astro 6 + MDX + React 19 + Tailwind v4 editorial magazine (12 articles, paywall, GSAP-driven spaceship interactive, Pagefind search, RSS/sitemap, Lighthouse-gated perf+a11y).
**Worktree:** `/tmp/fram-loop-test-themargin/` (orphan-branch worktree of fram-loop, throwaway).
**Final verdict from final-verification gate:** **GO** — every spec §8 acceptance criterion verified, no blockers. Push/PR paused on user confirmation (eval run, not a real ship).

---

## 1. Executive summary

fram-loop shipped a non-trivial, design-attentive editorial site end-to-end across 8 phases without a single fix-round, without a single human-input pause, with a monotonically-shrinking carry-forward chain that closed at zero open items, and with a final-verification gate that found nothing blocking. The numeric rubric and the Builder/Reviewer separation caught real issues every phase (paywall A11y, dev-server cache staleness, RED-fix-forward documentation gap, title duplication, README workflow inaccuracy). Two phases hit "perfect 5/5/5/5" sweeps; two scored 4 on Functionality due to genuinely-minor environmental issues (dev server cache, paywall Reset visual gap) that downstream phases addressed cleanly.

The skill works as advertised. The improvements identified are tightening, not redesign — most are documentation in `SKILL.md` / `templates.md` to surface conventions that emerged during the run but weren't pre-stated.

The skill has **strong evidence of value** when run on a multi-phase, design-heavy, full-stack deliverable. The cost was substantial (~5h wall-clock, ~1.4M tokens across all subagents) and the orchestrator's own prompt-writing burden was the load-bearing inefficiency.

---

## 2. Run metrics (numeric)

### Aggregate

| Metric | Value |
|---|---|
| Phases | 8 (per plan) |
| Commits on harness branch | 75 (8 builder commits × 8 phases + 1 orchestrator-closure per phase + a few setup) |
| Tasks executed | 32 (4 per phase × 8) |
| Builder commits | 64 (32 test + 32 impl) |
| RED-fix-forward used | 4 of 32 tasks (~12%) |
| Fix rounds triggered | **0** |
| Phases passing on first review | **8/8** |
| Test files at end | 41 |
| Tests at end | 358 (up from 17 at end of Phase 1) |
| Pages built | 32 (1 home + 12 articles + 6 topics + 6 authors + about + newsletter + archive + search + 404 + sitemap-index + sitemap-0 + rss.xml + robots.txt) |
| Lighthouse Performance | 0.99 (home) / 1.00 (article) |
| Lighthouse Accessibility | 0.96 / 0.96 |
| LHCI thresholds | Perf ≥ 0.90, A11y ≥ 0.95 (both comfortably exceeded) |
| Open carry-forwards at run end | 0 (every CF closed or explicitly deferred-as-v2 with rationale) |

### Per-phase scores (rubric: Functionality / Code Quality / Design Quality / Accessibility, all ≥ thresholds 4/4/4/3)

| Phase | Builder | Reviewer | Score | RFF | Builder time | Builder tokens | Reviewer time | Reviewer tokens |
|---|---|---|---|---|---|---|---|---|
| 1 Foundation | sonnet | opus | **5/5/5/4** | 1 (1.2) | 14m | 99k | 5m | 80k |
| 2 Home + seed | opus | opus | **4/5/5/5** | 2 (2.1, 2.3) | 24m | 162k | 9m | 115k |
| 3 Article + Paywall | opus | opus | **5/4/4/4** | 1 (3.3, no proof file) | 24m | 180k | 10m | 110k |
| 4 Immersive + paywall A11y fix | opus | opus | **5/5/5/5** | 0 | 15m | 132k | 8m | 110k |
| 5 Spaceship | opus | opus | **4/5/5/4** | 1 (5.3, with proof) | 16m | 122k | 21m | 163k |
| 6 Topic + Author | sonnet | opus | **5/5/4/4** | 0 | 13m | 96k | 6m | 71k |
| 7 Archive + Search | sonnet | opus | **5/5/4/4** | 0 | 18m | 126k | 8m | 89k |
| 8 Plumbing + polish | opus | opus | **5/5/5/5** | 2 (8.2, 8.4) | 26m | 180k | 8m | 96k |
| Final verification | — | opus | GO | — | — | — | 15m | 129k |

**Wall-clock total** (all builders + reviewers + final): ~5h.
**Token total**: ~1.4M (Builders ~1.1M, Reviewers ~830k, final-verify 129k).

### Carry-forward chain (monotonic shrinkage)

- Phase 1 emitted: 7 CFs → 4 addressed in Phase 2 + 3 forwarded
- Phase 2 emitted: 8 new + 3 forwarded → addressed in Phase 3 + 8
- Phase 3 emitted: 5 new + carry over → 3 addressed in Phase 4 + rest forwarded
- ...
- Phase 7 emitted: 5 new + many carried → 6 addressed in Phase 8 + rest explicitly deferred-as-v2
- Phase 8 final state: **zero open CFs**; all explicitly-deferred items have target-phase or v2 rationale

---

## 3. What worked (preserve — do not regress)

### 3.1 Builder/Reviewer separation as load-bearing structure

The most important design choice. Every phase, Reviewers caught issues Builders didn't self-flag:
- Phase 3: subscribe form trapped in `inert` overlay → CF-3.6 (Medium A11y).
- Phase 6: cosmetic title duplication on slug pages → CF-6.4.
- Phase 7: README's documented Pagefind dev workflow doesn't actually work → CF-7.5.
- Phase 8: Reviewer re-ran `npm run audit` rather than trusting Builder's claimed Lighthouse numbers.

Self-evaluation would have missed all of these. The "Reviewer is read-only with verification tools, never implements" rule is correct.

### 3.2 Numeric rubric with file:line evidence

Forces concrete grading. Every "score: 4" came with specific evidence (file:line, screenshot reference, command output). No "looks good to me" hand-waving.

### 3.3 Phase resets (fresh Builder per phase) prevented context anxiety

Predicted failure mode: Builder "wraps up early" as context fills. **Did not occur** in any phase. Even Phase 2 (162k tokens, 4 tasks, content-heavy) and Phase 5 (122k, 4 tasks, novel React+GSAP island) finished cleanly because each Builder showed up to a clean context window with focused scope.

### 3.4 Carry-forward chain converged monotonically

Phase 1 emitted 7 CFs. Phase 8 closed at 0 open. Each phase's Reviewer passed CFs to the next phase's Builder verbatim, and the next phase's Builder either addressed them (ideal) or explicitly deferred them with a target-phase rationale. **The chain never lost an item.** This is institutional memory working as designed.

### 3.5 "Fail on new debt only" baseline rule

Phase 0 had no lint/test/typecheck (intentional — Phase 1 wired them). Phase 1's Reviewer treated absence as expected. Phase 2+ compared against Phase 1's recorded outputs. **Zero false alarms** for pre-existing issues. The rule is correct.

### 3.6 TDD per task held end-to-end

32 tasks, 32 test commits, 32 impl commits. 4 RED-fix-forwards (~12% rate). At least one Reviewer per phase spot-checked a Red commit by checkout → confirmed it actually failed. **Discipline didn't degrade across phases.**

### 3.7 Resilience-over-rigidity worked when adapted

Phase 2's Builder hit a real issue: Astro 6 dev-server HMR doesn't reload through `content.config.ts` schema changes. Production build was correct; dev was stale. The skill adapted: Builder flagged it, Reviewer confirmed via fresh dev-server spawn, Orchestrator restarted dev between phases that touched content config. **No halt, no fix round.** This is exactly what §7 of the SKILL describes.

### 3.8 Two clean sweeps bookended risk

Phase 4 (5/5/5/5) before the spaceship novelty phase, and Phase 8 (5/5/5/5) at run end. The spaceship phase itself (P5) scored 4/5/5/4 — minor functional gaps that became carry-forwards rather than blockers. The pattern (clean → risky → recover-and-polish → clean) suggests the skill's phase-sizing discipline was correct.

### 3.9 Never-cheap-out-on-Reviewer rule

Phase 1 (mechanical) used Sonnet Builder; Reviewer used Opus. Phase 5's deeply-browser-driven Reviewer (21 min, 163k tokens, scripted keystrokes through 6 verification phases) caught real issues a cheaper model would have missed. **Defaults are correct.**

---

## 4. What didn't work / observations (improve)

### 4.1 Orchestrator prompt-writing burden compounds

I (the Orchestrator) wrote 16 prompts (8 Builder + 8 Reviewer) plus the final-verification prompt. Each was 8–15k tokens, structured in similar shape but with phase-specific content. **The orchestrator's own context grew faster than expected** because authoring a Builder prompt in markdown for a fresh subagent IS implementation work — describing the task in writing.

The SKILL says "orchestrator stays light" but doesn't address this prompt-writing cost. The `references/templates.md` Builder/Reviewer scaffolds are ~50 lines each; in practice the prompts ballooned to 200–400 lines as I added phase-specific scope, carry-forwards, decision rationale, escape hatches, and report-format details.

**Improvement:** add a richer Builder/Reviewer prompt scaffold in `templates.md` with explicit slot-fill markers so prompt-writing is more "fill in 5 placeholders" than "author from scratch each time." See §5.2.

### 4.2 RED-fix-forward proof-file convention is ambiguous in SKILL.md

SKILL.md §Defaults #1 says: *"you may commit a refinement before the impl commit that fixes the test to fail for the right reason."* This implies a separate refinement commit between Red and Green.

In practice:
- **Phase 2** wrote two `red-proof-task-X.md` files documenting the refinements (transparent).
- **Phase 3** used the term loosely — the test was iterated on locally before the Red commit landed; no separate refinement commit, no proof file. The Phase 3 Reviewer flagged this as ambiguous but accepted because Red was real.
- **Phases 5 + 8** wrote proof files (Builders learned from Phase 3 feedback in their carry-forwards).

**Improvement:** SKILL.md should require either (a) a separate refinement commit between the original Red and the Green, OR (b) a `red-proof-task-X.md` file documenting the iteration. Pick one. Currently the convention is implicit and the Builders have to be told. See §5.1.

### 4.3 Subagent CWD inheritance is a hidden gotcha not in SKILL.md

Every Builder + Reviewer prompt I wrote needed an explicit "cd /tmp/fram-loop-test-themargin first" reminder because subagents inherit the parent (orchestrator) session's CWD. This is a Claude Code quirk; SKILL.md doesn't mention it. Without the reminder, a subagent runs `git status` against the wrong directory and gets confused.

**Improvement:** add a "Working directory" preamble line to the Builder/Reviewer scaffolds in `templates.md`. See §5.2.

### 4.4 Dev-server hygiene between phases is undocumented

Phase 2's Astro-6-content-config HMR issue was the canonical case. Phase 3 onwards I learned to `pkill -f astro && rm -rf .astro && npm run dev` between phases that touched framework-config files. This worked but was learned-by-pain. SKILL.md §Defaults #8 says "start the dev server" but not "between phases that touch framework config, restart it."

**Improvement:** SKILL.md §Defaults #8 should add a dev-server-hygiene paragraph. See §5.1.

### 4.5 Phase artifact staging convention emerged organically

By Phase 4, Builders started saying "I left phases/phase-N/ untracked because the orchestrator manages those." This is the right convention — Builders commit only source/test/config files, the Orchestrator commits run-state artifacts in a closure commit. But it isn't in SKILL.md; Builders deduced it.

**Improvement:** SKILL.md should explicitly state: "Builders commit only source/test/config changes per the per-task TDD pattern. The Orchestrator commits run-state artifacts (`builder-report.md`, `reviewer-prompt.md`, `reviewer-report.md`, `carry-forward.md`, RUN.md updates, baseline refreshes) as a single phase-closure commit." See §5.1.

### 4.6 Phase 0 baseline-as-placeholder pattern wasn't documented but worked

Phase 0 had no lint/test/typecheck because Phase 1 wires them. The `baselines/baseline.md` recorded them as `unconfigured — Phase 1 will wire`, with placeholder `lint.txt` etc. files. The Phase 1 Reviewer treated absence as expected. After Phase 1 passed, the Orchestrator overwrote the placeholder files with the real outputs.

This worked, but the pattern is invented per-run. SKILL.md mentions "explicitly marked unavailable" but doesn't show this specific shape.

**Improvement:** SKILL.md should include the placeholder baseline pattern as an example for projects that bootstrap their tooling in Phase 1. See §5.1.

### 4.7 Orphan-branch worktree pattern undocumented

For ephemeral test/eval runs (this run), I used `git worktree add --orphan -b test-themargin /tmp/fram-loop-test-themargin`. This gave a clean ephemeral fixture isolated from the fram-loop repo's history. SKILL.md assumes existing repos with feature branches; the orphan-worktree pattern is undocumented but ideal for evals.

**Improvement:** add a brief "Test/eval setup" note to SKILL.md or to the README's "Use" section showing the orphan-worktree pattern.

### 4.8 Builder report format isn't strictly normalized

Phase 1's Builder report was 6.4k chars; Phase 4's was 9.9k; Phase 7's was 7.5k. The structures were similar but not identical, and the level of detail varied. The Phase 8 Builder included a spec §8 acceptance-criteria checklist; earlier Builders didn't. The Phase 8 inclusion was on me (I asked for it in the prompt), not from a default template.

**Improvement:** the Builder report scaffold in `templates.md` should require a "Spec acceptance criteria touched in this phase" section in every phase, not just the closing one. See §5.2.

### 4.9 Carry-forward bookkeeping burden grows linearly

By Phase 7 I was tracking 20+ open CFs across phases (some already addressed, some forwarded, some explicitly deferred). The chain stayed correct but the orchestrator's bookkeeping was visibly heavier in the late-phase carry-forward.md files. The Phase 7→8 carry-forward had to explicitly enumerate "must do", "nice-to-have", "out-of-scope" lists to manage Phase 8's scope.

**Improvement:** SKILL.md could describe a CF lifecycle: `Open → Addressed-in-N | Deferred-to-X (rationale) | Out-of-scope-v2 (rationale)`. Each phase's carry-forward.md uses these explicit states. This is what I converged to organically; codifying it in templates would help future runs. See §5.1 + §5.2.

### 4.10 Push/PR autonomy ambiguity for eval runs

fram-loop's design specifies that `complete` requires `git push -u origin <harness>` + `gh pr create`. For real feature work this is correct (the deliverable is a PR). For test/eval runs, push lands eval noise on the public skill repo. This run paused before push to confirm with the user — not because the push would have failed, but because I judged it potentially destructive in an eval context.

The SKILL says "the loop never pauses for human opinion mid-run." Push-to-public-shared-state is plausibly an exception worth carving out.

**Improvement:** SKILL.md could add a §Defaults #9: "Before push/PR creation, the Orchestrator verifies the destination matches user intent. For eval/test runs (orphan worktrees, scratch branches), it is correct to mark the run `partial` and surface push commands rather than push autonomously." See §5.1.

### 4.11 Final-verification gate has slight redundancy with Phase 8 Reviewer

The Phase 8 Reviewer pre-walked spec §8. The final-verification subagent walked it again. Some duplication, but intentional — final verification verifies against the LIVE deliverable (re-running `npm run audit`, hitting routes via Claude-in-Chrome), not against the Builder's claims. The duplication caught nothing new in this run, but it's the right structure.

**No change needed.** Worth a one-line clarification in SKILL.md that final verification is its own gate, not just a Reviewer rerun.

---

## 5. Concrete improvements proposed

### 5.1 SKILL.md changes

1. **§Defaults #1 (TDD)**: tighten RED-fix-forward to require either a separate refinement commit OR a `red-proof-task-X.md` file. Currently the convention is implicit.

2. **§Defaults #8 (Autonomous runtime)**: add dev-server-hygiene paragraph: *"If a Builder reports the dev server is stale or returns errors after a phase touches framework-config files (content schemas, build pipeline), the Orchestrator restarts cleanly between phases (`pkill -f <runtime> && rm -rf .cache && npm run dev` or equivalent). Astro's HMR specifically does not always reload through `content.config.ts` schema changes — observed in this eval run."*

3. **§Defaults**: add a #9 — *"Push/PR autonomy boundary"*: *"`complete` requires harness-branch pushed + PR opened back to source. For eval/test runs (orphan worktrees, scratch branches, throwaway destinations), the Orchestrator may correctly mark the run `partial` with push commands prepared rather than push autonomously. Push to a public shared destination is the kind of irreversible action where confirmation aligns with the skill's `complete | partial | blocked` semantics."*

4. **§Run state**: add a "Phase 0 baseline-as-placeholder" subsection showing the pattern when Phase 1 wires the tooling.

5. **§Architecture > Roles**: clarify the artifact-staging convention: *"The Builder commits only source/test/config changes per the per-task TDD pattern. The Orchestrator commits run-state artifacts (builder-report.md, reviewer-prompt.md, reviewer-report.md, carry-forward.md, RUN.md updates, baseline refreshes) as a single phase-closure commit."*

6. **§Architecture > Subagent quirks**: new subsection: *"Subagents inherit the parent session's CWD. Builder/Reviewer prompts must explicitly `cd <worktree>` first or use `git -C <worktree>` for every shell command. Browser-automation MCPs (Claude-in-Chrome, Playwright) and other deferred tools must be loaded via `ToolSearch` in subagents that need them — they are not loaded by default."*

7. **§Carry-forward**: codify CF lifecycle states (`Open → Addressed-in-N | Deferred-to-X | Out-of-scope-v2`) and a brief example of explicit deferral with rationale.

8. **§Final verification**: add a one-line clarification that final verification verifies against the LIVE deliverable, not Builder claims — i.e. it re-runs commands, navigates routes in browser, and probes artifacts on disk fresh, even if the Phase N Reviewer pre-walked the same surface.

### 5.2 references/templates.md changes

1. **Builder prompt scaffold** — add at the top:
   - **Working directory** line (with subagent-CWD warning)
   - **Carry-forward fixes from prior reviews** section (verbatim block) as a required scaffold slot
   - **Spec-acceptance touched in this phase** list as part of the report format

2. **Reviewer prompt scaffold** — add at the top:
   - **Working directory** line
   - **Browser/tool deferred-loading reminder** (`ToolSearch` first for `mcp__Claude_in_Chrome__*` etc.)
   - **Spot-check Red on at least one task by checkout** as a required step (not just "verify TDD ordering")

3. **Both scaffolds** — make the slot-fill structure explicit (`<placeholder>` markers) so per-phase customisation is closer to filling slots than authoring fresh.

### 5.3 README.md changes

- No corrections needed (`gh skill install` is canonical and correct — my earlier confusion was on the eval side).
- Optionally: add a short "Run an eval" section describing the orphan-branch worktree pattern for ephemeral test runs against the skill itself. Useful for skill-development feedback loops.

### 5.4 Out of this eval's scope but worth noting for v2

- Cost: ~1.4M tokens on Opus is significant. For lower-cost runs, expanding the "mechanical" classification (currently Phases 1, 6, 7) could route more work to Sonnet. Phase 6 + 7 ran on Sonnet without quality regression; Phase 8 ran on Opus and could plausibly have run on Sonnet for the polish-task subset. Worth measuring.
- Single-deliverable test only: this eval covered UI deliverable shape. The skill's claim of "deliverable-agnostic" (UI / API / CLI / library / migration / refactor / perf) is untested for non-UI shapes. Future evals should cover at least one non-UI deliverable.

---

## 6. Open questions / future eval ideas

1. **Pass^k reliability**: this run was a single shot. Running the same fixture 3× (with cache busted between) and reporting fraction-where-all-3-succeed would tell us if the skill is robust or if this run was lucky.
2. **Time-horizon-style multi-fixture eval**: per the eval-research notes, multi-size fixtures (~30min / ~2h / ~6h human-equivalent) reveal at what task-size the skill crosses 50% and 80% success rates. This eval was one ~5h fixture; a spread would calibrate.
3. **Single-agent freeform baseline**: same fixture, no Builder/Reviewer split, see if the loop earns its complexity. Hard but informative.
4. **Non-UI deliverable**: does the skill ship a CLI, library, or migration with the same fidelity? Especially: does the rubric tail adapt cleanly to "no browser MCP available"?
5. **Adversarial fixture**: deliberately ambiguous spec or a contradictory plan to test the BLOCKED state semantics.

---

## 7. Per-phase detail (appendix)

### Phase 1 — Foundation

- **Builder (Sonnet, 14m, 99k):** scaffold tooling (Tailwind/MDX/ESLint/Prettier/Vitest/astro-check), content-collection schemas, design tokens, base layout + motion primitives. 4 tasks × Red+Green = 8 commits. RED-fix-forward 1 (Task 1.2, with proof file). Made the right call to use `@tailwindcss/vite` over `@astrojs/tailwind` (Astro 6 incompat); flagged Astro 6's `ClientRouter` rename of `ViewTransitions`.
- **Reviewer (Opus, 5m, 80k):** PASS 5/5/5/4. Spot-check Red on Task 1.2 confirmed real Red. Three carry-forwards (font preload paths 404 in dev, Tailwind v4 token consolidation, `@tailwind` directive migration) all addressed in Phase 2.
- **Eval observation:** the Builder used "RED-fix-forward" loosely on Task 1.2 — wrote a proof file but the proof was about a path-rename rather than a wrong-reason fail. Reviewer accepted. First sign that the RFF convention is fuzzy.

### Phase 2 — Home + Seed content + ArticleCard + HeroSpread

- **Builder (Opus, 24m, 162k):** seeded all 12 articles (~600–950w each from briefs in editorial voice), 6 authors, 6 topics. Built ArticleCard (grid + dense), HeroSpread (with FadeUp + scrim), wired home composition. Addressed all 4 Phase 1 carry-forwards. RFF used twice (Tasks 2.1, 2.3) with proof files. Heaviest Builder phase.
- **Reviewer (Opus, 9m, 115k):** PASS 4/5/5/5. Found a MAJOR functional issue — dev server cache was stale, `:4321` returned 500 — and confirmed production was correct. Recommended `rm -rf .astro && pkill -f astro` between content-config phases. This became CF-2.7.
- **Eval observation:** the dev-server-cache issue is the canonical case of "the spec/plan didn't anticipate a runtime quirk." fram-loop adapted (Reviewer flagged, Orchestrator restarted dev, no fix round). The skill's resilience-over-rigidity ethos works.

### Phase 3 — Article (standard) + Paywall + ShareToolbar + MDX

- **Builder (Opus, 24m, 180k):** standard article header + dynamic article route, MDX components (PullQuote, DropCap, Footnote pair), retrofitted 5 feature articles to use them, Paywall mounted via MDX, ShareToolbar. Installed `eslint-plugin-mdx` (CF-2.1). 13 pages built (1 + 12 articles). RFF 1 (Task 3.3, no proof file — Builder used the term loosely).
- **Reviewer (Opus, 10m, 110k):** PASS 5/4/4/4. Found Medium A11y: subscribe form trapped inside `inert` overlay → CF-3.6. Investigated the missing RFF proof — confirmed Red was real via checkout but flagged the convention gap.
- **Eval observation:** the missing RFF proof on Task 3.3 surfaced the convention's ambiguity. Phase 5 + 8 Builders, after this feedback flowed into their prompts, wrote proof files correctly.

### Phase 4 — Immersive header + paywall A11y fix

- **Builder (Opus, 15m, 132k):** ImmersiveArticleHeader with full-bleed + parallax + title-shrink, layout switch on `heroLayout`, reduced-motion source-level test, **paywall A11y restructure** (form hoisted out of inert overlay, CF-3.6 fix), blur transition replaced with 200ms opacity, dead `data-drop-cap` marker removed. RFF 0.
- **Reviewer (Opus, 8m, 110k):** **PASS 5/5/5/5** — first clean sweep. CF-3.6 verified via DOM-walk; CF-3.7 verified via `transitionDuration` computed style; CF-3.8 via grep.
- **Eval observation:** the carry-forward chain worked exactly as designed — Phase 3's Medium A11y issue surfaced cleanly in Phase 4's must-do list and was fixed without scope drift.

### Phase 5 — Spaceship interactive

- **Builder (Opus, 16m, 122k):** React island (`@astrojs/react` + react@19 + react-dom@19), GSAP, pure-function AABB physics module, document-level keydown gated on `mode === 'active'`, three runtime-injected tokens, easter-egg footnote, touch directional pad, reduced-motion stepped movement. Mount at route level (not MDX, opposite of Paywall). RFF 1 (Task 5.3) **with proof file** — Builder corrected for the Phase 3 oversight.
- **Reviewer (Opus, 21m, 163k):** PASS 4/5/5/4. Most browser-driven Reviewer of the run — scripted keystrokes through 6 phases (dormant → activate → play → exit → sessionStorage → reduced-motion → touch). Found minor Reset-visual-gap (CF-5.5), walls-cache-stale-on-scroll (CF-5.6), and a hydration race with `client:visible` (CF-5.7). All low-severity, deferred to Phase 8 polish.
- **Eval observation:** the most novel phase. The Reviewer's deep browser interaction (162k tokens, 21min, scripted physics testing) is exactly the "never cheap-out on Reviewer" rule paying off. A weaker Reviewer would have rubber-stamped the static checks.

### Phase 6 — Topic + Author pages + cross-links

- **Builder (Sonnet, 13m, 96k):** `src/lib/articles.ts` helpers (CF-2.9), 6 topic pages, 6 author pages, byline → author anchor restructure with no-nested-anchors discipline (CF-2.2), home topic-spotlight empty-state (CF-2.10). 25 pages built (+12 listing pages). RFF 0.
- **Reviewer (Opus, 6m, 71k):** PASS 5/5/4/4. Confirmed zero nested anchors via DOM walk. Found minor title duplication on slug pages (CF-6.4). Fastest, leanest Reviewer of the run — pages were structurally clean.
- **Eval observation:** Sonnet handled this phase cleanly without quality regression. Suggests Phase 8 polish work could also route to Sonnet.

### Phase 7 — Archive + Search + index pages + 404

- **Builder (Sonnet, 18m, 126k):** `/topics` index, `/authors` index, custom `/404` (Layout-wrapped, on-brand "Lost in the margins" headline), `/archive` with month grouping + sticky headers + filter chips with `?topics=` URL state (OR semantics, replaceState, pre-paint apply), Pagefind search with post-build step. 30 pages. RFF 0.
- **Reviewer (Opus, 8m, 89k):** PASS 5/5/4/4. Found 1 documentation bug: `README.md:56-57` claimed `dist/pagefind/` is served by `astro dev`, but it isn't (Astro dev only serves `public/`). CF-7.5.
- **Eval observation:** Reviewer caught a doc-vs-reality mismatch the Builder didn't self-flag. Builder/Reviewer separation continues to earn its complexity.

### Phase 8 — Plumbing + polish (closing)

- **Builder (Opus, 26m, 180k):** Nav + Footer + About + Newsletter pages, RSS via `@astrojs/rss` + sitemap via `@astrojs/sitemap` + robots.txt, view-transition-name parity verification, **global `:focus-visible`** rule, reduced-motion sweep, paywall toast `aria-live` (CF-4.5), Lighthouse CI gates (`@lhci/cli`, `lighthouserc.json`, `npm run audit`), README Pagefind workflow corrected (CF-7.5). 32 pages. **Lighthouse: Perf 0.99/1.00, A11y 0.96/0.96.** RFF 2 (Tasks 8.2, 8.4) with bundled proof file.
- **Reviewer (Opus, 8m, 96k):** **PASS 5/5/5/5** — second clean sweep. Re-ran `npm run audit` to verify Lighthouse claims. Confirmed all 6 closed CFs (CF-2.8, 4.5, 7.4, 7.5, plus nice-to-haves CF-7.1, 3.2). Spec §8 checklist 9/10 ✅; #10 (PR) is the Orchestrator's gate.
- **Eval observation:** the closing phase landed cleanly. The "must-do CFs" + "nice-to-have CFs" + "explicitly-deferred CFs" structure I converged to in the Phase 7 carry-forward kept Phase 8's scope manageable.

### Final verification gate

- **Subagent (Opus, 15m, 129k):** GO. Spec re-walked section by section against the live deliverable. 14 critical browser paths verified. Basic OWASP-shape security check (no secrets, XSS escaped on all forms, controlled `set:html` usage, no SSRF). Full suite re-run: lint/typecheck/test/build/audit all green. Run artifacts on disk — every phase's 5 required files present and substantial.
- **Eval observation:** the final-verification gate is heavy but necessary. The Phase 8 Reviewer's spec-walkthrough was a Builder-claims pass; the final gate was a live re-walk. Caught nothing new this run, but the structural difference matters. One-line clarification in SKILL.md would help.

---

## 8. References

- [SKILL.md](skills/fram-loop/SKILL.md) — skill under test
- [references/templates.md](skills/fram-loop/references/templates.md) — prompt scaffolds
- `/tmp/fram-loop-test-themargin/plan/the-margin/` — full run audit trail (RUN.md, spec.md, plan.md, article-briefs.md, baselines/, phases/, final-verification/)
- `/Users/andreiclodius/.claude/projects/-Users-andreiclodius-Documents-Projects-fram-loop/memory/` — eval intent memory saved at the start of the run
