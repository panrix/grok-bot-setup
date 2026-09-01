# GO packet — whole Heavy stack (docs bench)

**For:** external builder / factory QC. CoS will review after you open the **implementation** PR.  
**This branch:** docs only. Do **not** implement here. Open a **new** BlockedPath PR **branched from #4**, not a parallel fight over `xai-prompt-session.cjs`.

Read first: `docs/STORY-heavy-stack-2026-09-01.md` (the one-pager). Then `docs/DESIGN-heavy-oauth-keepalive-2026-09-01.md` (M1–M5).

## Story so far (all three)

1. Weekly Cursor → Heavy: **https://github.com/BlockedPath/grok-bot-setup/pull/3** (`bin/fleet-heavy`). Dry-run default. `--go` is the only live flick. Do not re-arm `host-hook-watchdog.sh`.
2. Policy + models (yesterday): **https://github.com/BlockedPath/grok-bot-setup/pull/4**  
   CoS-reviewed SHA `3494a97`. GitHub head later moved — confirm SHA. **Not on the live hook.**  
   Matrix: CoS + Development **high** grok-4.6; Finance **low**; default **medium**; Pump UUID **Codex only** (`127.0.0.1:10531`), fail closed.  
   Keep CoS review: policy-missing, Pump row required, allowlist `extractAgentId`, no `SAND_XAI_*` env writes.  
   Still open: empty `agentId` → Heavy medium. Slice-0 keys-only dump before Pump GO.
3. OAuth keepalive (this packet): host `/health` 200 vs dead access `key`. Human login: `/home/box/.grok/bin/grok login --device-auth`. Add M1–M5 **on top of #4**. M3 is the same session file as the policy overlay.

## Implement

Branch from #4. Keep policy tests. Add keepalive tests. Cite #3, #4, and this docs PR. No `--go`, no watchdog start, no recover enable, no tokens in logs.

## After you build

Comment: tests run / tests **not** run (no live login, no host restart). Ping CoS.
