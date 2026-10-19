# Security

> Decided at bootstrap (2026-10-19). Baseline rules agents must honor in every plan and review.

## Secrets
- Secrets never enter the repo, specs, prompts, or chat. `.env` is gitignored; provide `.env.example`.
- Agents never print secret values, even when debugging.
- JWT signing keys (ES256) and the LiveKit API key/secret (also used to verify LiveKit webhooks) come from
  `dotnet user-secrets` locally and environment variables in deployment. `appsettings*.json` holds
  no secret values.

## Input & output
- Validate every request at the API boundary (FluentValidation or DataAnnotations) → 400
  ProblemDetails. Domain still guards its own invariants.
- Data access via EF Core only; raw SQL only parameterised (`FromSql` with interpolation), never
  string concatenation.
- React escapes by default; `dangerouslySetInnerHTML` is forbidden (ESLint `react/no-danger`).
- Security headers on every response: CSP (`default-src 'self'`; `connect-src` adds only our
  LiveKit server's `wss://` origin),
  `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`,
  `frame-ancestors 'none'`; HTTPS only with HSTS in deployment.
- Errors never leak stack traces, SQL, provider bodies or tokens (`docs/conventions.md`, Error handling).
- Logs: IDs only — never names, emails, passwords, tokens; the SignalR `access_token` query value
  is stripped from request logs.

## AuthN / AuthZ
- **AuthN (ADR-0003):** JWT issued by the Identity module, ES256 with `kid`; access token 15 min,
  kept in SPA memory, sent as Bearer; refresh token 7 days, opaque, rotated on every use, stored
  hashed, revocable, in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to `/auth/refresh`
  plus a required `X-Requested-With` header. Validation: issuer, audience, lifetime, signature,
  clock skew ≤ 30 s. Reuse of a rotated refresh token revokes the whole token family.
- Passwords via ASP.NET Core Identity hashing; lockout after 5 failed attempts.
- **No self-registration in v1** — Admins create accounts.
- **Default-deny:** a fallback authorization policy requires an authenticated user everywhere;
  `[AllowAnonymous]` only on login, refresh and health. The SignalR hub uses the same policies.
- Role policies (Admin / Instructor / Student) at the API layer **and** resource checks in the
  Application layer: an Instructor acts only on their own sessions (BR-8, BR-6); a Student joins
  only enrolled courses inside the join window (BR-3, BR-4).
- **Join tokens (ADR-0004):** the API issues a per-user LiveKit access token (TTL ≤ 5 min to
  connect, identity = our user id, one room) only after BR-3/BR-4 pass and no rejoin block exists;
  Students never receive API credentials or reusable links.
- **Webhooks:** LiveKit callbacks are verified (signed JWT + body hash), rejected outside a
  timestamp window (replay), and processed idempotently.

## Personal data (KVKK)
KVKK (Law No. 6698) **applies** — decided at bootstrap 2026-10-19; legal/DPO confirms details
(privacy-notice text, legal bases, VERBİS obligation) before go-live.
- **Minimisation:** we store only name, email, role, enrollments, session times, attendance
  (join/leave, removal flag) and the correction audit. Nothing else without a spec that names
  the purpose. No video, audio or chat is recorded or stored (recordings are out of scope).
- **Adults only (18+) in v1.** The Admin confirms this when creating a Student account (a
  required attestation field); there is no guardian flow. Minors → new ADR + spec first.
- **Privacy notice (aydınlatma metni):** shown at first login and reachable from every page;
  the text is supplied by legal. Its version and the acknowledgement time are stored per user.
- **Retention (BR-9):** attendance records and their correction audit are **anonymised** at the
  Course's end date + 2 years (period configurable, default 2 years, set by legal); deactivated
  user accounts are anonymised on the same schedule. Anonymised = no user identity, aggregates kept.
- **Data subject requests:** an Admin can export a user's data (JSON) and anonymise a user on
  request; both actions are audited.
- **Location:** all personal data (DB, backups, LiveKit server, logs) stays on servers in Turkey.
  No third-party service that receives personal data — including IP addresses — without a new
  ADR (no external analytics, no externally hosted fonts/scripts in the SPA).
- **Logs:** IDs only, never names or emails (see Input & output); log retention ≤ 90 days.

## Dependencies
- A new NuGet/npm package needs a line in the plan (why, license, maintenance status) and human
  approval at the plan gate.
- Licenses: OSI-approved, permissive preferred (MIT / Apache-2.0 / BSD); no GPL/AGPL in the shipped app.
- Full `scripts/check` fails on known vulnerabilities: `dotnet list package --vulnerable
  --include-transitive` and `npm audit --audit-level=high` (step `audit`).

## Review lens
Security is a mandatory dimension of every independent review (see `prompts/review.md`), not a
separate afterthought phase.
