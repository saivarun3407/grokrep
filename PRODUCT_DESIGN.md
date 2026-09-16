# PRODUCT_DESIGN — GrokRep (working name)

**Status:** v2 product spec (2026-09-15). Canonical.  
**Competitor:** Cursor **Grok Bot** (hosted cloud computer + Grok, no model picker).  
**Rename before public launch.** “GrokRep” / “your own @grok” is trademark risk.

---

## 0. One-liner

**Grok Bot, but you pick the model.**  
We provision a **sandbox VM** and a **multi-LLM model gateway**. You chat, create unlimited bot *configs*, put them in groups, and run a small company. You pay us. We pay the VM vendor and the labs.

```
User  --$-->  US  --$-->  sandbox (Firecracker / E2B / Fly)
                 └──$-->  models (OpenRouter and/or Anthropic, OpenAI, Google, xAI)
```

This is a **reseller + sandbox** company (Cursor-legal: our API keys, our machines). It is **not** “log into the user’s Claude Pro.” BYO key is an optional valve when they blow the included pool — not signup.

---


### Build law (engineering)

Authoritative build constraints (do not reinvent in code comments):

- [`SPECS/turn-protocol.md`](./SPECS/turn-protocol.md) — turn + bot state machines, idempotency, moderation-before-post
- [`SPECS/tool-policy.md`](./SPECS/tool-policy.md) — path jail, phased tools, egress
- [`SPECS/exceptional-bar.md`](./SPECS/exceptional-bar.md) — pass/fail + MVP build order
- [`DECISIONS/`](./DECISIONS/) — ADRs (e.g. sandbox vendor)

## 1. Problem

Grok Bot already sold the package people want: **a cloud employee that stays on when the laptop closes.** Two gaps:

1. **Brain is assigned.** Grok Bot has no model picker. Cursor IDE does; Grok Bot does not. Support bot on Claude, research bot on Grok, ops bot on GPT — impossible in one Grok Bot workspace.
2. **You live in Cursor’s account and Grok’s stack.** Switching the company off Grok means leaving the product.

We sell the same “it just runs” feeling with a **per-bot provider + model** field.

---

## 2. What we ship (the product)

Four nouns. If it isn’t one of these, it isn’t v1.

| Noun | What the user sees |
|---|---|
| **Workspace** | One company / personal account. Billing lives here. |
| **Sandbox** | One isolated Linux VM per workspace. Files, terminal, browser, bot processes. Sleeps idle, wakes on chat/summon. |
| **Chat** | Threads against a chosen model, with tools that run **in the sandbox**. |
| **Bots** | Named agents: persona + model + tools + channels. Unlimited *configs*. **Running** bots are capped by plan (they share the VM + token pool). |
| **Groups** | Rooms: humans + bots. Company ops = several bots in one group, each with its own model. |

**Models** are not a fourth app — they are a field on chat and on each bot, backed by **our** gateway.

---

## 3. Who it’s for

**ICP #1 — Grok Bot switcher**  
Wants a hosted always-on agent. Will not install Claude Code. Will not leave a Mac mini on. Wants Claude for writing, Grok for search-y tasks, GPT for structured ops.

**ICP #2 — small team / “run the company”**  
Support bot on Telegram, research bot in a group, founder in the same room. 2–20 people. Pays per seat + shared pool.

**Not ICP #1:** “I already pay Claude Max; don’t bill me tokens.” That’s the BYO toggle after they’re hooked, not the homepage.

---

## 4. Principles

1. **Signup has no CLI and no API key.** Sandbox exists and a cheap model answers in <10 minutes. Laptop can close.
2. **Two meters, always visible:** sandbox (CPU/hours) and models (tokens). Never “$20 unlimited VM + unlimited Opus.”
3. **New bots default to a cheap model.** Picker is the headline; flagship is an opt-in click.
4. **Unlimited = configs, not warm processes.** Pro: e.g. 3 running bots, rest cold-start.
5. **Our keys on the default path.** User keys optional, same VM, their bill.
6. **Summoned, labeled, no engagement farming.** Bots reply when asked. Disclosure on. No auto-follow/mass-DM.
7. **Sandbox is the computer, gateway is the brain.** Don’t put model credentials in the VM image.

---

## 5. User journeys

### 5.1 First 10 minutes (success test)

1. Sign up (email / Google). No card on Free; card on Pro.
2. Control plane creates a **sleeping** sandbox. First chat **wakes** it (target: <20s cold).
3. Chat opens with default cheap model. User sends “hello.” Reply streams.
4. User opens picker, switches to a mid-tier model, sends again. Badge shows provider + remaining pool.
5. **New bot:** name, one-line job, model, Telegram. We show BotFather steps or a deep link. `/start` on Telegram → bot replies from the sandbox.
6. User closes the laptop. Telegram still answers until the pool or VM-hour cap hits.

