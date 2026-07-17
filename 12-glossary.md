# 12 — Glossary of Networking & Domain Terms

← [Improvements](11-improvement-recommendations.md) | [Index](README.md)

Written for a software engineer with limited networking background. Terms are grouped by
theme rather than alphabetically so related concepts sit together.

## Router platforms & roles

- **PE (Provider Edge) router** — a router at the edge of a service-provider network that
  connects customer sites to the core. All devices SSM manages are PE routers.
- **SEP / NG-SEP / Small PE** — Telstra's naming for edge tiers. **SEP** = ASR 9010,
  **NG-SEP** = ASR 9903 (next-gen), **Small PE (SPE)** = smaller NCS 540 / NCS 55A2 boxes.
- **ASR 9010 / ASR 9903** — Cisco Aggregation Services Router chassis models. In code:
  `A9K`/`A9010` and `A9903`.
- **NCS 540 / NCS 55A2** — Cisco Network Convergence System routers used as Small PE.
- **IOS-XR** — Cisco's carrier-grade router operating system running on these platforms. Its
  install/upgrade model (packages, `install add/activate/commit`) is central to SSM.
- **RR (Route Reflector)** — a router that redistributes BGP routes to reduce the number of
  peerings; `nodeinfo.py` carries RR info.

## Satellites (nV / Fabric)

- **Satellite (nV satellite)** — a small access device (Cisco ASR9000v / 9000v2) that hangs
  off a host ASR9K and is managed as if it were line cards of the host. "nV" = network
  virtualization.
- **Single-Home (SH) satellite** — connected to **one** host PE. In code: `ShSat`.
- **Dual-Home (DH) satellite** — connected to **two** host PEs for redundancy. `DhSat`.
- **Satellite migration** — moving a satellite from one host/topology to another; SSM
  provides pre-check, interim-check, interface-description check, and post-check for this.

## Software upgrade concepts (IOS-XR)

- **Image / ISO** — the base OS software bundle installed on the router.
- **RPM / package** — individual software packages (IOS-XR uses an RPM-based packaging
  model). Optional feature sets are separate packages.
- **SMU (Software Maintenance Upgrade)** — a patch/hotfix for a specific bug (like a
  targeted software patch), identified by a Cisco defect ID (e.g. `CSCvt27458`).
- **install add / activate / commit** — the three-phase IOS-XR upgrade sequence:
  1. **add** — copy the package onto the device and unpack it (no effect yet);
  2. **activate** — make the package live (may require reload);
  3. **commit** — make the change persist across reloads.
  Understanding this sequence is essential to reading the upgrade `work_*` modules.
- **FPD (Field-Programmable Device)** — firmware on hardware components (line cards, etc.)
  that may need a separate `fpd upgrade` after a software upgrade. See `test_fpdUpgrade.py`.
- **BB / Broadband template, "BB uplift"** — a standard configuration template version;
  "uplift" = bringing a device up to the current template. Compliance tracks "BB Tmpl Ver".
- **Rollback** — reverting to the pre-change software/config if validation fails.

## Health-check & validation concepts

- **Pre-check** — capture device health/state **before** a change (baseline).
- **Post-check** — capture the same state **after** and compare to pre-check; differences
  flag potential impact.
- **Interim-check** — a mid-procedure check (e.g. after migrating one side of a redundant
  pair) before completing the change.
- **Redundancy / red-check** — verifying that a redundant path exists so traffic survives
  the change. SSM's `redcheck` runs a script on RDNBoss (`xr-steer.ssm.pl`).
- **Traffic steering** — deliberately moving traffic off a device/path before working on it.
  The `xr-steer` script generates the CLI for this (applied manually by the operator).
- **Convergence** — the time for the network to stabilize after a topology change (e.g. the
  Mgmt EVPN post-check note says to allow 3 minutes for convergence).

## Layer-2 / EVPN / VRRP concepts (Mgmt EVPN workflow)

- **VLAN (Virtual LAN)** — a logical layer-2 segment identified by a VLAN ID.
- **Bridge Domain (BD)** — an IOS-XR construct grouping interfaces into one L2 broadcast
  domain (roughly a VLAN inside the router). Mgmt EVPN post-check verifies MAC learning on
  mgmt BDs.
- **EVPN (Ethernet VPN)** — a control-plane technology for extending L2 domains across an
  IP/MPLS core using BGP. "Mgmt EVPN migration" moves management connectivity onto EVPN.
- **AC (Attachment Circuit)** — the customer/edge-facing interface bound into a BD/service.
- **Split-horizon** — an L2 loop-prevention rule: a frame received on one member of a group
  is not sent back out other members of the same group. The A9K "WIP-Mgmt split-horizon"
  flow applies this to management ACs.
- **VRRP (Virtual Router Redundancy Protocol)** — two routers share a virtual IP; one is
  **Master**, one is **Backup**. Mgmt EVPN root migration is done Backup-side first
  (interim-check), then Master-side (post-check).
- **BUM (Broadcast, Unknown-unicast, Multicast) traffic** — the flooded traffic classes in
  L2. "A9903 BUM remediation" fixes how the device handles this traffic.
- **MAC learning** — a switch/BD associating MAC addresses with ports; post-checks confirm
  MACs are being learned on the new mgmt BDs.
- **HSRP bundle** — Hot Standby Router Protocol group on a bundle interface; referenced by
  the traffic-steering redundancy list.

## Operations / change-management terms

- **CRQ (Change Request)** — a change-management ticket (BMC Remedy). A flow requires a CRQ
  in "Implementation in Progress" status. Format: `CRQ` + 12 digits (15 chars total).
- **VO** — Telstra's orchestration/automation platform that validates CRQ status and posts
  it back to SSM's `/api/addcrq` webhook. See [Architecture](02-architecture.md).
- **RDNBoss** — the jump host (bastion) with privileged ("trad") login to network devices.
  SSM reaches all routers **through** RDNBoss.
- **TRAD** — the login/access mechanism on RDNBoss used to authenticate to devices. A TRAD
  bug on the new RDNBoss is worked around by `xr-steer/telnet_ios.py`.
- **ServiceDb** — an inventory/service database API (device metadata & service data).
- **SAMURAI** — an internal API providing management-VLAN data for the Mgmt EVPN workflow.
- **Dry run** — execute the procedure without making real changes (`config.ini`
  `dryrun_only`).

## Device-access terms (how SSM talks to routers)

- **pexpect** — the Python library SSM uses to drive interactive terminal sessions
  (spawn SSH/telnet, `sendline`, `expect(prompt)`), rather than a structured API.
- **Prompt** — the device's CLI prompt string (e.g. ending in `#`) that pexpect waits for to
  know a command finished.
- **TextFSM** — a template engine that turns semi-structured Cisco CLI text output into
  structured rows/dicts. Templates live in `textfsm/`. Essential for validation comparisons.
- **Jump host / bastion** — an intermediate server you connect to first, then hop onward to
  the target device. Here that is RDNBoss.

## Related documents

- Platforms & flows in context → [Project Overview](01-project-overview.md)
- Upgrade/validation procedures in detail → [Device Workflows](07-device-workflows.md)
- External systems (VO, SAMURAI, RDNBoss, ServiceDb) → [Architecture](02-architecture.md)
