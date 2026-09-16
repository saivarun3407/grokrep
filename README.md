# GrokRep (working name)

**Grok Bot, but you pick the model.**  
Hosted **sandbox VM** + **multi-LLM gateway**. Chat, bots, groups. User pays us; we pay infra and labs.

Not a Claude-Pro login wrapper. Not “your own @grok.” Rename before launch.

## Docs

| File | What |
|---|---|
| [PRODUCT_DESIGN.md](./PRODUCT_DESIGN.md) | Canonical product spec |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Control plane, sandbox, gateway, data model |
| [SPECS/turn-protocol.md](./SPECS/turn-protocol.md) | Build law: turn + bot state machines, idempotency |
| [SPECS/tool-policy.md](./SPECS/tool-policy.md) | Build law: path jail, tools, egress |
| [SPECS/exceptional-bar.md](./SPECS/exceptional-bar.md) | Acceptance criteria + MVP build order |
| [DECISIONS/](./DECISIONS/) | ADRs (sandbox vendor, …) |
| [RESEARCH.md](./RESEARCH.md) | Historical research (not implementation truth) |
| [WHY_NOBODY_HAS_DONE_IT.md](./WHY_NOBODY_HAS_DONE_IT.md) | Historical / rejected BYO-social path |

Build law for engineering lives in **SPECS/** and **DECISIONS/**. Product truth is **PRODUCT_DESIGN.md**.

## Product in one diagram

```
User --$--> us --$--> sandbox VM
              └──$--> Claude / GPT / Gemini / Grok (our keys)
```

Grok Bot = cloud computer + **Grok, no picker**.  
We = cloud computer + **per-bot model**.

## Status

Design only. No app code in this repo yet. MVP is `PRODUCT_DESIGN.md` §15.
