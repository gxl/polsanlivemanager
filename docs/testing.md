# Testing

> Decided at bootstrap (2026-10-19).

## The contract
- Every acceptance criterion maps to at least one test (criterion ↔ test map lives in the plan).
- Tests assert **behavior**, not implementation details or mere status codes.
- The whole suite runs inside `scripts/check` — one command, everywhere.
- No coverage-percentage gate: the plan's criterion ↔ test map is the gate.

## Frameworks & layout
| Level | Tool | Where | Category / profile |
|---|---|---|---|
| Unit (domain + application, BR-n) | xUnit v3 + Shouldly, fake `IClock` | `tests/<Project>.Tests/` | `Category=Unit` → fast |
| Architecture (FD-1…FD-6) | NetArchTest | `tests/Architecture.Tests/` | `Category=Unit` → fast |
| Integration (API → DB, ProblemDetails codes, auth) | `WebApplicationFactory` + SQLite in-memory per test, fake VideoProvider | `tests/PolsanLive.Api.Tests/` | `Category=Integration` → full only |
| Frontend unit/component | Vitest (`vitest run`) + Testing Library | `web/src/**/*.test.ts(x)` | fast |
| E2E (v1 loop) | Playwright (TS), Chromium headless, `--workers=1` | `web/e2e/` | full only |

- Naming: `Method_Condition_ExpectedResult` (`docs/conventions.md`).
- Every .NET test has a `Category` trait. The fast/full split is a **selector**:
  `--filter "Category=Unit"` (fast) and `--filter "Category!=Unit"` (full only) — a test without
  the trait still runs in full (Two speeds, rule 4).
- **Never** call a real video provider or the network in check: VideoProvider is replaced by a
  fake behind its Contracts interface.
- **App-under-test lifecycle (e2e): owned by Playwright `webServer`** — it starts the published API
  (serving `web/dist`) on a fixed port with a throwaway SQLite file and the fake provider, waits
  for `/health`, and stops it at the end, also on failure/timeout. `reuseExistingServer: false`.

## What must be tested
- Every business rule (BR-n in `docs/domain.md`) with its boundaries (e.g. BR-5 at exactly
  `start+10:00` and `+10:01`; BR-4 at `start−10:00` and `end`; BR-9 on the day the retention
  period ends), tagged `[Trait("Rule","BR-n")]`.
- Every forbidden dependency FD-1…FD-6 (FD-5 via ESLint).
- AuthZ: every endpoint/hub method has a test for an unauthorized role (default-deny), incl.
  "instructor of another session cannot remove" (BR-8) and "non-enrolled student cannot join" (BR-3).
- Every `Error.code` the API can return, as ProblemDetails.
- Token lifecycle: access expiry, refresh rotation, reuse of a rotated refresh token is rejected.
- E2E smoke of the v1 loop: schedule → join → remove → attendance shows status + removal flag.

## Protected-tests rule
Weakening asserts, deleting, or skipping tests to reach green is forbidden. A red test triggers
`prompts/recovery/red-test.md` (R-02) — first decide what is wrong: code, test, or spec.

## Characterization tests
Before refactoring untested code (`workflows/refactor.md`) and for brownfield change requests
(`workflows/change-request.md`, Preserved behavior), pin the current behavior first — warts
included. They are written from observation, not from what the code "should" do.

## Evidence for UI criteria
A screenshot is evidence for a UI criterion; for change requests, before/after screenshots that
also show the preserved behavior. Screenshots go into the PR, not the repo (Evidence policy).
A screenshot proves a criterion only against the spec's "User experience": every state it lists
(empty, loading, error, success, no permission), compared with the linked design if there is one.

## Evidence policy
1. **Evidence is referenced, not stored.** Each acceptance criterion's evidence is a pointer: the
   test's name and the command that reproduces it (`./scripts/check`). The criterion ↔ evidence
   table in the spec/plan is the only evidence artifact that lives in the repo.
2. **Reproducible evidence is never filed.** No saved check output, console dumps, coverage reports
   or log copies in the repo — git history and CI runs already archive every execution.
3. **Non-reproducible evidence** (a screenshot of a manual UI check, a one-off measurement, an
   external-system confirmation) goes into the pull request (description or attachment); the
   evidence table links to it. It is the exception, not the rule.
4. **Never clone or copy the repository inside its own working tree.** Clean-clone verification
   runs in a temporary directory outside the repo; only the one-line result (command + green/red)
   is recorded in the evidence table. The clone is always deleted, never kept.
5. **There is no `evidence/` (or `.evidence/`) directory in this workspace.** An agent that feels
   the need to create one is about to violate rule 1. `scripts/doctor` warns on both violations.

## Resource hygiene
1. **A check run leaves nothing running.** No survivors of any kind: build/compile daemons,
   package-manager services, test hosts and parallel workers, watch modes, browsers, the application
   under test, containers or emulators started for the run. Agents run check an order of magnitude
   more often than humans; per-run residue compounds into machine starvation.
2. **The residue test** (the rule's definition, not advice): run `./scripts/check` twice back to
   back; after each run, process count and memory return to the baseline measured before the first.
   `./scripts/residue` runs it (`--fast` for the fast profile) and names every survivor with its
   command line; it lists anything started meanwhile too, so run it on an otherwise idle machine.
   Rerun it whenever `scripts/check.conf` changes.
3. Warm-process caches trade memory for speed for humans who build rarely; with an agent, trade the
   seconds back for a flat memory profile.
4. Interactive modes (watchers, dev servers, REPLs) never belong in check. e2e runs single-worker
   and headless by default; one component owns the app-under-test lifecycle — started for the run,
   stopped at the end, including on failure and timeout.

## Two speeds, one contract
1. `./scripts/check` is still the one way to ask "is this good?"; `--fast` is a speed of the
   contract, not a second contract.
2. **Fast green is a working state; only full green is evidence.** Per step, per finding, per
   iteration → `--fast`. Segment exits, PROVE steps, recovery-ramp exits, the verify table and CI
   → `CHECK GREEN (full, …)`. `scripts/doctor` warns when a shipped spec, or in CI the pull
   request description, shows `CHECK GREEN (fast, …)` and no `(full, …)`.
3. Fast: compile and the quick tests (unit, lint). Full only: e2e, long integration suites, and
   any single step over ~a minute — the minute is a per-step threshold, not a total for the fast
   profile; fast has no fixed total. Bootstrap decides per stack and tags the steps `[fast]` in
   check.conf. When fast grows too long to iterate with, take steps out in this order: lint
   first, then formatting, then unit tests; compilation stays longest.
4. **Split a test step by a selector, never by a list.** The fast step selects with a filter
   (a category, trait, tag or namespace) and the full-only step runs the same filter inverted,
   so "fast ∪ the rest = every test" holds by construction: a test added later still runs in
   one of them. Splitting by naming projects or files leaves a silent hole — a new one that is
   not added to check.conf runs in neither profile while check stays green.
5. Durations come from check itself (`<== <step> (n s)` per step, the total in the green line).
   Decide a split, or report a profile's duration, on the median of at least 3 runs — single
   runs vary too much to decide on.

## Determinism
Flaky tests are fixed, not retried or skipped — see R-03. Evidence of a fix: 5 consecutive green runs.
