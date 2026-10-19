# ADR 0001 — Modular monolith on .NET 10 + React/TypeScript

- Status: Accepted
- Date: 2026-10-19

## Context
PolsanLive Manager is a new, standalone web app for live education sessions at pilot scale
(< 1k users, tens of concurrent sessions) with one external integration (a video platform).
The team needs fast delivery now and the option to split or swap the video provider later.

## Decision
Build one ASP.NET Core (.NET 10 LTS) deployable organised as a modular monolith (Identity,
Courses, Scheduling, Attendance, VideoProvider) with boundaries enforced by architecture tests,
and a React + TypeScript (Vite) SPA served by the same host.

## Consequences
- Buys: one deploy, one DB, in-process calls and transactions; boundaries (FD-1…FD-6) are
  machine-checked, so a later split follows existing seams; strong typing for time and IDs.
- Costs: discipline overhead (Contracts projects, events instead of direct calls); one process
  means one scaling unit; two toolchains (dotnet + node) in check and CI.

## Alternatives considered
- **Layered monolith** — faster first week, but no module seams; tangles as features grow.
- **Microservices** — operational cost far exceeds benefit at pilot scale.
- **Node/NestJS, Java/Spring, Python/FastAPI** — viable; .NET chosen for typing, decimal/UTC
  handling, LTS horizon (Nov 2028) and mature boundary tooling (NetArchTest).

## Revisit triggers
A module needs independent scaling or release cadence; > 50k users; a second team owns a module.
