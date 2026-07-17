# 01 — Project Overview

← [Index](README.md) | Next: [Architecture](02-architecture.md) →

## Executive summary

SSM (Sep-Software-Management) is an internal web application used by a network operations
team to automate repetitive, high-risk work on Cisco IOS-XR provider-edge (PE) routers. A
router upgrade or migration is a long, error-prone manual procedure: an engineer must SSH
to the device, run dozens of health-check commands, capture the "before" state, perform the
change, run the same checks again, and compare "before" vs "after" to prove the change was
non-impacting. SSM encodes those procedures as **flows** and **checks** driven from a
browser, records every command and its output, and stores results in a database for
auditability.

The tool targets three router families used in Telstra's "SEP / NG-SEP / Small PE" edge:

| Platform label  | Hardware                            | Role in SSM                                                        |
| --------------- | ----------------------------------- | ------------------------------------------------------------------ |
| **A9K / A9010** | Cisco ASR 9010 (SEP)                | Upgrades (historical), WIP-Mgmt split-horizon, Mgmt EVPN migration |
| **A9903**       | Cisco ASR 9903 (NG-SEP)             | Upgrade to XR 7.8.2, BUM remediation ("BB uplift")                 |
| **SPE**         | Cisco NCS 540 / NCS 55A2 (Small PE) | Upgrades (historical), compliance                                  |

It also validates **satellites** (Cisco ASR9000v "nV satellites" hanging off an A9K host)
and orchestrates **Single-Home (SH)** and **Dual-Home (DH)** satellite migrations.

## Business purpose

- **Reduce human error** on change windows by scripting the exact command sequence.
- **Enforce validation** — a change is only "good" if post-check matches pre-check.
- **Provide auditability** — every stage's device transcript is logged per CRQ (change
  request) and retained (compressed after 90 days, archived after 180).
- **Track compliance** — daily jobs report which devices are not on the target software
  version / broadband template version, surfaced as dashboard charts and tables.
- **Gate work on change management** — a flow will not run unless the associated CRQ ticket
  is in the correct state, checked via the external VO platform.

## What a user actually does

1. Logs in (LDAP-backed; see [Architecture](02-architecture.md)).
2. Picks a platform section from the sidebar (A9K, SPE, Satellite, Mgmt EVPN).
3. Either:
   - **Runs a flow** (runner page): selects a device + CRQ, presses Run, and watches
     animated stage boxes progress; can Pause / Stop / Rerun. Used for upgrades &
     migrations. See [Device Workflows](07-device-workflows.md).
   - **Runs a check** (workflow page): selects one or many devices, presses Run, and gets a
     comparison table. Used for pre-check / post-check / interim-check health snapshots.
4. Opens the **Log Viewer** to read the raw device transcript for any stage.

## Technology stack

| Layer         | Technology                                                         | Notes                                             |
| ------------- | ------------------------------------------------------------------ | ------------------------------------------------- |
| Language      | Python 3.11.9                                                      | Single interpreter, `pyenv`-managed on the server |
| Web framework | FastAPI 0.110 + Uvicorn 0.29                                       | ASGI, but the app is largely synchronous          |
| Auth          | fastapi-login 1.10 (cookie JWT) + python-ldap / ldap3              | 8-hour token; LDAP group check                    |
| Templating    | Jinja2                                                             | Server-rendered HTML                              |
| Frontend      | black-dashboard-flask theme, jQuery, select2, Chart.js, DataTables | No SPA framework, no build step for app JS        |
| Database      | MongoDB (`ssm_db`) via pymongo 4.7                                 | Local instance                                    |
| Device I/O    | pexpect 4.9 (SSH/telnet via jump host)                             | No netmiko/napalm                                 |
| CLI parsing   | TextFSM 1.1                                                        | Cisco output → structured rows                    |
| Scheduling    | `schedule` 1.2 (in-process) + systemd                              | Compliance jobs                                   |
| Process mgmt  | systemd (3 services) + Apache httpd reverse proxy                  | See [Services](05-services.md)                    |
| External APIs | VO webhook (CRQ), SAMURAI (mgmt VLANs), ServiceDb                  | See [Architecture](02-architecture.md)            |

## Key characteristics a new developer must internalize

- **The "flow number" is load-bearing.** Ranges of integers map to specific orchestrators
  and campaigns (e.g. A9903 flows 1–10 = upgrade, 11–20 = BUM remediation). Much of the
  historical upgrade code is commented out / disabled because those campaigns finished; the
  numbering is kept for continuity. See [Runtime Flow](04-runtime-flow.md).
- **Everything goes through a jump host (RDNBoss).** SSM does not talk to routers directly;
  it spawns a `pexpect` session to the jump host and drives CLI from there.
- **State lives in MongoDB, not memory.** Flow locks, pause/terminate signals, stage
  status, and logs are persisted, so the UI (which polls every 2s) reflects DB state.
- **`MODE=model` is the offline/test harness.** A set of fake "model" nodes and canned data
  in `nodeinfo.py` lets the whole app run without touching real devices.

## Gaps / needs confirmation

- Exact MongoDB schema and indexes (inferred from `work_db.py`; no schema file exists).
- The daily log-cleanup script is documented in `README.md` but the actual cron entry /
  file location on the server is not in the repo.
- `authinfo.py` (secrets) is git-ignored; only `authinfo.py.sample` is available.