If step 5 requires `claude auth login`, we have failed the Grok Bot fight.

### 5.2 Create a company group

1. New **Group**: “Ops.”
2. Add humans (invite). Add bots: `Support` (Claude Haiku), `Research` (Grok), `Scribe` (GPT mini).
3. Human @mentions a bot in the group, or the group has a routing rule (keyword → bot).
4. All tool use (files, browser) happens in the **shared workspace sandbox** with per-bot filesystem prefixes or a shared `/workspace`.

### 5.3 Hit the cap

At **80%** of the token pool: banner. Choices: upgrade, buy extra usage, **paste own key** (Anthropic/OpenAI/Google/OpenRouter). Same sandbox. At **100%**: Platform models stop; BYO and already-running cheap fallback keep going if configured. Never silent 500s.

---

## 6. Feature spec

### 6.1 Chat

- Threads, streaming, retry, stop, edit-and-resend.
- Model picker: provider grouping, cost hint (low / mid / flagship), remaining pool.
- Tools (run in sandbox): read/write files under `/workspace`, terminal (allowlisted), web_search, browse_verify, view_image. v1: no arbitrary outbound except allowlisted fetch + connectors.
- Citations / tool-call trace in a side rail (“searching… verifying…”).
- History stored in control plane, not only on the VM (VM is ephemeral).

### 6.2 Bots

**Config (unlimited):**

| Field | Notes |
|---|---|
| Name, avatar, one-liner | |
| Persona | Versioned. Diffs shown. likes / dislikes / tone / hard limits. |
| Model | Provider + model id from live catalog. |
| Tools | Subset of sandbox tools. |
| Channels | In-app, Telegram, later Discord/Bluesky. |
| Autonomy | Auto-reply vs approval queue. |
| Disclosure | On, non-removable. |
| Blocklist | Topics, users, URL policy. |

**Runtime state machine** (see `SPECS/turn-protocol.md`):

```
config → starting → running → draining → stopped
                ↘ error
running → paused
paused → starting
error → starting | stopped
```

- Only **`running`** bots count toward the plan’s running-bot cap.
- **`paused`** = kill switch (workspace admin or us).
- Logs: every summon, model used, tokens, sandbox CPU, output after moderation.

**Agent loop** (portable @grok contract, clean-room — do not copy xAI prompt text):

persona (static) → tools (thread/context, search, browse-verify, media) → draft under channel budget → **moderation stage** → post or queue.

Rules we keep: retrieved text is **data**, not instructions; match language of the summon; no markdown on social channels; never tag the summoner; never condition on the bot’s own prior posts.

### 6.3 Groups

- A group is a thread with **members** (users + bots).
- Each bot keeps its own model.
- **MVP:** @mention only (no automatic router). A cheap router bot that assigns mentions to specialists is **deferred** past MVP.
- Company view: list of groups, which bots are running, pool burn per bot.

### 6.4 Sandbox (user-visible)

Status light: **Sleeping / Waking / Ready / Capped**.  
Disk: `/workspace` persisted across sleeps (object storage snapshot).  
Terminal: optional for Pro+, not on Free.  
“This is your cloud computer. Bots live here. It sleeps to save your hours.”

### 6.5 Models (user-visible)

Picker on chat and on each bot.

**Platform (default):** our gateway, counts against included pool.  
**BYO (optional):** user key, same sandbox, does not count against our pool; still counts VM hours.

Never: “Sign in with Claude.ai / ChatGPT / Gemini” in our hosted app.

### 6.6 Team

- Invite by email. Roles: Owner, Admin, Member, Viewer.
- Shared sandbox + shared pool (Team plan).
- Each human is a *seat*. We do **not** share one consumer Claude login across the org.
- Audit log: bot start/stop, persona publish, model change, connector add.

---

## 7. Sandbox VM (product contract)

| | Free | Pro | Plus | Team |
|---|---|---|---|---|
| VMs | 1, aggressive sleep | 1 workspace | 1 warmer, or 2 | shared + extra on higher SKU |
| Size | 1 vCPU / 1 GB | 2 vCPU / 2 GB | 4 vCPU / 4 GB | admin-chosen |
| Sleep | 5 min idle | 15 min idle | optional always-warm | always-warm add-on |
| Wake SLO | <30s | <20s | <10s warm | <10s |
| Disk persist | 1 GB | 10 GB | 50 GB | pooled |
| Concurrent **running** bots | 1 | 3 | 10 | 10 × seats (cap) |
| Network | allowlist | allowlist + connectors | same | same |

