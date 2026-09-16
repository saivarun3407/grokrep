# Turn protocol

**Status:** Build law (MVP)  
**Date:** 2026-09-16  
**Owns:** crash recovery, idempotency, ordering of tools → inference → moderation → post

## States

```
queued → waking → running_tools → inferring → moderating → posting → committed
                                                              ↘ failed
any non-terminal → failed (on timeout / crash after retries exhausted)
```

Bot runtime (separate from turn):

```
config → starting → running → draining → stopped
                ↘ error
running → paused
paused → starting
error → starting | stopped
```

Only `running` bots count toward the plan's running-bot cap.

## Identity

| Field | Rule |
|---|---|
| `turn_id` | UUID, created **before** wake |
| `idempotency_key` | Stable per inbound event, e.g. Telegram `update_id`, Discord message id, in-app `client_message_id` |
| Unique | `(channel, idempotency_key)` in `delivery_attempts` |

## Ordering

1. Insert turn row (`queued`) with idempotency key.  
2. Acquire wake lock for `sandbox_id` (Redis). Wake if sleeping → `waking`.  
3. Daemon runs tools under path jail → `running_tools`. Writes go to `/workspace/tmp/<turn_id>/`.  
4. Gateway completion → `inferring`.  
5. Moderation pipeline on **final** text → `moderating`.  
6. External channels: post only after pass → `posting` → `committed`.  
7. Promote `tmp/<turn_id>/` into `bots/<bot_id>/` only on `committed`.  
8. On moderation fail: `failed`, no channel post; in-app shows blocked notice.

### Streaming

- **In-app:** may stream tokens to UI; moderation still runs on final; failed → replace/flag.  
- **External (Telegram etc.):** buffer full text → moderate → post. Never stream outbound.

## Crash / retry

| Last status | On retry |
|---|---|
| `queued` / `waking` / `running_tools` / `inferring` | Re-run from that stage; tools re-exec into same tmp dir (idempotent writes preferred) |
| `moderating` | Re-run moderation on stored draft |
| `posting` | Check `delivery_attempts`; if success exists → mark `committed` without re-post |
| `committed` | No-op |
| `failed` | Manual or explicit user retry creates a **new** turn_id unless product says otherwise |

## Timeouts (defaults; tune in staging)

| Phase | Timeout |
|---|---|
| `starting` bot | 60s → `error` |
| `waking` | plan wake SLO × 2 |
| full turn | 120s MVP (raise for browse later) |

## Heartbeats

Daemon heartbeats control plane. Miss > 2× interval → sandbox `error`; next job re-wakes.

## Filesystem layout

```
/workspace/
  shared/                 # explicit opt-in cross-bot
  bots/<bot_id>/          # default tool root
  tmp/<turn_id>/          # wiped after commit or failed GC
```

## Anti-patterns

- Long-poll Telegram inside the VM  
- Posting before moderation on external channels  
- Provider API keys on the sandbox  
- Snapshot during unlocked `running_tools`
