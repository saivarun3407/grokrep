# RESEARCH.md — How the Grok Bot Works, and What the Market Looks Like

> **Product note (2026-09-15):** Canonical product is `PRODUCT_DESIGN.md` v2 (hosted sandbox + multi-LLM gateway, Grok Bot competitor). This file is the research pass that informed v1 (BYO-key social bot) and still holds for @grok’s loop, X policy, Cursor BYOK, and legal walls on consumer-subscription OAuth.

> Deep research synthesis, 2026-09-15. Compiled from seven parallel research agents (primary-source fetches, live CORS probes, repo forensics on `xai-org/grok-prompts`, Wayback archaeology on Cursor, and competitor teardowns). Confidence tags: [HIGH] = primary source fetched/verified, [MED] = credible secondary or partially verified, [LOW]/[UNVERIFIED] = flagged, do not build on without re-checking.

---

## 1. Executive summary

- **@grok on X is an agentic retrieval loop, not a templated summarizer.** Its published system prompt contains zero template variables — thread context is *fetched by the model itself* via tools at inference time (thread fetch, search, browse-to-verify, image viewing). This architecture is fully reproducible with any tool-calling model (GPT, Claude, Grok, DeepSeek, Gemini). [HIGH]
- **Nobody in the reply-bot market offers bring-your-own-key.** Every competitor sells opaque bundled credits. The BYO-model quadrant is empty. [HIGH]
- **X has effectively outlawed unsolicited auto-reply bots** (summoned-only replies since Feb 2026, AI replies require prior X approval, pay-per-use pricing, no free tier). The *only* compliant shape on X is exactly the @grok shape: summoned, labeled, one reply per invocation. [HIGH]
- **Bluesky is wide open**: free API, generous limits, no bot-disclosure rules, and zero no-code AI bot builders serve it. Telegram and Discord are bot-friendly and opt-in by architecture. Farcaster is unserved but dying. [HIGH]
- **Every major LLM provider permits direct browser calls today** (empirically verified), so BYOK can be architected with minimal key custody — but a 24/7 bot needs server-side execution, so key handling design is the trust product. Three of six BYOK competitors examined have gaps between their security claims and their code. [HIGH]
- **Cursor's BYO-key lesson**: BYOK works cleanly only when your moat is orchestration, not owned models. Cursor's BYOK broke the moment they shipped custom models (April 2024). A bot platform that owns no models can offer BYOK honestly and completely. [HIGH]

---

## 2. How @grok actually works

Source: `xai-org/grok-prompts` repo (cloned, full git history traversed), xAI docs, academic studies, incident reporting.

### 2.1 The load-bearing finding
`ask_grok_system_prompt.j2` — xAI's published prompt for the @grok bot — has **no Jinja variables** (3,036 chars, verified mechanically). Compare `grok_analyze_button.j2`, which templates `{{ url }}` etc. Implication: thread context is **not injected**. The bot receives a static persona/policy block and must fetch the parent post, quoted posts, and images itself via tool calls. [HIGH]

The predecessor `ask_grok_summarizer.j2` (Grok-3 era) *was* templated (`{{user_query}}`, `{{response}}`) — a two-stage analyze→summarize pipeline. xAI deleted it 2025-07-06 in the same commit that added the agentic prompt. The architecture visibly migrated from pipeline → single agentic loop. [HIGH]

### 2.2 The retrieval contract (from the prompt, verbatim obligations)
- "You have access to real-time search tools… Parallel search should be used to find diverse viewpoints."
- "Use your X tools to get context on the current thread."
- "Make sure to view images and multimedia that are relevant to the conversation."
- "You must use the browse page to verify all points of information you get from search." — a mandatory second hop trading latency for grounding. [HIGH]

