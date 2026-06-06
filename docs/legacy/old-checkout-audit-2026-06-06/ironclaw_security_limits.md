# IronClaw System Audit – Overview (Environment, Capabilities, and Limits)

Owner: Sherlock / IronClaw  
Context: High-level security and capability overview while `.env` cleanup is in progress. This document is *not* a full secret audit; it focuses on architecture, posture, and process.

---

## 1. Scope & Purpose

This audit describes:

- The **host and network posture** of the VPS running OpenClaw/IronClaw.
- The **runtime characteristics** of IronClaw/OpenClaw (ports, processes, data dirs).
- The **assistant’s capabilities and hard limits** with respect to this VPS.
- Known **security improvements already applied**.
- **Deferred work** (notably `.env`/secrets consolidation) and how to approach it later.

It is intentionally **read-only and descriptive**: no file changes, no live commands.

---

## 2. Host & Network Posture (Summary)

**Host:**
- Provider: Hostinger VPS.
- OS: Ubuntu 24.04.3 LTS (noble).

**Firewall (UFW):**
- Status: **active**.
- Publicly allowed:
  - `22/tcp` – SSH
  - `80/tcp` – HTTP
  - `443/tcp` – HTTPS
- Explicitly denied:
  - `3389/tcp` (RDP)
  - `8088/tcp`, `3847/tcp`, `4173/tcp`
  - `5353/udp`
  - `50051` (v4/v6), with a specific allow for `127.0.0.1:50051` only
- Interpretation:
  - Public attack surface is limited to SSH + web ports.
  - Custom/gRPC-style ports are bound to localhost or blocked externally.

**SSH posture:**
- `PasswordAuthentication yes`
- `PubkeyAuthentication yes`
- `PermitRootLogin yes`
- User preference: keep root login enabled due to Hostinger operational needs.
- Mitigations:
  - UFW restricts SSH to port 22 only.
  - Fail2ban is in place for SSH/unauthorized access attempts.
  - Strong, unique passwords are assumed for `root` and `sherlockhums`.

**Risk note:**  
SSH is more permissive than ideal (root + passwords), but this is an explicit trade-off accepted by the owner. Fail2ban + UFW + strong passwords are relied on for mitigation.

---

## 3. IronClaw / OpenClaw Runtime Overview

### 3.1. Binaries and processes

- IronClaw CLI:
  - Installed at: `/home/sherlockhums/.cargo/bin/ironclaw`
  - `ironclaw --help` and `ironclaw run --cli-only` both work.
- Observed processes (examples):
  - `ironclaw` – Rust binary, listens on localhost.
  - `openclaw-gateway` – gateway/orchestrator process.
  - `python app.py` – application process in `lobsterenv`.
  - `ollama serve` – model server.
  - Xorg/xrdp, terminals, etc. – expected interactive environment components.

### 3.2. Ports and listeners

From `ss` output (representative):

- `ironclaw`:
  - `tcp LISTEN 127.0.0.1:3000` – CLI/backend bound to localhost only.
- `openclaw-gateway`:
  - `tcp LISTEN 127.0.0.1:3334`
  - `tcp LISTEN 127.0.0.1:18789–18792`
  - `udp UNCONN 0.0.0.0:5353` (mDNS-like)
- No IronClaw/OpenClaw services are listening on `0.0.0.0` for TCP.

**Interpretation:**
- IronClaw/OpenClaw control surfaces are **not directly exposed to the internet**; they are bound to `127.0.0.1` and further protected by UFW.
- The only externally reachable services are whatever you run on 80/443 (and SSH on 22).

### 3.3. Data and configuration directories

- Main OpenClaw “warehouse”:
  - `~/.openclaw/`
  - Contains: agents, credentials, logs, workspace, node_modules, config files (`openclaw.json` and backups), memory, etc.
- IronClaw state directory:
  - `~/.ironclaw/` (exact contents not fully enumerated, but used for IronClaw’s own state/DB).
