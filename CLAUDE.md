# fram-loop — agent instructions

This repo is the source of the **fram-loop** skill — a two-agent autonomous loop (Builder + Reviewer + Orchestrator) for shipping multi-phase coding deliverables. The skill itself is at [skills/fram-loop/SKILL.md](skills/fram-loop/SKILL.md). This `CLAUDE.md` is for agents working *on* the skill (editing it, releasing it, evaluating it), not for agents *running* the skill on a project.

## Repo layout

```
fram-loop/
├── LICENSE
├── README.md             # user-facing skill description, install instructions, lineage
├── EVALS.md              # eval retrospective (per-run findings + improvement proposals)
└── skills/
    └── fram-loop/
        ├── SKILL.md      # the skill — protocol, defaults, phase structure, rubric
        └── references/
            └── templates.md  # Builder + Reviewer prompt scaffolds
```

Keep root minimal. Per-eval test deliverables (e.g. *The Margin*) live as standalone repos under `~/Documents/Projects/fram-ateliers/`, not in this repo. Do not commit `site/`, deliverable folders, or scratch worktrees here — `gh skill install` only ships `skills/fram-loop/` to install users, but anyone browsing or cloning the repo would see clutter.

## Release process — read before tagging

The canonical installer is `gh skill install` (built into `gh ≥ 2.90`, shipped April 2026, validates against the [agentskills.io](https://agentskills.io/specification) spec). Two non-obvious things about its resolver:

1. **Tag ≠ Release.** `gh skill install TheRealClodius/fram-loop fram-loop` (no version pin) resolves to the **latest GitHub Release**, NOT the latest pushed tag. A tag with no Release is invisible to the resolver. Always cut a Release alongside the tag.
2. **Bundled assets ride along — but only if they exist at the tagged commit.** `references/templates.md` is part of the skill. If you tag a commit that predates a bundled asset, install users get a stale skill missing that asset. Tag late, not early.

### Cutting a release

```bash
# 0. Bump the plugin manifest version to match the tag (marketplace users only
#    receive updates when this bumps — see .claude-plugin/plugin.json "version")
#    Then sanity-check: claude plugin validate .   (one known CLAUDE.md warning is expected)

# 1. Commit your changes on main
git add skills/
git -c user.name="Andrei Clodius" -c user.email="andrei@fram.design" commit -m "feat: <change summary>"

# 2. Tag it (annotated)
git tag -a v0.X.Y -m "v0.X.Y — <one-line summary>"

# 3. Push commit + tag
git push origin main
git push origin v0.X.Y

# 4. Cut the GitHub Release (THIS is what makes the version installable)
gh release create v0.X.Y --title "v0.X.Y — <title>" --notes "<release notes>"

# 5. Verify
gh release list                                                      # v0.X.Y should be Latest
gh skill preview TheRealClodius/fram-loop fram-loop                  # fetches from the latest Release — confirm content matches what you just tagged
# (gh ≥ 2.92 removed `gh skill install --dry-run`; `gh skill preview` is the replacement)
```

If you forget step 4, `gh skill install` will keep resolving to whatever was the last Release. Symptom: install users report missing files or stale conventions.

### Versioning policy

- **Patch (`v0.X.Y` → `v0.X.Y+1`)** — clarifications, bug fixes in protocol, README tweaks, prompt-scaffold tightening.
- **Minor (`v0.X` → `v0.X+1`)** — new conventions (new §Defaults entries), changed rubric thresholds, breaking changes to the prompt scaffolds users may copy.
- **Major (`v0` → `v1`)** — first time the skill ships in a state we'd recommend without caveats.

## Install / update workflows users will hit

Documenting here so changes to the skill don't break user expectations:

| Command | Behavior |
|---|---|
| `gh skill install TheRealClodius/fram-loop fram-loop` | Installs latest Release into project scope (`.agents/skills/fram-loop/`) when run inside a git repo, or with `--scope user`/`--agent claude-code` overrides. Default agent is `github-copilot` non-interactively — `--agent claude-code` lands in `.claude/skills/`. |
| `gh skill install TheRealClodius/fram-loop fram-loop@v0.1.1` | Pin to a specific Release/tag. |
| `gh skill update fram-loop` | Compares local `metadata.github-tree-sha` to the latest Release; updates if newer. **Requires the install to have metadata** — only canonical `gh skill install` injects it; third-party installers (`gh upskill`) leave the metadata block out. |
| `gh skill update fram-loop --force` | Re-downloads even when local matches remote. |

Known cosmetic issues (do not break, but worth knowing):

- `gh skill update` reports each install twice when `.claude/skills/fram-loop` is a symlink to `.agents/skills/fram-loop` (the scanner counts both as separate installs).
- Project-scope `gh skill install` puts the skill in `.agents/skills/` (the agentskills.io cross-host standard), but Claude Code only auto-discovers `.claude/skills/`. The setup in `~/Documents/Projects/fram-ateliers/the-margin/` symlinks the two so Claude Code finds it.

## Editing the skill

The skill is a prose contract executed by an autonomous loop. Edits to `SKILL.md` change behaviour in production-like runs, sometimes in surprising ways. Discipline:

- **Match the existing voice** — the skill prose is intentionally direct, slightly opinionated, explains *why*. Don't soften it into corporate-doc language.
- **Defaults #N entries** are load-bearing — Builders and Reviewers cite them by number. Do not renumber existing entries; append new ones with the next number.
- **§Goal, §When to use, §When NOT to use** are the trigger surface for the skill description optimization. Changes there affect whether the skill auto-fires correctly. Validate against eval queries (see EVALS.md §6 for ideas) before releasing.
- **`references/templates.md` scaffolds are copy-pasted by Orchestrators** — keep slot-fill placeholders explicit (`[…]` or `<...>` markers) so per-phase customisation is filling slots, not authoring fresh.
- **Carry-forward state machine** is `[Open] → [Addressed-in-Phase-K] | [Deferred-to-Phase-K] | [Out-of-scope-v2]` — see SKILL.md §Carry-forward. Don't introduce new states without updating the Builder/Reviewer report scaffolds.

## Running an eval against the skill

The pattern that worked in eval run #1 (see [EVALS.md](EVALS.md)):

1. Pick a deliverable spec'd from a real or invented brief (multi-phase, design-heavy if possible).
2. `git -C <fram-loop> worktree add --orphan -b <eval-branch> /tmp/<eval-dir>` — orphan-branch worktree gives a clean ephemeral fixture.
3. Pre-stage `<eval-dir>` with the spec, the harness-shaped plan, and any seed content. The skill expects these as input.
4. From inside `<eval-dir>`, install the skill at user or project scope.
5. Trigger the skill with a phrase from §When to use ("use the fram-loop", "spin up the fram-loop", etc.).
6. Observe — do not intervene mid-run except for genuine BLOCKED states. fram-loop's resilience-over-rigidity rule means most "stuck" rounds resolve themselves.
7. After final verification, **don't autopush** to origin — eval runs aren't real ships. Mark `partial` and surface the push commands. (Captured as §Defaults #9 in v0.1.1.)
8. Extract the deliverable to a permanent standalone repo if worth keeping (`git clone --single-branch --branch <harness> file://<fram-loop> <dest>`). Tear down the worktree.
9. Write the retrospective to `EVALS.md` with metrics (phases, scores, tokens, fix rounds, RFF count) and a concrete improvements list.

## What `gh skill install` ships to install users

Only `skills/fram-loop/` (the matched skill folder per `skills/*/SKILL.md`). NOT the rest of the repo. README.md, LICENSE, EVALS.md, CLAUDE.md (this file), and any other root-level content are visible to repo browsers/cloners but invisible to install users.

This means it's safe to put eval artifacts, design notes, and developer-facing docs at the repo root — they never leak into the skill's runtime footprint.
