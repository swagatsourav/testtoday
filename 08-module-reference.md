# 08 — Module Reference

← [Device Workflows](07-device-workflows.md) | Index | Next: [File Reference](09-file-reference.md) →

Purpose and responsibilities of every Python module, grouped by tier. For line-level detail
on the largest files see [File Reference](09-file-reference.md).

## Tier 1 — Web / entry

### `app.py`
FastAPI application: route definitions, cookie-JWT auth (`fastapi-login`), Jinja rendering,
`/api/*` endpoints, and flow dispatch. Imports every `work_*` module at startup. See
[API Reference](06-api-reference.md) and [Runtime Flow](04-runtime-flow.md).

### `schemas.py`
Pydantic request/domain models. Note the `BaseModelDirect` base class overrides
`__getitem__`/`__setitem__`/`keys`/`items` so models can be used like dicts (`fi["crq"]`) —
this is why `**flowinfo.model_dump()` and `flowinfo["platform"] = ...` both appear in the
work modules. Key models: `FlowInfo` (the running-flow state object), `WFData` (check
request), `MigrationData`/`NodeData` (satellite/EVPN migration state, with a `.create()`
factory), `ExecResult`/`StageResult`/`CheckInfo` (check outputs), `PostFlowInfo`/`CrqData`/
`apiMgmtData`/`PageConfig`.

## Tier 2 — Config / inventory

### `params.py`
Central configuration, **mode-switched** by `MODE` env var (`production`/`model`). Imports
secrets from `authinfo.py` (`AI`). Defines:
- Auth groups (`AUTHGROUP`, `OWNERS`, `OWNER_AUTHGROUP`), `SECRET`, `PORT`, SSL file names.
- MongoDB creds + DB name (`ssm_db` prod / `ssm_model_db` model).
- **`JUMPHOST` / `SHBOSS`** dicts (RDNBoss + SharedBoss host lists, prompts, jump-host login
  and device login creds) — the connection targets for all device I/O.
- `SDB` (ServiceDb URL + API key), `SAMURAI` URL.
- **`PL`** — per-platform upgrade definitions (GISO image filename/path/md5, target `XrVer`,
  satellite version, BB template versions, and `comply` = compliance target map). Platforms:
  `ASR9903`, `ASR9010`, `NCS55A2`, `NCS540`.
- **`DC`** — disk-check definitions per platform (RSP/RP locations, min free GB, cleanup
  find commands). `CLICAP` — the CLI capture command set. `CLILIST` — config command blocks.
- `AREA2STATE` (OSPF area → Australian state), `IMGINFO`, `XRVER`, `SATVER`.

> **This is the file you edit for a new campaign** (new target version, new image, new
> compliance rule). Values are campaign- and environment-specific.

### `nodeinfo.py`
Static network facts + offline fixtures (it is **not** the live inventory — that comes from
MongoDB via the compliance `rtype` refresh). Holds: `NTP`/`DNS` server IP lists, `ISIS`
config, `RR` (Route-Reflector info, mode-switched prod/model), `SCP_PROBLEM_LIST`, and the
`MODEL_*` fixtures (`MODEL_A9K_LIST`, `MODEL_A9903_LIST`, `MODEL_SPE_LIST`, `MODEL_ShSat_LIST`,
`MODEL_DhSat_LIST`, `MODEL_MGMTVLAN`/`MODEL_MGMTVLAN_DATA`/`MODEL_MGMTDATA`) that back
`MODE=model`. Referenced by `app.py` (mgmt endpoints) and the work modules.

### `config.ini`
Tiny runtime toggle file read for the WIP-Mgmt flow: `dryrun_only`, `allow_infrabs`.

### `authinfo.py` (git-ignored; `.sample` committed)
Holds all secrets: `SECRET`, `BASE_URL`, Mongo creds, device/jump-host creds
(`RDNBOSS_USER/PASS`, `USER/PASS`, model equivalents), `SDB_API_KEY`, `SL_TOKEN`.

## Tier 3 — Shared engine