**Sleep:** snapshot disk, pause machine. **Wake:** restore, attach gateway sidecar.  
**Isolation:** one tenant per VM (Firecracker/E2B-class). No shared kernel with other customers.  
**Egress:** default deny except: model gateway, connector webhooks we own, allowlisted fetch for browse_verify, package mirrors we pin.  
**No GPU in v1.** Computer-use via headed browser in the VM if we add it later (Plus).


**Filesystem layout** (product contract; tools jail against this — `SPECS/tool-policy.md`):

```
/workspace/
  shared/                 # explicit opt-in cross-bot
  bots/<bot_id>/          # default tool root
  tmp/<turn_id>/          # wiped after commit or failed GC
```

Promote `tmp/<turn_id>/` into `bots/<bot_id>/` only when the turn is **committed** (`SPECS/turn-protocol.md`).

Implementation choice (engineering, not user-facing): E2B or Fly Machines or Firecracker on our metal. Pick one in `ARCHITECTURE.md`. User only sees “cloud computer.”

---

## 8. Model gateway (product contract)

**We are the API customer.** Keys in a KMS vault. Runtime workers decrypt per request. Keys never written into the sandbox.

**v1 catalog (fetch live IDs at boot; never hardcode retired names):**

| Tier | Role | Examples (illustrative) | New-bot default? |
|---|---|---|---|
| **Low** | High volume, support, router | Haiku / Flash / GPT mini / cheap Grok | **Yes** |
| **Mid** | Daily driver | Sonnet / GPT standard / Gemini Pro / Grok standard | Picker |
| **Flagship** | Hard tasks | Opus / GPT flagship / Grok heavy | Overage or Plus |

Exact IDs from provider `/models` + OpenRouter. If a name dies (`grok-4`, `deepseek-chat`), drop it at runtime.

**Routing:**

```
chat/bot request
  → workspace policy (allowed models, BYO?)
  → if BYO key for that provider: use it (user bill)
  → else: our key, debit token pool
  → if pool empty: 402-style UX (upgrade / BYO)
  → moderation on output
  → bill meters
```

**Prompt cache** where the provider supports it (Anthropic native). Static persona + tools = cache prefix. Don’t send Anthropic through a shim that drops cache.

**OpenRouter** as the long-tail aggregator so we don’t sign every lab on day one. Direct Anthropic + OpenAI as soon as volume pays for native cache and better errors.

---

## 9. Connectors

Bots speak **from the sandbox** (outbound) or via **control-plane webhooks** that enqueue work onto the VM.

| Phase | Channel | Notes |
|---|---|---|
| **MVP** | In-app chat + groups | Always on |
| **MVP** | Telegram | User’s BotFather token stored encrypted. **Webhook to control plane only** (24/7 accept → enqueue → wake VM). Long-poll inside the sandbox is **not** an MVP option (VMs sleep). Consent on `/start`. |
| **2** | Discord | Registered app, slash + @mention. No self-bots. |
| **2** | Bluesky | App password/OAuth on workspace. Self-label `bot`. Reply only if tagged. |
| **3** | X | User’s X API keys. Summoned-only. Confirm AI-reply approval in X console first. User pays X per-use. URLs in replies off by default ($0.20/post). |
| Skip | Farcaster | Unstable |

Cron / webhook inbound = phase 2 (same wake path).

---

## 10. Pricing

Two meters: **Sandbox** and **Models**. Gross margin target **>50%** on Pro after both. If not: raise price or shrink pool.

| Plan | Price (test) | Sandbox | Models (our keys) |
|---|---|---|---|
| **Free** | $0 | Tiny, sleep-fast, hour cap | Tiny pool, **low** tier only |
| **Pro** | **$24/mo** | 1 VM, sleep 15m, 3 running bots | Included pool; low+mid in picker; flagship = overage |
| **Plus** | **$79/mo** | Warmer VM, 10 running bots | Larger pool; all tiers |
| **Team** | **$24/seat/mo** + shared pool | Shared VM(s), roles, audit | Pooled; admin allowlist |

**Overage:** prepaid packs or metered at visible rates (pass-through list + margin, shown before click).  
**BYO:** $0 extra on tokens; VM hours still count.  
**Unlimited bots** in marketing = unlimited configs. Running-bot cap on the same page as the price.

Homepage sentence:

> $24/mo. Cloud computer + models you choose. Claude, GPT, Gemini, Grok. Caps shown. Not unlimited Opus.

---

## 11. Compliance and safety

- **EU AI Act Art. 50** (in force 2 Aug 2026): disclose AI on first interaction; machine-readable mark on generated text (grace to 2 Dec 2026 for systems already on market). Implement as: visible “AI” badge + metadata header on API/exports.
- Telegram `/start` consent: messages go to **our** model providers (named).
- Discord: official bot only.
- Bluesky: `bot` self-label; tagged-only.
- Never sell fake engagement.
- **Moderation** is a named pipeline stage (classifier + blocklist + disclosure check), not a prompt paragraph. Kill switch per bot and global.
- Prompt injection: tool results are data; no side-effect tools in v1 (no follow, mass-DM, arbitrary shell as root, payments).
- Persona publishes go through a diff UI.

