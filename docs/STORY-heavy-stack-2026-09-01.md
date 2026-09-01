# Whole stack — cutover, policy/models, oauth keepalive

**Date:** 1 Sep 2026  
**For:** external builder. CoS reviews after.  
**Not live.** No `--go`. Recover paused. Watchdog stays dead.

This is **one story**. Yesterday’s overlay and today’s keepalive both edit `xai-prompt-session.cjs`. Do **not** open a third isolated implementation PR that fights #4 for that file. Start from **BlockedPath #4** and add keepalive on that bench.

## The three benches

| Bench | PR | What | Status |
|---|---|---|---|
| 1. Weekly Cursor → Heavy | https://github.com/BlockedPath/grok-bot-setup/pull/3 | `bin/fleet-heavy` gated `--go` | Open. Dry-run default. |
| 2. Policy + models | https://github.com/BlockedPath/grok-bot-setup/pull/4 | Per-agent effort + Pump on Codex | Open. **Not copied onto the live hook.** CoS-reviewed head `3494a97`. GitHub head later moved — verify SHA before you branch. |
| 3. OAuth keepalive | this docs PR | Probe / refresh attempt / fail-closed expired key | Design only. Implement **on top of #4**. |

## What we already built (policy / models) — do not rebuild

Live host is still **one** Heavy env (`SAND_XAI_REASONING_EFFORT=medium`, `grok-4.6`) for every agent. Overlay is code + JSON, not deployed.

Policy file (on box after deploy): `/home/box/agent-data/agent-inference-policy.json`  
Hook: `xai-prompt-session.cjs` `resolveAgentInference` / `extractAgentId`.

| Agent | UUID (office) | Provider | Model | Effort |
|---|---|---|---|---|
| CoS | `5cadd652-c086-4ef2-8b64-e9c466e848b8` | grok-heavy | grok-4.6 | **high** |
| Development | `7a4ecb2d-9742-493e-bffb-87deb7a722b1` | grok-heavy | grok-4.6 | **high** |
| Finance | `01da7977-b202-4a08-a775-834768e6150e` | grok-heavy | grok-4.6 | **low** |
| Pump Desk | `8ae9a103-cfa2-406d-9bf3-eea00ca5b3a9` | **codex** | local `127.0.0.1:10531` | n/a (not Heavy) |
| Everyone else / unknown UUID | — | grok-heavy | grok-4.6 | **medium** + WARN |

Hard rules already in #4 (CoS review, keep them):

1. Policy **missing** → not grok-heavy for Pump (`policy-missing`). Fail closed.
2. Pump UUID **without a row** → `policy-row-missing`. Never silent Heavy.
3. `extractAgentId` **allowlist only**: `agentId`, `agent.id`, `metadata.agentId`. Ignore `source_agent_id` / `target_agent_id` (those are routing, not the speaker).
4. Policy reads into **locals**. Never write `SAND_XAI_*` into `process.env` (that would retune the whole office).
5. Codex auth missing → throw `CODEX_AUTH_MISSING`. No Cursor fallback. `ensure-xai-inference.sh` / `patch-xai-fail-closed.sh` rethrow hook errors.

**Still open on #4 (do not skip):** empty `agentId` still resolves default Heavy medium. Before Pump GO, log-only slice 0: dump `sessionOptions` **keys only** (no bodies) from a desktop terminal after copying the module. If the live host does not pass `agentId`, Pump cannot be fenced.

Tests already there: `bash ./scripts/test-agent-inference-policy.sh`. Keep them green when you add keepalive.

## What broke overnight (keepalive)

Host `/health` 200. Access `key` in `~/.grok/auth.json` expired `2026-08-31T21:00:38Z`. Hook `grokSessionToken()` only reads `key`, never `expires_at`, never `refresh_token`. Mute. Human fix: `/home/box/.grok/bin/grok login --device-auth` (~6h).

Existing `scripts/token-expired.py` + `scripts/refresh-grok-if-expired.sh` did not run (watchdog dead). Recover is hook-only and **paused**. Do not re-arm `host-hook-watchdog.sh` (it `restart-host` / SIGTERM’d the office).

Modules to add (see `docs/DESIGN-heavy-oauth-keepalive-2026-09-01.md`):

- **M1** `bin/grok-auth-status` — probe, file script, no pasted python, never print tokens.
- **M2** keep `refresh-grok-if-expired.sh` — skip / `grok models` / `refresh-failed`. Stub both outcomes. Unproven that `grok models` rotates `expires_at`.
- **M3** same `xai-prompt-session.cjs`: if `expires_at` past, do **not** send the dead key (`GROK_AUTH_EXPIRED`). This is why it belongs on #4’s file, not a parallel PR.
- **M4** ping-only CoS recipe in docs. Quiet if ok. Ping if `<2h` or expired.
- **M5** `bin/grok-auth-cutover-status` — hook + oauth + watchdog pid + recover armed? Never `--go`.

## How to implement (one bench)

1. Branch from **#4 head** (`feat/agent-inference-policy` on `panrix/grok-bot-setup`, confirm SHA).
2. Keep policy JSON + tests.
3. Add M1–M5. M3 goes into the existing session module next to `resolveAgentInference`.
4. `npm test` + `bash ./scripts/test-agent-inference-policy.sh` + `bash ./scripts/test-grok-auth-keepalive.sh`.
5. Open the implementation PR into **BlockedPath/grok-bot-setup** (not this docs-only fork PR). Cite #3, #4, and this packet.
6. Comment: tests run / tests **not** run (no live login, no host restart, no `--go`).
7. Ping CoS to critique like yesterday.

## Two doors (still)

- Hook missing / Cursor wipe → `adapters.sh recover` / `fleet-heavy --go` (Ricky, desktop terminal). **Paused.**
- Token dead / health 200 → M1, optional M2, else human device-auth.

Do not weld them. Policy overlay is a **third** knob (who gets which model/effort) and only works if oauth is alive **and** `agentId` is present.

## Hard no

No `--go` from chat. No starting host-hook-watchdog. No enabling grok-session-recover. No tokens in logs. No pasted python one-liners. No Cursor fallback. No silent Heavy for Pump.
