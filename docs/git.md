# Git

> Decided at bootstrap (2026-10-19). Hosted on GitHub; default branch `main`.

## Branching
- `feature/<spec-no>-<short-name>` — **no branch without a spec.** Also for CHANGE-lane mini-specs.
- Refactors: `refactor/<spec-no>-<short-name>` (mini-spec with Changed behavior = none).
- Fixes: `fix/<report-id>-<short-name>` (bug fixes carry a report, not a spec number);
  incidents: `incident/<date>-<short-name>`; trivial changes: `trivial/<short-name>` (see below).

## Commits
- Conventional Commits, with a plan reference: `feat(scheduling): overlap check [plan 0001/2]`.
- Scopes: `identity`, `courses`, `scheduling`, `attendance`, `video`, `api`, `web`, `e2e`,
  `build`, `docs`.
- Agent commits follow the same standard: the agent writes the message, the human approves.

## Forbidden
- Direct commits to the default branch.
- Force push, history rewriting on shared branches. Undo = `git revert` (see recovery R-11).

## Trivial changes
<!-- Applies only to requests the change-request triage rubric
     (workflows/change-request.md) classifies as TRIVIAL: no acceptance criterion changes, no shared code. -->
- Policy: **(1) PR + one reviewer, no spec** — chosen at bootstrap (2026-10-19).
  Branch `trivial/<short-name>`, squash-merged after one approving review.
- Always: one narrow commit, `scripts/check` green. A second file or a new test means re-triage.

## Pull requests
- PR template checklist completed; `scripts/check` green in CI; squash-merge.
- `main` is protected on GitHub: PR required, the `check` and `doctor` CI jobs required,
  at least one approving review, no force push.
