# GO packet — Heavy OAuth keepalive (docs bench)

**For:** external builder / factory QC. CoS will review after you open the implementation PR.  
**This branch:** docs only. Do **not** implement on this PR unless Ricky says so — open a **new** implementation PR that cites this one.

## Story so far (read these first)

1. Weekly Cursor → Heavy cutover is gated CLI: **https://github.com/BlockedPath/grok-bot-setup/pull/3** (`bin/fleet-heavy`). Dry-run default. `--go` is the only live flick. Last week a watchdog SIGTERM killed the office — do not re-arm `host-hook-watchdog.sh` as the oauth fix.
2. Per-agent effort + Pump on Codex overlay: **https://github.com/BlockedPath/grok-bot-setup/pull/4** (head `3494a97`). Review-only. **Not copied onto the live hook.** Empty `agentId` still defaults Heavy medium. Slice-0 live dump of `sessionOptions` keys still required before Pump GO.
3. Overnight 31 Aug / 1 Sep mute: host `/health` 200, OAuth access `key` expired. Human fix: desktop-terminal `/home/box/.grok/bin/grok login --device-auth`. Token lasts ~6h.

This packet is **bench 3**: keepalive modules so mute does not hide behind health 200. Separate from #4.

## Implement (new PR, not this docs PR)

See `docs/DESIGN-heavy-oauth-keepalive-2026-09-01.md`. Modules M1–M5. Hard no: no `--go`, no starting watchdog, no enabling grok-session-recover, no tokens in logs.

Reuse existing `scripts/token-expired.py` and `scripts/refresh-grok-if-expired.sh`. Do not rewrite from scratch.

## Two doors

- Hook missing / Cursor wipe → `adapters.sh recover` (Ricky GO; may restart host). That is #3 / recover. **Paused.**
- Token dead / health 200 → M1 probe, optional M2 `grok models`, else human device-auth. That is this design.

Do not weld the doors.

## After you build

Comment on the implementation PR: tests run, tests **not** run (no live login, no host restart). Ping CoS to critique like #4.
