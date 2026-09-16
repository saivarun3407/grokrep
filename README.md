# GrokRep (working name)

**Grok Bot, but you pick the model.**  
Hosted **sandbox VM** + **multi-LLM gateway**. Chat, bots, groups. User pays us; we pay infra and labs.

Not a Claude-Pro login wrapper. Not “your own @grok.” Rename before launch.

## Docs

| File | What |
|---|---|
| [PRODUCT_DESIGN.md](./PRODUCT_DESIGN.md) | Canonical product spec |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Control plane, sandbox, gateway, data model |
| [RESEARCH.md](./RESEARCH.md) | 2026-09-15 research (Grok bot internals, market, legal) |
| [WHY_NOBODY_HAS_DONE_IT.md](./WHY_NOBODY_HAS_DONE_IT.md) | Why BYO-key social bots never launched (background) |

## Product in one diagram

```
User --$--> us --$--> sandbox VM
              └──$--> Claude / GPT / Gemini / Grok (our keys)
```

Grok Bot = cloud computer + **Grok, no picker**.  
We = cloud computer + **per-bot model**.

## Status

Design only. No app code in this repo yet. MVP is `PRODUCT_DESIGN.md` §15.