- Permissions (current target posture):
  - `chmod 700 ~/.openclaw`
  - `chmod 700 ~/.ironclaw`
  - `.env` and related files under `~/.openclaw` set to `chmod 600`.

### 3.4. Database & migrations (IronClaw)

- IronClaw uses a migration-based database schema with migrations in the source repo:
  - `~/ironclaw/migrations/V1__initial.sql` through `V9__flexible_embedding_dimension.sql`.
- Previous issue:
  - Error: `Migration failed: migration V9__flexible_embedding_dimension is missing from the filesystem`
  - Resolved by aligning the installed CLI with the source migrations (reinstall from repo).
- Current state:
  - `ironclaw run --cli-only` starts successfully.
  - Migrations are in sync with the binary.

**Risk note:**  
Migration errors affected **availability**, not confidentiality. Resolution indicates DB schema and binary are now consistent.

---

## 4. System Health Snapshot (High Level)

Based on earlier snapshots:

- **Memory:**
  - ~7.8 GiB total.
  - Sufficient free/available RAM under normal load.
  - **Swap: 0B** – no swap configured.
    - Implication: under heavy or spiky load (e.g., large models, stress tests), the kernel may kill processes (OOM) instead of swapping.
    - This is an availability concern, especially for load simulations.

- **Disk:**
  - `/dev/sda1` ~96G total, ~53% used (~51G used, ~46G free).
  - Boot and EFI partitions have comfortable free space.
  - No immediate disk pressure.

- **Top processes:**
  - `openclaw-gateway`, `python app.py`, `ollama serve`, `ironclaw`, Xorg/xrdp, terminals.
  - All appear expected for the environment; no obvious malicious or runaway processes in the sampled view.

---

## 5. Security Improvements Already Applied

These changes materially improved your posture:

1. **Warehouse permissions tightened:**
   - `~/.openclaw` and `~/.ironclaw` set to `700` so only the owner can read/write/execute.
   - `.env` files under `~/.openclaw` set to `600` (owner read/write only).

2. **Secrets rotation and cleanup (owner-managed):**
   - `.env` tokens/keys have been rotated to fresh values.
   - `.env` issues are being handled as a separate, owner-driven cleanup project.
   - This audit intentionally does **not** treat current `.env` contents as an incident.

3. **IronClaw DB migrations fixed:**
   - Resolved the missing `V9__flexible_embedding_dimension` migration.
   - Ensures IronClaw can start and operate with a consistent schema.

4. **Network exposure confirmed minimal:**
   - Verified that IronClaw/OpenClaw listeners are on localhost only.
   - Confirmed UFW rules limit external exposure to SSH and web ports.

---

## 6. Assistant Capabilities & Hard Limits on This VPS

### 6.1. What IronClaw (as an assistant) *can* do

Within the constraints of this environment:

- **Advise on commands and configurations:**
  - Suggest `find`, `ss`, `ufw`, `chmod`, `chown`, `fail2ban-client`, etc.
  - Design scripts for audits, monitoring, and load tests (to be run by you).
- **Interpret outputs you paste:**
  - Analyze `ls -l`, `ps aux`, `ss -tulpn`, `ufw status`, config snippets, etc.
  - Classify risk and suggest mitigations.
- **Help design policies and structure:**
  - Propose secret consolidation strategies.
  - Recommend directory structures and permission models.
  - Outline backup, logging, and monitoring approaches.
- **Generate ready-to-save documentation:**
  - Markdown reports (like this one).
  - Checklists for future audits.
  - Sample configs and command bundles.

### 6.2. What IronClaw *cannot* do (hard limits)

- **No direct shell or file access to the VPS:**
  - Cannot run commands, read files, or modify anything on the server directly.
  - All actions must be executed by you and outputs pasted back for analysis.

- **No automatic secret inspection or redaction:**
  - I only see what you paste.
  - I cannot scan your filesystem for secrets on my own.
  - I rely on you to avoid pasting full secret values.

