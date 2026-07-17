# 10 — Debugging Guide & Local Development

← [File Reference](09-file-reference.md) | Index | Next: [Improvements](11-improvement-recommendations.md) →

## Run modes (the most important thing to learn first)

SSM is controlled by environment variables read in `params.py`. These let you run the whole
app **without touching real devices**, which is essential for local development.

| Env var | Values | Effect |
|---------|--------|--------|
| `MODE` | `production` / `model` | `model` uses fake "model" nodes and canned data from `nodeinfo.py` (`MODEL_*`) instead of real inventory/devices |
| `MODELSKIP` | `Y` / `N` | In model mode, `Y` **skips** most device work (fast dry pass); `N` executes most work against the model node |
| `AUTHSKIP` | `True` | Bypass LDAP authentication entirely (offline UI testing) |
| `PORT` | e.g. `8443` | uvicorn listen port (falls back to `params.PORT`) |

Common invocations (from `README.md`):

```bash
# Full offline run: no auth, model nodes, skip device work
AUTHSKIP=True MODE=model MODELSKIP=Y python app.py

# Offline but actually exercise work against a model node
AUTHSKIP=True MODE=model python app.py

# Auth on, model data
MODE=model python app.py
```

When started in model mode, the startup log line reads `### SSM App started [Model] ###` or
`[Model_Modelskip]` so you can confirm the mode from `logs/ssm.log`.

## Local development setup

> There is **no** Dockerfile, CI config, or automated test runner in the repo. Development
> is manual. The steps below are the minimum to get the app running locally.

1. **Python 3.11.9** (the server uses `pyenv`). Create a virtualenv:
   ```bash
   python3.11 -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   ```
   Note: `python-ldap` needs system OpenLDAP dev headers; if `pip install` fails, install
   `openldap`/`libldap` dev packages (macOS: `brew install openldap` then point
   `CPPFLAGS`/`LDFLAGS` at it), or run with `AUTHSKIP=True` and skip LDAP for pure UI work.
2. **Secrets:** `cp authinfo.py.sample authinfo.py` and fill in values (git-ignored).
3. **MongoDB:** a local instance is required for stateful features (flow status, logs,
   stats). For pure UI/render testing in model mode much works without it, but any `work_db`
   call needs Mongo up on `localhost:27017`.
4. **Run:** `AUTHSKIP=True MODE=model MODELSKIP=Y python app.py`, then browse
   `http://localhost:8443/` (locally you hit uvicorn directly, not the `/xr/` Apache path).

## The `test/` scripts (manual, per-primitive)

`test/*.py` are **standalone runnable scripts**, not a pytest suite. Each sets
`MODE=model`, appends the parent dir to `sys.path`, and exercises one primitive against a
model node. They are the best executable documentation of how a function is called.

```bash
# Run a single stage of a flow against a model node
python test/test_stagex.py -n <modelnode> -s <stage> -f <flow> -m a9903

# Verify jump-host connectivity (pexpect)
python test/test_jumphostconnect.py

# Verify CLI-output parsing (TextFSM)
python test/test_parseCliOutput.py
```

Notable test scripts and what they prove:

| Script | Exercises |
|--------|-----------|
| `test_stagex.py` | Running any `stageN` of any `work_*` module by flow/stage |
| `test_jumphostconnect.py` | `functions.jumphost_connect` + pexpect send/expect loop |
| `test_parseCliOutput.py` | TextFSM parsing of Cisco output (largest test, 11KB) |
| `test_checkDeviceUp.py`, `test_checkDiskSpace.py`, `test_checkInstall.py` | Individual health checks |
| `test_isoActivate.py`, `test_installCommit.py`, `test_fpdUpgrade.py` | Upgrade primitives |
| `test_checkSatelliteStatus.py`, `test_upgradeDhSatellites.py`, `test_walkSpeRings.py` | Satellite/SPE topology logic |
| `test_compareServiceDb.py`, `test_runServiceDb.py` | ServiceDb integration |
| `test_checkCrq.py` | CRQ validation |

## Reading logs during debugging

- App log: `logs/ssm.log` (root logger, configured by `init_logger("ssm","logs")`).
- Compliance logs: `logs/compliance_a9k.log`, `logs/compliance_spe.log`.
- Per-device/stage transcripts: date- and CRQ-encoded filenames in `logs/` — the raw device
  session (what pexpect saw). These are what the in-app Log Viewer shows via
  `/api/getlogfiles` + `/api/showlogfile`, and what `/api/getlog?id=<oid>` renders per stage.
- Test-run logs are named `TEST_<...>` by the test scripts' `init_logger` calls.

## Troubleshooting cheat-sheet

| Symptom | Likely cause | Where to look |
|---------|--------------|---------------|
| App won't start / import error | An error in any `work_*` module (all imported at startup) | `logs/ssm.log`, run `python app.py` in foreground |
| A flow "already locked, running, do nothing" | Stale MongoDB lock (e.g. after a restart mid-flow) | `work_db` lock records; terminate/rerun via UI |
| Flow button disabled in UI | Form validation (CRQ must be 15 chars starting `CRQ`, node selected) | `scripts.html` `disableSubmitButton` |
| Device commands hang / timeout | Jump-host (RDNBoss) connectivity or prompt mismatch | `test_jumphostconnect.py`, pexpect `expect` timeouts in `functions_xr` |
| Old KexAlgorithms SSH failure to RDNBoss | SSH client too new for old device | add `~/.ssh/config` block from `README.md` |
| Empty compliance dashboard | Compliance service didn't run / Mongo empty | `logs/compliance_*.log`, `systemctl status` |
| CRQ not accepted | VO webhook didn't post, or `genKey` mismatch | `/api/addcrq` handler, VO task config in `votask/` |
| 401 on `/api/*` | Missing/expired cookie (8h token) | re-login; check `P.SECRET` unchanged |

## Verifying a change safely

Because real runs touch production routers, follow the model-mode-first discipline:

1. Reproduce with `MODE=model` (+ `MODELSKIP=Y` for a dry pass).
2. Exercise the specific primitive with the matching `test/test_*.py` script.
3. Only then, if unavoidable, run a **non-impacting** stage (e.g. stage 1 / a pre-check)
   against a real device — `test_stagex.py`'s docstring explicitly warns to use only
   non-impact stages against production nodes.

## Gaps / needs confirmation

- No CI/CD, linting, or coverage tooling exists — adding it is a recommendation (see
  [Improvements](11-improvement-recommendations.md)).
- Exact `MODEL_*` fixtures in `nodeinfo.py` and how completely they cover each platform
  (pending inventory deep-dive).
