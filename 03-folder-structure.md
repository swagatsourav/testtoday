# 03 — Folder Structure

← [Architecture](02-architecture.md) | Index | Next: [Runtime Flow](04-runtime-flow.md) →

The repository is **flat**: almost all Python lives in the root directory rather than in
packages. This is a deliberate constraint to note — there is no `src/`, no package
`__init__.py` hierarchy, and modules import each other directly by filename. This keeps
`sys.path` simple (the app runs from `/opt/ssm`) but means module boundaries are by
convention, not enforced.

```
ssm/
├── app.py                    # FastAPI entry point: routes, auth, API, UI dispatch
├── params.py                 # Central config/constants (ports, hosts, groups, node models)
├── schemas.py                # Pydantic request models (PostFlowInfo, WFData, CrqData, ...)
├── nodeinfo.py               # Device inventory model + MODEL_* fake data for offline mode
│
├── functions.py              # General helpers: logging, auth wrapper, log-file viewer, jumphost connect
├── functions_xr.py           # (3000 lines) Core IOS-XR device automation primitives
├── stages.py                 # Shared stage building blocks used across work_* modules
│
├── work_a9k.py               # A9K orchestrator + stages + pre/post/red-check + prework
├── work_a9010_isis.py        # A9010 ISIS install flow (disabled campaign)
├── work_a9010_wipmgmt.py     # A9010 WIP-Mgmt split-horizon flow (ACTIVE, flows 21-35)
├── work_a9903.py             # A9903 upgrade orchestrator + stages (flows 1-10)
├── work_a9903_bum.py         # A9903 BUM remediation flow (flows 11-20)
├── work_spe.py               # SPE orchestrator + stages + pre/post-check/prework
├── work_satellite.py         # SH/DH satellite pre/post-check + migration
├── work_mgmt_evpn.py         # Mgmt EVPN migration pre/post-check
├── work_db.py                # (1350 lines) ALL MongoDB access: locks, maps, logs, stats, CRQ
│
├── authenticator.py          # LDAP authentication logic
├── cmn_ldap.py               # LDAP connection/query helpers
├── compliance_a9k.py         # Standalone daily service: A9K compliance check
├── compliance_spe.py         # Standalone daily service: SPE compliance check
├── config_check.py           # Config validation helper
│
├── config.ini               # Small runtime toggles (a9010_wipmgmt dryrun/allow flags)
├── authinfo.py.sample       # Template for git-ignored authinfo.py (secrets)
├── requirements.txt         # Pinned Python deps
├── restart_ssm-tools.sh     # Restart all 3 systemd services after a deploy
│
├── README.md                # Deploy guide + systemd unit definitions + ops runbook
├── README_A9K.md            # A9K flow notes
├── README_A9903.md          # A9903 flow notes
├── README_SPE.md            # SPE flow notes
├── README_MGMT_EVPN.md      # Mgmt EVPN migration notes
│
├── templates/               # Jinja2 HTML (black-dashboard theme)
│   ├── layouts/             #   base.html, base-fullscreen.html
│   ├── includes/            #   sidebar(s), navigation, scripts.html (AJAX glue), modal
│   ├── index_a9k.html       #   A9K compliance dashboard (charts + tables)
│   ├── index_spe.html       #   SPE compliance dashboard
│   ├── runner.html          #   Flow runner UI (Run/Pause/Stop, animated stages)
│   ├── workflow.html        #   Pre/post-check UI (bulk node select, result table)
│   ├── logfiles.html        #   Log viewer
│   └── login.html
│
├── static/                  # Frontend assets (black-dashboard-flask theme)
│   ├── js/core/             #   jquery, popper, bootstrap (vendor)
│   ├── js/plugins/          #   chartjs, perfect-scrollbar, bootstrap-notify (vendor)
│   ├── js/                  #   black-dashboard.js, themeSettings.js (theme)
│   ├── css/, scss/, fonts/, img/
│
├── textfsm/                 # TextFSM templates: parse Cisco CLI output → structured rows
│
├── test/                    # Standalone manual test scripts (run against MODEL nodes)
│   └── test_*.py            #   one per primitive: test_stagex, test_jumphostconnect, ...
│
├── votask/                  # VO (orchestration platform) CRQ-check task definition
│   ├── VOTaskExport-SSM_CRQ_CHECK.json
│   └── README.md            #   webhook contract for /api/addcrq
│
├── xr-steer/                # Artifacts deployed to RDNBoss jump host
│   ├── xr-steer.ssm.pl      #   Perl: redundancy check for traffic steering
│   ├── telnet_ios.py        #   telnetlib workaround for RDNBoss TRAD bug
│   └── README.md
│
├── cert/                    # SSL cert/key location (README says "Not in use")
└── logs/                    # Per-device / per-service log output (git-ignored content)
```

## Grouping the modules mentally

Think of the Python files in five tiers:

1. **Web / entry** — `app.py` (routes + dispatch), `schemas.py` (request bodies).
2. **Config / inventory** — `params.py`, `nodeinfo.py`, `config.ini`, `authinfo.py`.
3. **Shared engine** — `functions.py`, `functions_xr.py`, `stages.py`, `work_db.py`.
4. **Per-platform work** — the `work_*.py` family. Each owns one platform × campaign.
5. **Auxiliary services** — `compliance_a9k.py`, `compliance_spe.py` (run as their own
   systemd services), plus `authenticator.py` / `cmn_ldap.py` (auth) and `config_check.py`.

## What is safe to ignore when reading

- `static/js/core/` and `static/js/plugins/` — third-party vendor libraries, unmodified.
- `static/css`, `static/scss/black-dashboard/`, `static/fonts` — theme assets.
- `*.min.js`, `*.js.map` — compiled/minified vendor output.

## What is deceptively important

- `work_db.py` and `functions_xr.py` are the two largest files and hold most of the real
  logic — do not skip them. See [File Reference](09-file-reference.md).
- `params.py` and `nodeinfo.py` encode environment- and campaign-specific facts (target
  software versions, jump-host details, model-node data) that change per campaign.

## Gaps / needs confirmation

- `logs/`, `sessions/`, `db_backup/`, `config/`, `temp/` are git-ignored (see `.gitignore`)
  — their runtime contents are not in the repo.
- systemd unit files live in `/etc/systemd/system/` on the server (documented in
  `README.md` but not committed here).
