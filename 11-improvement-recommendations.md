# 11 — Improvement Recommendations

← [Debugging Guide](10-debugging-guide.md) | Index | Next: [Glossary](12-glossary.md) →

Recommendations are grouped by theme and tagged **[High] / [Med] / [Low]** by priority.
Each cites concrete evidence from the codebase so you can verify before acting. Items marked
**(pending)** will be sharpened once the module/engine deep-dives complete.

## Security

- **[High] Unauthenticated API endpoints.** `/api/mgmtvlans/{state}`, `/api/mgmtdata`,
  `/api/mgmtdatatable` have **no** login dependency (`app.py` ~L776–847). They proxy to
  SAMURAI and return infrastructure data. Add `Depends(manager)` or an equivalent gate.
- **[High] TLS verification disabled.** `/api/mgmtdata` calls SAMURAI with
  `requests.post(..., verify=False)` (`app.py` ~L818). Enable verification or pin the CA.
- **[High] Wide-open CORS.** `allow_origins=['*']` with `allow_credentials=True`
  (`app.py` L54–60) — this combination is rejected by browsers for credentialed requests and
  is unsafe intent. Restrict to the actual origin(s).
- **[High] Secret & credential handling.** `authinfo.py` holds secrets in source form;
  `xr-steer/telnet_ios.py` reads a device password from `~/.pswd.txt` and uses **plaintext
  telnet** (port 23). Move secrets to a vault/secret manager; prefer SSH over telnet.
- **[Med] Webhook auth strength.** `/api/addcrq` is gated only by `genKey(node+crq)==key`.
  Confirm `genKey` uses a keyed HMAC with a strong secret, not a plain hash (pending
  `functions.py` review). Consider signature + timestamp to prevent replay.
- **[Med] App port bound to `0.0.0.0:8443`.** Reachable on all interfaces though only Apache
  should reach it. Bind to `127.0.0.1` and let Apache be the sole ingress.
- **[Low] Hardcoded operator identity.** `telnet_ios.py` hardcodes user `n113272`; systemd
  units run as that user. Parameterize for portability and least-privilege.

## Architecture

- **[High] Import-time coupling / no plugin model.** `app.py` imports every `work_*` module
  at startup (L26–39), so one bad module downs the whole web service, and adding a platform
  means editing `app.py`'s imports **and** the `flow`-range `if/elif` dispatch in
  `/api/runflow`. Introduce a registry (platform → orchestrator + flow ranges + stages) so
  platforms self-register and dispatch is data-driven.
- **[High] Flat namespace.** ~20 top-level modules with no packages. Group into packages
  (`web/`, `engine/`, `platforms/`, `data/`, `integrations/`) to make boundaries explicit
  and reduce accidental cross-imports.
- **[Med] Two god-modules.** `functions_xr.py` (3007 lines) and `work_db.py` (1349 lines)
  concentrate most logic and risk. Split by concern (connection vs parsing vs upgrade
  primitives; locks vs logs vs stats vs CRQ) behind narrow interfaces (pending deep-dive to
  propose exact seams).
- **[Med] Flow-number semantics are implicit.** Integer ranges encode campaign + platform.
  Replace magic ranges with named campaign objects; keep the numbers only as display labels.
- **[Low] Dead/disabled code retained inline.** Several routes and orchestrator branches are
  commented out or `return False` (completed campaigns). Also confirmed dead:
  `authenticator.py` (the `Account01Authenticator` class is imported nowhere — live auth is
  `cmn_ldap.LDAP`), and `work_a9010_isis.py` (only referenced in commented-out code). Move
  these to an archive/branch or guard behind a feature flag to reduce reader confusion.

## Reliability

- **[High] Dangling flow locks on restart.** Locks live in MongoDB with no owner/heartbeat;
  a process restart mid-flow leaves a flow "locked, running" forever until manual
  terminate/rerun (`app.py` runflow logic; [Runtime Flow](04-runtime-flow.md)). Add a lock
  TTL/heartbeat and startup reconciliation that clears orphaned locks.
- **[High] No graceful shutdown.** No shutdown hook closes in-flight `pexpect` sessions on
  SIGTERM; a deploy/restart can abandon a device session mid-command. Add a FastAPI shutdown
  handler and cooperative cancellation.
- **[Med] Cooperative pause/terminate depends on check points.** If an orchestrator blocks
  on a long `expect`, pause/stop won't take effect until the next stage boundary. Document
  and, where feasible, shorten the polling interval / add timeouts (pending confirmation of
  check points).