- **No direct integration with Hostinger control panel:**
  - Cannot change root/SSH settings, firewall, or snapshots from the provider side.
  - Can only advise on what to configure.

- **No guaranteed view of full system state:**
  - My assessment is limited to the commands and files you choose to share.
  - There may be services, configs, or data I do not know about.

**Implication:**  
All security actions are **human-in-the-loop**. I can design, explain, and review, but you (or another trusted agent with shell access) must apply and validate changes.

---

## 7. Deferred / Future Work (to revisit after `.env` cleanup)

These items are intentionally postponed until your `.env` cleanup is finalized:

1. **Runaway `.env` and secrets mapping (IronClaw Audit v2 directive):**
   - Systematic, read-only discovery of `.env`-style and `secrets.*` files across:
     - `$HOME`, `~/.openclaw`, `~/.openclaw/workspace`, `~/.openclaw/workspace-main`.
   - Classification of each file (critical/medium/low) based on indicators.
   - Identification of duplication and “runaway” env usage.
   - Design of a consolidation strategy:
     - 1–2 canonical secret locations.
     - Per-project env boundaries.
     - Git ignore patterns and backup policies.

2. **Periodic env/secret drift detection:**
   - A Lobster/cron-style job that:
     - Re-runs the discovery logic.
     - Flags new `.env`/`secrets.*` files in unexpected locations.
   - Comparison of inventories between runs.

3. **Load testing and observability:**
   - Install and configure:
     - `htop` or `glances` for CPU/RAM/process view.
     - `iotop` for disk I/O.
     - `bmon` or `nload` for network traffic.
   - Design safe load tests:
     - HTTP/API load on services behind 80/443.
     - Controlled CPU/RAM stress (`stress-ng`) while monitoring, mindful of no swap.
   - Document thresholds and “red lines” (e.g., max CPU, RAM, or load average to tolerate).

4. **SSH/fail2ban configuration review:**
   - Confirm `sshd` jail is active and effective.
   - Optionally move toward:
     - Root via key only (`PermitRootLogin prohibit-password`), if/when operationally acceptable.
   - Document a recovery plan using Hostinger console in case of SSH lockout.

---

## 8. Current Risk Summary (High-Level)

**Confidentiality:**
- `.env` and secret files are now permission-restricted (`600`), and tokens have been rotated.
- `~/.openclaw` and `~/.ironclaw` are not world-readable.
- IronClaw/OpenClaw control surfaces are bound to localhost.
- Residual risk: historical `.env` sprawl and backups may still exist; detailed mapping is deferred until cleanup is complete.

**Integrity:**
- No evidence (from the data shared) of tampering or unauthorized processes.
- IronClaw DB migrations are consistent; schema matches the binary.
- Config backups (`openclaw.json.bak.*`) exist and should be kept under `700` directories.

**Availability:**
- IronClaw previously blocked by a migration mismatch; now resolved.
- No swap means heavy load or misbehaving processes could cause OOM kills.
- Disk has comfortable free space; no immediate risk of full disk.

---

## 9. Recommended Next Checkpoint

Once `.env` cleanup is finalized by the owner and other agents:

1. Re-run a **focused v2 env audit**:
   - Use the `ironclaw-audit-v2-directive.md` as the playbook.
   - Generate JSON + Markdown artifacts describing env file locations, classifications, and consolidation recommendations.

2. Decide on a **canonical secrets layout**:
   - For example:
     - `$HOME/.env.openclaw` for global platform secrets.
     - Per-project `.env` files under `~/.openclaw/workspace/...` for local concerns.
   - Reduce duplication and legacy backups.

3. Implement a **lightweight monitoring + load test plan**:
   - So you can safely observe the system under stress and detect regressions early.

---

This document is intended as a stable reference for how the system and assistant are currently configured and constrained. It can be updated after the `.env` cleanup and v2 audit to reflect new findings and decisions.
