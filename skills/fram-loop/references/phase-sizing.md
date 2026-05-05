# Phase sizing — what "coherent" means

The §Phase Structure table sets 2–4 tasks as the right size. Whether 5+ is OK depends on whether the tasks are *coherent* — sharing a feature surface, gating on each other, or addressing one spec section.

## Coherent (5+ tasks fine in one phase)

- **One form component.** Five fields of one form, sharing validation logic, single user story.
- **Wiring one tool.** Install + config + autofix script + lint script + CI integration for one ESLint plugin. Sequential dependency chain — splitting leaves broken intermediate state.
- **One DB migration.** Create table + indexes + foreign keys + seed data + read-path updates. All gating on the table's existence.
- **One API endpoint.** Route + handler + validation schema + tests + error responses. One feature surface.

## Incoherent (split into separate phases)

- **Five unrelated bug fixes.** No shared surface, no gating relationship.
- **Mixed scope.** Bug X + feature Y + refactor Z bundled together.
- **Multiple endpoints** unless they share a common base implementation.
- **Cleanup work bundled with feature work.** Each obscures the other's scope.

## The principle

Tasks belong in the same phase when they gate on each other or address one spec section / feature surface. If they're "stuff that happened to be on the list," split them.

When in doubt, prefer 3–4 task phases — that's the sweet spot for Builder context legibility and Reviewer review surface. 5+ is fine when the tasks are tightly coupled; if you find yourself writing connective tissue in the phase description to explain how the tasks relate, that's a sign they don't.
