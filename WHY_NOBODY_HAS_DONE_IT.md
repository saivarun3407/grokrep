# Why Nobody Has Done It

> **HISTORICAL / REJECTED PATH — not build truth.**  
> Explains why BYO-key *social* bots stayed empty and why we **reject** logging into a user’s Claude Pro / ChatGPT. **Do not implement those.**  
> Ship **`PRODUCT_DESIGN.md` (v2)** only. See `SPECS/exceptional-bar.md`.

## 0. The headline: the category was never born, not killed

There is no graveyard of failed BYO-key social bot startups — because there were no launches. GitHub, name+description search: `"bring your own key social media bot"` → **0 repos**. The entire `byok twitter` universe is **6 repos, the most-starred has 4 stars**, all 2026-vintage hobby projects, none hosted, none monetized. [HIGH] So "nobody has done it" is literally true, and the interesting question is what stopped anyone from starting. Seven walls, ranked:

## 1. The customer-overlap problem (the deepest reason)

The people who *have* API keys are developers — and developers don't need a product (ElizaOS, n8n, a weekend of Claude Code). The people who'd *pay* for a no-code product don't have API keys — they have ChatGPT Plus or Claude Pro subscriptions. **And "BYO subscription" is explicitly prohibited:**

- Anthropic Agent SDK docs, verbatim: *"Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products… Use the API key authentication methods instead."* [HIGH]
- Anthropic Consumer Terms ban automated access *"except when you are accessing our Services via an Anthropic API Key."* A social bot is definitionally automated access. Prohibited twice over. [HIGH]
- Enforced, not theoretical: request-fingerprinting against OpenCode (Jan 2026), legal action (Mar 2026), partial relent scoped to *personal local* use only (Apr 2026). Commercial products must use API keys. [MED-HIGH]
- OpenAI's third-party stance unverified (policy page 403'd) — do not assume it's looser. [LOW]

Cursor's early BYOK worked because its audience — developers — already had OpenAI keys. The social-bot audience mostly doesn't. That asymmetry, more than anything, explains the empty quadrant.

**Our answer (updated after T3/Hermes teardown):** do not pretend OpenRouter is ChatGPT Plus. The sentence “use the plan you already pay for” is real in two shapes only: (A) T3-style — spawn the **unmodified official CLI** the user already logged into (`claude auth login`, `codex login`, `grok login`); tokens never enter our vault. (B) Hermes-style — **lab-listed partner OAuth** (xAI actually published Hermes). Hosted “Sign in with Claude” is still a cease-and-desist. OpenRouter/Poe stay as a labeled extra bill, not the headline. Product copy may say the sentence **if** onboarding is “on this computer / this VPS.”

## 2. BYOK never solved the cost that actually binds (on X)

Platform API cost dwarfs model cost on X ($15/1k replies + $0.005/read + $0.20/URL-post vs ~$0.09–2.80/1k replies of inference). Reads dominate — a bot must read to find its summons. BYO-LLM-key gives a vendor zero relief there: either eat X costs as COGS (recreating exactly the bundled-credit economics that killed Hypefury at ~$1.4M/yr), or erect a *second* credential wall ("bring your own X developer account + prepaid credits") that is far worse friction than an OpenAI key. [HIGH] So on the platform where the audience was, BYOK was never the unlock — which is why nobody bothered.

**Our answer:** launch where platform cost is zero (Bluesky, Telegram, Discord); offer X only as a premium BYO-X-credits mode for users who clear X's own wall themselves.

## 3. Platform risk caused a visible mass extinction, then a lockout

Commit-data fact: of the 10 most-starred archived Twitter-bot repos, **six stop dead in 2023** — the API-paywall extinction event. Tombstones: @RemindMe_OfThis (419★, 4.5 years of service): *"dead, thanks to the Twitter API changes"*; @this_vid (666★, millions of users): **suspended** April 2023 — the ban-risk datapoint. [HIGH] Then Feb–Apr 2026: X outlaws unsolicited programmatic replies outright. Everyone who might have built ambitiously watched five-year-old beloved bots die and the category leader exit. Survivors are ToS-grey browser extensions whose architecture structurally can't do autonomy.

## 4. Incumbent margin structure forbids BYOK

Bundled credits ARE the business: ReplyGuy charges $39–499/mo for metered replies; the string "API key" appears nowhere on its site. [HIGH] Offering BYOK would vaporize the margin on their heaviest users. Classic innovator's dilemma — the incumbents *can't* do it, and nobody neutral had a reason to.

## 5. No visible demand pull

April 2025, HN: a founder stood in exactly this spot — named the ElizaOS credential-surrender problem, asked "is anyone actually looking for something like this?" **One reply, zero points.** [HIGH that it happened; LOW as a market signal — one post isn't a market test.] But entrepreneurs scan for pull, and this category showed none.

## 6. Mindshare was captured by crypto, then poisoned

"Social AI agent" = ElizaOS (token migration → April 2026 class action), Virtuals (token launchpad), Farcaster agents (platform now in collapse). Serious builders and capital read the category as crypto-degen territory and went to coding agents instead. Wordware's viral Twitter moment ended in a pivot; the product survives only as a third-party revival. [MED]

## 7. The window is simply new

Every enabling condition is 2025–2026: tool-calling maturity across all providers, OpenRouter PKCE, Bluesky at scale with zero bot rules, X's summoned-only rule clarifying what's legal, Hypefury vacating 120k accounts, and xAI publishing @grok's prompt (the architecture blueprint). "Nobody has done it" is partly "nobody could have until about a year ago."

## The honest synthesis

The gap is real but it is not free money — it's guarded by two walls the incumbents never solved: **key onboarding for non-developers** (wall 1) and **X's platform cost** (wall 2). T3 and Hermes showed wall 1 has a legal door: run the official client on hardware the user controls (and, for Grok, get listed by xAI). That door does **not** open a multi-tenant cloud that intermediates Pro/Plus tokens. Our design now takes that door: local/VPS runtime + official CLIs as the headline, OpenRouter/API as the escape hatch, Bluesky/Telegram-first + BYO-X-credits for wall 2. The extinction record (wall 3) is why compliance-by-design isn't overhead — it's the survival trait every dead bot lacked.

PRODUCT_DESIGN.md pitch is now **"use the plan you already pay for, on this machine"** — T3/Hermes shape — not "log into Claude on our website."

## Open gaps (search-budget casualties)
- Product Hunt / Indie Hackers launch archaeology (no open API; needs search budget).
- OpenAI's third-party subscription policy (403'd).
- Crypto agent-launchpad consumer wrappers that folded (needs crypto-native sources).