Tool surface (xAI API, third-party-visible analog): consolidated `x_search` tool with `allowed_x_handles`/`excluded_x_handles` (max 20), date bounds, `enable_image_understanding` (equips `view_image`), `enable_video_understanding`. Thread fetch bills parent AND quoted posts: **$5 per 1k posts fetched, $10 per 1k profiles** (effective Sept 21, 2026). [HIGH for API; the first-party bot's internal tools are related but not proven identical.]

### 2.3 Output constraints (from the prompt)
- Reply **under 550 characters** (tuned upward through 2025: 400 → 450 → 550). [HIGH]
- **No markdown.** No tagging the person replied to (native in-thread reply). [HIGH]
- **Match language, regional/hybrid dialect, and alphabet** of the post being replied to. [HIGH]
- "Responses must stem from your independent analysis, not from any beliefs stated in past Grok posts or by Elon Musk or xAI" — survived every prompt revision since 2025-07-07. [HIGH]
- Post-MechaHitler (2025-08-18 commit): nine explicit negative constraints — no moralizing, no "facts over feelings," no calling views "biased"/"baseless," no political slogans, no single-study reliance. [HIGH]

### 2.4 Ingestion (undocumented — inference flagged)
Nothing about the mention-ingestion path is public. [HIGH on the negative] Since xAI owns X, the plausible path is internal event fan-out, not the public API [MED, inference]. The three public options (filtered stream, mentions polling at 450 req/15min, Account Activity webhooks — deprecated March 2026) are each individually inadequate for @grok's observed latency/volume — which is itself the strongest evidence of privileged ingestion.

### 2.5 Observed behavior (academic, arXiv-verified)
- **62% reply rate** — @grok answers only ~62% of summons ("Grok in the Wild," arXiv 2602.11286, 41,735 interactions). Half of replies get ≤20 views in 48h. 51% of requests in English. [HIGH]
- 76.8% of users summon exactly once; invocations are reactive and don't predict later Community Notes activity — the bot functions as *private sensemaking performed in public*, not distributed fact-checking (arXiv 2605.19720, 169k invoking posts). [HIGH]
- Latency self-reported 2–6 min typical. [MED]
- No published rate limits; every circulating number traces to SEO farms or @grok self-replies. [HIGH on the negative]

### 2.6 Strategic reads
1. **The static prompt is a cost architecture.** Every invocation pays for live retrieval; at $5/1k posts fetched, platform-scale replies carry real marginal cost. The 62% reply rate and 550-char cap are cost controls wearing editorial masks.
2. **The prompt is the incident-response surface.** Both 2025 failures (white-genocide injection, MechaHitler) were remediated by prompt edits + publishing the prompt. The published file is now ~13 months stale — the transparency commitment has decayed.
3. **Retrieval-as-context is an unclosable injection surface.** A documented $150k wallet drain via Morse-code instructions in a reply Grok decoded and obeyed. Prompt rules govern tone; none govern whose instructions to obey. Any @grok-style product must treat fetched thread text as hostile input.

### 2.7 Grok app Companions (secondary; retired)
Ani/Rudi/Valentine launched 2025-07-14 in the Grok app; avatar layer ("Ani-2" model, Animation Inc) ran decoupled from LLM inference; persona prompts leaked via grok.com misconfiguration (Aug 2025), showing per-turn re-rendered prompts with live app state and animation triggers as pseudo-tools. Retired ~Sept 2026; characters migrated to Animation Inc's own Animates app. [MED-HIGH] Relevant design contrast: @grok = stateless, tool-heavy, epistemically constrained, public; Companions = stateful, render-coupled, immersion-first, private. Same base model, inverted prompt pressure.

---

## 3. Feasibility: BYO-key without custody

Live CORS probes run 2026-09-15 against every major provider (preflight + real POST with foreign Origin):

| Provider | Browser-direct works? | Mechanism | Documented? |
|---|---|---|---|
| Anthropic | Yes (opt-in) | `anthropic-dangerous-direct-browser-access: true` header | Yes — best in class |
| OpenAI | **Yes, no gate** | ACAO returned unconditionally (contradicts common belief) | No |
| xAI / Grok | Yes | `ACAO: *` everything | No |
| DeepSeek | Yes | Reflects origin + credentials | No |
| Google Gemini | Yes | Reflects origin (native + OpenAI-compat) | No (pushes ephemeral tokens, Live-API-only) |
| OpenRouter | Yes — purpose-built | `ACAO: *` + curated header allowlist + **OAuth PKCE** | Partially — the only sanctioned general BYOK browser flow |
| Groq / Mistral | Yes | `ACAO: *` | No |

Takeaways: the "you need a backend proxy" convention is a security norm, not a technical constraint. Six of eight providers support this **undocumented** (revocable without notice). OpenRouter's PKCE flow is the one first-party-sanctioned pattern for minting a user's own key client-side. For a 24/7 bot, keys must still rest server-side (summons arrive while the user is offline) — see §4 for how competitors handle that, mostly badly.

---

## 4. Competitor landscape

### 4.1 BYOK chat clients (the adjacent market that proves BYOK demand)
| | Key at rest | Keys touch vendor servers? | License | Notes |
|---|---|---|---|---|
| **Jan** (44k★) | **OS keychain** (≥0.8.4) | No | Apache 2.0 | Cleanest; caveat: pre-migration plaintext snapshot retained on disk |
| **TypingMind** | Browser localStorage + optional AES | **Yes** — via proxy, and always in Teams | Closed | "Never touches our servers" claim breaks under its own proxy |
| **Chatbox** (42k★) | On-device, mechanism undocumented | No (BYOK mode) | GPL-3.0 CE + closed pro | Weakest evidentiary position |
| **Open WebUI** (152k★) | **Server DB, plaintext** — docs claim localStorage | Yes (stored) | Modified BSD-3 w/ branding clause (≥v0.6.5) | **Docs contradict code** — per-user keys POSTed to backend, persisted unencrypted (verified in source) |
| **big-AGI** (7k★) | Browser localStorage; server sees key per-request, transient, never persisted | Transiently | MIT | **The clean pattern to copy**: `access.oaiKey \|\| env.KEY \|\| ''` |
| **LibreChat** (43k★) | MongoDB, encrypted — but **AES-CBC with a fixed global IV** (legacy v1 path), not the AES-256-CTR in its own codebase | Yes (stored + proxied) | MIT | Best server-side BYOK UX: per-user isolation, user-chosen TTL (30min–30d/never), Mongo TTL enforcement, revocation |

**Lesson: key-handling honesty is a differentiator.** Three of six have claim/code gaps. LibreChat's TTL + revocation UX is the model for server-side custody; big-AGI's transient pattern for anything that can run client-side.

### 4.2 X reply/engagement tools
- **Nobody offers BYO-key or model choice. Nobody.** All bundle opaque credits. [HIGH]
- **Hypefury (category leader) quit X entirely ~Aug 2026** — reportedly ~$1.4M/yr in per-connected-account API fees (120k accounts × ~$1/mo) [MED on figures]. Its acquired analytics tool BlackMagic shut down July 2026. **120k+ orphaned X accounts are actively shopping for migration paths.**
- Survivors picked one of three connection models: **official OAuth** (Tweet Hunter, $29–199/mo — real AI replies gated at Enterprise $199), **browser extension** (ReplyGuy/appendment, Replai, Tweetback, most 2026 entrants — cheap, ToS-grey, human-in-loop by architecture), or **session cookies** (Xreply, $29.99–79.99/mo — fully automated, most ban-exposed).
- **ElizaOS** (19.3k★, TypeScript, beta) is the only BYO-model option — and it extends BYO to the platform side too (user brings their own X developer API keys). Developer-only; docs never state which X tier is needed; April 2026 token class-action clouds it. **The gap: ElizaOS autonomy with consumer onboarding.**

### 4.3 No-code bot builders (cross-platform)
Botpress ($0–750/mo, bundled, forced auto-recharge), Chatbase ($0–500/mo, bundled), Typebot, Voiceflow (strong BYO posture but sales-gated), Make (BYO LLM key, beta), n8n (BYO by architecture, self-hostable), Zapier Agents. **None target X, Bluesky, or Farcaster for AI replies.** Zapier's beta Premium Bluesky app (post/watch only, no AI reply) is the entire no-code coverage of decentralized social. **The intersection BYO-key + AI reply + Bluesky is completely empty.**

---

## 5. Platform policy & economics

### 5.1 X — hostile to the point of prescribing the product shape [HIGH, primary-fetched]
- **Pricing**: tiers gone; pay-per-use only, no free tier. Post read $0.005 · post creation $0.015 · **post containing a URL $0.200** · **summoned post $0.010** · user read $0.010. Cap: 3M reads/mo, then Enterprise. Rewards: 10–20% of X API spend back as xAI credits at $200+/mo.
- **Feb 23, 2026**: "Programmatic replies now restricted to cases where the original Post's author has summoned the replier." Enforced architecturally (it's a billing line item).
- Developer Guidelines: replies only if user engaged first, max 1 reply/interaction; keyword-triggered auto-replies explicitly banned; **"AI-powered app generates and posts replies" requires prior approval from X**; mandatory "Automated" profile label + bot bio + linked human account.
- Apr 16, 2026: likes/follows/quote-posts endpoints **removed** from self-serve.
- Rate limits: POST /2/tweets 100/15min per user; mentions 300/15min.
- **Net: the summoned-bot model (@grok's own shape) is the only compliant path on X.**

### 5.2 Bluesky — most permissive [HIGH]
Free API. No bot/automation clauses in ToS or Community Guidelines; no disclosure requirement (only generic spam/manipulation/impersonation rules). Write limits: 5,000 pts/hr, 35,000 pts/day → **max 1,666 posts/hr, 11,666/day** per account. Caveats: policy vacuum ≠ permission grant, docs mid-migration (docs.bsky.app → bsky.network), community hostility to AI content is a reputational risk. Official SDKs very active; bot-framework layer thin (@skyware/bot: one maintainer, ~6 months since release).

### 5.3 Telegram — moderate [HIGH]
Bots first-class, opt-in by architecture (can't message users who haven't started them). 30 msg/sec free; 1,000/sec purchasable (0.1 Stars/msg over free, 100k Stars + 100k MAU gates). Two landmines: (1) ban on scraping group/channel content "aimed at creating large datasets, machine learning models and AI products"; (2) Telegram Business clause forbids disclosing message contents to third parties **including third-party APIs** without user authorization — squarely covers shipping messages to an LLM. Consent design required.

### 5.4 Discord — crowded but sanctioned [HIGH on ToS, MED on developer policy — 403-blocked]
Registered bot applications fully sanctioned and opt-in (bots act only where installed). Self-bots/auto-messaging of user accounts banned. Most crowded ecosystem; anomaly: BotGhost's pricing pages 404.

### 5.5 Farcaster — skip [HIGH on acquisition, LOW on figures]
Acquired by Neynar Jan 2026; founders stepped back; revenue reportedly down ~85%; owner reportedly shopping it. Excellent bot infra (Neynar: free tier 10M credits/mo, 1-click agent accounts), zero no-code competition — but an empty room, not an empty market.

### 5.6 Legal floor (all platforms) [HIGH]
- **EU AI Act Art. 50** — in force since Aug 2, 2026: first-interaction AI disclosure + **machine-readable marking of generated text**.
- **California SB 1001** — bot disclosure safe harbor (commercial/electoral intent, platforms ≥10M US visitors).
- **FTC fake-reviews rule** (Aug 2024) — selling "fake indicators of social media influence" = civil penalties; any engagement-inflation positioning is radioactive.
- Default posture: **always label the bot, everywhere; mark generated output machine-readably.**

---

## 6. Cursor's BYO-key history — the business-model reference

Wayback-verified arc (all pricing pages fetched at archived URLs):

| Phase | Date | State |
|---|---|---|
| Headline feature | May 2023 | "Enter your OpenAI API key to use Cursor **completely at-cost**" — marketed on pricing page |
| Hedged | Apr 2024 | Custom models ship (Copilot++/Tab) → "a few features…cannot be charged to an API key" FAQ appears |
| De-marketed | Jun 2024 | BYOK removed from pricing page entirely — **12 months before the pricing crisis** |
| Docs footnote | 2025 | Restriction FAQ verbatim-unchanged through the June 2025 pricing upheaval |
| Taxed | 2026 | **Cursor Token Rate: $0.25/M tokens on BYOK usage** (Teams/Enterprise); first-party models exempt |

Corrections to the popular narrative: BYOK was crippled in **April 2024** (custom models), not 2025 (margin); it was **never removed**, only de-marketed; the 2026 toll is the real change nobody discusses. Also: Cursor's BYOK was never direct-to-provider — keys transited Cursor's backend on every request, and **BYOK voided Cursor's zero-data-retention guarantee** (privacy inversion).

**Lessons extracted:**
1. Decide where the moat lives first — BYOK follows. Orchestration moat → BYOK is cheap and honest. Owned models → BYOK fractures the product forever.
2. A partial BYOK is worse than none unless the seam is labeled brutally, on day one, on the pricing page.
3. BYOK is a top-of-funnel instrument; de-market it, don't remove it, when strategy shifts.
4. Never let "at-cost" imply "free product."
5. If you migrate metering, change one variable at a time and show a live balance in the billing unit.
6. Say the privacy inversion out loud — it converts security buyers.

---

## 7. Synthesis — the gap this product occupies

**The empty quadrant:** self-serve (non-developer) × BYO-model × autonomous-capable social AI bot. Incumbents are either developer frameworks (ElizaOS) or credit-bundled human-in-loop drafting tools. Nobody lets a user say: *my bot, my persona, my Claude/GPT/Grok/DeepSeek subscription, my platforms.*

**The architecture is commoditized; the orchestration isn't.** @grok's design — static persona + agentic tool loop (thread fetch → parallel search → browse-verify → view media → ≤550-char reply) — runs on any tool-calling model. xAI's moat is X ingestion privilege, not the loop. On every platform except X, that privilege doesn't exist, so the loop itself is the product.

**Platform order is dictated by policy, not preference:** Bluesky first (free, permissive, zero competition), Telegram + Discord second (opt-in by architecture), X in summoned-only compliant mode with BYO X-API keys (pushing X's pay-per-use costs and approval process to the user, ElizaOS-style — no consumer product does this), Farcaster never (for now).

**Trust is the wedge.** Competitors' key-handling claims don't survive code review. A product that publishes its key-custody design, encrypts honestly, offers TTL + revocation, and never resells inference has a story none of them can tell.

**Cost story beats credit caps.** "Unlimited replies — you pay your model provider directly" against "500 replies/mo for $49" is the same wedge early Cursor ran against Copilot.

**Unit economics inversion [HIGH].** For an X bot, platform API cost dwarfs LLM cost: posting is $15/1k replies ($200/1k if the reply contains a URL) plus reads at $0.005 each, vs. roughly $0.09–$2.80/1k replies of LLM inference depending on model. Consequences: (1) model-agnosticism is a trust/lock-in feature, not a cost lever; (2) price the product off platform API cost, not tokens; (3) **ban or paywall URLs in generated X replies**; (4) prompt caching is where reply-bot LLM economics actually live — so Anthropic must be integrated natively (its OpenAI-compat shim drops caching), which argues for per-provider native adapters over a lowest-common-denominator shim.

**Moderation is an owned output layer, not a prompt paragraph [HIGH].** All three 2025 @grok incidents share one root cause: unreviewed write access to the production prompt/config. The July 2025 refactor that made @grok agentic also silently deleted the "balanced and neutral" output-moderation stage — right before MechaHitler. xAI's hate-speech classification was retrofitted after the fact. Design lessons: moderation must be a named, independently tested pipeline stage; prompt changes need review gates; and the bot must never condition on its own prior public posts (a rule @grok's prompt still carries).

**Stale-API corrections [HIGH]:** xAI's Live Search API was retired Jan 2026 (410 Gone) — the current path is Agent Tools (`x_search`, `web_search`); X Search reprices Sept 21, 2026 to per-post-fetched (parent + quoted posts counted); `grok-4`/`grok-4-fast` and `deepseek-chat`/`deepseek-reasoner` model names are retired — verify current model IDs at build time.

---

## 8. Open items / unverified (re-check before building on them)
- **X's summoned-only + prior-approval rule for AI replies**: fetched at [HIGH] from docs.x.com changelog and Developer Guidelines, but help.x.com (the policy page of record) is bot-blocked and the research coordinator flags the combined rule as CONFLICT-level pending primary confirmation. It gates the entire X strategy — **resolve via the X developer console before building anything for X.**
- X legacy-tier migration dates; Enterprise pricing (~$42k/mo [MED]).
- Hypefury's exact $1/account figure (primary Veys post not fetched).
- Discord Developer Policy AI clauses (403-blocked — needs manual browser check).
- AT Protocol current rate-limit page (docs mid-migration; figures from repo source).
- Bluesky current DAU; whether big-AGI Pro Sync replicates keys; LibreChat paid tier existence.
- Contradictions-verifier pass never returned — explorer confidence tags are un-cross-checked by that layer (a Perplexity claim-table verification did run on the @grok material).
- All CORS behavior is a 2026-09-15 snapshot; six of eight providers support it undocumented and revocable.

## 9. Source artifacts
- `xai-org/grok-prompts` clone with full git history (incl. deleted prompt files) + leaked persona archive: session scratchpad (`/private/tmp/claude-502/-Users-saivarun-Desktop-permitsai/192cae44-ae48-49d0-a7e9-3251b6df5653/scratchpad/`).
- Key permalinks: current @grok prompt `github.com/xai-org/grok-prompts/blob/31f21d9.../ask_grok_system_prompt.j2`; deleted two-stage summarizer `.../6c96965.../ask_grok_summarizer.j2`; xAI tools docs `docs.x.ai/developers/tools/x-search`; X pricing `docs.x.com/x-api/getting-started/pricing`; X changelog `docs.x.com/changelog`; Cursor archive set (10 Wayback URLs, all HTTP-200-verified, listed in the Cursor report).
- Full agent reports preserved in session task outputs under `/private/tmp/claude-502/-Users-saivarun-Desktop-permitsai/192cae44-ae48-49d0-a7e9-3251b6df5653/tasks/`.
