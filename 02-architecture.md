# 02 — Architecture

← [Project Overview](01-project-overview.md) | Index | Next: [Folder Structure](03-folder-structure.md) →

## High-level architecture

SSM is a **monolithic Python application with three processes** plus several external
dependencies. There is no microservice decomposition; the "services" are just three
entry-point scripts sharing the same codebase and MongoDB.

```mermaid
	graph TB
	    subgraph Browser
	        UI[Black-dashboard UI<br/>jQuery + Chart.js + select2]
	    end
	    subgraph Server["Server (/opt/ssm, user n113272)"]
	        AP[Apache httpd<br/>reverse proxy /xr/ → :8443]
	        subgraph P1["ssm-tool.service"]
	            APP[app.py<br/>FastAPI + uvicorn :8443]
	            WORK[work_*.py<br/>orchestrators & stages]
	            FX[functions_xr.py<br/>device automation]
	            APP --> WORK --> FX
	        end
	        subgraph P2["ssm-compliance-a9k.service"]
	            CA[compliance_a9k.py<br/>daily 07:10]
	        end
	        subgraph P3["ssm-compliance-spe.service"]
	            CS[compliance_spe.py<br/>daily 07:40]
	        end
	        DB[(MongoDB<br/>ssm_db :27017)]
	    end
	    subgraph External
	        RDN[RDNBoss jump host<br/>vusno238]
	        DEV[Cisco IOS-XR routers<br/>A9K / A9903 / SPE / satellites]
	        VO[VO platform<br/>CRQ webhook]
	        SAM[SAMURAI API<br/>mgmt VLAN data]
	        SDB[ServiceDb API<br/>inventory]
	        LDAP[LDAP directory]
	    end
	
	    UI -->|HTTPS| AP --> APP
	    APP --> DB
	    CA --> DB
	    CS --> DB
	    FX -->|pexpect SSH/telnet| RDN -->|trad| DEV
	    CA --> RDN
	    CS --> RDN
	    APP -->|LDAP bind| LDAP
	    VO -->|POST /api/addcrq| APP
	    APP -->|GET/POST| SAM
	    APP --> SDB
	    CA --> SDB
```

## Component responsibilities

| Component                          | Responsibility                                                                                               |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Apache httpd**                   | TLS termination, reverse-proxy `/xr/` → `localhost:8443`                                                     |
| **app.py**                         | HTTP routing, auth (cookie JWT), template rendering, `/api/*`, flow dispatch, thread-pool fan-out for checks |
| **work_*.py**                      | Per-platform orchestrators + stage functions (the actual procedures)                                         |
| **functions_xr.py**                | Low-level device automation: connect, send CLI, expect prompt, parse output                                  |
| **functions.py**                   | Cross-cutting helpers: logging, auth wrapper, log-file viewer, jump-host connect                             |
| **work_db.py**                     | The single MongoDB gateway: locks, maps, logs, stats, CRQ, on-demand nodes                                   |
| **compliance_a9k/spe.py**          | Standalone daily audits; write results consumed by the dashboards                                            |
| **authenticator.py / cmn_ldap.py** | LDAP authentication & group authorization                                                                    |
| **MongoDB**                        | All persistent state (flow locks, stage status, logs, stats, CRQ)                                            |

## Module dependency diagram

```mermaid
graph LR
    app.py --> schemas.py
    app.py --> params.py
    app.py --> nodeinfo.py
    app.py --> functions.py
    app.py --> functions_xr.py
    app.py --> work_db.py
    app.py --> work_a9k.py
    app.py --> work_a9903.py
    app.py --> work_a9903_bum.py
    app.py --> work_a9010_wipmgmt.py
    app.py --> work_spe.py
    app.py --> work_satellite.py
    app.py --> work_mgmt_evpn.py

    work_a9k.py --> functions_xr.py
    work_a9k.py --> stages.py
    work_a9k.py --> work_db.py
    work_a9903.py --> functions_xr.py
    work_a9903.py --> stages.py
    work_spe.py --> functions_xr.py

    functions_xr.py --> functions.py
    functions_xr.py --> params.py
    functions_xr.py --> nodeinfo.py
    stages.py --> functions_xr.py
    stages.py --> work_db.py

    compliance_a9k.py --> functions_xr.py
    compliance_a9k.py --> work_db.py
    functions.py --> authenticator.py
    authenticator.py --> cmn_ldap.py
```