### `functions.py`
Cross-cutting helpers (no device-CLI parsing logic — that's `functions_xr`):
- **Auth:** `authenticateUser` (LDAP via `cmn_ldap.LDAP`, robot bind + user bind + group
  check), `genKey` (SHA-256 of `SECRET+crq` — the webhook key).
- **Connectivity:** `jumphost_connect` (pexpect ssh to RDNBoss, random host, 40 retries),
  `ssh_connect_jumped` (hop to device), `ssh_connect`/`devConnect` (combined + logfile),
  `sessionClose`, `pingDevices`/`pingAccessNodes`, `collectRtype`.
- **ServiceDb:** `runServiceDb` (async check with progress polling via `TCPKeepAliveAdapter`),
  `sdbPrePost`, `compareServiceDb`.
- **CRQ:** `checkCrq` → `checkCrqData` (reads DB) + `checkCrqVo` (POSTs to VO webhook).
- **Logging/HTML:** `init_logger` (weekly rotation for `ssm*`, daily for others),
  `genLogFname`, `listLogFiles`, `showLogFile`, `commentLogfile`, `buildStageLog`,
  `buildStageLogDetail`, `buttons1`/`buttons2` (the Proceed/Rerun/Terminate modal buttons).
- **Parsing/util:** `parseCliOutput` (TextFSM wrapper), `pl2dt`/`dev2dt` (platform/name →
  devtype), `checkMw` (maintenance-window time check), `checkList`, `chunks`, `taskSuccess`,
  `dt2flow`, `attachLogs` (emails zipped logs to worklog mailbox per CRQ).

### `functions_xr.py`
The device-automation engine (~3000 lines). See [File Reference](09-file-reference.md) for the
full function catalog. Categories: CLI send/config (`cliSend`, `configureLines`,
`cliConfigAlignment`), base-info collection (`collectBaseInfo`, `captureIntfList`,
`captureAccessNodes`), health checks (`checkDeviceUp`, `checkSmuStatus`, `checkPlatformStatus`,
`checkSatelliteStatus`, `checkEcll`, `checkMainBundles`, `walkSpeRings`, `checkDiskSpace`), the
big check pipeline (`cliChecks`, `captureChecks`, `execChecks`, `compareChecks`,
`buildChecksText`), migration (`genMigrationData`, `compareMigration`), and upgrade primitives
(`imgFilePrepare`, `imgFileTransfer`, `xrFileCopy`, `checkFileReady`, `installCleanup`,
`cleanDiskSpace`, `fpdUpgrade`).

### `stages.py`
Reusable **stage building blocks** shared across `work_*` orchestrators:
- `stageInit` — start-time, devtype, logger, mark stage "in progress", connect to device.
- `stageCheckMulti` — the pre/post/interim health-check stage (runs `execChecks`, compares
  via `compareChecks`, writes the colored result + Proceed/Rerun buttons, pauses/terminates
  the flow on failure). This is the workhorse behind every check.
- `stageReachCheck` — access-node reachability (capture CDP/LLDP neighbors, ping from both
  jump hosts, compare pre vs post).
- `stageCheckBBVer` — verify Base-Build template version + optional config alignment.

### `work_db.py`
The **sole MongoDB gateway** (~1350 lines). See [File Reference](09-file-reference.md) for the
collection map. Function groups: flow lock/state (`getLock`, `lockFlow`, `unlockFlow`,
`markFlow`, `proceedFlow`/`rerunFlow`/`pauseFlow`/`terminateFlow`, `checkFlowPause`,
`checkFlowState`, `exitFlow`, `resetFlowMap`), stage/UI map (`markStage`, `getMap`, `getLog`,
`insertLog`, `setRealtimeStatus`/`getRealtimeStatus`), node inventory (`getNodeList`,
`getCrqList`, `dev2pl`, `ip2node`, `getpairnode`), check results (`addCheckResult`,
`getLatestCheckResult`, `getLatestPreCheckData`, `getWorkflowResults`, `genWFResultsTable`),
stats (`getStatsA9k`, `getStatsSpe`), CRQ (`getDbCrq`/`addDbCrq`, `*CrqLog`), flow/migration
data (`getFlowData`/`addFlowData`, `getDbMdata`/`addDbMdata`, `getMgmtEVPNData`/
`addMgmtEVPNData`, `getDbFnnList`/`addDbFnnList`), on-demand jobs (`runOndemand`,
`addOnDemand`), SSM worklog (`storeSsmLog`/`checkSsmLog`/`getSsmLog`).

## Tier 4 — Per-platform work modules

Each exports a `stages` list (labels for the UI), `stageN` functions, and an `orchestrator`.
See [Device Workflows](07-device-workflows.md) for the procedures.

| Module | Platform / campaign | Orchestrator | Flows |
|--------|--------------------|--------------|-------|
| `work_a9903.py` | ASR9903 upgrade → XR 7.8.2 R05 | `orchestrator` | 1–10 |
| `work_a9903_bum.py` | ASR9903 BUM remediation / BB uplift | `orchestrator` | 11–20 |
| `work_a9010_wipmgmt.py` | ASR9010 WIP-Mgmt split-horizon | `orchestrator` | 21–35 |
| `work_spe.py` | Small PE (NCS540/55A2) upgrade | `orchestrator` | 1–10 (disabled in app) |
| `work_a9k.py` | ASR9010 upgrade + shared A9K primitives | `orchestrator` | 1–4 (disabled) |
| `work_a9010_isis.py` | ASR9010 ISIS install | — | 11–20 (disabled) |
| `work_satellite.py` | SH/DH satellite migration + pre/post check | `orchestrator` | — |
| `work_mgmt_evpn.py` | Mgmt EVPN migration pre/post check (thin) | — (check-only) | — |

`work_a9k.py` is special: besides its own (disabled) upgrade orchestrator it holds **shared
A9K primitives** used by other flows and by satellite upgrades — `runTsScript`/`executeTs`/
`rollbackTs` (traffic steering via `xr-steer.ssm.pl`), `upgradeShSatellites`/
`upgradeDhSatellites`, `installCommit`, `smuActivate`, `isoActivate`, `runRedCheck`, and the
`runPreCheck`/`runPostCheck`/`runPreWork` used by A9K devices in `/api/workflow`.

## Tier 5 — Auxiliary services & auth

### `compliance_a9k.py` / `compliance_spe.py`
Standalone systemd services. `compliance_a9k.py`: connects to RDNBoss, runs `rtype` to get
the device list, sanity-checks counts vs DB (aborts update if >8 devices dropped or <95%
collected), refreshes `db.src.xtype`, prunes decommissioned nodes (after 10 misses), then
fans out `checkA9kDevices` across threads (20 devices/chunk) to collect base info + satellite
status into `db.sw` and `db.a9k.sats`. Scheduling via `schedule.every().day.at(t)` in a
background thread. `compliance_spe.py` is the SPE analogue (smaller).

### `cmn_ldap.py` (LIVE auth)
The `LDAP` class actually used by `authenticateUser` — confirmed by `functions.py:32`
(`from cmn_ldap import LDAP`). Methods: `connect`, `search`/`search_username`/`search_realname`,
`show_user`, `is_member`, plus a full `AccountError` exception hierarchy. Uses `python-ldap`.

### `authenticator.py` (DEAD CODE)
A self-contained `Account01Authenticator` class (LDAP against Telstra "ACCOUNT-01"). **Not
imported anywhere** — `functions.authenticateUser` uses `cmn_ldap.LDAP` instead. This is a
legacy/alternate implementation and a candidate for removal. See
[Improvements](11-improvement-recommendations.md).

### `config_check.py`
`confCheck(device, trad, platform, baseinfo)` — validates device running-config against the
expected Base-Build template; used by `stages.stageCheckBBVer`.

## Gaps / needs confirmation

- Which LDAP implementation is authoritative (`authenticator.py` vs `cmn_ldap.py`).
- `nodeinfo.py` internal structure (inventory helpers vs pure `MODEL_*` fixtures) — not yet
  read in full.
- `work_a9010_isis.py` is imported only in commented-out code; confirm it is fully retired.
