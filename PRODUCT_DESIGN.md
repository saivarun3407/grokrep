# PRODUCT_DESIGN.md — GrokRep (working name)

> **One-liner:** Your own @grok — a summonable, persona-driven AI reply bot for your social accounts, powered by your own AI account: an API key from OpenAI, Claude, Grok, or DeepSeek, or one-click connect via OpenRouter.
>
> ⚠️ **Wording rule (legal):** never say "use your ChatGPT/Claude subscription." Consumer chat subscriptions cannot legally power third-party bots — Anthropic's terms prohibit third-party claude.ai login and ban automated access except via API key, and they enforce it (see WHY_NOBODY_HAS_DONE_IT.md §1). The product is BYO **API key**, with OpenRouter OAuth as the no-friction path for non-developers.
>
> Positioning reference: what Cursor did in its early days for coding (bring your own key, we supply the orchestration), applied to grok-style social bots. All claims grounded in `RESEARCH.md` (2026-09-15).

---

## 1. Thesis

Three verified facts make this product possible and timely:

1. **The @grok architecture is reproducible with any model.** It's a static persona prompt plus an agentic tool loop (fetch thread → parallel search → browse-to-verify → view media → short reply). Every frontier model supports tool calling. xAI's real moat is privileged X ingestion — which doesn't exist on any other platform, so on Bluesky/Telegram/Discord the *loop itself* is the product.
2. **Nobody sells this.** The reply-bot market is 100% bundled-credit, zero model choice. The only BYO-model option (ElizaOS) is a developer framework. Self-serve × BYO-model × social bot is an empty quadrant.
3. **BYOK is honest for us in a way it never was for Cursor.** We own no models. Our moat is orchestration, connectors, and trust. Cursor's BYOK fractured the day they shipped custom models; ours never has to.

**What we are NOT building:** an unsolicited engagement-farming reply-guy tool. X outlawed that shape (summoned-only replies since Feb 2026, AI replies need X approval), and the FTC treats fake-engagement tooling as penalty territory. The compliant, durable shape is the @grok shape: **a bot people deliberately summon.**

## 2. Who it's for

- **Creators/brands** who want an interactive presence: followers mention `@mybot` (or DM it on Telegram, or slash-command it on Discord) and get in-character, context-aware answers.
- **Communities** wanting a resident expert bot grounded in live retrieval, not a stale FAQ.
- **Hypefury's ~120k orphaned X accounts** — motivated, currently unserved, shopping for a migration path today.
- **Tinkerers** priced out of credit caps who already pay for a model subscription and want "unlimited, at cost."

Non-technical first. Plain-English UI copy throughout (no "inference," "system prompt," or "webhook" in user-facing text — "your bot's personality," "how it answers," "connect your account").

## 3. Product shape

Creating a bot is a 4-step flow, AI-assisted by default, with live streaming progress at every step:

1. **Personality** — name, voice, what it's for. A guided persona builder (sectioned like the leaked Ani template: likes/dislikes, key phrases, tone, hard limits) with a chat preview pane. AI drafts the persona from a paragraph of description; user edits.
2. **Brain** — pick a provider and paste a key (or click "Connect OpenRouter" for the one-click OAuth path). Model dropdown auto-populated from the provider. Live "test reply" button.
3. **Platforms** — connect Bluesky / Telegram / Discord (X later, see §6). Each connector explains in one sentence what the bot can and can't do there and what it costs (X only).
4. **Rules** — reply length, languages, topics to avoid, autonomy level (auto-post vs. approval queue — default per platform, see §6), disclosure label (on, non-removable).

Then: a dashboard showing every summon, the retrieval steps the bot took (streamed live — "reading the thread… searching… verifying source…"), the reply, and cost meters (model tokens + platform API where applicable). Styling: white + green/blue, consistent with our home-page palette.

## 4. Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              CONTROL PLANE              │
                    │  persona studio · key vault · billing   │
                    │  dashboard · approval queue · audit log │
                    └───────────────┬─────────────────────────┘
                                    │
