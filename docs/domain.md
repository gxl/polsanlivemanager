# Domain

> The shared language between business, humans, and agents. Decided at bootstrap (2026-10-19).
> If a term isn't here, expect the AI to invent its own meaning for it — add it here first.

**Product:** PolsanLive Manager — a web app for running live (synchronous) education sessions.
**v1 loop:** an Admin schedules a Live Session → enrolled Students and the Instructor join through
an external video platform → attendance is recorded. Recordings, grading and analytics are out of
scope for v1 (`docs/architecture.md`, Deliberately out of scope).

## Ubiquitous language

| Term | Meaning | Notes / not to be confused with |
|---|---|---|
| **Admin** | Role that manages users, courses, enrollments and sessions; may act on any session | One user has exactly one role in v1 |
| **Instructor** | Role that runs Live Sessions of the courses assigned to them | |
| **Student** | Role that joins Live Sessions of the courses they are enrolled in | |
| **Course** | A named group of Live Sessions with an enrolled set of Students and an end date (e.g. "Math 9-A") | Not a single meeting |
| **Enrollment** | A Student's membership in a Course | Not Attendance |
| **Live Session** | One scheduled, timed meeting of a Course: `start`, `end`, one Instructor; status Scheduled / Live / Ended / Cancelled | Not the video room (that is the Meeting) |
| **Meeting** | The LiveKit room backing a Live Session (room name, state) — ADR-0004 | Ours = Live Session; the provider's = Meeting. Lives only in VideoProvider |
| **Join token** | Short-lived per-user LiveKit access token our API issues after BR-3/BR-4 pass | Not our JWT (ADR-0003); never reusable, never shared |
| **Join window** | The interval in which joining is allowed: `start − 10 min` to `end` (BR-4) | |
| **Attendance Record** | Per Student per Live Session: status Present / Late / Absent, join and leave times, removal flag | Not Enrollment |
| **Removal** | An Instructor/Admin action that removes a participant from a running Live Session (BR-8) | An action with a flag on the record — *not* an attendance status |
| **Correction** | An audited post-session change of an Attendance Record by an Instructor/Admin (BR-6) | |

## Business rules

- **BR-1:** A Live Session has `end > start` and lasts at least 15 minutes and at most 8 hours.
- **BR-2:** An Instructor never has two non-cancelled Live Sessions that overlap in time.
- **BR-3:** Only Students enrolled in the Course may join its Live Session; the session's Instructor
  and any Admin may always join.
- **BR-4:** Joining is possible only inside the join window: from 10 minutes before `start` until `end`.
- **BR-5:** Attendance is computed when the session ends: first join ≤ `start + 10 min` → **Present**;
  first join later → **Late**; never joined → **Absent**.
- **BR-6:** After a session has ended, its Attendance Records change only through an Instructor/Admin
  **Correction**, and every Correction is audited (who, when, old → new value, reason).
- **BR-7:** A Live Session can be cancelled only before `start`; a Cancelled session produces no attendance.
- **BR-8:** During a Live session the Instructor (of that session) or an Admin may **remove** a
  participant, with an optional reason. The participant's leave time is set to the removal moment;
  Present/Late are computed as in BR-5; the record is flagged "removed by <user> at <time>".
  The participant may rejoin unless the remover chose **block for this session**. "Idle" is the
  Instructor's manual judgement in v1 — no automatic idle detection.

- **BR-9:** Attendance Records and their Correction audit are anonymised when the Course's end
  date + the retention period (default 2 years) has passed; deactivated users are anonymised on
  the same schedule (KVKK — `docs/security.md`, Personal data). A Course therefore has an end date.
- **BR-10:** Only adults (18+) can be Students in v1: creating a Student account requires the
  Admin's attestation that the person is 18 or older.

Thresholds (10 min, 15 min, 8 h, 2 years) are policy values: one named constant each, changed only via a spec.

## Key domain invariants

- **No Attendance Record without an Enrollment** (or for the session's Instructor).
- **No silent edits to attendance after a session ends** — every change is a Correction with an audit entry.
- All domain time is UTC and comes from `IClock` (`docs/conventions.md`, Data rules).
