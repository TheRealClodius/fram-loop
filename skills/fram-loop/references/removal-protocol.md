# Removal protocol — when TDD doesn't fit

§Defaults #1 prescribes test-first commits per task. For dead-code removal, dependency cleanup, or feature deletion, the evidence shape is different — there's no behavior to assert "first." This protocol is the parallel discipline for subtraction work: evidence-based per-task verification, but the evidence is "no remaining callers" rather than "test fails Red."

## The three-step protocol

Per task:

1. **Reference search.** Run `grep` / language-server "find usages" / static analysis output showing 0 callers (or only the definition site). Commit this output alongside the removal as evidence.
2. **Remove + verify.** Full test suite + lint pass post-removal. No new unused-import / unreachable-code diagnostics introduced.
3. **Reviewer reruns the search.** Diff against the Builder's committed evidence. If the Reviewer's enumeration finds a caller the Builder's didn't, that's the issue surfacing.

The diff in step 3 is the audit. The discipline is: produce *something* the Reviewer can re-run.

## What counts as evidence

- Output of `rg "<symbol>"` / `grep -rn "<symbol>"` / language-server "find usages" export.
- Coverage report showing the symbol's lines at 0% prior to removal.
- Static-analysis output listing the symbol as unused (e.g. `ts-prune`, `vulture`, `unused`, language-specific dead-code analyzers).
- Any combination of the above.

The form depends on what's available in the project. Pick the most authoritative tool the language ecosystem provides.

## When this beats TDD

- **Dead-code removal** where there's no behavior to assert against.
- **Dependency cleanup** (removing unused npm packages, stale config sections, deprecated build steps).
- **Feature deletion** where the feature's tests are also being removed.
- **Type-only removals** (deprecated interfaces, narrowed type aliases).

## When TDD still applies

If the removal *changes* observable behavior (not just deletes unused code), TDD applies — write the test that asserts the new behavior, then make the change. The removal protocol is for pure subtraction: the system's surface is smaller, but everything that worked still works.

If you're not sure which applies, lean TDD — the cost of a redundant test is cheap; the cost of an undetected behavior change is not.
