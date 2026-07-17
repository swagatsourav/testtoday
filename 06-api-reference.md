# 06 — API & Route Reference

← [Services](05-services.md) | Index | Next: [Device Workflows](07-device-workflows.md) →

All routes are defined in [`app.py`](../app.py). There are two categories: **page routes**
(return server-rendered HTML) and **`/api/*` routes** (return JSON, called by frontend AJAX).
Auth is enforced per-route via a FastAPI dependency:

- `Depends(manager.optional)` — page routes: returns `None` if not logged in, and the
  handler renders the login page instead.
- `Depends(manager)` — most `/api/*` routes: rejects unauthenticated requests.
- **No auth dependency** — `/auth/token`, `/api/addcrq`, `/api/mgmtvlans`, `/api/mgmtdata`,
  `/api/mgmtdatatable` (see security note at end).

## Authentication routes

| Method | Path | Handler | Purpose |
|--------|------|---------|---------|
| POST | `/auth/token` | `login` | OAuth2 password form → LDAP auth → sets JWT cookie (8h) |
| GET | `/logout`, `/sat/logout`, `/mgmtevpn/logout` | `logout*` | Delete cookie, redirect |

`login` uses `authenticateUser(username, password, P.AUTHGROUP, P.OWNERS, P.OWNER_AUTHGROUP)`
unless `P.authskip` is set (offline test). Token created by `manager.create_access_token`,
stored as a cookie by `manager.set_cookie`.

## Page routes (HTML)

| Path | Template | Segment / purpose |
|------|----------|-------------------|
| `/` | login or redirect → `/a9k` | Landing |
| `/a9k` | `index_a9k.html` | A9K compliance dashboard |
| `/spe` | `index_spe.html` | SPE compliance dashboard |
| `/precheck`, `/postcheck` | `workflow.html` | Bulk node health check |
| `/redcheck` | `workflow.html` | A9K redundancy check (via RDNBoss script) |
| `/runa9903` | `runner.html` | A9903 upgrade (flows 1–10) |
| `/runa9903bum` | `runner.html` | A9903 BUM/BB uplift (flows 11–20) |
| `/logfiles` | `logfiles.html` | Log viewer |
| `/sat`, `/sat/precheckshsat` … `/sat/postcheckdhsat` | `workflow.html` | Satellite SH/DH pre/interim/intfdesc/post checks |
| `/sat/logfiles` | `logfiles.html` | Satellite log viewer |
| `/mgmtevpn`, `/mgmtevpn/runa9ksph` | `runner.html` | A9K split-horizon (flows 21–35) |
| `/mgmtevpn/precheck*`, `/postcheck*`, `/interimcheckroot` | `workflow.html` | Mgmt EVPN Tail/Root/RootPB checks |
| `/mgmtevpn/logfiles` | `logfiles.html` | Mgmt EVPN log viewer |

> **Disabled/commented routes:** `/runa9k`, `/runa9kisis`, `/runspe`, `/prework` are
> commented out in `app.py` — their upgrade campaigns are complete. The corresponding
> orchestrator calls in `/api/runflow` also `return False`. This is expected, not a bug.

Page routes deep-copy one of three `PageConfig` objects (`config_base`, `config_sat`,
`config_mgmtevpn`) which differ mainly in `ASSETS_ROOT` (relative path depth) and which
sidebar template to render.

## `/api/*` — data & control endpoints

### Read / list

| Method | Path | Params | Returns |
|--------|------|--------|---------|
| GET | `/api/getnodelist` | `devtype` (A9K/A9010/A9903/SPE/ShSat/DhSat/MgmtEVPN:*…) | Node list for select2 (or CRQ list for `MgmtEVPN:RootPost`) |
| GET | `/api/getlogfiles` | `device` | List of log files for a device |
| GET | `/api/showlogfile` | `fname` | Raw contents of one log file |
| GET | `/api/getmap` | `type` (=PETYPE) | Live flow/stage status map for UI polling |
| GET | `/api/getlog` | `id` (Mongo object id) | Single stage transcript (for modal) |
| GET | `/api/getstats` | `petype` (A9K/SPE) | Compliance chart + table data |

### Flow control (all `GET`, all require login, all keyed by `petype` + `flow`)

| Path | Handler | Effect |
|------|---------|--------|
| `/api/runflow` (POST, body `PostFlowInfo`) | `runflow` | Start (or resume if paused) a flow → dispatches to orchestrator by petype+flow range |
| `/api/proceedflow` | `proceedflow` | Resume a paused flow |
| `/api/pauseflow` | `pauseflow` | Request pause |
| `/api/terminateflow` | `terminateflow` | Request terminate/stop |
| `/api/rerunflow` | `rerunflow` | Re-run current stage |

`runflow` logic: reads `getLock(flow, worktype)`. If locked **and** paused →
`proceedFlow`. If locked and running → no-op. If unlocked → dispatch to the orchestrator
matching `worktype` + `flow` range (see [Runtime Flow](04-runtime-flow.md)).

### Bulk check

| Method | Path | Body | Effect |
|--------|------|------|--------|
| POST | `/api/workflow` | `WFData` | Dispatch pre/post/interim/intfdesc check across `devicelist` via `ThreadPoolExecutor(20)`, 5s stagger |
| GET | `/api/runredcheck` | `device` | Run A9K redundancy check |

`/api/workflow` routes each device by worktype + name substring to the correct
`runPreCheck* / runPostCheck* / runPreWork*` function. See handler `runworkflow`.

### CRQ & Mgmt EVPN integration

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/api/addcrq` | **none** (HMAC-style key check) | VO webhook posts CRQ status; validated by `genKey(node+crq)==key` then `addDbCrq` |
| GET | `/api/mgmtvlans/{state}` | **none** | Proxy to SAMURAI for mgmt VLAN list (or MODEL data) |
| POST | `/api/mgmtdata` | **none** | Proxy to SAMURAI for mgmt data |
| POST | `/api/mgmtdatatable` | **none** | `getMgmtData` + `addMgmtEVPNData`, build HTML tables of RDN-PE/NG-SEP/SEP mapping |

## Request bodies (`schemas.py`)

- `PostFlowInfo` — runflow: `flow`, `petype`, `device`, `crq`, `skipdh`, `skipsh`, `srcPE`, `dstPE`.
- `WFData` — workflow: `mode`, `worktype`, `devicelist`, `target`, `crq`, `flow`, `data`,
  `snlist`, `dnlist`, `srcPE`, `dstPE`, `rollback`, `checkmode`.
- `CrqData` — addcrq: `node`, `crq`, `key`, `data`.
- `apiMgmtData` — mgmt: `state`, `vlanlist`.
- `PageConfig` — template config: `ASSETS_ROOT`, `sidebar`, plus per-page `title`/`overview`.

(See [File Reference](09-file-reference.md) for exact field types — pending schema deep-dive.)

## Security notes

- **`/api/addcrq`, `/api/mgmtvlans`, `/api/mgmtdata`, `/api/mgmtdatatable` have no login
  dependency.** `addcrq` is gated by an HMAC-like `genKey` check; the mgmt endpoints have
  **no gate at all** and proxy to SAMURAI / return data. Flag for review — see
  [Improvements](11-improvement-recommendations.md).
- **CORS** is fully open (`allow_origins=['*']`, `allow_credentials=True`).
- `/api/mgmtdata` calls SAMURAI with `verify=False` (TLS verification disabled).

## Gaps / needs confirmation

- Exact field types/optionality of `schemas.py` models (pending schema deep-dive).
- `genKey` algorithm (in `functions.py`) — confirms the webhook auth strength.