┌───────────────┐   summon   ┌──────▼──────┐   tools   ┌──────────────────┐
│  PLATFORM     │──events───▶│ BOT RUNTIME │◀─────────▶│  TOOL LAYER      │
│  CONNECTORS   │◀──replies──│ (agent loop)│           │ thread_fetch     │
│ bluesky (jetstream)        └──────┬──────┘           │ platform_search  │
│ telegram (webhook)                │                  │ web_search       │
│ discord (gateway)          ┌──────▼──────┐           │ browse_verify    │
│ x (mentions, summoned-only)│ MODERATION  │           │ view_image       │
└───────────────┘            │ LAYER (owned│           └──────────────────┘
                             │ named stage)│
                             └──────┬──────┘           ┌──────────────────┐
                                    │                  │ PROVIDER ADAPTERS│
                                    └─── final post ◀──│ anthropic (native│
                                                       │  + prompt cache) │
                                                       │ openai · xai     │
                                                       │ deepseek · gemini│
                                                       │ openrouter · any │
                                                       │ OpenAI-compat URL│
                                                       └──────────────────┘
```

### 4.1 The agent loop (mirrors @grok, verified against its prompt)
Per summon: static persona prompt (never per-turn templated context) → model runs tools: `thread_fetch` (parent + quoted posts), `platform_search`, `web_search` (parallel, diverse viewpoints), `browse_verify` (mandatory before asserting a searched fact), `view_image` → drafts reply under a per-platform char budget → moderation layer → post (or approval queue).

Rules inherited from @grok's prompt because they're battle-tested: no markdown in replies; match the language/dialect of the summoning post; never tag the summoner explicitly; **never condition on the bot's own prior posts** (the one rule that survived every xAI incident); reply at the invocation point.

### 4.2 Moderation layer — owned, named, tested (the central Grok lesson)
All three 2025 @grok incidents trace to unreviewed prompt/config changes and a silently deleted moderation stage. So:
- Moderation is a **separate pipeline stage** with its own tests, not a prompt paragraph. Checks: hate/harassment classifier, platform-policy rules (per-connector), user's own topic blocklist, URL policy (see §7), disclosure marker present.
- Persona/prompt changes are **versioned with diffs** shown to the user; our own default-prompt changes go through review + changelog.
- Kill switch per bot; global pause per platform.

### 4.3 Prompt-injection defense (the unclosable surface, managed)
Everything the bot retrieves is attacker-writable ($150k was drained from a wallet-connected bot via Morse code in a reply). Mitigations: retrieved content is data, never instructions (structured tool results, injection-pattern stripping); **no side-effectful tools** in v1 (the bot can read and post one reply — it cannot follow, DM, transact, or browse arbitrary user-supplied URLs beyond the verify step's allowlisted fetch); moderation layer runs on output regardless of what the input claimed.

### 4.4 Provider adapter layer
- **Native adapters per provider** (Vercel AI SDK pattern), not a lowest-common-denominator OpenAI shim — because Anthropic's prompt caching (the single biggest LLM cost lever for a static-prompt bot) is lost through its compat shim. Cache the persona + tool definitions; only the summon context is fresh tokens.
- Escape hatch: any OpenAI-compatible base URL (covers DeepSeek, Groq, Mistral, local, and whatever ships next).
- OpenRouter as the **zero-friction default**: its OAuth PKCE flow is the only provider-sanctioned way to mint a user's own key in-browser — "Connect" button, no key pasting, no key ever typed into our UI.
- Model IDs fetched live from providers, never hardcoded (model names rot — verified twice in this research).

### 4.5 Key custody — the trust product
A 24/7 bot must hold keys server-side (summons arrive while the user sleeps). Competitors' claims here don't survive code review (Open WebUI stores "browser-only" keys in plaintext DB; LibreChat encrypts with a fixed global IV). We do it right and **publish the design**:
- Envelope encryption: per-user data key (AES-256-GCM, authenticated — no fixed IVs) wrapped by a KMS master key. Keys decrypt only in the runtime worker, held in memory per-request.
- User-chosen TTL (LibreChat's best idea: 30 days / 90 days / until revoked) + one-click revoke-all; keys never in logs; last-4 display only.
- Self-host tier reads keys from env/OS keyring and our servers never see them (big-AGI's transient pattern).
- Plain-English key page: "Your key is stored encrypted, used only to run your bot, never to bill you, and you can delete it any time. Your conversations go to your AI provider under *their* privacy policy, not ours." (Say the privacy inversion out loud — Cursor lesson #6.)

## 5. Compliance by design (non-negotiable defaults)
- **Disclosure everywhere, non-removable:** bot label in profile/bio on every platform (mandatory on X, voluntary-but-default elsewhere), "AI-generated" machine-readable marker on output (EU AI Act Art. 50, in force since Aug 2026), first-interaction disclosure. CA SB 1001 safe harbor comes free with this.
- **Never market engagement inflation.** No auto-likes, auto-follows, mass-DM — the endpoints barely exist anymore and the FTC penalty exposure is direct.
- **Telegram:** consent-gated ("this bot uses AI; your messages are sent to [provider] to generate answers") shown on `/start`; no group-content harvesting, ever (ToS ban on AI-dataset collection).
- **Discord:** registered bot application only; never a self-bot path.

## 6. Platform strategy (order dictated by policy, not preference)

| Phase | Platform | Connection | Autonomy default | Why |
|---|---|---|---|---|
| **1 — launch** | Bluesky | Official API via app-password/OAuth; Jetstream for mention events | **Auto-post** | Free API, 11k posts/day headroom, no bot rules, zero no-code competitors. The empty intersection. |
| **1 — launch** | Telegram | Bot API webhook (BotFather-registered by us, one click for user) | **Auto-post** (opt-in by architecture) | First-class bots, 30 msg/sec free, structurally can't be unsolicited. |
| **2** | Discord | Registered application, slash command + @mention | **Auto-post** | Sanctioned, opt-in; crowded but our BYO-model + retrieval loop is differentiated. |
| **3 — gated** | X | **User's own X developer account** (BYO X-API keys, ElizaOS-style), summoned-only mode, "Automated" label enforced | **Approval queue** default; auto only after user confirms X approval | The only compliant shape. User pays X's per-use costs ($0.01/summoned post) directly — we pass through zero platform cost, same BYO philosophy. **Blocked until we resolve the AI-reply approval rule in the X dev console (RESEARCH.md §8).** |
| Skip | Farcaster | — | — | Platform in collapse; revisit if ownership stabilizes. |

Approval queue everywhere as a user-selectable mode ("review before posting") — it's also our answer for cautious brands.

## 7. Business model (Cursor's lessons, applied in reverse)

**Subscription for orchestration; inference is never ours to sell.**

- **Free:** 1 bot, 1 platform (Bluesky), BYO key, approval-queue mode, disclosure label. Top-of-funnel, near-zero marginal cost to us.
- **Pro ~$15/mo:** 3 bots, all platforms, auto-post mode, persona studio, analytics, priority summon processing. Undercuts the $29–49 incumbent band *while removing their reply caps* — "unlimited replies, you pay your model provider at cost" is the wedge.
- **Team ~$49/mo:** shared bots, roles, audit log, approval workflows.
- **Self-host (open core):** runtime + connectors under Apache-2.0/MIT (no GPL/AGPL dependencies anywhere in the stack — commercial constraint; also why we build clean-room rather than forking Chatbox CE or Open WebUI, whose licenses bite); hosted control plane, persona studio, and team features are the paid layer. Branding stays (Open WebUI's clause shows the norm).

**Pricing truths we commit to on day one (each one is a Cursor scar):**
1. BYOK covers *everything* — there is no feature your key can't power. If that ever changes, the seam gets labeled on the pricing page the same day.
2. "At cost" never implies "free" — the subscription is for orchestration and is stated next to the BYOK offer in the same sentence.
3. One billing unit, live balance visible: bots + platforms. Never token-credits.
4. If we ever add a bundled-inference convenience tier, it's OpenRouter-backed pass-through with visible margin, added alongside BYOK — never replacing it.
5. **URLs in X replies are blocked by default** ($0.20/post URL tax — a 13× cost multiplier the user would eat); toggleable with a plain-English cost warning. On other platforms, links are fine.

**Cost model honesty:** on X, platform API cost dwarfs model cost (~$15/1k replies vs. ~$0.09–$2.80/1k) — which is exactly why user-pays-platform (BYO X keys) is the only structure that survives Hypefury's fate. On Bluesky/Telegram/Discord, platform cost is zero and the model bill goes straight to the user's provider. Our COGS ≈ hosting + retrieval infra, priced into the subscription.

## 8. MVP scope (phase 1)

**In:** Bluesky + Telegram connectors · persona studio with AI-drafted personas + live preview · provider adapters: Anthropic (native, cached), OpenAI, xAI, DeepSeek, OpenRouter (OAuth), custom base URL · agent loop with thread_fetch / platform_search / web_search / browse_verify / view_image · moderation layer v1 (classifier + blocklists + disclosure marker) · key vault with TTL + revocation · dashboard with live-streamed retrieval steps · approval queue · machine-readable AI marking.

**Out (deliberately):** X connector (gated on approval-rule verification) · Discord (phase 2) · voice/avatars (Companions were retired for a reason; text first) · any side-effectful tools (follow/DM/transact) · bundled inference credits · fine-tuning.

**Success test:** a non-technical user goes from signup → live summonable Bluesky bot on their own Claude subscription in under 10 minutes, and every reply survives our own red-team injection suite.

## 9. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Bluesky adopts X-like bot rules (policy vacuum closes) | High | Multi-platform from day 1; disclosure-by-default already exceeds any likely rule; approval-queue mode ready |
| X approval rule blocks the X connector entirely | Med | X is phase 3 and gated; product stands on Bluesky/Telegram/Discord without it |
| Prompt injection incident on a customer bot | High | §4.3 (no side-effectful tools, output moderation, structured retrieval); red-team suite in CI; incident kill switch |
| Provider revokes undocumented CORS / changes key policy | Low (server-side custody is the primary path) | Browser-direct is only used for key-validation UX; runtime is server-side regardless |
| Incumbent (Typefully/Buffer) adds BYOK | Med | They'd cannibalize their credit margin (innovator's dilemma); our retrieval loop + persona depth is the second moat |
| A model provider ships a native bot product | Med | They'll be single-model by construction — model-agnosticism is precisely what they can't copy |
| AI-content backlash on Bluesky (reputational) | Med | Summoned-only default, hard disclosure, quality bar via browse-verify — position as "answers when asked," never ambient spam |
| BYOK onboarding friction — non-developers don't have API keys (the reason this category stayed empty) | High | OpenRouter OAuth PKCE as the default "Connect" path: mints the user's own key in two clicks, no developer signup; raw key-pasting is the power-user path, not the main flow |
| Provider subscription-auth enforcement (Anthropic-style fingerprinting/legal action) | Low for us | We never touch consumer-subscription auth — API keys only; keep all copy free of "use your subscription" claims |

## 10. Why we win (one paragraph)

Every competitor either bundles a hidden model behind credit caps (SaaS reply tools), requires an engineering team (ElizaOS), or can't touch social platforms at all (no-code chatbot builders). We ship the @grok architecture — verified against xAI's own published prompt — as a consumer product, on the platforms where it's actually legal, powered by the AI subscription the user already has, with key custody we publish and competitors demonstrably fake. Cursor proved BYOK builds a beachhead when your moat is orchestration; unlike Cursor, our moat never stops being orchestration.
