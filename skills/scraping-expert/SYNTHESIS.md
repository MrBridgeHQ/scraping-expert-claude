# Scraping Expert — Knowledge Synthesis

> **Derived digest, not the source of truth.** One-page distillation of the `scraping-expert` skill
> for fast orientation. Authoritative content is `SKILL.md` + the files in `references/`. When this
> digest and the skill disagree, the skill wins — refresh this file.

## Central thesis
**Anti-bot is a cost function, not a wall.** No system open to the network is fully closed; the goal is not to be undetectable but to keep *cost-per-result below value-per-result* while staying under the trust threshold the target has set. Corollary: **scraping is the engineering of decisions made before the first line of code**, not a matter of selectors. The skill is **platform-agnostic** — its doctrine applies to a standalone script, a VPS cron job, a serverless function, a web-app backend, or an Apify Actor; Apify is one deployment target among others. One hard limit: **identity verification (IDV — government ID + selfie)**. We don't engineer around it; we stop.

## Two recommended operating rules
1. **Don't scrape from your local/dev machine.** All requests to the target run on remote infrastructure (Apify, your own servers/VPS, or a managed API). Local dev uses fixtures, never the network. "Test it on the live site" → run it on the remote infra, not locally.
2. **Keep scraper repositories private.** On Apify, deploy via `apify push`.

## The expert workflow — five phases (in order)
1. **Diagnose the target & define the plan** (before any code).
2. **Choose the cheapest tool that works** (start one rung *below* instinct; escalate on observed failure).
3. **Implement with resilience baked in** (selectors with fallbacks; sessions/proxy configured; cache by default; retries at the right layer; surface progress/degraded state).
4. **Handle errors & blocks as a first-class, categorized output** (not exceptions that crash the run).
5. **Test without the network** (fixtures + mock at the boundary).

### Phase 1 — Diagnose & define the plan (15-30 min, never from the local IP)
Six questions: (1) internal API? (2) static or dynamic? (3) which anti-bot vendor? (4) which proxy tier/country? (5) volume budget? (6) data shape / crawl topology. Tools: **Wappalyzer** (one-click vendor ID), DevTools, CreepJS/FingerprintJS (validate *your own* fingerprint).
**Watch yourself do it first.** Before any code, navigate the target as a human — the human script (homepage → search → result) becomes the scraper's navigation script, which is itself the bypass for behavioral detection.
**Actively probe, don't just identify.** Fire test requests through the research/scraping APIs and MCP tools available — **Firecrawl, ZenRows, ScrapFly, ScraperAPI, Tavily/Exa, the Apify console** — to observe how the protection *actually behaves*: plain fetch → 403 or silent-200? residential managed call succeeds where datacenter fails? does JS rendering change the result? Each probe (off the local IP) narrows the choice with evidence.
**The objective is not "which vendor" — it's the full plan**: tool rung + proxy tier/country + pacing/timing + concurrency budget + session/token-replay strategy + escalation plan. Phase 1 ends with that one-page plan, not a vendor name. Key trap: **a 200 is not a success** — anti-bot often returns 200 + empty/fake content; schema validation is the canary.

### Phase 2 — Cheapest viable tool (the ladder)
| Rung | Tool | Cost/req | When |
|---|---|---|---|
| 1 | `got-scraping` (or `httpx`/`curl_cffi`) | ~$0.0001 | internal API / static HTML, no JS |
| 2 | `CheerioCrawler` | ~$0.0002 | static HTML, light anti-bot |
| 3 | `PlaywrightCrawler` | ~$0.001-0.005 | JS-rendered, moderate anti-bot |
| 3b | `AdaptivePlaywrightCrawler` | between 2 and 3 | mixed page types (auto HTTP↔browser) |
| 4 | **Camoufox** | ~$0.005-0.02 | aggressive Cloudflare, Akamai, DataDome, HUMAN |
| 5 | Managed API | $0.001-0.05 | LinkedIn/Amazon/ticketing, low volume, time-to-market |

Camoufox = the only open-source tool at ~0% headless detection (Firefox patched at the C++ level), but heavy (2-4 GB, 5-60 s/page). Adjacents: **Patchright** (~67%, lighter), **noDriver** (raw CDP), **Stagehand** (LLM element selection; modes `dom`/`hybrid`/`cua`). Costs above are illustrative (Apify CU); the rung→resource shape holds on any host.

### Phase 3 — Resilience (structural, not extra try/catch)
Selectors with **fallback chains** (structured data > `#id` > semantic class > `data-*` > structure > XPath); **sessions + proxy rotation configured** (Crawlee `useSessionPool`, runs anywhere); **cache external responses** in a cache layer (KV Store / Redis / Postgres / disk, ~30-day TTL); retries with backoff at the right layer; **surface progress/degraded state** (status message, logs, or a run-status row).

