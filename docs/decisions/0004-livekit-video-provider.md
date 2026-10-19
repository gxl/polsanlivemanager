# ADR 0004 — LiveKit (self-hosted) as the video provider

- Status: Accepted
- Date: 2026-10-19

## Context
v1 needs an external video platform behind the VideoProvider adapter (ADR-0001) that supports:
(1) creating meetings via API, (2) short-lived per-user joins, (3) join/leave events to our server
(BR-5), (4) server-side removal of a participant (BR-8), (5) blocking rejoin for the session
(BR-8), (6) data kept under our control (student personal data; KVKK applicability pending).

## Decision
Use **LiveKit**, self-hosted (Turkish data centre or on-premises). A Meeting = a LiveKit room.
VideoProvider creates/closes rooms and calls `RemoveParticipant` through LiveKit's server API,
mints **per-user access tokens** (JWT signed with the LiveKit API secret, TTL ≤ 5 min to connect,
identity = our user id, room = the session's room) only after BR-3/BR-4 pass, and consumes LiveKit
**webhooks** (`participant_joined`, `participant_left`, `room_finished`), verified by their signed
JWT and body hash. "Block for this session" = VideoProvider refuses further tokens for that user
and room. The SPA renders the room with `livekit-client` / `@livekit/components-react`.

## Consequences
- Buys: every v1 requirement maps to a documented server feature; BR-8 is one server call;
  rejoin blocking needs no provider feature; no per-host licences; data stays on our servers.
- Costs: we build the in-app video room screen (from LiveKit components, no WebRTC code of our
  own); we operate a LiveKit server (TURN/TLS, ports, upgrades, capacity); the browser talks to
  the LiveKit media server directly (FD-5 exception, narrowed); no built-in whiteboard/chat/polls.

## Alternatives considered
- **BigBlueButton** — full classroom app, signed join URLs and webhooks (add-on); server-side
  removal/blocking not confirmed in the classic API. Runner-up.
- **Jitsi Meet** — JWT joins, but events and server-side removal need Prosody modules.
- **Zoom / Teams** — vendor cloud outside our control, reusable links, licensed, limited
  server-side removal.

## Revisit triggers
The spike (first VideoProvider feature) fails to confirm webhooks, `RemoveParticipant` or token
TTL behaviour; operating the server proves too costly (→ LiveKit Cloud, new ADR); whiteboard /
recordings become in scope.
