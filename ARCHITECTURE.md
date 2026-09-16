# ARCHITECTURE — GrokRep

Companion to `PRODUCT_DESIGN.md`. How the Grok Bot competitor is built: **control plane + sandbox VM + model gateway**.

Default stack (MVP): change only with a written decision.

| Layer | Default | Why |
|---|---|---|
| Control plane | TypeScript (Node 22+), Postgres, Redis | Fast web + queues |
| App | Next.js (web) | One repo for marketing + app |
| Sandbox | [E2B](https://e2b.dev) or Fly Machines | Firecracker isolation, pause/resume, no GPU |
| Models | OpenRouter first, direct Anthropic/OpenAI when volume pays | One bill, live `/models` |
| Secrets | KMS / Doppler + envelope encryption in DB | Keys never in VM image |
| Object store | S3-compatible | `/workspace` snapshots |
| Auth | Better Auth or Clerk | Email + Google |
| Payments | Stripe | Subscription + metered overage |

---

## 1. System

```
                    ┌─────────────────────────────────────┐
   Web / mobile     │           CONTROL PLANE             │
                    │  auth · workspaces · bots · groups  │
                    │  billing meters · connector ingress │
                    │  moderation · audit                 │
                    └──────────────┬──────────────────────┘
                                   │ wake / RPC / logs
                                   ▼
                    ┌─────────────────────────────────────┐
                    │     SANDBOX VM  (per workspace)     │
                    │  /workspace disk · bot workers      │
                    │  tools: files, shell, fetch         │
                    │  connectors: Telegram long-poll     │
                    │  NO provider API keys on disk       │
                    └──────────────┬──────────────────────┘
                                   │ inference over
                                   │ private sidecar
                                   ▼
                    ┌─────────────────────────────────────┐
                    │          MODEL GATEWAY              │
                    │  our keys (KMS) · BYO keys (vault)  │
                    │  catalog · cache · debit pool       │
                    │  OpenRouter / Anthropic / OpenAI …  │
                    └─────────────────────────────────────┘
```

**Rule:** sandbox cannot call `api.anthropic.com` with a key it holds. It calls **our gateway** with a short-lived workspace token. Gateway attaches the real key.

---

## 2. Control plane

Responsibilities:

- CRUD: users, workspaces, bots, groups, memberships, connector credentials (encrypted).
- Sandbox lifecycle: create, sleep, wake, snapshot, destroy.
- Job queue: inbound Telegram/webhook → `wake if sleeping` → `run turn` → `post result`.
- Meters: `sandbox_ms`, `prompt_tokens`, `completion_tokens`, `provider`, `model`, `bot_id`.
- Moderation: all bot outbound.
- Stripe webhooks → plan entitlements.

API shape (illustrative):

```
POST /v1/chat/completions     # OpenAI-compat, workspace-scoped
POST /v1/bots
POST /v1/bots/:id/start
POST /v1/groups
GET  /v1/usage
POST /v1/keys                 # BYO, encrypted at rest
POST /internal/sandbox/wake
```

Chat from the web UI uses the same completions endpoint the sandbox uses.

---

## 3. Data model (Postgres)

```
users
workspaces          plan, stripe_id, sandbox_id, pool_reset_at
workspace_members   role
sandboxes           provider_id, state (sleeping|waking|ready|error), last_active
bots                workspace_id, persona_version, model, status (config|running|paused)
bot_channels        bot_id, kind (telegram|discord|bsky|x), encrypted creds
groups              workspace_id
group_members       group_id, user_id | bot_id
threads             workspace_id, group_id?
messages            thread_id, role, model, tokens
usage_events        workspace_id, meter (sandbox|tokens), amount, bot_id?
byo_keys            workspace_id, provider, encrypted_blob
audit_events        actor, action, payload
persona_versions    bot_id, body, diff_from
```

`/workspace` files are **not** rows. They are a disk snapshot in object storage keyed by `sandbox_id`.

---

## 4. Sandbox lifecycle

```
signup
  → provision VM (paused)
  → empty /workspace snapshot

first chat or inbound connector
  → WAKE: resume VM, mount snapshot, start daemon
  → daemon heartbeats control plane

idle > plan.sleep_after
  → wait until no open turn holds the wake lock
  → SNAPSHOT /workspace only after last turn is **committed** (or failed) and sandbox is **idle under lock**
  → PAUSE VM
  → state = sleeping

destroy workspace
  → delete VM + snapshots
```

**Daemon inside VM (our agent):**

- HTTP to control plane (mTLS or signed workspace token).
- Runs bot workers (one process or one supervisor with N children).
- Executes tools locally.
- For inference: `POST gateway/v1/chat/completions` with workspace token — never a lab key.

**Network policy:** egress allowlist = gateway, Stripe-unrelated; GitHub/npm only if we explicitly add “install packages” later. MVP: no unrestricted internet.

**Resource caps:** cgroup CPU/RAM from plan. Kill runaway `yes` loops.

---

## 5. Model gateway

```
request (workspace token, model, messages, tools)
  → auth workspace
  → resolve model:
        if byo_keys[provider] && user opted that bot to BYO → user key
        else → platform key
  → check token pool (platform path only)
  → call provider (OpenAI-compat or native Anthropic for cache)
  → record usage_events
  → return stream
```

**Catalog job:** hourly `GET /models` from OpenRouter (and directs). Store `provider`, `id`, `tier` (low|mid|flagship), `input_price`, `output_price`. UI never hardcodes `grok-4`.

**Anthropic:** native Messages API + prompt cache for persona prefix when we go direct. OpenRouter is fine for MVP.

**402 path:** pool exhausted → UI: upgrade / overage / BYO. Gateway returns a typed error, not a model hallucination.

**Typed errors** (gateway → client; never hallucinate these as model text):

| Code | When |
|---|---|
| `pool_exhausted` | Platform token pool at 100% |
| `sandbox_capped` | Sandbox hours exhausted |
| `model_unavailable` | Catalog / provider down |
| `auth_failed` | Workspace or BYO key bad |
| `rate_limited` | Per-workspace or provider limit |
| `moderation_blocked` | Outbound failed moderation (channel path) |
| `turn_conflict` | Idempotency / wake-lock conflict |


---

## 6. Turn execution (chat or bot)

Canonical state machine, idempotency, and crash/retry: **`SPECS/turn-protocol.md`**.

```
queued → waking → running_tools → inferring → moderating → posting → committed
                                                              ↘ failed
```

Sketch:

```
1. Insert turn (`queued`) with idempotency key before wake
2. Acquire wake lock (Redis); wake sandbox if sleeping
3. Optional tools: daemon runs them into /workspace/tmp/<turn_id>/; results are data
4. Gateway completion (stream in-app only; buffer for external)
5. Moderation on final text
6. If FAIL: do not post externally; log; optional user-facing “blocked”
7. If channel: post; if approval queue: wait → committed
8. Persist messages + usage; promote tmp → bots/<id>/ only on committed
```

**Redis uses (explicit):** job **queue**, per-sandbox **wake lock**, **rate limit** counters. Not a substitute for Postgres truth.

Social channels: char budget + no markdown; **never stream outbound** before moderation. In-app chat: markdown OK; may stream tokens, still moderate final.

---

## 7. Connectors

**Telegram MVP (webhook-only):**

- User pastes bot token → encrypted in `bot_channels`.
- **Webhook to control plane only** — not long-poll inside the sandbox (VMs sleep).
- Control plane accepts webhooks **24/7**, **enqueues** a turn, then **wakes** the VM.
- Wake → turn protocol → moderate → reply. Sleep timer resets after committed/failed.

Do not keep VMs warm just for Telegram. See `SPECS/turn-protocol.md` anti-patterns.

---

## 8. Billing

- Stripe Customer on workspace (Team: one customer, N seats).
- Subscription = plan entitlements (JSON on `workspaces.plan`).
- Meters: Stripe metered items for overage **or** our own prepaid credits table. MVP: hard cap, no overage code yet — upgrade or BYO.
- Webhook: `invoice.paid` / `subscription.deleted` → pause sandboxes.

---

## 9. Security

| Threat | Control |
|---|---|
| Key leak from VM | Keys only on gateway; short-lived workspace JWT |
| Cross-tenant | One VM per workspace; no shared containers |
| Prompt injection | Tool results marked data; no side-effect tools v1 |
| Crypto mining in VM | CPU cap, no unrestricted egress, sleep |
| BYO key theft | Envelope encrypt, show last4, rotate UI |
| SSRF via browse tool | Allowlist + block RFC1918 |
| Webhook spoof | Telegram secret token; Discord signatures |

---

## 10. Repo layout (when code starts)

```
apps/web            Next.js app + marketing
apps/gateway        Model gateway worker
apps/control        API + workers (or monorepo packages/api)
packages/agent      Daemon image that runs IN the sandbox
packages/db         Drizzle schema
infra/              Fly/E2B templates, Terraform later
docs/               this folder’s markdown stays at repo root for now
```

Do not start this tree until MVP in `PRODUCT_DESIGN.md` §15 is the sprint. This file is the map, not a license to build everything.

---

## 11. MVP build order

Aligned with `SPECS/exceptional-bar.md` (do not reorder):

1. ADR sandbox driver + empty wake/sleep demo (`DECISIONS/001-sandbox-vendor.md`).
2. Gateway OpenRouter stream + typed errors + `usage_events`.
3. Auth + workspace + two meters UI.
4. Chat through gateway (no sandbox tools yet).
5. Daemon + path-jailed files tools + turn state machine + idempotency.
6. Telegram webhook → turn → moderate → reply.
7. Hard caps 80%/100%.
8. Groups + @mention only (no router).
9. Red-team + wake latency dashboard.

Each step is demoable alone.
