# Design — Heavy OAuth keepalive (probe / refresh / weekly cutover)

**Date:** 1 Sep 2026  
**Status:** design for a PR. Not live. No host restart. Recover stays paused.  
**Repo bench:** `BlockedPath/grok-bot-setup` (same factory as PR #3 / #4).  
**Driver:** Ricky. Overnight mute 31 Aug / 1 Sep with `/health` 200.

## Related PRs (the whole story — do not rebuild these)

- **PR #3** fleet-heavy cutover CLI (gated `--go`, no SIGTERM in dry-run): https://github.com/BlockedPath/grok-bot-setup/pull/3
- **PR #4** per-agent effort matrix + Pump on Codex (head `3494a97`, review-only, **not deployed**): https://github.com/BlockedPath/grok-bot-setup/pull/4
  - CoS review items 1–3 are in that head (policy-missing, Pump row required, allowlist `extractAgentId`).
  - Remaining live hole: empty `agentId` still defaults Heavy medium — log-only slice 0 before Pump GO.
- This PR is a **third bench**: oauth keepalive. Do not pile M1–M5 onto #4. Do not flick live.

## The issue (what actually broke)

Two different layers were mixed up.

| Layer | What it is | Last night |
|---|---|---|
| Host / hook | `host-main.cjs` + `xai-prompt-session.cjs` | Up. `/health` 200. Pid stayed. |
| Grok OAuth | `~/.grok/auth.json` access `key` | Expired `2026-08-31T21:00:38Z`. File still *present*. |
| Watchdog | `host-hook-watchdog.sh` every 30s | **Dead** (stale pidfile 9986). We killed it after it SIGTERM'd the office. |
| CoS recover | `@every 30m` `adapters.sh recover` | **Paused.** It only reinstalls the hook. It does not refresh oauth. |

The hook reads **only** `entry.key` from `auth.json` (`grokSessionToken()`). It never looks at `expires_at`. It never uses `refresh_token`. So a dead key is sent to `cli-chat-proxy.grok.com` until someone runs `grok login --device-auth` (browser).

The file **does** contain a `refresh_token` (non-empty). The grok CLI docs in the binary say the CLI auto-refreshes via that token. Our helper `scripts/refresh-grok-if-expired.sh` only runs `grok models` when `token-expired.py` says `1`. That helper never ran last night because the watchdog was dead. We have **not proven** that `grok models` actually rotates `key` + `expires_at`. Treat that as a hunch the PR must test with a stub CLI, then a later live GO.

`grok login --device-auth` is still the only **proven** human fix. No cron can click the Mac browser.

## Why the thing we already built didn't save us

Already on disk (do not rewrite from scratch):

- `scripts/token-expired.py` — `1` if expired, `0` otherwise. No secrets.
- `scripts/refresh-grok-if-expired.sh` — if expired, `grok models`. Comment: never `grok login`, never delete `auth.json`.
- `scripts/host-hook-watchdog.sh` — every 30s: call refresh helper, and if hook missing → `ensure-xai-inference` + **`adapters.sh restart-host`**.
- CoS routine **Grok-session recover** — hook recover only. Paused.

Three failure modes, all real:

1. **Wrong layer armed.** Recover/watchdog fix "hook wiped". Overnight was "token dead". Health 200 hid it.
2. **Watchdog is a host-restart gun.** We cannot re-arm it as the oauth fix. Last week's SIGTERM is why it is dead.
3. **Refresh is unproven.** `grok models` may no-op. Probe-only is honest until a stub test (and later a live GO) shows `expires_at` moved forward.

Paste-python into tmux also failed (`NameError: EOF`). Status must be a **file script**, never a one-liner.

## What we want (factory: benches, then QC)

Same as yesterday: **one job = one PR**. Critique before anything copies onto the live host.

Weekly Cursor → Heavy should become:

```
desktop terminal:
  adapters.sh recover          # hook only, still needs Ricky GO if it restarts host
  bin/grok-auth-status         # probe
  # if EXPIRED: grok login --device-auth  (human)
  # if ok: done
```

Overnight mute should become: CoS (or a cron **outside** the host-restart watchdog) pings Ricky *before* expiry. Never restarts the host from the oauth path.

## Modules (build these as separate files; tests per module)

### M1 — `bin/grok-auth-status` (probe)

- Calls `scripts/token-expired.py` (keep that module).
- Prints: `exists`, `has_key` (bool), `expires_at`, `seconds_left`, `ok|soon|EXPIRED`.
- **Never** prints `key`, `refresh_token`, email, or file dump.
- `soon` = under 2h (constant, documented).
- Exit: `0` ok, `1` soon, `2` expired/missing. Tests with fixture json (no live file).

### M2 — `scripts/refresh-grok-if-expired.sh` (refresh attempt)

Keep the contract: never `grok login`, never delete `auth.json`, never print tokens.

Change vs today (hunch, builder must verify):

- If not expired: exit 0, print `skip`.
- If expired or soon: run a **non-interactive** grok command that the CLI uses to mint a new access token (today: `grok models`). Capture only exit code.
- Re-run M1. If still expired: exit 3 `refresh-failed` (honest). Do not loop. Do not device-auth.
- Test with `FLEET_ALLOW_TEST_OVERRIDES=1` stub `grok` that rewrites fixture `expires_at` into the future vs a stub that no-ops. **Never hit live auth.x.ai in CI.**

### M3 — hook: fail closed on expired key (no 401 spam)

In `xai-prompt-session.cjs` `grokSessionToken` / `resolveAuth`:

- If `expires_at` is in the past, do **not** send the dead `key`. Return mode `none` or throw a coded error `GROK_AUTH_EXPIRED` with a one-line log: `expires_at=… seconds_left=…` (no token).
- Optional (v2, not this PR unless cheap): if `seconds_left < 10min`, try spawning M2 once per process with a mutex. Default **off**. Host process must not block 30s on `grok models` every chat.
- Tests in a new `scripts/test-grok-auth-keepalive.sh` (and js if needed): expired fixture → no token returned.

### M4 — ping-only CoS recipe (docs, not auto-enable)

`docs/GROK_AUTH_KEEPALIVE.md`:

- CoS routine prompt: run M1. Quiet if `ok`. Ping Ricky if `soon` or `EXPIRED` with the expiry timestamp and the one desktop command: `/home/box/.grok/bin/grok login --device-auth`.
- **Do not** call `adapters.sh recover`, `restart-host`, or start the host-hook watchdog from this routine.
- Recover stays a *hook* tool. Oauth stays a *token* tool. Two doors.

### M5 — weekly cutover wrapper (status only in this PR)

`bin/grok-auth-cutover-status` (name bikeshed ok):

- Prints hook present? (grep `createXaiPromptSession` in host-main, file exists).
- Prints M1 result.
- Prints watchdog pid alive? (info only; this PR does **not** start it).
- Prints recover enabled? (info only; this PR does **not** enable it).
- Exit non-zero if oauth expired. Never `--go`. Never SIGTERM.

Live `adapters.sh recover` / fleet-heavy remain **out of band** (existing PRs). This PR does not flick.

## Hard no (same as fleet-heavy)

- No `adapters use` / fleet-heavy `--go` from tests or from CoS.
- No starting `host-hook-watchdog.sh` (it restarts the host).
- No enabling grok-session-recover from this PR.
- No printing tokens.
- No pasting python one-liners in docs; file scripts only.
- No live `--device-auth` in CI.

## Success

- `bash ./scripts/test-grok-auth-keepalive.sh` green (stub fixtures).
- `docs/GROK_AUTH_KEEPALIVE.md` names M1–M5 and the two-door rule.
- Implementation is a **new** PR that cites this docs PR. CoS critiques after, like #4.
- **Not deployed.** Desktop GO later, after review.

## Open questions for the builder (verify, do not invent)

1. Does `grok models` actually rewrite `expires_at`? Stub both outcomes. If the real CLI cannot refresh without device-auth, M2 must report `refresh-failed` and docs must say "human login is the only path".
2. Does the hook cache `auth.json` in process, or re-read per session? Current live code re-reads. Keep that.
3. Do not merge oauth keepalive into the host-hook watchdog in this PR.