### Phase 4 — Errors & blocks as first-class output
Platform-agnostic principle: **emit a structured, categorized error record to your output sink** (Apify dataset row, Postgres error column, log line) instead of crashing the run. Two pieces of carry-everywhere doctrine:
- **Failure taxonomy:** `TARGET_BLOCKED` ≠ `EXTRACTION_FAILED` ≠ rate-limit ≠ `TARGET_NOT_FOUND` ≠ bad-input — each handled differently; **a 200 with empty/fake body is a silent block**, caught by schema validation.
- **Commit irreversible side effects only on confirmed success** (COMMIT the DB transaction / fire the webhook / charge — only after success, never on retry/partial/error).
*Apify-specific:* this becomes the graceful-exit (`exit SUCCEEDED`, errors in the dataset, never `Actor.fail()`) + PPE (`Actor.charge()` last) pattern, documented in `references/error-handling-ppe.md`.

### Phase 5 — Test without the network
Local suite makes **no network calls**. Mock everything crossing the process boundary (HTTP client, platform SDK, third-party SDKs). Fixtures captured from real remote runs (PII scrubbed), version-controlled, refreshed (30-90 days). Coverage = *critical path* (parsers, error paths, fallbacks), not 100%.

## The #1 lever — the hidden API
The highest-leverage move: discover the site already serves its data as clean JSON. 20 min in the Network tab beat 200 h of fighting anti-bot. Canonical example **Vivino** (`/api/explore/explore`…): a 4 GB Camoufox job becomes a lightweight HTTP client hitting JSON. Look for `/api/`, `/graphql`, `/_next/data/`, `/wp-json/`, `__NEXT_DATA__`, `window.__NUXT__`, and the **mobile backend (often less protected)**. "Scrape the source of truth, not the rendering of it."

## The anti-bot model (doctrine core)
**Four detection pillars:** (1) IP reputation, (2) TCP/TLS-JA3/JA4/browser fingerprint — *detection is consistency, not absolute values*, (3) behavior (mouse, scroll, typing, navigation flow, cookie banner), (4) active challenges (CAPTCHA, proof-of-work, JS).
**Four escalation levels:** HTTP + consistent fingerprint → sessions + proxy rotation → throttling + humanization (residential) → engine-level stealth or managed API. Highest-leverage optimization: **token replay** (solve the JS challenge once in a browser, replay the `cf_clearance`/`datadome` cookie over HTTP — **same IP + same UA mandatory**, 50-95% cost cut).
**Per-vendor playbook:** Cloudflare, Akamai (sensor data), PerimeterX/HUMAN (continuous validation → level 4), DataDome (<2 ms scoring), AWS WAF (configurable), Kasada/Shape (banking/ticketing), regional (GeeTest, YandexCaptcha, FriendlyCaptcha). **What no longer works in 2026:** random UA rotation, `puppeteer-extra-stealth` (stale since 2023), free proxies, manual `navigator.webdriver=false`, solving the CAPTCHA (you already failed upstream). Framing: **think like the defender's dashboard** — don't generate the spike that makes an analyst hit "block".

## Managed APIs — pay someone else
When: pathological target, low volume, time-to-market, credits already burned. Vendors: **Firecrawl** (AI/markdown, MCP-ready, default), **ZenRows**/**ScrapFly** (Cloudflare/DataDome), **ScraperAPI** (prototyping), **Bright Data Scraping Browser** (CDP, hardest). Patterns: thin adapter, ~30-day cache, failover, cost monitoring.

## Extraction resilience
**Never one selector** for a critical field. **Schema validation (Zod/Pydantic)** is the **anti-bot canary** (200 + missing fields = silent block). Avoid honeypots (`display:none`). Recovery when a selector breaks: snapshot → diff → update the chain + the fixture → document.

## Legal / ethics (any project)
Frameworks: ToS (contract), CFAA (public ≈ ok; behind login = no), copyright (facts ok, expression no), GDPR (PII = high risk). `robots.txt` respected by default. **Won't build:** bulk PII, paywall bypass, login into third-party accounts, republishing protected content, continuing after a cease-and-desist. **Hard limit: IDV.** 2026: anti-bot *targeting AI agents*, **Know Your Agent (KYA / MCP-I)**, regulation-driven enforcement (GDPR art. 32, PCI, HIPAA).

## The golden rules (platform-agnostic)
1. Internal API first. 2. Cheapest viable tool first. 3. Categorize errors & blocks before handling them (a 200 ≠ success). 4. Cache external calls by default.
*On Apify only, two more apply at the deployment layer:* always exit SUCCEEDED; PPE charges only on confirmed success.

## Position
`scraping-expert` is the **platform-agnostic doctrine / architecture layer**. It supplies diagnosis, tool-ladder, and anti-bot doctrine; implementation (Crawlee/Actor code, auditing an existing scraper, MCP server code, pricing/PPE, README/Store content, generic web SEO) is delegated to dedicated implementation tooling. Two references are **Apify-deployment doctrine**, segregated for that consumption: `references/apify-patterns.md` and `references/error-handling-ppe.md`.
