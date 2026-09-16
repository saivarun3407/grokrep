# ADR 001 — Sandbox vendor (MVP)

**Status:** Proposed (must close before coding sandbox lifecycle)  
**Date:** 2026-09-16  
**Context:** PRODUCT_DESIGN requires one isolated Linux VM per workspace with pause/resume and `/workspace` snapshots. ARCHITECTURE leaves E2B vs Fly Machines open.

## Decision rule (score, don't vibe)

| Criterion | Weight |
|---|---|
| Pause/resume reliability | 25 |
| Cold wake p95 | 20 |
| Snapshot/restore of `/workspace` | 20 |
| Per-VM network egress policy | 15 |
| Price at sleep + at 2vCPU warm | 10 |
| API maturity / escape hatch | 10 |

Score each vendor 1–5 × weight. Pick the higher total for MVP.

## Ship rules

1. **One provider for MVP.** Do not dual-run.
2. Abstract behind `SandboxDriver`: `create | pause | resume | snapshot | destroy | exec`.
3. Second provider only after **100 paying workspaces** and a new ADR.
4. Measure real wake p50/p95 in staging before marketing SLOs.

## Non-goals

- GPU
- Multi-tenant shared kernel
- Keeping VMs warm on Free/Pro by default

## Consequences

- Control plane owns lifecycle; daemon is an image we bake.
- Telegram must use control-plane webhooks (not long-poll inside a sleeping VM).