---

## 12. Non-goals (v1)

- Logging into the user’s Claude / ChatGPT / Gemini **consumer** account.
- Spoofing Claude Code / Codex OAuth.
- Shipping Tab/Composer clones (proprietary Cursor models).
- Unsolicited X reply farming.
- Fine-tuning.
- GPU training boxes.
- “Unlimited 24/7 flagship” on Pro.

---

## 13. Information architecture (app)

```
[Home]
  Chat          — threads, model picker
  Bots          — list, new, running/stopped, logs
  Groups        — rooms
  Computer      — sandbox status, disk, terminal (Plus)
  Usage         — two meters, BYO keys
  Settings      — workspace, team, billing, connectors
```

Mobile: Chat + bot approvals + meters. Bot *create* can wait for desktop/web.

---

## 14. Success metrics

| Stage | Metric | Target |
|---|---|---|
| Activation | Signup → first model reply | <10 min, >60% |
| Grok Bot parity | Laptop closed, Telegram bot still replies | True on Pro |
| Wedge | % of workspaces that use **≥2 providers** in 7 days | >40% |
| Economics | Gross margin Pro | >50% |
| Safety | Moderation stage on 100% of bot outbound | 100% |
| Cap UX | Hits 100% pool without a billed path (upgrade/BYO) | 0% |

Kill criterion: cannot hold >50% GM at $24 with sleep-on-idle VMs and cheap defaults.

---

## 15. MVP (build this, nothing else)

**In:**

1. Auth + workspace.  
2. One sandbox per workspace (smallest machine, sleep idle, persist `/workspace`).  
3. Model gateway: **two providers minimum** (e.g. OpenRouter low + mid). Picker on chat.  
4. Chat streaming.  
5. Create bot → in-app + Telegram.  
6. One group (**@mention only**).  
7. Two meters + hard stop.  
8. Output moderation + disclosure.  
9. **Turn idempotency** (`turn_id` + `(channel, idempotency_key)`) and **moderation-before-external-post** (`SPECS/turn-protocol.md`).  

**Out:** CLI login, X, Discord, Bluesky, terminal UI, flagship-included-unlimited, Teams SSO, Telegram long-poll-in-VM, group router.

**Demo script:** empty account → chat on model A → switch to model B → bot on Telegram → close laptop → phone still gets a reply.

---

## 16. Roadmap

| Phase | When | Ships |
|---|---|---|
| **0** | now | This spec |
| **1 MVP** | first slice | §15 |
| **2** | | Discord, Bluesky, Groups routing, BYO keys, Plus SKU |
| **3** | | Team seats, audit, always-warm, browser-in-sandbox |
| **4** | gated | X summoned-only + BYO X credits |
| **5** | optional | User-owned runtime (T3-style CLI) as a *second* product mode — not the Grok Bot fight |

---

## 17. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Token burn on agentic bots | Critical | Cheap default, running-bot cap, 80% banner, BYO valve, GM kill switch |
| VM cost if we don’t sleep | Critical | Idle sleep mandatory on Free/Pro; meter hours |
| Cursor adds a picker to Grok Bot | High | BYO + non-Grok default + groups; ship picker day one |
| OpenAI/Anthropic supply or ToS | High | Multi-provider gateway; OpenRouter long tail; our keys not user OAuth |
| Prompt injection / bot incident | High | No side-effect tools; moderation stage; kill switch |
| Trademark GrokRep | High | Rename before marketing |
| X API economics | High | X is phase 4, user pays X, URLs off |
| Wake latency > Grok Bot | Med | Keep-alive on Plus; show “waking…” honestly |

---

## 18. Copy

**Ship**

> Cloud computer + the model you choose.  
> Chat, bots, and groups. Claude, GPT, Gemini, Grok.  
> $24/mo includes a sandbox and a model pool. Caps are on the usage page.  
> Optional: use your own API key on the same computer.

**Never**

> Use your ChatGPT Plus / Claude Pro login.  
> Unlimited Opus.  
> Your own @grok.  
> We reuse your grok.com session.

---

## 19. Open decisions (do not block MVP)

- Final public name.  
- E2B vs Fly Machines vs self-hosted Firecracker (`ARCHITECTURE.md` picks a default).  
- OpenRouter-only in MVP vs Anthropic+OpenAI direct.  
- Exact token pool sizes (set after a 50-user cost probe, not from a guess in this doc).
