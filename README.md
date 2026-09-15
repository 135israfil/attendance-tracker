# Employee Attendance & PC Activity Tracker

A GPS-based attendance system paired with a privacy-conscious Windows desktop agent for PC activity monitoring — built, tested, and verified end-to-end across ten phases.

**Status:** complete and delivered. Source is private; this repo documents the build. See [`docs/development-plan.md`](docs/development-plan.md) for the full phase-by-phase log, including test counts, live-verification notes, and every non-obvious decision made along the way.

## Why I built this

Attendance and "is anyone actually working" are two different questions, and most tools that try to answer both end up doing neither well — GPS attendance apps that don't tell you if someone's PC is on, or activity monitors that record far more than anyone consented to. This system draws that line deliberately: attendance is GPS-verified but never blocking (a wrong GPS reading flags a punch, it doesn't lock someone out), and PC monitoring is scoped tightly enough that a CI script actively fails the build if anything resembling keylogging, screen capture, or clipboard access shows up in a dependency.

## What it does

- GPS check-in/check-out with configurable split shifts, half-day leave in 0.5-day increments, and admin-configurable weekends and holidays
- A Windows tray agent tracking active/idle time, foreground application, and website **domain only** — never a full URL, and window titles are read only as a cheap change-detection signal and discarded immediately, never stored
- Nightly absentee marking, activity retention policies, and full audit logging on every admin action
- Attendance and PC-activity reports with CSV/Excel export
- An installer (PyInstaller + Inno Setup) that autostarts the agent per-user, supports silent IT rollout, and cleanly uninstalls including its local offline queue

## Architecture

```
Employee phone/browser ──HTTPS──┐
Admin browser ──────────HTTPS──┤
                                ├──> nginx ──> Django + DRF (gunicorn) ──> PostgreSQL
Windows PC agent ───HTTPS+token─┘                    │
   └── local SQLite queue (offline buffer)           └──> (optional) Redis + Celery beat
```

One Django project rather than microservices — reporting constantly joins attendance and activity data, and a single database avoids distributed joins entirely. Three deliverables share it: a Django+DRF backend, a Vue 3 SPA (one build serving both admin and employee roles), and a Python tray agent packaged separately.

The part that took the most care: **split shifts**. An employee can have one shift (the common case, unchanged in behavior) or two, and a single pure, unit-tested matching function decides which punch belongs to which shift — nearest-start wins, a surplus punch is stored unmatched and flagged for review rather than rejected outright, so nobody's afternoon silently disappears because they punched an extra time by accident.

## Delivered and verified

Not just written — verified live against a running stack at every phase:

- **308 backend tests at 94% coverage**, 138 agent tests, zero missing migrations, clean linting on both trees
- A real agent enrolled, tracked activity across all three major browsers (including incognito/private windows), and correctly survived being killed without warning — the abandoned session closed at its last heartbeat on restart
- The installer was actually installed and uninstalled, not just read: a silent rollout landed the binary, autostart entry, and privacy notice in the right places, including the tricky part — writing the autostart registry key to the *logged-in user's* hive during an elevated install, not the administrator's
- A `pg_dump`/restore drill was actually run against a scratch database and came back with every table intact
- The full acceptance checklist (36 criteria) passed against the live stack, not inferred from the test suite

## Stack

Django + Django REST Framework, Vue 3 (Vite, Pinia, Vue Router), PostgreSQL, Python Windows tray agent (`pywinauto`/`uiautomation` for browser domain capture, `psutil` for process info, `keyring` for credential storage), Docker Compose + nginx + Let's Encrypt.

## What I'd do differently

The single highest-risk item — reading the active browser tab's domain via UI Automation across Chrome, Edge, and Firefox — was correctly run as a throwaway spike before any real code depended on it, and that's the one process decision here I'd keep unconditionally on the next project like this. If I were starting over, I'd write the CI privacy-guard script (the one that fails the build on anything screen-capture- or keylogging-adjacent) in phase 0 rather than during phase 0's wrap-up — it ended up gating every later phase, and it earns its place even earlier than it landed.
