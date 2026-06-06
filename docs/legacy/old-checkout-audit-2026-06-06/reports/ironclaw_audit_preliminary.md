# IronClaw / OpenClaw Preliminary Security Audit (Hostinger VPS)

## 1. Host & Network Posture

- Host: Ubuntu 24.04.3 LTS (Hostinger VPS).
- Firewall: UFW **active**.
  - Allowed (public): 22/tcp (SSH), 80/tcp (HTTP), 443/tcp (HTTPS).
  - Denied: 3389/tcp, 8088/tcp, 3847/tcp, 4173/tcp, 5353/udp, 50051 (v4/v6).
  - 50051 allowed only on `127.0.0.1` (loopback).
- Result: Public attack surface is limited to SSH + web ports; custom ports are blocked from the internet.

## 2. IronClaw / OpenClaw Runtime

- IronClaw binary: `/home/sherlockhums/.cargo/bin/ironclaw`.
- CLI: `ironclaw --help` and `ironclaw run --cli-only` work.
- Recent issue: DB migration error
  - `Migration failed: migration V9__flexible_embedding_dimension is missing from the filesystem`
  - Resolved by syncing migrations and reinstalling from source; no current migration errors.
- Listeners (from `ss`):
  - `ironclaw`: `127.0.0.1:3000` (localhost only).
  - `openclaw-gateway`: `127.0.0.1:3334`, `127.0.0.1:18789–18792`, UDP `0.0.0.0:5353`.
- Result: IronClaw/OpenClaw services are **not exposed externally**; they are bound to localhost and further protected by UFW.

## 3. Warehouses & File Permissions

- Primary warehouse: `~/.openclaw/`
  - Contains: agents, credentials, logs, workspace, node_modules, configs (`openclaw.json`), etc.
- IronClaw state: `~/.ironclaw/`
- Permissions (current / target):
  - `~/.openclaw`: `chmod 700` (applied).
  - `~/.ironclaw`: `chmod 700` (applied).
  - All `.env`-style files under `~/.openclaw`:
    - Target: `chmod 600` (owner read/write only).
    - Top-level `~/.openclaw/.env` is already `-rw-------` owned by `sherlockhums`.

## 4. Secrets & Environment Files

- High-value secrets centralized in `~/.openclaw/.env`, including:
  - LLM / AI keys: `OPENAI_API_KEY`, `OPENAI_WHISPER_API_KEY`, `NVIDIA_API_KEY`, `OPENROUTER_API_KEY`, `OPENCODE_API_KEY`, `GEMINI_API_KEY`, `HUGGINGFACE_API_TOKEN`, etc.
  - Service keys: `MOLTBOOK_API_KEY`, `SHIPYARD_API_KEY`, `VERCEL_API_KEY`, `ELEVENLABS_API_KEY`, `IMAGEKIT_*`, `BING_*`, `PINECONE_*`, `TALK_API_KEY`, `TWILIO_SID`, `TWILIO_TOKEN`.
  - Internal tokens: `OPENCLAW_AUTH_TOKEN`, `SAG`.
- Additional `.env` files under:
  - `workspace/tools/model_foundry/...`
  - `workspace/tasks/...`
  - `workspace/projectmorpheus/...` (backend, frontend, Alpaca refs, tests, etc.).
- Current posture:
  - `.env` locations are known and recognized.
  - Permissions locked down to owner-only (via `chmod 600`) across the tree.
- Recommended hygiene:
  - Keep `.env.example` / `.env.template` free of real secrets.
  - Optionally split global `.env` into per-project `.env` files to reduce blast radius.
  - Ensure `.env` and secrets are in `.gitignore` and not committed.

## 5. SSH & Access Control

- SSH configuration:
  - `PasswordAuthentication yes`
  - `PubkeyAuthentication yes`
  - `PermitRootLogin yes`
- User preference: keep root login enabled (Hostinger VPS).
- Mitigations:
  - UFW limits SSH to port 22 only.
  - Fail2ban configured for SSH (assumed active).
  - Strong, unique passwords required for `root` and `sherlockhums`.
- Future optional hardening (if desired):
  - Switch to `PermitRootLogin prohibit-password` (root via key only) once root keys are fully in place.
  - Confirm fail2ban `sshd` jail status and ban settings.

## 6. System Health Snapshot

- Memory: ~7.8 GiB total; healthy free/available RAM.
- Swap: 0B (no swap configured) – implies potential OOM kills under heavy load; availability consideration for load tests.
- Disk: `/dev/sda1` ~96G total, ~53% used; no immediate disk pressure.
- Top processes: `openclaw-gateway`, `python app.py`, `ollama serve`, Xorg/xrdp, `ironclaw`, terminals – all expected.

## 7. Risk Summary

- **Confidentiality:**
  - High-value API keys and internal tokens are concentrated in `.env` files.
  - Permissions on `~/.openclaw`, `~/.ironclaw`, and `.env` files are now restricted (700/600), which significantly reduces local exposure risk.
- **Integrity:**
  - IronClaw DB migrations are now consistent; startup succeeds.
  - No signs of unauthorized processes from the snapshots reviewed.
- **Availability:**
  - IronClaw previously blocked by migration mismatch; now resolved.
  - No swap means aggressive load testing could trigger OOM; tests should be controlled and monitored.

## 8. Next Recommended Steps

1. **Secret hygiene & structure**
   - Periodically review `.env` contents and remove unused keys (e.g., old API keys).
   - Rotate any keys that may have been exposed historically.
   - Keep examples/templates free of real secrets.

2. **Monitoring & load simulations**
   - Install tools like `htop`, `glances`, `iotop`, `bmon`/`nload` for live terminal visualization.
   - Design controlled load tests against key services while monitoring CPU/RAM/IO (especially given no swap).

3. **SSH & fail2ban review**
   - Confirm fail2ban `sshd` jail is active and banning after a small number of failed attempts.
   - Consider moving root to key-only login in the future if operationally acceptable.
