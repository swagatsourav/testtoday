# 09 — File Reference (important source files)

← [Module Reference](08-module-reference.md) | [Index](README.md)

Line-level detail for the files a maintainer will actually open. For higher-level grouping see
[Module Reference](08-module-reference.md). Line numbers are from the state at documentation
time and will drift — treat them as anchors, not guarantees.

## `app.py` (1030 lines)
Entry point. See [API Reference](06-api-reference.md) for the route table.
- L20 `os.environ['PETYPE']="XR"` set **before** `import params` (params branches on env).
- L26–39 imports every `work_*` orchestrator/stage set + `work_db` accessors at module load.
- L47 `manager = LoginManager(P.SECRET, '/auth/token', use_cookie=True)`; L63 `load_user`.
- L67 `login` — LDAP auth (or `authskip`), 8h token, cookie set.
- L855 `runworkflow` (`/api/workflow`) — the `ThreadPoolExecutor(20)` check fan-out with the
  device-name routing (`-spe-`/`-e-`, worktype `ShSat`/`DhSat`/`MgmtEVPN*`).
- L920 `runflow` (`/api/runflow`) — flow-range → orchestrator dispatch (most upgrade branches
  `return False` = disabled campaigns).
- L1019 `__main__` — `init_logger("ssm","logs")`, `uvicorn.run(host="0.0.0.0", port)`.

## `params.py` (492 lines)
Config, mode-switched. Highest-churn file for campaigns.
- L7–9 `mode`/`modelskip`/`authskip` from env. L10–26 auth groups.
- L42–125 `JUMPHOST` and `SHBOSS` (prod + model): host lists, prompts, jump login (`user`/
  `pswd`) and device login (`user2`/`pswd2`).
- L127–134 `SDB` (ServiceDb) + `SAMURAI` URLs.
- L188–192 `CMD0` (terminal setup sent on every device login). L237–251 `CLICAP` (capture cmd
  set). L253–349 `DC` (per-platform disk-check RSP/RP locations, min GB, cleanup finds).
- L352–464 `PL` — per-platform upgrade defs: `giso` (fname/path/md5), `XrVer`, `satver`,
  `label`, `comply` (compliance target versions→template versions). Edit here for a new target.
- L466–492 `IMGINFO`/`XRVER`/`SATVER` derived from `PL`.

## `schemas.py` (165 lines)
- L15–26 `BaseModelDirect` — dict-style access mixin (why models are indexed like `fi["crq"]`).
- L28–70 `NodeData` + `MigrationData` (satellite/EVPN migration state; `.create()` factory at
  L61 swaps src/dst on rollback; `__init__` sets type from `-ss-`/`-ds-` in target name).
- L77–94 `FlowInfo` — the running-flow state object passed as `**flowinfo.model_dump()`.
- L112–142 `PostFlowInfo` (runflow body), `WFData` (workflow body; `checkmode` derived in
  `__init__`).

## `functions.py` (967 lines)
Cross-cutting helpers. See [Module Reference](08-module-reference.md) for the grouped list.
- L55 `authenticateUser` — LDAP robot-bind + user-bind + group check via `cmn_ldap.LDAP`.
- L111 `genKey` — `sha256(SECRET+crq)`, the `/api/addcrq` webhook key.
- L121 `init_logger` — `ssm*` weekly rotation (52 backups), others daily (14).
- L193 `jumphost_connect` — pexpect ssh to RDNBoss, random host, 40 retries, failure
  classification. L251 `ssh_connect_jumped`. L397 `devConnect` (connect + attach logfile).
- L452 `runServiceDb` — ServiceDb async check with progress polling (`TCPKeepAliveAdapter`).
- L607 `compareServiceDb` — up→down / down→up diff.
- L753 `parseCliOutput` — TextFSM wrapper. L764 `genLogFname` — the CRQ/stage/date filename
  scheme the log-cleanup cron matches on.
- L815 `checkCrq` → L835 `checkCrqData` (DB read, 12h validity) + L878 `checkCrqVo` (POST to
  VO webhook `verify=False`). L905 `attachLogs` — email zipped logs per CRQ.

## `functions_xr.py` (3007 lines)
The device engine. Function catalog by category:
- **CLI/config:** `cliSend` (L28), `cliSendRun` (L54), `captureConf` (L60), `configureLines`
  (L87, honors `dryrun_only`), `cliConfigAlignment` (L154).
- **Base info:** `captureAccessNodes` (L176), `captureIntfList` (L231), `collectBaseInfo`
  (L250 — platform/version/label/tmplver, optional DB update), `collectPairBaseInfo` (L308).
- **Health checks:** `comparePwList` (L320), `checkMainBundles` (L339), `checkEcll` (L372),
  `checkSmuStatus` (L400), `checkPlatformStatus` (L447), `checkSatelliteStatus` (L482),
  `checkDeviceUp` (L530), `walkSpeRings` (L719), `checkDiskSpace` (L2502).
- **Check pipeline (the big ones):** `cliChecks` (L1064 — parse a device's captured CLI into
  the structured snapshot), `compareChecks` (L1374 — pre vs post diff), `captureChecks`
  (L1821), `execChecks` (L2063 — top-level per-device runner used by `stageCheckMulti`),
  `buildChecksText` (L2108).
- **Migration:** `genMigrationData` (L878), `compareMigration` (L2133).
- **Upgrade primitives:** `installCleanup` (L2444), `cleanDiskSpace` (L2479), `checkFileReady`
  (L2528), `xrFileCopy` (L2569), `imgFilePrepare` (L2589), `imgFileTransfer` (L2700),
  `fpdUpgrade` (L2814), `fpdUpgradePsu` (L2953).

