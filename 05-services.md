# 05 — Services, Ports, Cron & Log Management

← [Runtime Flow](04-runtime-flow.md) | Index | Next: [API Reference](06-api-reference.md) →

SSM is **three long-running processes**, all managed by systemd, all with
`WorkingDirectory=/opt/ssm`, running as user `n113272`, using a `pyenv` Python 3.11.

## systemd services

| Unit file | Command | Purpose | Port |
|-----------|---------|---------|------|
| `ssm-tool.service` | `python app.py` | Web UI + API (uvicorn) | 8443 |
| `ssm-compliance-a9k.service` | `python compliance_a9k.py "07:10"` | Daily A9K compliance check at 07:10 | none |
| `ssm-compliance-spe.service` | `python compliance_spe.py "07:40"` | Daily SPE compliance check at 07:40 | none |

Unit definitions are reproduced in the root [`README.md`](../README.md). Key details:

- **`ssm-tool.service`** sets `Environment=MODE='production'` and `Environment=PORT=8443`,
  and `Restart=on-failure`. `app.py` reads `PORT` (falls back to `params.PORT`) and calls
  `uvicorn.run(app, host="0.0.0.0", port=...)`.
- The two **compliance services** take the run time as a CLI argument (`"07:10"` / `"07:40"`).
  They use the in-process `schedule` library to run daily at that time — the systemd service
  stays alive and the scheduler fires the check (see the module bodies). `MODE='production'`.

### Restarting after a deploy

`restart_ssm-tools.sh` restarts all three, then waits 15s and prints `systemctl status` for
each. Deploy procedure (from README): `git pull` on the server, then `./restart_ssm-tools.sh`.

## Reverse proxy & ports

```
Browser ──HTTPS──▶ Apache httpd ──HTTP──▶ 127.0.0.1:8443 (uvicorn / app.py)
                   /xr/  ProxyPass
```

- Apache (`/etc/httpd/conf/httpd.conf`) proxies `/xr/` → `http://localhost:8443/`
  (`ProxyPass` + `ProxyPassReverse`, `SSLProxyEngine on`).
- Because the app sits under `/xr/`, the frontend computes `basePath` from the first URL
  path segment (`scripts.html`) so all `/api/*` calls resolve correctly behind the proxy.
- A `ufw` rule that forwarded 443→8443 is documented but marked **"[Not in use]"** — the
  Apache proxy is the live path.
- SSL cert/key handling in `app.py` (`cert/` dir, `params.SSL_CERT`/`SSL_KEY`) is present in
  code but README marks the certs **"[Not in use]"** — TLS is terminated at Apache.

**Port summary:** the only app-owned listening port is **8443** (uvicorn, bound `0.0.0.0`).
MongoDB listens on its default **27017** locally. Apache owns 443.

> ⚠️ **Security note:** uvicorn binds `0.0.0.0:8443`, so the app port is reachable on all
> interfaces, not just localhost — the Apache-only path is a convention, not enforced by the
> bind. CORS is also wide open (`allow_origins=['*']` with credentials). See
> [Improvements](11-improvement-recommendations.md).

## Scheduled / cron jobs

There are two kinds of scheduling:

1. **In-process (`schedule` library):** the compliance services schedule their daily run
   internally. No crontab entry needed for these — systemd keeps the process alive and the
   library triggers the job.
2. **OS cron (external to the repo):** a **daily log-cleanup script** is documented in
   `README.md`. It runs in `/opt/ssm/logs` and:
   - gzips task logs (`ssm.log.*`, `compliance_a9k.log.*`, `compliance_spe.log.*`) older
     than 90 days;
   - deletes zero-byte log files;
   - gzips per-device logs (`2*` = date-prefixed) without a CRQ older than 90 days;
   - zips per-device logs **with** a CRQ older than 180 days into `CRQ<n><device>.zip`,
     preserving mtime.

   The actual crontab entry / script file is **not committed** — see Gaps.

## Logging model

- `functions.init_logger(name, "logs")` configures the root logger; each service names its
  log (`ssm`, `compliance_a9k`, `compliance_spe`).
- Per-device / per-stage transcripts are written to `logs/` with date- and CRQ-encoded
  filenames (that is what the cleanup script pattern-matches on). The Log Viewer
  (`/api/getlogfiles`, `/api/showlogfile`) reads these back into the UI.
- Retention: 90 days → compressed; 180 days (with CRQ) → zipped & archived.

See [Runtime Flow](04-runtime-flow.md) for how logging ties into stage execution, and
[Debugging](10-debugging-guide.md) for reading logs during development.

## Dependencies external to these processes

| Dependency | Used by | Purpose |
|------------|---------|---------|
| MongoDB (`ssm_db`, local) | all | State, locks, logs metadata, stats, CRQ |
| RDNBoss jump host (`vusno238`) | work_* / functions_xr | Path to reach network devices |
| VO webhook | `/api/addcrq` | CRQ status validation |
| SAMURAI API | Mgmt EVPN routes | Mgmt VLAN data |
| ServiceDb API | compliance / work | Device inventory & service data |

## Gaps / needs confirmation

- The log-cleanup cron entry (schedule + exact path) is not in the repo — confirm on server
  (`crontab -l` for `n113272`, and `/etc/cron.*`).
- Whether the `schedule`-based compliance jobs also self-restart on failure or rely solely on
  systemd — confirm by reading `compliance_a9k.py` / `compliance_spe.py` main blocks
  (covered by the data/auth/compliance deep-dive, pending).
- MongoDB deployment details (auth, replica set?) — README shows `mongodump`/`mongorestore`
  with a `dbadmin`/`admin` user but not the full connection config.
