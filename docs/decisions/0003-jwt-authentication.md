# ADR 0003 — JWT authentication issued by the Identity module

- Status: Accepted
- Date: 2026-10-19

## Context
The SPA, the JSON API and the SignalR hub need one authentication mechanism. Accounts are
admin-provisioned (closed pilot); no SSO in v1. The team chose JWT over cookie sessions.

## Decision
The Identity module (ASP.NET Core Identity for users/passwords) issues **ES256-signed access JWTs**
(15 min lifetime, `kid` header for key rotation) and **opaque refresh tokens** (7 days, rotated
on every use, stored hashed server-side so they are revocable). The SPA keeps the access token
**in memory only** and sends it as `Authorization: Bearer`; the refresh token lives in an
`HttpOnly; Secure; SameSite=Strict` cookie scoped to `/auth/refresh`, which also requires an
`X-Requested-With` header. SignalR passes the access token as the `access_token` query value
(stripped from logs). JwtBearer validates issuer, audience, lifetime and signature (skew ≤ 30 s).

## Consequences
- Buys: stateless API/hub authorization; XSS cannot read the refresh token; stolen access tokens
  expire within minutes; revocation via refresh-token store; key rotation without downtime.
- Costs: a refresh endpoint and token-rotation logic to build and test; signing keys to manage
  as secrets; a page reload needs a silent refresh before the first API call.

## Alternatives considered
- **Cookie session auth** — recommended at bootstrap (simplest for a same-origin SPA); declined.
- **JWT in localStorage** — readable by any XSS; rejected for student data.
- **HS256** — any verifier could mint tokens. **External IdP** (Keycloak/Entra) — brings SSO and
  an extra service into v1; revisit with SSO.

## Revisit triggers
SSO or LMS integration requested; a second API consumer; multiple host instances (key distribution).