## `stages.py` (560 lines)
Reusable stage blocks. `stageInit` (L23), **`stageCheckMulti`** (L49 — the pre/post/interim
check workhorse; note it calls `pauseFlow`/`terminateFlow` directly at L288–299 based on
result), `stageReachCheck` (L352), `stageCheckBBVer` (L494, uses `config_check.confCheck`).

## `work_db.py` (1349 lines)
Sole MongoDB gateway. **Connection:** L21
`MongoClient('mongodb://USER:PASS@localhost/default_db?authSource=admin')`, `db = client[P.MONGODB_DB]`.

### MongoDB collection map (inferred from usage)
| Collection | Written by | Holds |
|-----------|-----------|-------|
| `db.lock` | `lockFlow`/`unlockFlow` | Active flow locks (node, flow, worktype, status) |
| `db.ops` | pause/rerun/terminate | Cooperative flow-control signals (`op`) |
| `db.guimap` | `markFlow`/`markStage`/`resetFlowMap` | Per-flow & per-stage UI state (color/status/glow/oid) |
| `db.rtstatus` | `setRealtimeStatus` | Live "marquee" status per node |
| `db.precheck` / `db.postcheck` (+ `.shsat`/`.dhsat`/`.mgmtevpn`) | `markStage`/`addCheckResult` | Check snapshots (pre/post, per work type) |
| `db.redcheck`, `db.prework` | `markStage` | Redundancy / prework results |
| `db.sw` | compliance | Per-device base info (version, template, pair) |
| `db.a9k.sats` | compliance | Satellite inventory (type/sn/sw/hosts) |
| `db.src.xtype` | compliance | Device inventory from `rtype` (name/type/ip/sw) |
| CRQ / flowdata / mdata / fnnlist / mgmtevpn / ssmlog | `*DbCrq`/`*FlowData`/`*DbMdata`/`*FnnList`/`*MgmtEVPNData`/`*SsmLog` | change tickets, per-flow data, migration data, service maps, worklogs |

### Function groups
Flow lock/state (L39–139: `getLock`, `lockFlow`, `unlockFlow`, `markFlow`, `proceed/rerun/
pause/terminateFlow`, `checkFlowPause`/`checkFlowState`, `exitFlow`), UI map (L140–303:
`markStage`, `getMap`, `getLog`, `insertLog`, `setRealtimeStatus`, `resetFlowMap`), inventory
(L303–383, L920–956: `dev2pl`, `getNodeList`, `getCrqList`, `ip2node`, `getpairnode`),
results/stats (L384–898: `addCheckResult`, `getLatestCheckResult`, `getWorkflowResults`,
`genWFResultsTable`, `getStatsA9k`/`getStatsSpe`), CRQ/log/flow/migration data (L978–1271),
on-demand jobs (L1272–1360: `runOndemand`, `addOnDemand`).

## `work_a9903.py` (298 lines) — reference orchestrator
Cleanest active example. `stages` list L29; `stage1`–`stage14` L47–89 (thin wrappers over
`stages.*` blocks); `orchestrator` L93. The orchestrator's guard sequence (lock → maintenance
window → CRQ → base-info/version check → stage loop) is documented in
[Device Workflows](07-device-workflows.md). Constants near top: `FINALSTAGE`, `maxhour=7`.

## Other work modules
- `work_a9903_bum.py` (288) — 11 stages, BUM remediation; same orchestrator shape.
- `work_a9010_wipmgmt.py` (402) — `stageApplySPH` (L37) is the real work; 5 stages;
  `orchestrator` L184; honors `config.ini`.
- `work_spe.py` (1214) — SPE upgrade; helpers `maxMetric`/`deMaxMetric` (L49/L96),
  `isoActivate`, `smartlicense`, `wCSCvt27458`; 10 stages; `runPreCheck`/`runPostCheck`/
  `runPreWork` used by `/api/workflow` for `-spe-` devices.
- `work_a9k.py` (1971) — disabled upgrade orchestrator **plus** shared primitives:
  `runTsScript`/`executeTs`/`rollbackTs` (L62/L277/L424, traffic steering via `xr-steer`),
  `upgradeShSatellites`/`upgradeDhSatellites` (L703/L719), `installCommit` (L539),
  `runRedCheck` (L863). `runPreCheck`/`runPostCheck` for `-e-` (A9K) devices.
- `work_satellite.py` (785) — SH/DH migration; `getSatInfo`, `stageMigration`,
  `stagePreCheckSat`/`stagePostCheckSat`, `orchestrator` L618.
- `work_mgmt_evpn.py` (149) — thin: only `runPreCheck`/`runPostCheck` delegating to
  `stageCheckMulti` with `MigrationData` + mgmt `extradata`.

## Service & auth files
- `compliance_a9k.py` (232) — `checkA9kDevices` (L27, per-chunk collector), `complianceRunner`
  (L80, rtype refresh + sanity gates + thread fan-out), `schedJob`/`schedRunner` (L211/215,
  `schedule` lib), `__main__` L224 (time arg, default 06:00). `compliance_spe.py` (98) is the
  SPE analogue.
- `cmn_ldap.py` (215) — the live `LDAP` class (`connect`/`search_username`/`show_user`/
  `is_member`). `authenticator.py` (246) — alternate `Account01Authenticator`; verify if used.
- `config_check.py` (147) — `confCheck` for Base-Build config validation.

## Gaps / needs confirmation
- Exact indexes on MongoDB collections (none declared in code; likely created manually).
- `nodeinfo.py` (840) not yet read line-by-line — inventory helpers + `MODEL_*` fixtures.
- `textfsm/` template inventory (only the satellite-status template is confirmed in use).
