# Architecture

> Decided at bootstrap (2026-10-19). Describes the architecture we *decided on*. Agents read this
> file before planning; change it only through an ADR (`docs/decisions/`).

## System overview
PolsanLive Manager is a **modular monolith** (ADR-0001): one ASP.NET Core (.NET 10) host serving
a JSON API, a SignalR hub, and the built React + TypeScript (Vite) SPA, backed by one **SQLite**
database (ADR-0002). Pilot scale (< 1k users, tens of concurrent sessions). Live video is **not**
built here: a self-hosted **LiveKit** server hosts the Meetings (ADR-0004), managed only through
the VideoProvider adapter. One deployable keeps operations trivial at pilot scale; enforced
module boundaries keep "split later" and "change provider later" cheap.

**Repository layout**
```
src/PolsanLive.slnx
src/Host/PolsanLive.Api/                 # composition root: endpoints, auth, SignalR, serves web/dist
src/SharedKernel/                        # typed IDs, Result/Error, IClock — no framework deps
src/Modules/<Module>/<Module>.Contracts/ # the ONLY public surface: interfaces, DTOs, events
src/Modules/<Module>/<Module>.Domain/    # entities, business rules (BR-n)
src/Modules/<Module>/<Module>.Application/
src/Modules/<Module>/<Module>.Infrastructure/  # EF Core DbContext for this module's tables
tests/<Project>.Tests/                   # xUnit; tests/Architecture.Tests for FD-1…FD-6
web/                                     # React SPA; web/e2e = Playwright
deploy/livekit/                          # LiveKit config as code: livekit.yaml, compose, runbook
```

## Deployment & operations
- **LiveKit (ADR-0004):** **Polsan IT operates** the production server (on-premises or a
  Turkish-hosted VM): provisioning, UDP/TCP ports, TLS + TURN, monitoring, backups, upgrades,
  and on-call for video outages. **The dev team owns the configuration as code** in
  `deploy/livekit/`; changes go through PRs like any code. API key/secret are provisioned by IT
  as secrets, never committed (`docs/security.md`).
- **Local development:** each developer starts LiveKit by hand in dev mode
  (`livekit-server --dev`, binary or container) when trying real video — documented in
  `deploy/livekit/README.md`. It is **never** started by `scripts/check` or CI, which use the
  fake VideoProvider (`docs/testing.md`); stop it when done.

## Modules / components and ownership

| Module | Single responsibility | Owns (tables prefixed) |
|---|---|---|
| **Identity** | Users, roles, login, JWT issuing, refresh tokens | Users, credentials, role assignment, refresh tokens (`identity_*`) |
| **Courses** | Courses, instructor assignment, enrollments | Course, Enrollment (`courses_*`) |
| **Scheduling** | Live Session lifecycle: create, cancel, start/end, join window, overlap (BR-1, 2, 4, 7) | Live Session (`scheduling_*`) |
| **Attendance** | Join/leave events, Removals, status computation, Corrections + audit (BR-3, 5, 6, 8) | Attendance Record, Correction audit (`attendance_*`) |
| **VideoProvider** | The only adapter to LiveKit: rooms, per-user access tokens, `RemoveParticipant`, rejoin blocks, webhooks in | Meeting = LiveKit room, rejoin blocks (`video_*`) |
| Host (`PolsanLive.Api`) | Composition, HTTP/SignalR endpoints, authN/authZ wiring | nothing (no tables) |
| `web/` | SPA UI for the three roles | nothing (talks only to our API) |

## Communication rules
1. A module calls another **only through that module's `*.Contracts`** (interfaces + DTOs), in-process.
2. Cross-module **reactions** use in-process **domain events** published via Contracts
   (e.g. `LiveSessionEnded` from Scheduling → Attendance computes statuses). The publisher never
   knows its subscribers.
3. **Real-time UI** goes through one SignalR hub in the Host. A Removal is a command to Attendance;
   Attendance emits `ParticipantRemoved`; VideoProvider reacts by ejecting the user from the Meeting.
4. Inbound provider **webhooks** (join/leave) enter **only through VideoProvider**, which verifies
   them and translates them into our own events; provider payload types never leave VideoProvider.
5. Each module persists through its **own `DbContext`** mapping only its own prefixed tables;
   one SQLite database file, one transaction per command, no cross-module joins.

## Forbidden dependencies (make them testable)
Asserted by `tests/Architecture.Tests` (NetArchTest), `Category=Unit`, so they run in `--fast`.
- **FD-1:** No module references another module's Domain/Application/Infrastructure — only its `*.Contracts`.
- **FD-2:** No `DbContext` maps another module's tables (table prefix = owning module).
- **FD-3:** Only `VideoProvider.*` references the LiveKit server SDK / LiveKit HTTP API or holds
  the LiveKit API key/secret.
- **FD-4:** `*.Domain` references no ASP.NET Core, EF Core or SignalR types.
- **FD-5:** `web/` calls only our API. Sole exception: the video room screen
  (`web/src/features/video/`) connects to the LiveKit **media server** with `livekit-client` /
  `@livekit/components-react`, using the short-lived token our API issued — it never calls
  LiveKit's server/room API and never holds API credentials. Enforced by ESLint
  `no-restricted-imports` (LiveKit packages allowed only in that folder) and review.
- **FD-6:** `*.Contracts` depend on nothing but the BCL and `SharedKernel`.

## Deliberately out of scope (v1)
Recordings · grading/assessment · analytics/reporting dashboards · minors and a parents/guardians role ·
self-registration · SSO / external LMS integration · automatic idle detection · building our own
video (WebRTC/media server) · multi-tenancy · mobile/desktop apps · microservices.
