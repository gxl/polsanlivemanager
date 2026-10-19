# Conventions

> Decided at bootstrap (2026-10-19). Only rules that are real: every rule here is either enforced
> by tooling (preferred) or checked in review. Aspirations don't belong here.

## Language
<!-- Set at bootstrap (step 0) and mirrored in AGENTS.md. Changing it later is allowed — edit both
     places; existing documents are not translated retroactively. -->
- Chat language (interviews, sessions): English
- Document language (docs/, specs, plans, ADRs, review and verify reports): English
- Always English (protocol, not prose): ANEW core files, template headings and field labels,
  `Status:` values (Draft / Approved / In progress / Shipped), `Approved by / on:`, `Source:`.
- Code identifiers, branch names, commit messages: English
- Agent messages (handoff summaries, refusals, `Next:` lines): chat language; command names, file
  paths and protocol fields verbatim.

## Language & framework versions
- .NET 10 SDK, pinned in `global.json` (`rollForward: latestFeature`); C# `LangVersion` latest.
- `Directory.Build.props`: `Nullable=enable`, `TreatWarningsAsErrors=true`,
  `AnalysisLevel=latest-recommended`, `ImplicitUsings=enable`.
- Central package management: `Directory.Packages.props` (versions live only there).
- EF Core 10 + `Microsoft.EntityFrameworkCore.Sqlite`; ASP.NET Core Identity; SignalR.
- Node 22 LTS, pinned in `web/.nvmrc` + `engines`; React 19; TypeScript `strict: true`; Vite.
- `package-lock.json` is committed; installs use `npm ci`.

## Naming
- Projects/namespaces: `PolsanLive.<Module>.<Layer>` (Layer ∈ Contracts, Domain, Application,
  Infrastructure); host `PolsanLive.Api`; shared `PolsanLive.SharedKernel`.
- C#: .NET conventions — PascalCase types/members, `_camelCase` private fields, `I` prefix for
  interfaces, `Async` suffix for async methods. One public type per file, file = type name.
- Tables: `<module>_<entity>` snake_case (FD-2 relies on the prefix).
- TS/React: `PascalCase.tsx` components, `camelCase.ts` modules, `useX` hooks; feature folders
  under `web/src/features/<module>/`.
- Tests: `Method_Condition_ExpectedResult`; class `<Subject>Tests`; business-rule tests carry
  `[Trait("Rule", "BR-n")]`; every test carries `[Trait("Category", "Unit"|"Integration")]`.
- Branches and commits: `docs/git.md`.

## Error handling
- Expected failures (validation, business-rule violation, not found, forbidden, conflict) are
  returned as `Result<T>` / `Error` from the Application layer — **never thrown**. Exceptions are
  for the truly unexpected only.
- Every `Error` has a stable dotted `code` (`<module>.<reason>`, e.g. `scheduling.overlap` for
  BR-2, `attendance.join_window_closed` for BR-4).
- The API maps `Error` → **RFC 9457 ProblemDetails** (`code` extension) with 400 / 403 / 404 /
  409 / 422 as appropriate.
- Unhandled exceptions → global handler → generic 500 ProblemDetails with `traceId`; details are
  logged server-side only. **Stack traces, SQL, provider error bodies and secrets never reach the client.**
- Frontend: one API client turns ProblemDetails into a typed error; the UI shows a message keyed
  by `code`, never raw server text.

## Data rules
- **Time:** stored and computed in **UTC** — `DateTime` with `Kind=Utc`, enforced by an EF value
  converter (SQLite cannot query `DateTimeOffset`, ADR-0002). Conversion to the user's time zone
  happens **only in the UI**. Domain code gets "now" from an injected `IClock`; `DateTime.Now` /
  `DateTime.UtcNow` are banned outside the clock implementation (analyzer / review).
- **IDs:** strongly typed `Guid` wrappers per entity (`LiveSessionId`, `CourseId`, …), generated
  as UUID v7 (`Guid.CreateVersion7()`).
- **Enums** (role, session status, attendance status) are persisted as strings.
- Policy thresholds (BR-1, BR-4, BR-5) are named constants, never inline literals.
- Money: none in v1; if introduced, `decimal` only.
- Personal data: store only name, email, role; logs carry IDs, never names or emails.

## Enforced by tooling
- Compiler: nullable reference types, warnings-as-errors, .NET analyzers (`latest-recommended`).
- `dotnet format --verify-no-changes` (whitespace, style, analyzers) — `scripts/check` step `format`.
- Architecture tests FD-1…FD-6 (NetArchTest) — `scripts/check` step `test-unit`.
- `web/`: ESLint (incl. `react/no-danger`, FD-5 `no-restricted-imports`), `tsc --noEmit`,
  `prettier --check` — `scripts/check` step `web-lint`.
- Everything else in this file is checked in review.