> The exact edges among `work_*`, `stages.py`, and `functions_xr.py` are being confirmed by
> the module deep-dives; this diagram shows the dominant direction of dependency: **app →
> work → (stages, functions_xr, work_db) → (functions, params, nodeinfo)**. Nothing depends
> back on `app.py` (it is the top of the graph).

## Architectural layers

```
┌─────────────────────────────────────────────────────────┐
│ Presentation:  templates/ + static/  (Jinja2 + jQuery)   │
├─────────────────────────────────────────────────────────┤
│ Web/API:       app.py  (routes, auth, dispatch)          │
├─────────────────────────────────────────────────────────┤
│ Orchestration: work_*.py  (flows, stages, checks)        │
├─────────────────────────────────────────────────────────┤
│ Engine:        functions_xr.py, functions.py, stages.py  │
├─────────────────────────────────────────────────────────┤
│ Data/Config:   work_db.py, params.py, nodeinfo.py        │
├─────────────────────────────────────────────────────────┤
│ Infra:         MongoDB · RDNBoss · LDAP · VO · SAMURAI   │
└─────────────────────────────────────────────────────────┘
```

## External integrations (why each exists)

| System        | Direction                | Purpose                                                       | Entry point in code                          |
| ------------- | ------------------------ | ------------------------------------------------------------- | -------------------------------------------- |
| **RDNBoss**   | SSM → jump host → device | The only path to reach routers; holds privileged device login | `functions.jumphost_connect`, `functions_xr` |
| **LDAP**      | SSM → directory          | Authenticate users & check group membership                   | `authenticator.authenticateUser`             |
| **VO**        | VO → SSM webhook         | Validate CRQ (change ticket) status                           | `POST /api/addcrq`                           |
| **SAMURAI**   | SSM → API                | Management-VLAN data for Mgmt EVPN migration                  | `/api/mgmtvlans`, `/api/mgmtdata`            |
| **ServiceDb** | SSM → API                | Device inventory / service data                               | compliance & work modules                    |
| **MongoDB**   | SSM ↔ DB                 | All persistent state                                          | `work_db.py`                                 |

See [Device Workflows](07-device-workflows.md) for the RDNBoss/device communication detail
and [Services](05-services.md) for ports and process management.

## Key architectural observations (for the maintainer)

- **Flat module namespace, import-time coupling.** `app.py` imports every `work_*` module at
  startup; a syntax/import error anywhere takes down the web service. There is no plugin
  registration — adding a platform means editing `app.py`'s import block and dispatch logic.
- **State in DB, not memory.** Flow control (lock/pause/terminate) is mediated through
  MongoDB, which is why the UI can poll and why a process restart leaves locks dangling.
- **Synchronous device I/O on a thread pool.** Long device operations run under
  `ThreadPoolExecutor` (checks) or inline in the request (flows) — not via async. FastAPI's
  async benefits are largely unused.
- **Two coupling points dominate:** `functions_xr.py` (every platform depends on it) and
  `work_db.py` (every stateful action goes through it). These are the highest-leverage — and
  highest-risk — files to change.

## Gaps / needs confirmation

- Precise inter-module edges (pending `work_*` and engine deep-dives) — will be finalized in
  [Module Reference](08-module-reference.md).
- Whether compliance services read ServiceDb directly or via `functions_xr` (pending
  compliance deep-dive).