- **[Med] Retry/rollback consistency.** Standardize retry and rollback across `work_*`
  modules (pending deep-dive to catalog current behavior per platform).
- **[Low] Monitoring/alerting.** No health endpoint or metrics. Add `/healthz`, export
  basic metrics (active flows, lock age, device-timeout counts), and alert on stuck flows.

## Performance

- **[Med] Async unused.** FastAPI is async but device I/O is synchronous on threads
  (`ThreadPoolExecutor(20)` for checks; inline for flows). This is acceptable for pexpect,
  but the fixed 20-worker pool + 5s stagger (`app.py` runworkflow) caps bulk-check
  throughput; make pool size and stagger configurable and tune per environment.
- **[Med] Repeated device queries.** Compliance and checks likely re-fetch inventory/parse
  the same outputs. Introduce short-lived caching for inventory (ServiceDb) and parsed
  results (pending deep-dive to confirm hotspots).
- **[Low] UI polling every 2s.** `runner.html` / `workflow.html` poll `/api/getmap` on a 2s
  timer regardless of activity. Consider Server-Sent Events or backoff when idle.

## Code quality

- **[Med] `eval()` in stage dispatch.** `test/test_stagex.py` uses `eval(func)(**flowinfo)`
  to call `stageN`. Even in a test harness, prefer `getattr(module, f"stage{s}")`. Check
  whether production dispatch uses similar patterns (pending).
- **[Med] Duplication across `work_*`.** The per-platform modules repeat stage scaffolding,
  pre/post-check structure, and device-comms boilerplate. Extract shared base
  stages/mixins (`stages.py` already hints at this — pending confirmation of coverage).
- **[Low] Naming & typos.** User-facing strings contain typos ("chekc", "oroginal");
  segment/worktype string-munging (`runmode.replace(...)`) is fragile. Introduce typed enums
  for worktype/mode/petype.
- **[Low] String-substring device routing.** `"-spe-" in device` / `"-e-" in device`
  (`app.py` runworkflow) encodes platform in the hostname. Centralize device→platform
  resolution in `nodeinfo.py`.

## DevOps

- **[High] No CI/CD, no automated tests.** `test/` is a set of manual scripts. Add a CI
  pipeline: lint (ruff/flake8), type-check (mypy), and convert `test/` into a pytest suite
  runnable in `MODE=model` without hardware.
- **[Med] No containerization.** Deployment is `git pull` + `restart_ssm-tools.sh` on a
  hand-configured host with `pyenv`. A Dockerfile + compose (app + MongoDB) would make envs
  reproducible and simplify onboarding.
- **[Med] Infra defined in docs, not code.** systemd units, Apache config, ufw rules, and
  the log-cleanup cron live in `README.md` prose. Commit them as versioned files
  (`deploy/systemd/*.service`, `deploy/httpd/*.conf`, `deploy/cron/*`).
- **[Low] Observability.** Centralize logs (they are per-file on the host) and add structured
  logging with correlation by CRQ/flow.

## UI / UX

- **[Med] Frontend modernization.** jQuery + inline `<script>` blocks in templates +
  string-built HTML tables (`/api/mgmtdatatable` returns HTML strings) are hard to maintain.
  Consider a light component approach (even Alpine/HTMX) and returning JSON that the client
  renders, keeping the black-dashboard look.
- **[Med] Accessibility.** SVG stage boxes, color-only status signaling, and marquee status
  text are not accessible. Add ARIA roles/labels, text status alongside color, and remove
  marquee. Ensure keyboard operability of Run/Pause/Stop.
- **[Low] Error handling in AJAX.** Many `error:` handlers are empty (`//Do Something`).
  Surface failures to the user (bootstrap-notify is already loaded).
- **[Low] Design consistency.** Inline styles are scattered through templates; consolidate
  into the existing SCSS to keep spacing/typography consistent.

## Suggested sequencing

1. **Security quick wins** (auth on mgmt endpoints, `verify=True`, CORS, bind localhost).
2. **Reliability** (lock TTL + startup reconciliation, graceful shutdown).
3. **DevOps foundation** (pytest in model mode + CI lint/type; commit infra files).
4. **Architecture** (registry-based platform dispatch; package layout) — enables safe
   refactor of the two god-modules.
5. **UI/UX modernization** — lower risk once the above are in place.

## Gaps / needs confirmation

Items tagged **(pending)** depend on the in-progress deep-dives of `functions_xr.py`,
`work_db.py`, and the `work_*` platform modules. This document will be revised to cite exact
functions and line numbers once those complete; see [Module Reference](08-module-reference.md)
and [File Reference](09-file-reference.md).
