# Exceptional product bar

**Status:** Acceptance criteria  
**Date:** 2026-09-16  
**Source:** Technical fix pack applied to PRODUCT_DESIGN / ARCHITECTURE gaps.

## Research canon

- `PRODUCT_DESIGN.md` is the only product truth.  
- `RESEARCH.md` §7 and `WHY_NOBODY_HAS_DONE_IT.md` are historical / rejected paths for consumer OAuth and “use Claude Pro login.” Do not implement those.  
- Every build choice cites an ADR under `DECISIONS/`.

## Pass/fail (v1)

You ship an exceptional v1 when **all** are true:

1. Laptop closed, Telegram bot answers, moderated, disclosure on.  
2. Two providers used in one workspace in the demo script.  
3. Kill VM mid-turn → retry → **no double reply**, disk not corrupt.  
4. Pool hit → clear upgrade/BYO path, zero silent 500s.  
5. Keys never appear in sandbox filesystem or logs (verified by test).  
6. New bot defaults to low tier; flagship is opt-in.  
7. p95 wake and $ burn visible on Usage.  
8. Prompt injection suite in CI is green.  
9. One sandbox vendor behind a `SandboxDriver` interface.  
10. Research-rejected paths cannot be enabled by a feature flag.

## Cap UX

- **80%** tokens: banner + projected hours-left at current burn.  
- **100%** platform tokens: stop platform models; BYO continues if configured; one screen: upgrade / pack / paste key.  
- **100%** sandbox hours: sleep VM; queue up to N inbound then reject with “computer capped” (never silent drop).

## Autonomy defaults

- New bots: **approval queue ON** for any external channel.  
- Auto-reply only after explicit owner toggle (and disclosure + moderation).  
- Kill switch: workspace-global + per-bot, both &lt;2s effect.

## MVP build order (do not reorder)

1. ADR sandbox driver + empty wake/sleep demo  
2. Gateway OpenRouter stream + typed errors + `usage_events`  
3. Auth + workspace + two meters UI  
4. Chat through gateway (no sandbox tools yet)  
5. Daemon + path-jailed files tools + turn state machine + idempotency  
6. Telegram webhook → turn → moderate → reply  
7. Hard caps 80%/100%  
8. Groups + @mention only (no router)  
9. Red-team + wake latency dashboard  

## Observability (Usage page)

- Tokens by bot, model, day  
- Sandbox hours / wake count / p50–p95 wake latency  
- Moderation block rate  
- Failed turns by reason  
- Burn vs pool projection
