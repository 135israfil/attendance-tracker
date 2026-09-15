# Development Plan — Employee Attendance & PC Activity Tracker

Derived from `README.txt` v1.0. This plan is the build order; the README stays the
requirements source of truth. Priority order inherited from §39:
**Reliability > Simplicity > Features.**

**Progress:** Phases 0–9 complete (2026-08-19). All 36 §37 criteria verified — see the
Phase 9 note for the run.

A full audit against every README section followed, which turned up one requirement that
had been decided away rather than built: §5's "the company should decide whether
outside-radius check-ins are permitted" was a fixed policy, not a setting. It is now
`require_check_in_inside_radius` (D2), default off so the shipped behaviour is unchanged.
The audit also found the README's own §4, §9 and §27 still describing the pre-D7 schema —
one check-in per day, work hours as two columns on Employee — so those sections were
amended to match what was built. §28's `/api/agent/activity/` remains folded into
`sync/`, which §28 explicitly permits and §3 explains.

Remaining before release: a production deployment against the runbook, which needs a
host; and confirming the Fri–Sat weekend default against the real office (§9 below).
Spike findings: `claudedocs/SPIKE-REPORT.md`.
Setup instructions live in `DEVELOPMENT.md`.

---

## 0. Locked decisions

Resolved before schema work, since each one touches the data model:

| # | Question | Decision | Consequence |
|---|----------|----------|-------------|
| D1 | Single vs multi company (§30) | **Single company** | No `Organization` model, no org scoping. One `OfficeConfig` singleton row. |
| D2 | Outside-radius check-in (§5) | **Allow and flag by default, blockable by setting** | `location_status` = INSIDE / OUTSIDE stored per punch, and admins filter on it. §5 asks the company to decide, so `require_check_in_inside_radius` on `AppSettings` turns the flag into a refusal. Default off: phone GPS is wrong often enough that a hard block locks genuine staff out, and a refused punch leaves no record of someone who did turn up. Check-in only — blocking check-out would strand anyone who left the building before closing their punch. |
| D3 | Agent process model (§11) | **User-session tray app** | Autostart via `HKCU\...\Run`. Session 0 isolation would make active-window and idle detection impossible, so no Windows Service. |
| D4 | Weekend days | **Admin-configurable** | Set of weekdays on `AppSettings`, default Fri–Sat. Read by the absentee job and every report. |
| D5 | Activity retention window | **Admin-configurable** | `activity_retention_days` on `AppSettings`, default 90. Applies to raw `Activity` rows only — attendance is retained indefinitely (see A6). |
| D6 | Half-day leave | **Admin-configurable** | `allow_half_day_leave` toggle. Leave amounts stored as `Decimal` in 0.5 steps, not integers. |
| D8 | Private browsing (incognito / InPrivate) | **Tracked, same as normal** | Verified readable on all three browsers. No separate flag, no exclusion. Carries an obligation: the §31 employee notice must say so explicitly — people assume private browsing is exempt. |
| D7 | Split shifts | **Admin-configurable, per employee** | Work schedule moves from two columns on `Employee` to an `EmployeeShift` table, and attendance moves from one punch pair per day to an `AttendancePunch` table. This is the expensive one — see §2 and the Phase 3 note. |

### Assumptions (change these here if wrong)

- **A1 — Timezone:** office is `Asia/Dhaka` (UTC+6), matching the README's lat 23 / long 90 examples. All datetimes stored UTC; the attendance *date* is derived in office-local time. Configurable in settings, not hardcoded at call sites.
- **A2 — Scale:** ~50–200 employees, one office. Sizing target for indexes and report queries, not a hard cap.
- **A3 — Employee credentials** are created by the admin (email + generated initial password). No self-registration, no email verification flow in the MVP.
- **A4 — Absent** is *derived*, not entered: a working day with no attendance row and no leave marking. Materialised nightly so reports stay simple.
- **A5 — Deployment** is a single Linux VM with Docker Compose behind nginx + Let's Encrypt. Not Kubernetes, not serverless.
- **A6 — Retention applies to activity only.** D5 deletes raw `Activity` sessions on a schedule. Attendance, leave and audit records are the legal/HR trail and are never auto-deleted. Say so explicitly if the retention policy is ever challenged.
- **A7 — Half-day granularity is 0.5.** Quarter-days are not supported; `Decimal(4,1)` covers the range.
- **A8 — Split shift means 2 shifts.** The `EmployeeShift` table imposes no limit, but the UI and late-attribution rules are designed and tested for one or two shifts per day.

---

## 1. Architecture

```
Employee phone/browser ──HTTPS──┐
Admin browser ──────────HTTPS──┤
                                ├──> nginx ──> Django + DRF (gunicorn) ──> PostgreSQL
Windows PC agent ───HTTPS+token─┘                    │
   └── local SQLite queue (offline buffer)           └──> (optional) Redis + Celery beat
```

Three deliverables, one backend:

1. **backend/** — Django + DRF, session auth for the web app, device-token auth for agents.
2. **frontend/** — Vue 3 SPA (Vite, Pinia, Vue Router), one build serving both admin and employee roles.
3. **desktop-agent/** — Python tray app, packaged with PyInstaller, installed by Inno Setup.

Rationale for a single Django project: §32 explicitly rules out microservices, and the
reporting queries join attendance and activity constantly — one database avoids
distributed joins entirely.

---

## 2. Data model

Refines §27. Deltas from the README are marked **[+]** and explained.

### accounts

**User** — custom user model, created in the very first migration (swapping later is a
migration nightmare).
- `email` (login field, unique), `password`, `role` ∈ {ADMIN, EMPLOYEE}, `is_active`
- `AbstractBaseUser` + `PermissionsMixin`

### employees

**Employee** — one-to-one with User.
- `user`, `employee_id` (unique, e.g. `EMP-001`), `name`, `phone`
- `department`, `designation`, `joining_date`
- `leave_total`, `leave_used` — **`Decimal(4,1)`** per D6, so a half day is 0.5. README
  says "leave balance"; storing total and used makes §10's "Total 12 / Used 4 /
  Remaining 8" display exact and auditable. `leave_remaining` is a derived property,
  never a column.
- `status` ∈ {ACTIVE, INACTIVE}
- **No `work_start_time` / `work_end_time`** — replaced by `EmployeeShift` below (D7).

**EmployeeShift** **[+], from D7** — `employee`, `sequence` (1, 2, …), `start_time`,
`end_time`, unique on `(employee, sequence)`.

An employee with one row is a normal 09:00–18:00 worker; two rows is a split shift
(09:00–13:00, 14:00–18:00). The single-shift case stays the default everywhere — new
employees get one row seeded from the `AppSettings` defaults, and the admin UI only
offers "add second shift" when `allow_split_shift` is on. Modelling this as rows rather
than as nullable second-shift columns means the late-minute logic loops over shifts
once instead of branching on "is this a split-shift employee" at every call site.

**LeaveAdjustment** **[+]** — append-only log of balance changes (`employee`, `delta`
as `Decimal(4,1)`, `reason`, `created_by`, `created_at`). §10 lets admins add and deduct
balance; without a log, "why is my balance 6.5?" is unanswerable.

### attendance

**AppSettings** — singleton, the concrete backing for §35. Named for what it is now that
it carries policy as well as geography.

- *Office:* `name`, `latitude`, `longitude`, `radius_meters`
- *Office, D2:* `require_check_in_inside_radius` (bool, default off) — §5's company-level choice; off means an outside punch is recorded and flagged, on means it is refused
- *Schedule:* `default_start_time`, `default_end_time`, `timezone`
- *Schedule, D4:* `weekend_days` — set of weekday numbers, default {Fri, Sat}
- *Schedule, D7:* `allow_split_shift` (bool, default off) — controls whether the admin UI
  offers a second shift; existing shift rows keep working if it is turned back off
- *Leave, D6:* `allow_half_day_leave` (bool, default off)
- *Activity:* `idle_threshold_seconds` (300), `heartbeat_interval_seconds` (60), `sync_interval_seconds` (60)
- *Activity, D5:* `activity_retention_days` (default 90, minimum enforced so nobody sets 0 by accident)

**Attendance** — one row per employee per date, holding the *day-level* summary.
- `employee`, `date` — **unique together**
- `first_check_in_at`, `last_check_out_at` (UTC) — denormalised from the punches for
  fast report and list queries; recomputed whenever a punch changes
- `worked_seconds` — sum of closed punches
- `late_minutes` — sum across shifts (see below)
- `status` ∈ {PRESENT, LATE, ABSENT, LEAVE}
- `leave_fraction` **[+], from D6** — `Decimal(2,1)`: 0, 0.5 or 1.0. Kept separate from
  `status` so a half-day-leave morning plus a worked afternoon is representable as
  `PRESENT` + 0.5 rather than needing a `HALF_DAY` status that reports then have to
  special-case. `status = LEAVE` means `leave_fraction = 1.0`.
- `is_manually_adjusted`, `adjusted_by`, `adjustment_note` **[+]** — §9 allows admin
  correction; §30 requires audit logging of important admin actions.

**AttendancePunch** **[+], from D7** — the actual in/out pairs.
- `attendance`, `sequence` — unique together
- `check_in_at`, `check_in_lat/lng/accuracy_m/distance_m/location_status`
- `check_out_at`, `check_out_lat/lng/accuracy_m/distance_m/location_status`
- `matched_shift` (FK `EmployeeShift`, nullable) — which shift this punch is attributed to
- `scheduled_start` **[+]** — snapshot of the matched shift's start time at check-in.
  Without it, editing someone's schedule silently rewrites months of late history.
- `late_minutes` — for this punch

**Punch → shift matching rule.** On check-in, pick the employee's shift whose
`start_time` is nearest to the punch, excluding shifts already matched today; fall back
to the lowest unmatched sequence if nothing is close. Late minutes are
`max(0, punch_time − matched_shift.start_time)`, and the day's `late_minutes` is their
sum. For a single-shift employee this collapses to exactly the §8 behaviour, so nothing
about the common case changes. Matching is a pure, unit-tested function — the ambiguous
cases (an early first punch, a missed shift, three punches against two shifts) are
decided there and nowhere else.

**Holiday** **[+]** — `date`, `name`. Needed to answer "is this a working day?" before
A4 can mark anyone absent, alongside `AppSettings.weekend_days` (D4).

### computers

**Computer**
- `employee`, `computer_name`, `operating_system`, `agent_version`
- `enrollment_code` **[+]**, `enrollment_expires_at` **[+]** — admin pre-creates the
  computer row and hands the employee a one-time code. The README's §28 register
  endpoint has no stated trust anchor; without this, anyone who can reach the API can
  enrol a device and post activity.
- `device_token_hash` **[+]** — token is hashed at rest and shown exactly once at
  registration, like an API key. Never stored in plaintext.
- `last_seen_at`, `status` ∈ {ONLINE, IDLE, OFFLINE, DISABLED}

### activities

**Activity** — the session model from §18.
- `employee`, `computer`
- `activity_type` ∈ {PC_ON, PC_OFF, ACTIVE, IDLE, APPLICATION, WEBSITE}
- `application_name`, `executable_name`, `website_domain`
- `start_time`, `end_time`, `duration_seconds`
- `client_event_id` (UUID) **[+]** — generated on the agent, **unique per computer**.
  This is the entire anti-duplicate mechanism §19 asks for: sync is an upsert on
  `(computer, client_event_id)`, so replaying a batch after a dropped response is safe.
- Indexes: `(employee, start_time)`, `(computer, activity_type, start_time)`

Note the tracks run in parallel: an `APPLICATION` session and a `WEBSITE` session and
an `ACTIVE` session can all cover the same wall-clock minute. Nothing sums across
types — only within a type.

**DailyActivitySummary** **[+], deferred to Phase 8** — per employee per day rollup of
pc_on / active / idle seconds plus top app and top domain. Only build it if dashboard
queries measurably slow down; premature otherwise.

### audit

**AuditLog** **[+]** — `actor`, `action`, `target_type`, `target_id`, `payload`,
`created_at`. Required by §30. Written on: employee create/disable, attendance
correction, leave adjustment, office config change, device enable/disable.

---

## 3. API surface

Aligned with §28, with the enrollment split made explicit.

### Agent (device-token auth, `Authorization: Device <token>`)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/agent/register/` | Body: enrollment code + machine info → returns device token (once) |
| POST | `/api/agent/heartbeat/` | Liveness + current state (active/idle) → returns config version |
| POST | `/api/agent/sync/` | Batch of activity sessions, idempotent on `client_event_id` |
| GET  | `/api/agent/config/` | Idle threshold, intervals, kill switch |

`/api/agent/activity/` from §28 is folded into `sync/` — one code path for online and
offline means the offline path is exercised on every single request instead of only
during outages. That is the reliability-first choice.

### Attendance (session auth, employee role)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/attendance/check-in/` | lat, lng, accuracy → opens the next punch for today |
| POST | `/api/attendance/check-out/` | lat, lng, accuracy → closes the open punch |
| GET  | `/api/attendance/my/` | Own history with punches, date-filtered |
| GET  | `/api/me/summary/` | Employee dashboard payload (§24) |

Check-in and check-out keep the same signature they had before split shifts — the punch
sequence and shift matching are entirely server-side, so the phone UI stays two buttons.
The employee page shows CHECK IN when no punch is open and CHECK OUT when one is, and
for a split-shift employee that simply happens twice in a day.

### Admin (session auth, admin role)

`/api/employees/` CRUD · `/api/employees/{id}/shifts/` · `/api/settings/` · `/api/holidays/` ·
`/api/attendance/` (list, filter, correct punches) · `/api/leave-adjustments/` ·
`/api/computers/` (list, enrol, disable) · `/api/activity/` ·
`/api/dashboard/` · `/api/reports/attendance/` · `/api/reports/activity/` ·
`/api/reports/{type}/export/?format=csv|xlsx`

---

## 4. Phased build

Sizes are rough relative buckets (S < M < L), not time estimates. Each phase ends with
a demoable, tested increment.

### Phase 0 — Foundations · S — **DONE**
Repo layout per §33. Docker Compose (postgres + backend). Django project, custom User
model + first migration, settings split via env vars, pytest + factory_boy + coverage,
ruff + black, pre-commit, CI running lint and tests. Vue app scaffolded and talking to
the API. Health endpoint.

**Exit:** `docker compose up` gives a working stack; CI is green.

**Delivered.** Verified: `docker compose up --build` brings both services to healthy,
`/api/health/` returns 200, 7 tests pass against real Postgres at 93% coverage, ruff and
black clean, no missing migrations. Frontend builds and eslint is clean at
`--max-warnings 0`. The §6 privacy guard shipped early as `scripts/check_privacy.py`
(wired into CI and pre-commit) and was verified to both pass clean and fail on a planted
violation. Git repo initialised; files staged, not yet committed.

### Phase 1 — Risk spikes · M — **DONE**
Two throwaway spikes that can change scope, so they run first:

1. **Browser domain capture.** Read the active tab's URL from Chrome, Edge and Firefox
   via UI Automation (`uiautomation` / `pywinauto`), reduce to bare domain, discard the
   rest in the same function. Test against all three browsers, incognito windows, and
   Chrome with accessibility off. **This is the single highest-risk item in the
   project** — see §7 Risks for what happens if it fails.
2. **Idle + foreground app.** `GetLastInputInfo` for idle, `GetForegroundWindow` →
   `GetWindowThreadProcessId` → `psutil` for app and exe name. Verify behaviour on a
   locked workstation and over RDP.

**Exit:** a written spike report recording, per browser, whether domain capture works
and how reliably. Scope decision made and recorded here before Phase 6 starts.

**Delivered** — full results in `claudedocs/SPIKE-REPORT.md`. Domain capture **works on
all three browsers**, normal and private, with no accessibility prefs and no per-profile
changes required. 12/12 trials pass. Foreground app, executable and idle detection all
work. Six findings now bind later phases:

- **Title-gated polling is required, not optional.** A UIA read costs 46ms; a window
  title read costs 0.001ms (~42,000× cheaper). Polling the title and paying for UIA only
  when it changes turns 2.32% of a core into ~0.0001%. Phase 6 designs to this.
- **Window titles must never be stored.** The sampler surfaced titles like
  `PLAN.md - Visual Studio Code` — document and file names, which is file-activity
  monitoring under §31. Titles are a change signal only.
- **Tick counters wrap every ~49.7 days.** `GetLastInputInfo`/`GetTickCount` subtraction
  must be masked to `0xFFFFFFFF` or idle time goes negative on long-uptime machines.
- **`process.exe()` can be denied** for elevated processes; `process.name()` still works,
  and §15 only requires the executable "where available".
- **Match on control type, not just name.** Chromium exposes the address bar as an
  `EditControl`, Firefox as a `ComboBoxControl` named `Search or enter address` — Firefox
  has *zero* EditControls, so a name-only matcher finds nothing there. Firefox also reads
  2–3× slower (110–259ms), which makes title-gating more important, not less.
- **"Window exists but is not yet readable" is normal.** A browser can take many seconds
  after launch before its UIA tree answers. The collector retries instead of recording a
  gap.

### Phase 2 — Auth & employee management · M — **DONE**
Login/logout, role-based permissions, employee CRUD, disable (never hard-delete),
department/designation, `EmployeeShift` management (D7), leave balance in half-day
decimals + `LeaveAdjustment` (D6), holidays, audit log wired into all of the above.

**Settings screen (§35), now load-bearing** — office location and radius, timezone,
default hours, weekend days (D4), split-shift toggle (D7), half-day-leave toggle (D6),
idle threshold, heartbeat and sync intervals, activity retention days (D5). Every one of
these reads from `AppSettings` at use time; none are duplicated into code constants.

**Exit:** §37 criteria "Admin can create employees", "Employee can login", plus a test
that changing weekend days changes which dates the absentee job treats as working.

**Delivered.** 67 tests pass at 92% coverage; ruff, black, eslint and the privacy guard
are clean; no missing migrations. Verified against the running stack, not just in tests:
CSRF → admin login → employee created with an auto-seeded shift; employee login blocked
from `/api/employees/` (403) and from writing settings (403) while still able to read
them (200, needed by the Phase 3 check-in page).

Implementation notes worth carrying forward:

- **Policy toggles are enforced server-side.** D6 and D7 are refused by the API when
  disabled, not merely hidden in the UI — tested both ways round.
- **Disabling an employee deactivates the login.** Otherwise an "inactive" employee could
  still authenticate and check in. Tested by attempting a login afterwards.
- **Employees are never deleted** — `DELETE` returns 405 — because attendance and
  activity history must outlive the person leaving.
- **Audit entries survive actor deletion** (`on_delete=SET_NULL`), tested by deleting the
  admin and asserting the entry remains with its target intact.
- **Shortening retention (D5) requires explicit confirmation** at the API, with a second
  confirmation in the UI. Lengthening it does not.
- **`AppSettings` writes are idempotent**: `objects.create()` folds into the singleton
  rather than raising a duplicate-key error.
- **Leave adjustments take a row lock** so two concurrent adjustments cannot both read
  the same starting balance.
- `AuditLog` lives in `core` rather than its own app, since §33 does not list one and
  every app writes to it.

### Phase 3 — GPS attendance · M→**L** (D7) — **DONE**
Haversine distance helper (pure function, unit-tested). Check-in opens the next
`AttendancePunch`, runs shift matching, snapshots `scheduled_start`, computes
`late_minutes`, stores coordinates and the inside/outside flag per D2. Check-out closes
the open punch and recomputes the day's `worked_seconds`, `first_check_in_at`,
`last_check_out_at` and total `late_minutes`. Rejects a second check-in while a punch is
open, and rejects more punches than the employee has shifts. Own-history view. Admin
correction at punch level with audit trail. Nightly `mark_absentees` command (A4) reading
`weekend_days` + `Holiday`, run from cron — Celery not required for this.

Half-day leave (D6): admin marks a date as half-day, which sets `leave_fraction = 0.5`
and deducts 0.5 from `leave_used`; the employee can still punch for the other half and
the day resolves to PRESENT or LATE with the fraction recorded.

The size bump from M to L is entirely D7 — the punch table, the matching function and
its edge cases, and the day-level recompute on every punch change.

**Delivered.** 125 tests pass at 91% coverage; all ten §37 Attendance criteria met.
Verified live as well as in tests: a 19:42 UTC check-in filed correctly against the
**next** local date (Dhaka is UTC+6), distance 55.6m INSIDE, a double check-in refused,
and a check-out 2.2km away recorded as OUTSIDE rather than blocked (D2).

**Rule change from the Phase 3 sketch:** a surplus punch is **stored unmatched and
flagged**, not rejected. The earlier line said "rejects more punches than the employee
has shifts", which contradicted the risk table and was the worse behaviour — refusing
someone's third punch means their afternoon goes unrecorded entirely. Surplus punches
now carry no `matched_shift`, contribute 0 late minutes, and set `needs_review` on the
day so an admin sees them. A `MAX_PUNCHES_PER_DAY = 10` ceiling guards against a stuck
client, and admins can filter the list on `needs_review`.

Other decisions made while building:

- **Check-out searches today *and* yesterday.** Someone who checks in at 23:50 and out
  at 00:10 must be able to close the punch they actually opened; without this their day
  would stay open forever and a second Attendance row would appear.
- **Worked time is the sum of closed punches**, never last-out minus first-in — tested
  with a split shift where the two differ by a full hour of lunch.
- **`recompute()` re-queries punches instead of using the related manager.** The admin
  queryset uses `prefetch_related`, so `attendance.punches.all()` returned a *cached*
  pre-edit list and corrections silently recomputed from stale rows. Caught by a test.
- **Corrections recalculate against the stored `scheduled_start` snapshot**, so fixing a
  punch cannot retroactively apply a schedule change made afterwards.
- **Leave marking reuses `LeaveAdjustment`** (target=USED) rather than a parallel
  mechanism, so the leave trail has one shape.
- **The absentee job is a management command, not Celery** — it runs once a day and
  needs no broker. It refuses to process today or a future date, which would flag
  everyone who simply has not arrived yet.

Responsive employee check-in page: big CHECK IN / CHECK OUT buttons, browser geolocation
permission prompt, clear error states for permission denied, timeout, and low accuracy.

**Exit:** all ten §37 Attendance criteria.

### Phase 4 — Agent v0.1: registration & liveness · M — **DONE**
Tray app skeleton, config file in `%PROGRAMDATA%`, device token in Windows Credential
Manager via `keyring` (not plaintext on disk). Enrollment-code registration flow.
Heartbeat loop. Server marks a computer OFFLINE when three intervals pass with no
heartbeat, and **closes any open session at the last heartbeat time** — crashes and
power cuts never send a shutdown event, so server-side inference is mandatory, not a
nicety. Startup / shutdown / sleep / wake via `WM_QUERYENDSESSION` and
`WM_POWERBROADCAST`.

**Exit:** §37 "agent installed", "computer registered", "PC online/offline status".

**Delivered.** 169 backend tests at 93% coverage plus 43 agent tests; ruff, black, eslint
and the privacy guard clean on both trees; no missing migrations. Verified live against a
running server, not only in tests: a real agent enrolled with a lowercase, dash-less code,
reported `Windows 11` and its version, opened a PC_ON session, and — after being killed
without warning — closed the abandoned session at the dead run's last heartbeat on
restart, then closed its own session on a reported shutdown.

Decisions made while building:

- **Lifecycle events ride the heartbeat.** `POST /api/agent/sync/` is Phase 5 work, so
  rather than invent a fourth endpoint, `heartbeat` carries an optional `event`
  (STARTUP / SHUTDOWN / SLEEP / WAKE / LOCK / UNLOCK). A shutdown notice therefore
  travels the one channel already proven to work, at the one moment there is no time to
  discover that a second channel is broken.
- **`STARTUP` is sent by the service as its first heartbeat, not by the system
  collector.** Emitting it from the Windows message pump raced the heartbeat thread:
  whichever request landed first refreshed `last_seen_at`, and the previous run's
  abandoned session then absorbed the downtime it existed to exclude. Caught during live
  verification, not by a test — the unit tests were happy with both orderings.
- **The `Activity` model ships now, in full.** Phase 4 only writes PC_ON rows, but
  building the table to the §2 shape immediately avoids reshaping it under Phase 5's
  sync path, and `close()` clamps a backwards clock to a zero-length session rather than
  a negative one.
- **The admin computer list sweeps before it answers.** The cron job is the backstop;
  without the inline sweep the one screen whose job is to say whether a PC is on would
  cheerfully claim ONLINE twenty minutes after the machine died.
- **Disabling a device clears the token hash**, and enabling issues a fresh code. A
  machine disabled because its credential leaked must not resume on that credential.
  Issuing a *new* code deliberately does not revoke the old token, so re-imaging a PC
  never knocks a working one offline in the meantime.
- **Device auth is opted into per view, never global.** A device token is authority to
  report one machine's own activity; in `DEFAULT_AUTHENTICATION_CLASSES` it would
  silently become authority against every session endpoint. Tested by pointing a device
  token at `/api/employees/` and `/api/attendance/my/`.
- **Agent throttles are scoped separately** (`agent-register` 30/min, `agent` 120/min).
  The browser defaults are wrong in both directions here: an anonymous 60/min would be
  spent by a NAT-shared office mid-rollout, and 2000/hour is far more than a heartbeat
  loop needs.
- **`config_version` is milliseconds, not seconds.** Two settings saves inside one second
  are ordinary, and at second resolution the agent would see no change at all.
- **The fleet kill switch is `AppSettings.agent_enabled`.** Agents keep reporting
  liveness while it is off, so a fleet stopped centrally can be started again centrally.

Not in this phase and deliberately so: the SQLite offline queue, the session builder and
`POST /api/agent/sync/` all belong to Phase 5, and the installer to Phase 9. Until then
the agent is started by hand and lifecycle events are best-effort — which the server
already treats as the normal case.

### Phase 5 — Agent v0.5: sessions, offline buffer, sync · L — **DONE**
The core of the agent.

- Collectors produce state changes; a **session builder** converts them into closed
  sessions. Sessions close on: state change, idle transition, app switch, and a **hard
  15-minute cap** so a crash loses at most 15 minutes rather than a whole day.
- Every session gets a `client_event_id` UUID at creation.
- Local SQLite queue: `INSERT` on creation, `synced` flag set only after a 2xx.
- Sync service: batch upload, exponential backoff, resume from the queue on startup.
  Replays are harmless by construction (idempotent upsert).
- Backend ingestion: validates ownership, rejects disabled devices, clamps absurd
  durations and future timestamps.

**Exit:** §37 active time, idle time, application usage and duration, offline storage,
and offline sync. Test by pulling the network cable mid-session.

**Delivered.** 192 backend tests at 93% coverage plus 85 agent tests; ruff, black and the
privacy guard clean on both trees; no missing migrations. Verified live against a running
server with the cable-pull case simulated as a server URL nothing answers on: eight
minutes of activity collected while the upload failed, nothing marked synced, then a
clean drain on recovery producing exactly the §14/§15 shape — 5m Visual Studio Code, 3m
Google Chrome, one 8m ACTIVE session spanning both, and IDLE once the user walked away.
Replaying every one of those payloads over the wire returned `accepted=0, duplicates=4`.

Decisions made while building:

- **Idle closes the application track.** A PC left on overnight with an editor focused
  would otherwise report eight hours of "using Visual Studio Code". Idle time is real and
  recorded on its own track; it is simply not application usage.
- **A gap in the samples closes sessions at the last sample, not at the resumption.** If
  the laptop slept for twenty minutes, nothing was observed in between — the same
  principle the server applies when it closes a PC session at the last heartbeat. Without
  it, a lunchtime sleep is silently counted as work.
- **The server derives duration from the timestamps** and ignores any the client sends. A
  client-supplied duration that disagrees with its own start and end is a bug report, not
  a data point.
- **A bad session is named individually in the response and dropped locally.** Rejecting
  a whole batch for one malformed row would leave that row at the head of the queue
  forever, retried every cycle, blocking everything behind it — an outage caused by one
  bad record.
- **`PC_ON` and `PC_OFF` cannot be submitted by an agent.** They are derived from
  heartbeats; accepting them from a client would let a broken agent rewrite its own
  uptime.
- **The domain field is re-validated server-side as a bare host**, even though the
  collector truncates before anything is written. §17 is worth two lines of defence: the
  agent's truncation is the mechanism, this is what happens if it ever regresses.
- **Three independent loops** — heartbeat, sampler, sync. A server outage must not stop
  collection, and a stuck collector must not stop the machine reporting that it is alive.
- **Absurd durations are clamped, not rejected.** The agent caps at 15 minutes, so an
  hour-long session means a clock stepped; clamping keeps the time rather than discarding
  someone's afternoon.
- **Confirmed uploads are purged after a day.** The grace period is for people, not
  correctness: when someone asks why an afternoon looks wrong, the last day of uploads is
  still on the machine. (Written, then found unwired during review, then wired and
  tested — the test asserts the loop calls it, not just that the method exists.)

### Phase 6 — Agent v1.0: website domains · M — **DONE**
Gated on the Phase 1 spike result. Domain only, per §16 and §17 — the URL is truncated
to its host in the collector, before it is ever written to disk or sent anywhere. Only
the foreground tab of the foreground window counts. Explicit unit tests asserting that
paths, query strings and fragments never appear in stored data.

**Exit:** §37 website usage tracked, website duration calculated.

**Delivered.** 127 agent tests (41 of them on this phase) and 192 backend tests; ruff,
black, eslint and the privacy guard clean; no missing migrations. Verified live against a
real Chrome window launched with a throwaway profile — the production collector read
`https://example.com/private/account?id=123&token=sup3rs3cret#billing` and returned
`example.com`, with no path, query, token or fragment surviving, and ten consecutive
samples cost exactly **one** UIA read.

The live check refuses to read any window whose process it did not start, which is the
same discipline the Phase 1 spike used: verifying browser capture must not mean reading
somebody's actual tabs.

Decisions made while building:

- **Title-gated reads, as the spike demanded.** The collector caches on the window
  title's signature and pays for UI Automation only when it changes — 1 read per 10
  samples in the live check, against 10 without it. The title itself is hashed and
  discarded in the same function that reads it; it is a change signal, never data.
- **An unreadable window holds the previous domain for five samples, then gives up.**
  "Window exists but is not yet readable" is normal after a browser launches; recording
  a gap there would mean real browsing shows as nothing. Ten seconds is the ceiling, so
  a genuinely closed tab is never credited with minutes it did not have.
- **Website sessions run alongside application sessions, not instead of them.** "Chrome
  for two hours" and "github.com for twenty-five minutes" are both true, and §18's tracks
  never sum across each other.
- **Idle closes the website track too.** A tab left open over lunch is not lunch spent
  reading it.
- **The privacy guard now rejects `uiautomation`'s dual-use half.** The library is
  required for §16 and cannot be banned outright, so `scripts/check_privacy.py` gained a
  call-level check that fails the build on input synthesis (`SendKeys`, `Click`, …),
  bitmap capture (`CaptureToImage`, `BitmapFromWindow`, …) and clipboard access. Verified
  by planting a violation and watching CI-equivalent checks fail, then removing it.
- **`psutil` was imported by the Phase 5 application collector but never declared** in
  `requirements.txt`. Found while adding `uiautomation`; a PyInstaller build would have
  shipped an agent that could not name a single application.

### Phase 7 — Dashboards & activity page · M — **DONE**
Admin dashboard (§20) — today's attendance counts and PC status counts. Employee list
(§21). Employee activity page (§22) with date selector, attendance summary, PC summary,
top applications, top websites, and the timeline (§23). Employee dashboard (§24),
strictly own-data.

**Exit:** all nine §37 Dashboard criteria, plus a test proving employee A cannot read
employee B's data through any endpoint.

**Delivered.** 223 backend tests at 93% coverage; ruff, black, eslint and the privacy
guard clean; frontend builds. Verified live against a seeded day matching §22's example:
the timeline came back as `09:01 PC started · 09:05 Google Chrome / github.com ·
09:42 Visual Studio Code · 10:30 Idle · 10:45 Visual Studio Code`, and every isolation
probe from an employee session returned 403.

Decisions made while building:

- **Sessions are clipped to the day, not summed by stored duration.** A PC on from 22:00
  to 02:00 gives two hours to each date. The stored `duration_seconds` is the length of
  the session; a day view needs its overlap with that day, and they are different
  numbers. Open sessions count up to now — or to midnight if the day is in the past, so
  a session nobody closed does not keep growing every time a report is opened.
- **Nothing sums across activity types**, and the tests say so explicitly. ACTIVE,
  APPLICATION and WEBSITE sessions routinely cover the same minute; adding them would
  produce a number that means nothing.
- **"Not recorded" is separate from "absent" on the dashboard.** Absence is derived
  nightly (A4); at 10am a missing row means someone has not arrived yet, and calling
  that absent would be wrong for most of the working day.
- **The roster is three queries regardless of headcount.** The obvious per-employee loop
  is 4×N and looks perfectly healthy on a developer's five-row database. There is a test
  asserting the query count so it stays that way.
- **Every dashboard payload carries the office timezone**, and the frontend formats with
  it rather than the viewer's locale. Found during live verification: the timeline was
  rendering 03:01 for a 09:01 event. The *date* a session belongs to is already decided
  in office-local terms, so a display in the viewer's zone could put the morning on the
  wrong day entirely.
- **`/api/me/summary/` has no employee parameter at all** — not one that is validated,
  one that does not exist. The employee comes from `request.user`, so there is nothing
  to tamper with, and the isolation test asserts that passing `?employee=` to it changes
  nothing.
- **Refusing another employee's id is a 403, not a silent fall-back to own data.**
  Quietly returning something else hides the attempt from the caller and from whoever
  reads the logs.
- **The timeline is capped at 200 entries** and reports how many were dropped. §23 is a
  human-readable narrative, not an audit log, and a busy machine produces thousands of
  app switches a day.

### Phase 8 — Reports & export · S — **DONE**
Attendance report and PC activity report with the §25 filters and columns. CSV and
Excel export (`openpyxl`), streamed for large ranges. Add `DailyActivitySummary` only
if measured query times justify it.

§25's attendance columns say "Check-in" and "Check-out" singular; with D7 those are
`first_check_in_at` and `last_check_out_at`, plus a punch count and an expandable
per-punch detail row. Worked time comes from `worked_seconds`, not from
last-minus-first, so a two-hour lunch gap is not counted as work.

**Exit:** all four §37 Reports criteria.

**Delivered.** 278 backend tests at 93% coverage (55 new); ruff, black, eslint and the
privacy guard clean; no missing migrations; frontend builds. Verified live against a
seeded day: the attendance report returned Ayesha 09:02–18:00 worked 8:58 late 2 Inside
office (41 m), Bashir 08:55–17:30 worked 8:35 Outside office (812 m), and Farhana's
split shift as worked 8:00 across two punches against a 10:00 elapsed span. The activity
report gave Ayesha PC on 9:00 / active 7:00 / idle 2:00 with Visual Studio Code 5:00 and
github.com 1:30. Both exports were downloaded through the browser and their contents
checked; every employee-session and anonymous probe against all four endpoints returned
403.

`DailyActivitySummary` was **not** added. The measured shape did not justify it: the
activity report is one query per requested day covering the whole roster, so cost scales
with the range the caller chose rather than with headcount, and there is a test asserting
the query count stays flat as employees are added. A rollup table would have to be kept
correct against late-arriving agent syncs and midnight-crossing sessions, which is real
complexity bought against a cost nobody has measured as a problem yet. Phase 9's
retention job caps the table's size independently.

Decisions made while building:

- **Bashir's PC-on time is 10:30 on the 18th and 2:00 on the 19th**, from one machine
  left on from 22:00 to 02:00. The report clips to the office-local day using the same
  `overlap_seconds` and `day_window` the dashboards use — imported, not reimplemented —
  so the two screens cannot drift apart about how long a day was.
- **Worked time is `worked_seconds`, never last-check-out minus first-check-in.** For a
  split shift those differ by the length of the lunch break, and there is a test that
  asserts the elapsed span is 10 hours while the reported figure is 8.
- **An export ignores the page size.** The JSON endpoints page at 100–200 rows; the
  export takes the whole filtered range. A CSV that silently stopped at row 200 would be
  worse than no export, so a test drives seven employees through a two-row page size and
  asserts all seven reach the file.
- **CSV streams; Excel cannot.** The CSV response carries no `Content-Length` — it is
  generated row by row through a `StreamingHttpResponse`. An xlsx is a zip whose central
  directory is written last, so it is built in openpyxl's write-only mode and sent whole;
  CSV is the format to reach for on a genuinely large range.
- **Excel gets real durations, CSV gets text.** The xlsx writes seconds as a fraction of
  a day under an `[h]:mm` format, so a column of PC time actually sums; `[h]` rather than
  `h` because a fortnight of activity is not "3:30". CSV has no cell types, so it writes
  `H:MM`. The difference is inherent to the formats, not an oversight.
- **The CSV starts with a byte-order mark.** Without it Excel on Windows reads a UTF-8
  file as the system code page, and the first column is people's names.
- **Only employees with recorded activity get a row.** A zero row for every employee on
  every weekend day is noise; "no row" already reads as "nothing recorded". Farhana, who
  has attendance but no agent, is correctly absent from the activity report and present
  in the attendance one.
- **Exports are audited, ordinary reads are not.** An export takes the whole workforce's
  attendance and activity out of the system into a file that then travels on its own. The
  entry is written when the stream is exhausted rather than when it starts, so a
  cancelled download is not logged as a completed one, and it records the row count.
- **The date range is capped at 366 days.** A mistyped year is the realistic way an
  unbounded report happens, and the failure without a cap is a timeout rather than a
  message.
- **`URL_FORMAT_OVERRIDE` is now off.** DRF reads `?format=` during content negotiation
  and 404s when it names a renderer that is not installed; only JSONRenderer is, so the
  feature could never do anything but reject a request — and it collided with §26's
  documented `?format=csv|xlsx`, which is a parameter of ours rather than a renderer
  selector.
- **Grouping by employee is offered alongside grouping by day.** §25 lists a date-range
  filter but no date column for the activity report, which reads two ways; both are
  available and a test asserts the grouped totals equal the sum of the daily rows, so
  they are two views of one number rather than two numbers.

### Phase 9 — Hardening & release · M — **DONE**
HTTPS + HSTS, secure cookie flags, DRF throttling on auth and agent endpoints,
password hashing verified, dependency audit, `.env` secret handling. Retention job (D5)
deleting `Activity` older than `activity_retention_days`, run nightly, with a dry-run
mode and a logged count of deleted rows — a delete job nobody can audit is a liability.
Installer:
PyInstaller onefile → Inno Setup, `HKCU\Run` autostart, silent install flags for IT
rollout, uninstall that removes the local queue. Data-retention job per §31. Deployment
runbook, backup and restore procedure, admin operating guide, and an employee-facing
notice covering what is and is not collected.

**Exit:** deployed, backed up, documented, and a full pass over the §37 checklist.

**Delivered.** 308 backend tests at 94% coverage (30 new, in `tests/test_hardening.py`)
and 138 agent tests; ruff, black, eslint and the privacy guard clean; no missing
migrations; `manage.py check --deploy --fail-level WARNING` clean in a production
configuration with one documented silence (W021, HSTS preload — see the note in
`settings.py`). Dependency audit run over all three halves: `pip-audit` on the backend
and agent requirements and `npm audit --omit=dev --audit-level=high` on the frontend all
reported nothing, and the audit runs as its own CI job so a newly-published advisory
against a pinned version fails visibly rather than looking like the tests broke.

**The §37 checklist passes 36/36**, verified live against a running stack rather than
inferred from the test suite. The 32 criteria reachable over HTTP were driven end to end
through the real API — session + CSRF for the browser half, `Device <token>` for the
agent half — on a scenario seeded through the public endpoints: employee created, logged
in, checked in at 55.6 m INSIDE and out at OUTSIDE, late minutes derived server-side,
a punch corrected by an admin (the day flips to manually-adjusted and late is
recomputed), a computer enrolled and registered, a four-session batch synced and then
replayed to prove idempotence (second attempt accepted 0), 40 min active / 15 min idle /
30 min VS Code / 18 min github.com read back on both the admin and employee views, and
both reports filtered by date and exported as CSV (UTF-8 BOM) and xlsx (a real zip). The
remaining four are not HTTP-shaped and were verified directly: a machine silent past the
cutoff swept to OFFLINE with its open session closed; the agent's SQLite queue holding
three sessions across a process restart and draining on sync; and the installer, below.

Verified while running, not just written:

- **The installer was actually installed and uninstalled**, not read. A silent
  `/VERYSILENT /SERVER=…` run exited 0, put the exe, the privacy notice and an
  uninstaller in `C:\Program Files\OfficeTracker`, wrote
  `HKCU\...\Run\OfficeTrackerAgent` pointing at the installed binary, and seeded the
  server URL so an unattended rollout leaves machines needing only their enrolment code;
  the installed exe answered `--status`. The uninstaller then removed all three — binary,
  autostart and `C:\ProgramData\OfficeTracker` with the local queue in it. The
  `runasoriginaluser` trick in `[Run]` is the part that had to be proven rather than
  assumed: an elevated install writing an Inno `[Registry]` HKCU entry would have
  configured autostart for the *administrator's* hive, leaving it silently absent for the
  person who actually logs in. The value landed in the interactive user's hive.
- **The backup and restore procedure was executed, not just documented.** A `pg_dump -Fc`
  taken exactly as `docs/backup-and-restore.md` writes it, restored into a scratch
  database, came back with all six tables identical to the source (7 employees, 7
  attendance days, 8 punches, 6 computers, 25 activity rows, 9 users). A runbook nobody
  has run is a hypothesis.
- **All three scheduled jobs ran against real data**: `purge_activity --dry-run` reported
  its window, cutoff and count without deleting; `mark_absentees` marked 4; and
  `mark_offline_computers` marked 1 and closed its session.

Decisions made while building:

- **Deleting is judged by when a session ended, not when it started.** A session that
  began before the cutoff but ran past it is partly inside the window people were
  promised, so it is kept; open sessions have not ended at all and are never eligible.
- **The retention job keeps its own floor.** `AppSettings` already validates a minimum,
  but the module that does the deleting does not delegate its own safety — a settings row
  written before that validator existed, or edited straight in the database, must not be
  able to wipe everything. Seven days, enforced in `retention.py` and again in the
  command's `--days` override.
- **A run that deleted nothing is still audited.** "The job ran and found nothing" and
  "the job did not run" are different facts, and only one of them is a problem; an audit
  trail that cannot tell them apart makes a silent failure look like a quiet week.
- **Rows go in batches of 5,000.** One `DELETE ... WHERE start_time < cutoff` over a year
  of a fifty-PC fleet is a single long transaction holding locks the agent sync also
  wants. Batching keeps each transaction short and the job interruptible without losing
  the work already done.
- **HSTS preload is off by default.** A wrong HSTS header is remembered by every
  visitor's browser and cannot be withdrawn, so the aggressive settings are opt-in for a
  first deploy (`DJANGO_SECURE_HSTS_SECONDS=0`) and preload stays a deliberate decision
  rather than a default someone inherits.
- **Login is throttled far harder than everything else** (10/min against 60/min anon),
  because it is the only endpoint where brute forcing is useful. Agent registration gets
  its own 30/min scope: a whole office can sit behind one NAT address during a rollout,
  so the browser default would have throttled the rollout itself, and 30/min is still
  hopeless odds against a ten-character single-use code.
- **Two `S105` hits in the agent's uninstall cleanup are silenced, not restructured.**
  `actions["device_token"] = "cleared"` is a status label in a report dict, flagged only
  because the key name contains "token"; contorting the code to dodge the heuristic would
  have been worse than a `noqa` that says why.

Mapping §34 to concrete modules:

```
desktop-agent/agent/
  main.py            tray icon, lifecycle, single-instance lock
  config.py          server URL, token (keyring), intervals, idle threshold
  collectors/
    idle.py          GetLastInputInfo
    application.py   GetForegroundWindow -> PID -> psutil name/exe
    browser.py       UI Automation address bar -> domain only
    system.py        startup/shutdown/sleep/wake/lock via window messages
  sessions.py        state changes -> closed sessions (+ 15 min cap)
  storage/queue.py   SQLite, client_event_id, synced flag
  sync/client.py     batched upload, backoff, resume
```

Polling: ~2 s for foreground app and idle state — fast enough to attribute short app
switches, cheap enough to be invisible in Task Manager. Verify CPU cost in Phase 5.

Config is server-driven: `GET /api/agent/config/` lets admins change the idle threshold
and intervals per §35 without redeploying the agent, and carries a kill switch so a
misbehaving fleet can be stopped centrally.

---

## 6. Privacy & security guardrails

§31's prohibitions are enforced structurally, not by discipline:

- The browser collector's public function returns a **domain string**, never a URL. The
  full URL exists only as a local variable and is never logged. Validated in Phase 1
  against 22 synthetic cases covering paths, query strings, tokens, fragments, embedded
  credentials, ports, `file:///` paths and browser-internal schemes.
- **Window titles are never stored** (Phase 1 finding). They carry document and file
  names, so retaining them would be file-activity monitoring under §31. Titles may be
  read as a cheap change-detection signal and must be discarded immediately.
- `uiautomation` is dual-use — it can also synthesise input and grab bitmaps. It is
  permitted because address-bar reading requires it, but only its tree-reading APIs may
  be used; Phase 6 should narrow the guard to reject its input and bitmap calls.
- No module imports anything capable of screen capture, key hooking, clipboard access,
  or audio/video. A CI grep for `keyboard`, `mss`, `PIL.ImageGrab`, `pyautogui`,
  `sounddevice`, `cv2`, `win32clipboard` fails the build.
- Employee endpoints filter on `request.user.employee` at the queryset level, so
  cross-employee reads fail even if a view forgets a check.
- Device tokens hashed at rest; disabled devices rejected at authentication.
- Audit log on every admin mutation.
- Retention job deletes raw activity older than the configured window.

Non-technical but required: employees must be told tracking is on, per §31.

---

## 7. Risks

| Risk | Impact | Response |
|------|--------|----------|
| ~~Browser URL capture is unreliable~~ — **retired for Chrome and Edge** by the Phase 1 spike | — | Verified working, incognito included, no accessibility flag needed. |
| ~~Firefox domain capture unverified~~ — **retired**: verified working on Firefox 154 | — | Needed a control-type fix (ComboBox, not Edit), no accessibility prefs, no per-profile changes. 12/12 trials pass across all three browsers. |
| Browser updates break address-bar UIA names | Website tracking silently stops | Substring + case-insensitive matching across several known names, plus a value-pattern fallback. Add a health signal so a browser that stops yielding domains is visible rather than quietly absent. |
| Phone GPS accuracy indoors is 30–100 m | False "outside office" flags | D2 already flags rather than blocks. Store `accuracy_m` and show it beside distance so admins can judge. |
| Employees close the tray app | Activity gaps | Accepted for the MVP — D3 chose visibility over lock-in. Gaps are visible as missing heartbeats; escalate only if it becomes a real problem. |
| Unsigned installer trips SmartScreen | Rollout friction | Budget for a code-signing certificate, or document the IT-managed silent-install path. |
| Activity table growth (~50 PCs × hundreds of sessions/day) | Slow reports | Sessions not per-second rows (§18), correct indexes, rollup table ready in Phase 8, retention job in Phase 9. |
| Timezone and DST errors | Wrong dates, wrong late minutes | UTC storage + explicit office timezone, `freezegun` tests around midnight boundaries. Asia/Dhaka has no DST, which helps. |
| ~~Split-shift punch attribution is ambiguous~~ (D7) — **handled in Phase 3** | — | Matching is one pure function in `shift_matching.py` with every edge case enumerated as a test: nearest-start wins, ties go to the earlier shift, matched shifts are not reused, and a surplus punch is stored unmatched with `needs_review` set rather than rejected or guessed at. Admin punch-level correction is the escape hatch. |
| Admin sets a destructive retention window (D5) | Irreversible activity loss | Enforced minimum, confirmation on lowering the value, dry-run mode, deletion counts logged to the audit trail, and the job never touches attendance or leave (A6). |

---

## 8. Testing

- **Pure logic, heavily tested:** haversine distance, late-minute calculation, local-date
  derivation, session merging, idempotent sync, absentee marking. These are where
  correctness actually lives.
- **API tests** per endpoint: happy path, permission denial, cross-employee access denial,
  invalid input.
- **Agent unit tests** with mocked win32 calls; a documented manual matrix on real
  Windows 10 and 11 for install, autostart, sleep/wake, network loss and recovery.
- **End-to-end smoke:** check in → agent produces activity → dashboard reflects it →
  report exports it.
- Coverage gate on `backend/` business-logic modules; no gate on UI glue.

---

## 9. Still open

Nothing blocking. The four previously-open questions are now D4–D7: all admin-configurable,
all resolved in the model above.

Two things to watch rather than decide now:

1. **Defaults matter more than the settings.** Most installs will never touch the
   settings screen, so the shipped defaults (Fri–Sat weekend, 90-day retention,
   half-day off, split shift off) are the behaviour almost everyone gets. Confirm the
   weekend default against the actual office before go-live.
2. **D7 pushes past the README.** §9's "if employee checks in" and §25's singular
   check-in/check-out columns both assume one punch per day. The plan honours the
   configurable behaviour asked for, and the single-shift path is unchanged — but the
   README should be updated to match if it stays the requirements source of truth.
