# Agent inference policy (effort matrix + optional per-agent provider)

**Status:** overlay on Heavy. Design: `kb/.../DESIGN-per-agent-effort-router-2026-08-31.md`. GO packet 31 Aug wins on conflicts.

## What it does

`xai-prompt-session.cjs` reads `/home/box/sand-data/agent-inference-policy.json` (override with `SAND_AGENT_INFERENCE_POLICY`) on each session.

- Keys are **agent UUIDs** (never display names).
- Default: Heavy `grok-4.6` effort **medium**.
- Per-agent `effort` / `provider` / `model` as locals — **never** written into `process.env` (avoids Pump poisoning CoS).
- Unknown UUID → medium + WARN.
- `provider: codex` (Pump) → `http://127.0.0.1:10531/v1` with dummy Bearer `openai-oauth`. **Fail closed** if `~/.codex/auth.json` missing or proxy down — no Grok fallback, no Cursor fallback.
- Hook install (`ensure-xai-inference.sh`) **rethrows** on create failure (no Cursor catch).

## Install on the box (desktop terminal)

```bash
cp /home/box/grok-bot-setup/examples/agent-inference-policy.json \
  /home/box/sand-data/agent-inference-policy.json

# Deploy updated module + fail-closed hook (one restart — Node require cache)
cp /home/box/grok-bot-setup/xai-prompt-session.cjs /home/box/sand-host/xai-prompt-session.cjs
# Re-run patch only if hook still has Cursor fallback:
#   bash /home/box/grok-bot-setup/scripts/ensure-xai-inference.sh
# Then ONE host restart from desktop terminal, e.g.:
#   /home/box/grok-bot-setup/adapters.sh restart-host
# (Only after Ricky GO for restart. Not from CoS.)
```

After first deploy, **policy JSON edits do not need restart** (mtime cache).

## Smoke

1. Finance short ping → log `effort=low provider=grok-heavy`
2. CoS one line → `effort=high`
3. Development one line → `effort=high`
4. Pump (after `codex login --device-auth` + `adapters start openai-oauth`) → `provider=codex`

```bash
grep '\[sand-xai\] agent=' /tmp/sand-host-manual.log | tail -20
```

## Tests

```bash
bash ./scripts/test-agent-inference-policy.sh
```

Never points at live adapters / `--go`.
