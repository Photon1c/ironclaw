# Old IronClaw Checkout Capture — 2026-06-06

Source checkout:

```text
/home/sherlockhums/ironclaw
```

Canonical checkout:

```text
/home/sherlockhums/apps/ironclaw
```

Purpose: preserve useful audit/report artifacts from the older checkout before preparing to remove or archive it.

## Captured files

- `ironclaw_security_limits.md`
- `reports/ironclaw_audit_preliminary.md`

## Intentionally not copied

- `.env`
- `.env.delete`
- other secret/config files
- old `.git` metadata
- build/target artifacts

## Old checkout git state at capture time

```text
## cursor/llm-backend-abstraction-a385...upstream/cursor/llm-backend-abstraction-a385
?? ironclaw_security_limits.md
?? reports/
```

## Old checkout remotes at capture time

```text
origin	https://github.com/nearai/ironclaw.git (fetch)
origin	https://github.com/nearai/ironclaw.git (push)
upstream	https://github.com/Photon1c/ironclaw (fetch)
upstream	https://github.com/Photon1c/ironclaw (push)
```

## Old checkout latest commits at capture time

```text
dd50dc5 (HEAD -> cursor/llm-backend-abstraction-a385, upstream/cursor/llm-backend-abstraction-a385) Merge branch 'main' into cursor/llm-backend-abstraction-a385
906d618 fix: add type annotation for Vec<String> to fix Windows build (#452)
4a7339f chore: release v0.13.0 (#385)
dc7d9cc fix(channels): add host-based credential injection to WASM channel wrapper (#421)
a21dba0 refactor: rename WasmBuildable::repo_url to source_dir (#445)
```

## Notes

The older checkout was on `cursor/llm-backend-abstraction-a385`; the canonical checkout under `/home/sherlockhums/apps/ironclaw` is the current synced main/fork workspace. This capture is documentation-only and does not imply the old branch should be merged.
