# -Health-Monitoring-Software-for-Astronauts-on-Space-Missions
Astronaut self-assessment app for logging, tracking, and understanding health indicators on long-duration space missions.
# Astro Health Companion

A self-assessment tool that lets astronauts record their own health indicators on long-duration missions,
see whether each reading is in range, get specific guidance, track trends, and keep a daily health routine.

`docs/CHALLENGE_MAPPING.md` maps every challenge requirement to the feature that meets it.

> Prototype. Demo data and illustrative ranges only. Not a diagnostic tool. Not for clinical use.

## Features

| Area | What it does |
|---|---|
| **Today** | Overall status, one colour-coded card per indicator, routine progress, points, streak, badges, flight surgeon notice |
| **Log** | Record a reading. Checked on save, with status, message, and next step. Input limits match the server |
| **Trends** | Any indicator over time, reference band shaded, each reading coloured by status, delete and CSV export |
| **Body map** | Select a body area to see its role in space and the measurements that check it, coloured by status |
| **Routine** | Daily checklist. Logging a measurement completes its task. Points, full-day bonus, streaks, badges |
| **Learn** | For each indicator: what it is, why spaceflight affects it, how to measure it in space, what to do at each level |
| **Escalation** | Three out-of-range readings in a row for one indicator raise a "notify the flight surgeon" notice |

## Indicators

| Indicator | Unit | Accepted range | Spaceflight hazard |
|---|---|---|---|
| Systolic / diastolic blood pressure | mmHg | 40–300 / 20–200 | Microgravity |
| Resting heart rate | bpm | 20–250 | Microgravity |
| Blood oxygen saturation (SpO₂) | % | 50–100 | Hostile closed environment |
| Blood glucose | mg/dL | 10–800 | Inadequate nutrition |
| Body temperature | °C | 25–45 | Hostile closed environment (infection risk) |
| Sleep last night | hours | 0–24 | Isolation and confinement |
| Mood | 1–5 (whole) | 1–5 | Isolation and confinement |

## Quick start

Requires Python 3.10 or newer. No runtime dependencies.

    make run                 # http://localhost:8000
    # or without make:
    cd backend && python3 server.py

Configuration (environment variables):

| Variable | Default | Purpose |
|---|---|---|
| `HOST` | `127.0.0.1` | Bind address. Use `0.0.0.0` in containers |
| `PORT` | `8000` | Listen port |
| `DB_PATH` | `data/astro.db` | SQLite database file (created on first run) |
| `LOG_LEVEL` | `INFO` | Python logging level |
| `MAX_BODY_BYTES` | `4096` | Maximum request body size |
| `CREW_PATH` | `backend/crew.json` | Crew roster file |

### Docker

    make docker
    make docker-run          # http://localhost:8000, data kept in the astro-data volume

The image runs as a non-root user, has a health check on `/api/health`, and stores data under `/data`.

## Tests

    make test                # 40 backend tests: unit, persistence, HTTP end-to-end (stdlib only)
    npm install && make test-ui   # browser-level UI tests with jsdom
    make check               # syntax checks

CI (`.github/workflows/ci.yml`) runs all of the above, then builds the image and smoke-tests the container.

## API

| Method | Path | Body or query | Purpose |
|---|---|---|---|
| GET | `/api/health` | | Liveness |
| GET | `/api/crew` | | Crew roster and environments |
| GET | `/api/indicators` | | Definitions, ranges, guidance |
| GET | `/api/tasks` | | Routine task list |
| GET | `/api/summary` | `astronaut` | Latest status per indicator, overall, flags, points, badges |
| GET | `/api/history` | `astronaut`, `indicator` | Readings over time (includes `id` for deletion) |
| GET | `/api/routine` | `astronaut`, optional `day` (YYYY-MM-DD) | Routine for a day |
| GET | `/api/export.csv` | `astronaut` | Full log as CSV attachment |
| POST | `/api/log` | `{astronaut, indicator, value}` | Validate, store, and evaluate a reading |
| POST | `/api/routine/complete` | `{astronaut, task}` | Mark a task done |
| POST | `/api/demo/seed` | `{astronaut}` | Replace the astronaut's log with 14 days of demo history |
| DELETE | `/api/readings/<id>` | `astronaut` (query) | Delete a reading (for corrections) |

Errors return JSON: `400` for invalid input (with a message), `404` for unknown records or routes,
`413` for oversized bodies, `405`/`404` for unsupported methods, `500` for unexpected failures.

Example:

    curl -X POST localhost:8000/api/log -H 'Content-Type: application/json' \
         -d '{"astronaut":"A1","indicator":"sbp","value":136}'

returns `"status": "caution"`, a message, and the action to take.

## Status rules

Each indicator has an **ok** range and a **crit** range. A value outside ok but inside crit is a
*caution*. Outside crit it is *critical*. A value outside the accepted range is rejected as an input error,
not classified, so a typo like 900 mmHg cannot appear as a critical result.
All ranges live in `backend/guidance.py`.

## Project layout

    astro-health-monitor/
    ├── backend/
    │   ├── server.py          HTTP server, routing, security headers, static files
    │   ├── tracker.py         Validation, check-ins, routine, points, streaks, export, demo data
    │   ├── guidance.py        Indicator definitions, bounds, status rules, guidance text
    │   ├── db.py              SQLite schema and connection handling
    │   ├── config.py          Environment-based settings
    │   ├── crew.json          Crew roster
    │   ├── requirements.txt   (none: standard library only)
    │   └── tests/             test_guidance, test_tracker, test_api
    ├── frontend/
    │   ├── index.html         Shell and navigation
    │   ├── styles.css         Beige, white, and black theme, responsive, reduced-motion aware
    │   └── app.js             Views, body map, trend chart, forms, API client
    ├── tests/ui/ui.test.mjs   Browser-level UI tests (jsdom) against a live server
    ├── docs/                  Challenge mapping, architecture, pitch and demo script
    ├── .github/workflows/ci.yml
    ├── Dockerfile, .dockerignore, Makefile
    ├── SECURITY.md, CHANGELOG.md, ANTIGRAVITY_PROMPT.md
    └── package.json           (test dependency only: jsdom)

## Data and privacy

- Readings are stored in SQLite at `DB_PATH`. Back up this file like any mission record.
- Logs contain method and path only, never bodies or query strings.
- There is no authentication yet. See `SECURITY.md` before any deployment beyond a closed demo network.

## Roadmap

- Authentication and per-role access (astronaut, flight surgeon, behavioural health officer)
- TLS termination guidance and reverse-proxy configuration
- Wearable and cuff integration, so readings arrive without manual entry
- Camera-based input, for example a device read-out
- Personal baselines from pre-flight readings
- Offline-first sync for when the link to Earth drops
