# SSM Developer Documentation

> **SSM** (Sep-Software-Management) is a Python/FastAPI web tool that automates
> operational work on Telstra's Cisco IOS-XR provider-edge routers — software
> upgrades, SMU deployment, pre/post-check validation, compliance auditing, and
> guided migrations.

This documentation set reverse-engineers the application so a new developer (from a
software background, not necessarily networking) can maintain, extend, and modernize it
without relying on the original authors.

## How to read this (recommended order)

| # | Document | What it covers |
|---|----------|----------------|
| 01 | [Project Overview](01-project-overview.md) | Business purpose, executive summary, technology stack |
| 02 | [Architecture](02-architecture.md) | High-level & runtime architecture, component/dependency diagrams |
| 03 | [Folder Structure](03-folder-structure.md) | Directory-by-directory map |
| 04 | [Runtime & Request Flow](04-runtime-flow.md) | Startup sequence, request lifecycle, flow/stage execution model |
| 05 | [Services & Ports](05-services.md) | systemd services, ports, cron/scheduled jobs, log management |
| 06 | [API Reference](06-api-reference.md) | Every HTTP route and `/api/*` endpoint |
| 07 | [Device Workflows](07-device-workflows.md) | Upgrade, pre/post-check, validation, rollback (networking explained) |
| 08 | [Module Reference](08-module-reference.md) | Purpose & responsibilities of every Python module |
| 09 | [File Reference](09-file-reference.md) | File-by-file detail for important source files |
| 10 | [Debugging & Local Dev](10-debugging-guide.md) | Local setup, MODE flags, troubleshooting |
| 11 | [Improvement Recommendations](11-improvement-recommendations.md) | Architecture, code quality, security, DevOps, UI/UX |
| 12 | [Glossary](12-glossary.md) | Networking & domain terms for software engineers |

## The one-paragraph mental model

A user logs into a dark-themed dashboard and either (a) triggers a **flow** — a
multi-stage automated procedure against a single router, controlled by Run/Pause/Stop
buttons — or (b) triggers a **pre/post-check** — a health snapshot run in parallel across
many devices. The FastAPI backend (`app.py`) dispatches these to per-platform work
modules (`work_a9k.py`, `work_a9903.py`, `work_spe.py`, …). Those modules connect to
routers **through a jump host (RDNBoss)** using `pexpect` (SSH/telnet), send Cisco CLI
commands, parse the text output with **TextFSM** templates, and record every step in
**MongoDB** and per-device log files. Two additional **systemd services** run daily
**compliance checks** and publish results back to the dashboard. Change-request (CRQ)
validation is delegated to an external orchestration platform (**VO**) via a webhook.

## Status of this documentation

Some runtime components live outside the repository (systemd unit files, the log-cleanup
cron script, RDNBoss-side deployments, MongoDB data). Where evidence is incomplete this is
called out explicitly in each document under a **"Gaps / needs confirmation"** heading —
these are the files or runtime facts to gather next.
