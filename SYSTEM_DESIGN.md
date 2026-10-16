# System Design: francispanman.com — Portfolio, Project Showcase, Dashboards & Agentic Lab (October 2026)

**Build francispanman.com as one GitHub monorepo deployed to a single Cloudflare Worker with Static Assets (not Pages). Serve plain HTML, CSS and JS from `/`, `/projects/<slug>/` and `/dashboards/<slug>/`, and add a small `/api/*` Worker for forms and proxies. Put the agentic sandbox on its own `lab.` Worker, gated by Cloudflare Access. Use Cloudflare Email Routing plus Gmail "Send mail as" for hello@. Everything except Claude API tokens fits on $0 free tiers.**

## TL;DR

- **Hosting:** Cloudflare's Workers Best Practices doc now says to use Workers instead of Pages for new projects. Static-asset requests are free and unlimited. The Workers Free plan's 100,000 dynamic requests/day and 10 ms CPU per request are far more than a portfolio's contact form, currency proxy and a gated Claude lab will use. No framework is needed until about project 20; when you outgrow vanilla, the natural step up is Astro, which Cloudflare acquired in January 2026.
- **Email and stores:** Inbound mail to hello@francispanman.com is free and unlimited via Email Routing. Cloudflare's new Email Sending (public beta since April 16, 2026) requires the Workers Paid plan ($5/month) for arbitrary recipients. So send through Gmail "Send mail as" (free) or Resend's free tier (3,000/month, 100/day). One exception is free on every plan: Workers can email *verified destination addresses*, which makes contact-form notifications to your own Gmail $0. For the stores, link out rather than using subdomains — Shopify does not support Cloudflare proxy setups, so any Shopify subdomain must be a DNS-only (grey-cloud) CNAME to `shops.myshopify.com`.
- **Agentic lab:** Gate lab.francispanman.com with Cloudflare Access (free up to 50 users). Route Claude calls through AI Gateway, and defend any public AI demo with Turnstile, the Workers Rate Limiting binding, a daily budget counter in D1 and an Anthropic spend limit. Default public demos to Workers AI's 10,000 free neurons/day, which hard-fail rather than bill on the Free plan.

---

## 1. Executive summary

Francis needs four things from one domain:
- a career portfolio that tells a coherent "EA/PA + GIS + data + AI-assisted builder" story;
- a home for 19 freeCodeCamp vanilla JS projects as he ships them;
- a small set of GIS and ops dashboards;
- a private sandbox for Claude, MCP and n8n experiments.

He also wants a custom email address and a clean relationship with two commerce ventures.

The design is deliberately boring at its core:

- **One repo, one Worker, mostly static files.** Every HTML page, project and dashboard is a static asset, served free and cached at the edge. The only code that runs per request is a small API Worker for forms, an API-key proxy and (later) AI endpoints.
- **Isolation by risk, not by topic.** Low-risk public content shares one deployment. Anything that spends money (Claude API) or holds secrets lives in a separate `lab` Worker behind Cloudflare Access, so a bug in the portfolio can't expose the lab and vice versa.
- **Defer databases until a write path exists.** Phase 1 uses JSON files committed to git. D1 arrives only when the contact form, newsletter or changelog need to write.
- **The build order follows the learning order.** For example, Form Validator → Contact Form → Turnstile-protected `/api/contact`.

Expected monthly cost: $0 for Cloudflare. The domain renewal and any Claude API tokens are the only spend. The first paid trigger is sending email to arbitrary recipients from Cloudflare, or needing more than 10 ms CPU per request; either costs $5/month (Workers Paid).

---

## 2. Requirements

### 2.1 Functional requirements

| ID | Requirement | Phase |
|---|---|---|
| F1 | Portfolio pages: Home, About, Work (EA/PA, GIS, Web Dev bootcamp, Data Analytics bootcamp), AI-Assisted Dev, Now, Contact | 1 |
| F2 | Project gallery listing all 19 freeCodeCamp builds with status, live link, repo link, "what I learned" note | 1 |
| F3 | Each project runnable at a stable URL (`/projects/<slug>/`) without breaking when other projects change | 1 |
| F4 | Contact form that reaches Francis's inbox, protected from bots | 1 |
| F5 | Custom email: receive at hello@ (and aliases), reply/send as hello@ | 0 |
| F6 | Dashboards: at least one GIS map, one ops/research-admin, one data-analytics dashboard | 2 |
| F7 | Agentic lab: private chat/agent UIs calling Claude API, Workers AI, MCP servers; scheduled agents | 3 |
| F8 | A public, read-only MCP server exposing portfolio data | 3 |
| F9 | "Built with Claude Code" changelog | 1–3 |
| F10 | Clear links to Sicario Tactical PH and ChinaAll.store | 4 |

### 2.2 Non-functional requirements

- **Cost:** $0/month baseline; every paid upgrade must be a deliberate, documented decision.
- **Security hygiene:** no secrets in git, ever; least-privilege API tokens; private things behind Access; abuse-resistant public endpoints.
- **Simplicity:** no build step in Phase 0–1. Wrangler is the only required npm tool. Every file is understandable by someone mid-way through a vanilla JS course.
- **Performance:** static pages served from edge cache; Lighthouse ≥ 90 on portfolio pages.
- **Maintainability at 2 h/day:** any single change (add a project, add a dashboard) fits in one 2-hour session, including deploy.
- **Observability and accessibility:** know when the contact form fails or AI spend nears a cap; semantic HTML, keyboard navigation, alt text.

### 2.3 Constraints

- Domain and DNS already on Cloudflare (registrar + authoritative DNS), which unlocks Email Routing, Access, Redirect Rules and Workers Custom Domains with no migration.
- GitHub user `panunch`, SSH auth; languages Python, JS, HTML, CSS.
- Workers Free plan limits: 100,000 requests/day (resets 00:00 UTC), 10 ms CPU per HTTP request, 50 subrequests per invocation, 3 MB Worker size, 100 Workers per account, 5 Cron Triggers per account.
- Learner preference: build it himself, so the document gives patterns and reasons rather than a finished codebase.

---

## 3. Proposed architecture

### 3.1 Component diagram

```
                        ┌──────────────────────────────────────────────┐
  Visitors / Recruiters │         Cloudflare edge (francispanman.com)   │
  ─────────────────────►│                                              │
                        │  DNS + Redirect Rule: apex ──301──► www      │
                        │  ┌────────────────────────────────────────┐  │
                        │  │ Worker "fp-www" (Custom Domain: www)   │  │
                        │  │  ├─ Static Assets (free, unlimited)    │  │
                        │  │  │   /  /about /work /ai /now          │  │
                        │  │  │   /projects/<slug>/  (19 fCC builds) │  │
                        │  │  │   /dashboards/<slug>/ (maps, charts) │  │
                        │  │  └─ Worker code (run_worker_first:     │  │
                        │  │      /api/*)                           │  │
                        │  │      POST /api/contact ─► Turnstile    │  │
                        │  │      GET  /api/fx (key proxy+cache)    │  │
                        │  │      POST /api/demo/ask ─► Workers AI   │  │
                        │  └───────┬──────────────┬────────────────┘  │
                        │     ┌────▼───┐   ┌──────▼──────┐            │
                        │     │  D1    │   │ send_email  │──► Gmail   │
                        │     └────────┘   └─────────────┘            │
                        │  ┌────────────────────────────────────────┐  │
  Francis only ────────►│  │ Cloudflare Access (OTP / Google login) │  │
                        │  └───────────────┬────────────────────────┘  │
                        │  ┌───────────────▼────────────────────────┐  │
                        │  │ Worker "fp-lab" (lab.francispanman.com)│  │
                        │  │  chat UI, agents, Cron daily brief     │  │
                        │  └──────┬──────────────┬──────────────────┘  │
                        │  ┌──────▼─────┐  ┌─────▼──────┐              │
                        │  │ AI Gateway │  │ Workers AI │              │
                        │  └──────┬─────┘  └────────────┘              │
                        │  Worker "fp-mcp" (mcp.francispanman.com)     │
                        │  Email Routing: MX ─► hello@ ─► Gmail        │
                        └─────────┼────────────────────────────────────┘
                                  ▼
                           Anthropic Claude API (spend-capped key)
```

### 3.2 Subdomain / route map

| Hostname / path | What it is | Hosted by | Proxy | Access |
|---|---|---|---|---|
| `francispanman.com` | 301 → `https://www.francispanman.com` | Redirect Rule | Proxied (orange) | Public |
| `www.francispanman.com/` | Portfolio pages | `fp-www` static assets | Custom Domain | Public |
| `www…/projects/<slug>/` | 19 fCC builds (+ later projects) | `fp-www` static assets | — | Public |
| `www…/dashboards/<slug>/` | GIS / ops / analytics dashboards | `fp-www` static assets | — | Public (sensitive ones → move to lab) |
| `www…/api/*` | Contact, subscribe, FX proxy, public AI demo | `fp-www` Worker code | — | Public, rate-limited |
| `lab.francispanman.com` | Agentic sandbox, private dashboards, admin | `fp-lab` Worker | Custom Domain | **Cloudflare Access** |
| `mcp.francispanman.com/mcp` | Public read-only MCP server | `fp-mcp` Worker | Custom Domain | Public, rate-limited |
| MX at apex | Email Routing MX/SPF/DKIM | Cloudflare Email Routing | DNS-only | n/a |
| `sicario.francispanman.com` | **Not recommended**; if used, CNAME → `shops.myshopify.com` | Shopify | **DNS-only (grey)** | Public |
| `chinaall.store` | Its own domain; linked from `/ventures` | Shopify / other | n/a | Public |

**Why `api`, `projects` and `dashboards` are paths, not subdomains:** same-origin means no CORS, one deploy, shared CSS and navigation, one TLS hostname, and consolidated SEO. A subdomain is earned only by a different trust boundary (`lab`, `mcp`) or a different vendor (Shopify).

---

## 4. Data flow

**A. Page or project view (≈99% of traffic).** Browser → Cloudflare edge → `fp-www`. Because Workers serve static assets ahead of Worker code unless `run_worker_first` matches, `/projects/quiz-game/` is answered straight from the asset store. No Worker invocation, no quota consumed.

**B. Contact form.**
1. The browser runs fCC-style client validation, then the Turnstile widget produces a token.
2. `POST /api/contact` reaches the Worker.
3. The Rate Limiting binding allows ≤ 3 per IP per 60 s.
4. The Worker calls Siteverify at `https://challenges.cloudflare.com/turnstile/v0/siteverify` with the secret.
5. On success it inserts a row into D1 `contact_messages`, then `send_email` → Francis's verified Gmail. This is free on every plan.
6. It returns `{ok:true}`. On any failure, the D1 row still exists, so nothing is lost.

**C. API-key proxy (Currency Converter).** Browser → `GET /api/fx?base=USD` → Worker checks the Cache API (1 h TTL) → on a miss it calls the FX provider with a key from Worker secrets → caches → returns. The key never reaches the browser.

**D. Lab agent run.** Francis → `lab.` → the Access login page (one-time PIN to his email) → JWT cookie → `fp-lab` → AI Gateway → Anthropic API (or a Workers AI binding) → response streamed back. Waiting on the network doesn't count toward the 10 ms CPU limit, so proxying an LLM call fits the Free plan.

**E. Scheduled agent (daily brief).** Cron Trigger (e.g. 22:00 UTC = 06:00 Manila) → `fp-lab` scheduled handler → fetch sources (calendar ICS, RSS, PAGASA bulletin parser, GitHub activity) → Claude summarises → `send_email` to Francis's verified Gmail.

**F. Email.** Sender → MX → Cloudflare Email Routing → rule `hello@` → Francis's Gmail. Replies go out from Gmail via "Send mail as hello@francispanman.com" over smtp.gmail.com.

**G. Deploy.** `git push` (SSH) to `main` → Workers Builds → `wrangler deploy` → live. Pushing a feature branch creates a Preview with its own URL. Preview URLs are public by default.

---

## 5. Storage and service choices (free-tier limits)

| Service | Use here | Free allowance (Oct 2026) | What happens at the limit | Recent change to note |
|---|---|---|---|---|
| **Workers Static Assets** | All HTML/CSS/JS, project folders, GeoJSON | Asset requests free and unlimited | n/a | Now Cloudflare's recommended path over Pages |
| **Workers (Free)** | `/api/*`, lab, MCP | 100,000 requests/day; 10 ms CPU/request; 50 subrequests; 3 MB script; 100 Workers; 5 Cron Triggers | Requests error until 00:00 UTC reset; no bill | Paid = $5/mo with 10M requests and 30M CPU-ms included |
| **Workers Builds** | Git push → deploy | 3,000 build minutes/month; 1 concurrent build; 20-min timeout (Paid: 6,000 min, 6 concurrent) | Builds queue/fail | Pages' old "500 builds/month" no longer the relevant number |
| **Previews** | Branch previews | 100 Previews per Worker (500 paid); 100 deployments per Preview | Least-recent Preview deleted | Old "preview URLs" renamed **Version URLs**; Previews are public by default |
| **D1 (SQLite)** | Forms, subscribers, changelog, AI budget counter | 5M rows read/day; 100K rows written/day; 5 GB total | Queries stop until reset | Stable pricing |
| **Workers KV** | Feature flags, cached lookups | 100K reads/day; 1,000 writes/day; 1 GB | Writes fail | Write cap makes KV wrong for counters or logs |
| **R2** | Large files (PMTiles, rasters, PDFs) | 10 GB-month; 1M Class A + 10M Class B ops/month; no egress fees | Billing requires Paid | — |
| **Durable Objects (SQLite-backed)** | Agent state, chat memory | 100K requests/day; 13,000 GB-s/day; 5 GB storage | Errors until reset | On free tier since 2025; must use `new_sqlite_classes` |
| **Workers AI** | Public AI demos, embeddings | 10,000 neurons/day | **Hard-fails** on Free (safe) | Per-model pricing now shown, still billed in neurons ($0.011/1k on Paid) |
| **AI Gateway** | Front door for Claude calls | Analytics, caching, rate limiting free | — | **Changed Sept 24, 2026:** new gateways' logs follow Workers Logs pricing; pre-existing gateways keep 100,000 logs total (Free) |
| **Rate Limiting binding** | Per-IP/per-key limits in code | No extra charge; Free and Paid | `limit()` returns `success:false` | GA Sept 19, 2025; needs Wrangler ≥ 4.36.0 |
| **Email Routing** | hello@, aliases, catch-all | Inbound unlimited; 200 rules/domain; 200 destination addresses/account; 25 MiB/message | — | Now part of "Cloudflare Email Service" |
| **Email Sending** | Outbound from Workers | Free plan: only to *verified destination addresses* (free, uncounted). Arbitrary recipients: Workers Paid, 3,000/month included, then $0.35/1,000 | — | **Public beta since Apr 16, 2026**; SMTP beta since June 2026 |
| **Cloudflare Access (Zero Trust Free)** | Gate lab, previews, admin | Up to 50 users; 24-h logs | User 51 blocked, not billed | Sign-up may ask for a payment method on file |

Turnstile and Web Analytics are free and need no capacity planning here.

**Default rule:** static JSON in git first → D1 when you need writes → R2 when a single file is bigger than you'd want in the repo (a few MB) → KV only for read-mostly config → Durable Objects only for per-agent state.

---

## 6. Deep dives

### 6(a) The portfolio site

**Recommendation: plain multi-page HTML/CSS/JS, no build step, for Phases 0–2.** You are learning vanilla web dev, and the 19 fCC projects are vanilla. A framework would add a second mental model (components, build output, adapters) before you have pain that justifies it. Every page is a real file at a real URL, which is also the most robust structure for SEO and for Workers Static Assets.

**Repo structure (monorepo `panunch/francispanman.com`):**

```
francispanman.com/
├── .gitignore                 # .dev.vars, .env*, node_modules/, .wrangler/, *.pem, .DS_Store
├── .dev.vars.example          # documents required secrets with fake values (committed)
├── wrangler.jsonc             # fp-www config
├── package.json               # only devDependency: wrangler
├── src/worker.js              # /api/* router (~150 lines by Phase 2)
├── migrations/0001_init.sql   # D1 schema (Phase 1b)
├── public/                    # = assets.directory
│   ├── index.html  about/  work/  ai/  now/  contact/  ventures/
│   ├── projects/
│   │   ├── index.html         # gallery rendered from /data/projects.json
│   │   ├── quiz-game/         # each fCC project self-contained
│   │   └── …
│   ├── dashboards/            # ph-flood-risk/, grants-tracker/
│   ├── data/                  # projects.json, changelog.json, simplified GeoJSON
│   ├── assets/css/site.css    # shared design tokens (CSS custom properties)
│   ├── assets/js/nav.js       # shared header/footer injection (tiny, no deps)
│   ├── 404.html               # double-duty: fCC "404 page" project
│   ├── _headers               # security + cache headers
│   └── _redirects             # /p/* shortcuts, old-URL redirects
├── lab/                       # separate Worker, own wrangler.jsonc (Phase 3)
└── mcp/                       # separate Worker (Phase 3)
```

**Minimal `wrangler.jsonc` (illustrative; confirm field names against current docs):**

```jsonc
{
  "name": "fp-www",
  "main": "src/worker.js",
  "compatibility_date": "2026-10-01",
  "assets": {
    "directory": "./public",
    "binding": "ASSETS",
    "not_found_handling": "404-page",
    "run_worker_first": ["/api/*"]
  },
  "routes": [{ "pattern": "www.francispanman.com", "custom_domain": true }],
  "observability": { "enabled": true }
  // Phase 1b adds: d1_databases, ratelimits, send_email
}
```

**www vs apex:** make `www` the canonical host (Worker Custom Domain). Redirect the apex with a Cloudflare Redirect Rule (301, preserve path and query). The apex needs a proxied DNS record for the rule to fire. MX records for Email Routing sit on the apex independently, so the redirect doesn't affect mail.

**Content strategy (the "story"):** lead with *"I turn messy operations and spatial data into tools — increasingly by building with Claude."* Each Work section pairs a credential with an artifact:
- EA/PA → grants tracker
- GIS → flood-risk map
- Data analytics → PSA indicators dashboard
- Web dev → the 19 projects
- AI → lab write-ups and changelog

**When to graduate to a framework.** Move to Astro when any two of these are true:
- You're hand-copying the same header into more than ~15 pages and `nav.js` injection feels hacky.
- You want Markdown-authored write-ups/blog posts.
- You want image optimisation or a content collection for projects.

Astro is the natural step: it outputs static HTML by default and keeps your vanilla JS islands. Astro joined Cloudflare in January 2026 but remains open-source, MIT-licensed, and multi-target. Eleventy is the lighter-weight alternative if you want templating without a component model. Migration is incremental, because existing `/projects/*` folders can be copied into Astro's `public/` untouched.

### 6(b) Hosting the 19 freeCodeCamp builds

**Pattern: one folder per project under `public/projects/<slug>/`, each fully self-contained** (its own `index.html`, `style.css`, `script.js`). Use **relative paths only** (`./style.css`, not `/style.css`) so a project works both locally and at its sub-path.

**Workflow per project (fits inside a session):**
1. Build in the fCC flow (locally or in a scratch repo).
2. Copy the folder to `public/projects/<slug>/`.
3. Add one entry to `public/data/projects.json` (slug, title, order, status, skills, learned, date).
4. Commit on a branch → Preview URL → check → merge → live.

**Projects that need special handling:**

| Project | Issue | Solution |
|---|---|---|
| Recipe Finder (MealDB) | Public API, free test key | Call directly from browser; no secret involved |
| Currency Converter | Most FX APIs need a key | `GET /api/fx` Worker proxy, key in `wrangler secret`, Cache API 1 h |
| Contact Form / Contact Page with validation | Needs a backend | Point both at `/api/contact` (Turnstile + D1 + send_email) |
| Newsletter Signup | Storing emails = personal data | `/api/subscribe` → D1 with double opt-in token; no bulk sending until needed |
| 404 page | — | Promote it to the site's real `404.html` via `not_found_handling` |
| Kanban, Expense Tracker, To-Do, Bookmark Saver | localStorage only | Fine as-is; note "data stays in your browser" in the UI |
| Coming Soon countdown, Scroll Progress | — | Reuse on the real site (countdown for Phase 2 launch; scroll bar on long write-ups) |

Keep your raw fCC practice repo separate if you like, but the *published* copy lives in the monorepo. Nineteen separate deployments would mean 19 configs and domains for files that cost nothing to serve from one place.

### 6(c) Dashboards

**Tooling (all static-hostable, CDN or vendored, no build):**

| Need | Pick | Why / version note |
|---|---|---|
| Simple slippy map + GeoJSON choropleth | **Leaflet 1.9.4** (stable, BSD-2-Clause) | Easiest learning curve; 2.0 is still alpha (ESM-only, drops global `L`), so stay on 1.9.4 |
| Vector tiles, large datasets, PMTiles | **MapLibre GL JS v6** (BSD) | Now ESM-only and requires WebGL2; use when GeoJSON gets too heavy |
| Standard charts | **Chart.js 4.5.x** (MIT) | Canvas, simple API, great for ops KPIs |
| Exploratory/analytic charts | **Observable Plot 0.6.17** (ISC) | Grammar-of-graphics, concise; import via jsDelivr `+esm` |
| Heavy scientific/interactive | Plotly.js | Large bundle; use only if you need 3D or statistical charts |

**Philippine data sources (with licensing realities):**

- **HDX COD-AB Philippines (PSA/NAMRIA):** ADM0–ADM4, i.e. down to barangays, with 42,048 ADM4 features. The shapefile zip is about 883 MB. Pre-process it offline with mapshaper or ogr2ogr (Python is fine): filter to one province/city, simplify, and export GeoJSON under ~2 MB. Treat the boundaries as indicative, not official. Check the dataset's license field (typically CC BY-IGO) and credit it.
- **UP NOAH hazard maps:** flood (5/25/100-year return periods), storm-surge and landslide shapefiles under **ODbL**. ShareAlike means any derived layer you publish must also be ODbL. A community PMTiles repackaging (~5.17 GB, ODbL) exists on BetterGov.ph. It's a mirror, not official, but PMTiles on R2 plus MapLibre is the ideal static-hosting pattern.
- **PSA OpenSTAT:** open with attribution; PxWeb API plus CSV downloads. Download CSVs at build time rather than calling the API from the browser (CORS untested).
- **PAGASA:** **no free open-data API.** Raw climatological data is by request form, with terms of use and historically a fee. For live weather context, link to bulletins or use a community parser and treat it as unofficial.
- **OpenStreetMap via Geofabrik:** the Philippines extract is about 582 MB PBF, plus shapefile/GeoPackage. Attribution "© OpenStreetMap contributors" is mandatory (ODbL).

**Dashboard architecture:** data → a Python preprocessing script in `tools/` (committed) → small static GeoJSON/CSV/JSON in `public/data/` → client-side render. Only the grants tracker needs writes, and its private version lives in the lab (D1 behind Access). The public version uses anonymised sample data. This keeps your employer's real deadlines and data off the public internet.

### 6(d) Agentic sandbox with abuse protection

**Separation:** `fp-lab` is its own Worker on `lab.francispanman.com`, protected by a Cloudflare Access self-hosted application.
- Policy: allow only Francis's email addresses; login via one-time PIN or Google.
- Unauthenticated requests never reach the Worker. Access is free for up to 50 users.
- Also put Access in front of custom-domain Preview URLs. workers.dev preview URLs send `X-Robots-Tag: noindex` but are otherwise public.

**What lives in the lab:**
- Chat UI → AI Gateway → Claude API (key in `wrangler secret put ANTHROPIC_API_KEY`).
- Agents SDK experiments on Durable Objects (SQLite-backed, available on Free).
- Cron-triggered agents (daily brief).
- n8n integration: run n8n locally or self-hosted and call lab webhooks protected by an Access **service token** (machine-to-machine), not by obscurity.
- Private dashboards (real grants tracker).

**MCP server (public, `fp-mcp`):** read-only tools such as `list_projects`, `get_project(slug)`, `get_resume_section(name)` and `search_changelog(q)` over the same JSON/D1 data.

**Version note:** As of July 2026, `McpAgent` is deprecated and feature-frozen in the Cloudflare Agents SDK. Start new servers with the stateless handler `createMcpHandler()` (Agents SDK v0.20.0+); many tutorials still show `McpAgent`.

**Public AI demo endpoint, layered defence (cheapest checks first):**

1. **Size guards:** reject bodies over ~4 KB; cap prompt characters; force `max_tokens` (e.g. 400) server-side; ignore any client-supplied model or token settings.
2. **Turnstile:** single-use tokens that expire after 5 minutes, verified server-side (mandatory).
3. **Rate Limiting binding:** e.g. `limit: 5, period: 60` keyed on `cf-connecting-ip` + route. Counters are per Cloudflare location and eventually consistent, so treat this as a speed bump, not an accounting system.
4. **Global daily budget in D1:** `UPDATE ai_budget SET calls = calls + 1 WHERE day = ? AND calls < ?`. If 0 rows change → 503 "demo resting until tomorrow". This is the real cost ceiling (D1 is right here; KV's 1,000 writes/day is not).
5. **Model choice:** public demos default to **Workers AI**. On the Free plan, usage past 10,000 neurons/day *fails rather than bills*, a built-in circuit breaker. Claude-powered features stay in the Access-gated lab.
6. **AI Gateway** in front of every Claude call: caching of identical prompts, gateway-level rate limiting, and analytics.
7. **Provider-side cap:** a dedicated Anthropic API key for this site with a monthly spend limit set in the Anthropic Console, so even a total bypass has a bounded blast radius.
8. **Kill switch:** an env var `AI_DEMO_ENABLED` checked first; flip it in the dashboard without redeploying.

### 6(e) Custom email

**Receive (free):** enable Email Routing for the zone. Cloudflare adds MX and SPF records automatically. Create rules:
- `hello@` → personal Gmail
- `francis@` → same
- `jobs@` → same, with a Gmail filter label
- catch-all → drop, or forward during the first month to learn what arrives

Each rule maps to one destination; to fan out to several inboxes, route to an Email Worker that calls `forward()` per destination.

**Send — three options, ranked:**

| Option | Cost | Deliverability/auth | Verdict |
|---|---|---|---|
| **Gmail "Send mail as"** via smtp.gmail.com:587 TLS + App Password (requires 2-Step Verification) | $0 | Works; but mail is DKIM-signed by gmail.com, so DMARC alignment is weak and some recipients show "via gmail.com" | **Use now** for human replies |
| **Resend free** | $0 for 3,000/month, 100/day, 3 domains | Proper DKIM on your domain | **Use for app mail** to non-verified recipients (newsletter confirmations) |
| **Cloudflare Email Sending** | Workers Paid $5/mo, 3,000 included, then $0.35/1,000; free to verified destinations on any plan | Auto SPF/DKIM/DMARC; public beta | **Use free path now** for self-notifications; revisit once GA or if you upgrade to Paid |

**DNS authentication (final state):**
- **SPF** (one record only; merge includes): `v=spf1 include:_spf.mx.cloudflare.net include:_spf.google.com ~all`. Add Resend's include on its own sending subdomain (e.g. `send.francispanman.com`) as Resend instructs, rather than stacking everything on the apex.
- **DKIM:** Email Routing manages its own; Resend gives you a DKIM CNAME/TXT on its subdomain. Gmail SMTP adds none for your domain.
- **DMARC:** Start with `v=DMARC1; p=none; rua=mailto:dmarc@francispanman.com` (route `dmarc@` to Gmail, or use Cloudflare's DMARC Management view). Stay at `p=none` while Gmail send-as is your main sender. Move to `p=quarantine` only once all outbound goes through an aligned sender (Resend or Cloudflare Email Sending).

### 6(f) Store subdomains (Sicario Tactical PH, ChinaAll.store)

**Recommendation: outbound links from a `/ventures/` page, not subdomains.**
- Each store's brand equity, SEO and customer trust belong to its *own* domain. `sicario.francispanman.com` tells shoppers "this is a hobby page of some person", which undercuts a tactical-gear retail brand.
- A Shopify store can have only one primary domain. Adding your subdomain as a secondary just redirects to the primary, so it buys nothing.
- Shopify's Help Center says every record pointing to Shopify must be DNS only; Cloudflare proxy setups (including Orange-to-Orange) are not supported. A subdomain would therefore lose Cloudflare's proxy features on that host anyway.

**If you still want one** (e.g. Sicario has no domain yet and you want a branded URL today):
1. In Cloudflare DNS add `CNAME sicario → shops.myshopify.com`, **Proxy status: DNS only (grey cloud)**.
2. In Shopify admin → Settings → Domains → Connect existing domain → `sicario.francispanman.com` → Verify.
3. Shopify issues the TLS certificate. Don't enable Cloudflare proxy, and remove any wildcard `*` record that could conflict.

ChinaAll.store already has its own domain. Link to it and keep its DNS in its own Cloudflare zone (same account is fine).

---

## 7. Deep dive: data model, API design, caching, error handling

### 7.1 Data model (D1, introduced in Phase 1b)

```sql
CREATE TABLE projects (
  slug TEXT PRIMARY KEY, title TEXT NOT NULL,
  track TEXT CHECK (track IN ('fcc','dashboard','lab','career')),
  status TEXT CHECK (status IN ('planned','building','shipped')),
  shipped_on TEXT, repo_url TEXT, live_path TEXT, tags TEXT,
  built_with_claude INTEGER DEFAULT 0, learned TEXT
);
CREATE TABLE changelog (
  id INTEGER PRIMARY KEY AUTOINCREMENT, date TEXT NOT NULL,
  project_slug TEXT REFERENCES projects(slug),
  tool TEXT CHECK (tool IN ('claude-code','claude-api','manual','n8n')),
  summary TEXT NOT NULL, commit_url TEXT
);
CREATE TABLE contact_messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT, created_at TEXT DEFAULT (datetime('now')),
  name TEXT, email TEXT, message TEXT, ip_hash TEXT, notified INTEGER DEFAULT 0
);
CREATE TABLE subscribers (
  email TEXT PRIMARY KEY, created_at TEXT DEFAULT (datetime('now')),
  confirm_token TEXT, confirmed_at TEXT
);
CREATE TABLE ai_budget (day TEXT, endpoint TEXT, calls INTEGER DEFAULT 0,
  PRIMARY KEY (day, endpoint));
```

Store `ip_hash` (SHA-256 of IP + secret salt), never raw IPs. Add an index only when a query scans: D1 bills rows *read*, not queries.

### 7.2 Route design

| Method & path | Auth | Purpose | Responses |
|---|---|---|---|
| `GET /api/health` | none | Uptime check | 200 `{ok:true, version}` |
| `POST /api/contact` | Turnstile + RL 3/60s | Store + notify | 200 / 400 / 403 / 429 / 500 |
| `POST /api/subscribe` | Turnstile + RL | Double opt-in | 202 / 400 / 409 / 429 |
| `GET /api/fx?base=` | RL 30/60s | Key-hiding proxy | 200 (cached) / 400 / 502 |
| `POST /api/demo/ask` | Turnstile + RL + budget | Workers AI demo | 200 / 403 / 413 / 429 / 503 |
| `lab./api/chat` | Access JWT | Claude via AI Gateway (streaming) | 200 / 401 / 502 |
| `lab./hooks/n8n/*` | Access service token | Automation callbacks | 200 / 401 |
| `mcp./mcp` | none, RL | MCP (Streamable HTTP) read-only | per MCP spec |

Router: a plain `switch` on `url.pathname` in `worker.js`. No Hono needed until you have more than ~10 routes, which is itself a lesson in when dependencies earn their place.

### 7.3 Caching

- **Static assets:** cached at the edge automatically. In `_headers`, use long `immutable` caching only for versioned filenames (e.g. `site.v3.css`), and short `max-age=300` for HTML and `/data/*.json` so deploys show up quickly.
- **Upstream API proxies:** Workers Cache API keyed on the normalised URL; 1 h for FX, 24 h for slow-moving PSA data.
- **AI:** AI Gateway caching for identical prompts (useful for demo "example questions").
- **Never cache** `/api/contact`, `/api/subscribe` or anything behind Access.

### 7.4 Error handling

- **One JSON envelope everywhere:** `{ ok:false, error:{ code:"RATE_LIMITED", message:"Try again in a minute." } }`. Use correct status codes (400 validation, 403 Turnstile fail, 413 too large, 429 with `Retry-After`, 502 upstream, 503 budget/kill-switch).
- **Never forward upstream error bodies** (they can leak keys or quotas). Log them; return a generic message.
- **Write-then-notify:** persist to D1 *before* sending email, so a mail failure loses nothing. Mark `notified=1` after success and retry unsent rows from a Cron.
- **Front-end:** every fetch has a timeout (`AbortController`), disables the submit button while pending, and renders human-readable errors. This is the fCC Form Validator lesson applied to production.
- **Observability:** enable Workers Logs; use `console.error` with a short code. Check weekly in the dashboard. Cloudflare Web Analytics covers traffic without cookies.

---

## 8. Scale and reliability (within free tiers)

- **Traffic headroom:** static pages are unmetered. Only `/api/*` counts toward 100,000/day. Even a viral post (50k views) with 1–2 API calls per 10 visitors uses under 10% of the quota.
- **CPU:** 10 ms is the tightest Free limit. I/O waits (D1, fetch to Claude) don't count, but JSON-parsing large payloads or crypto-heavy JWT checks can exceed it. Keep the Worker thin and push heavy GIS processing to offline Python. Exceeding it produces errors (1102/1101), the signal to upgrade.
- **Quota exhaustion behaviour:** Workers Free returns errors until 00:00 UTC (08:00 Manila); Workers AI and D1 stop. Design for it: the contact form front-end falls back to showing the hello@ address when the API errors.
- **Availability and backups:** Cloudflare is a single point of failure, which is acceptable for a portfolio. The repo is the source of truth, and static files can be redeployed elsewhere in an hour. Run `wrangler d1 export` monthly (a lab Cron can push it to R2). D1 also has Time Travel restore.

---

## 9. Security

**Repository hygiene**
- `.gitignore` from day one: `.dev.vars*` (except `.dev.vars.example`), `.env*`, `node_modules/`, `.wrangler/`, `*.pem`, `*.key`, `.DS_Store`, `tools/raw/` (large source data).
- Secrets only via `wrangler secret put NAME` (production) and `.dev.vars` (local). Commit a `.dev.vars.example` listing names with fake values.
- Turn on GitHub **secret scanning + push protection** for the repo. Use SSH keys with a passphrase. Add 2FA on GitHub, Cloudflare and Google (Gmail App Passwords require 2-Step Verification anyway).
- **Cloudflare API tokens** for any CI or automation: scoped to one account, the Workers/D1 permissions needed and the one zone. Never use the Global API Key.

**Edge and app**
- `_headers` baseline: `Content-Security-Policy` allowing only self plus the CDNs you use (`challenges.cloudflare.com` for Turnstile, jsDelivr/unpkg for libraries, tile hosts for maps); `X-Content-Type-Options: nosniff`; `Referrer-Policy: strict-origin-when-cross-origin`; `Permissions-Policy` (deny camera/mic/geolocation unless a dashboard needs location); `X-Frame-Options: DENY`.
- Enable DNSSEC (one click on Cloudflare Registrar), "Always Use HTTPS" and HSTS once stable. **Exception:** never on a Shopify-pointed host.
- Access on `lab.` and on custom-domain Previews; service tokens for n8n; validate the `Cf-Access-Jwt-Assertion` in the lab Worker as defence in depth.
- Turnstile + Rate Limiting binding on every public POST. The Free zone plan's single WAF rate-limiting rule is better spent as a coarse backstop on `/api/*`.
- **LLM-specific:** treat model output as untrusted (escape before inserting into the DOM). Give MCP tools read-only scopes. Never let a public endpoint choose the model, system prompt or tools.

---

## 10. Trade-off analysis

| Decision | Alternatives | Why chosen | Revisit when |
|---|---|---|---|
| **Workers Static Assets** for hosting | Cloudflare Pages; GitHub Pages; Netlify/Vercel | Cloudflare's stated default for new projects; static and API in one deploy; new features land on Workers only | Never likely; Pages continues to work if a tutorial forces it |
| **Vanilla multi-page HTML** | Astro; Eleventy; React/Next | Matches the fCC curriculum; zero build; every URL is a file | ~15+ pages with copy-pasted chrome, or you want Markdown posts → Astro |
| **Monorepo, one `fp-www` Worker** | Repo per project; Worker per project | One deploy, shared styles, no CORS; projects are just folders | A project needs its own runtime, secrets or trust boundary → give it its own Worker |
| **Separate `lab` Worker behind Access** | Same Worker with password check | Bugs in public code can't reach Claude key; Access is free and battle-tested | Never; add more Access apps as needed |
| **D1 for counters/budget** | KV; Durable Objects | 100K writes/day vs KV's 1,000; SQL is a skill he's building | Need strongly consistent per-user counters at high rates → Durable Object |
| **Workers AI for public demos; Claude in lab** | Claude everywhere | Free tier hard-fails instead of billing | You want public Claude demos → add Paid plan + stricter budget |
| **AI Gateway in front of Claude** | Direct API calls | Free caching, analytics, rate limits, provider switching | Log needs exceed free allowance under new Sept 2026 pricing |
| **Stateless MCP handler** | `McpAgent` on Durable Objects | `McpAgent` deprecated (v0.20.0); stateless is simpler and cheaper | Need per-session state → keep state in D1/DO explicitly |
| **Gmail send-as + Resend free** | Cloudflare Email Sending (Paid); Google Workspace; Zoho | $0 now; good enough for a personal brand | Email Sending reaches GA, or you upgrade to Workers Paid → consolidate on Cloudflare and tighten DMARC |
| **Stores via outbound links** | Shopify subdomains | Protects store brands; Shopify blocks CF proxy anyway | A store needs a temporary branded URL → grey-cloud CNAME |
| **Leaflet 1.9.4 first** | MapLibre v6 from day one | Lowest learning curve, no WebGL2 requirement | Datasets > ~5 MB GeoJSON or you want PMTiles/vector tiles → MapLibre |
| **Stay on Workers Free** | Workers Paid $5/mo | Fits all current needs | Sending to arbitrary recipients from Cloudflare, CPU > 10 ms, > 5 crons, or Durable-Object-heavy agents |

---

## 11. Phased build roadmap (2-hour sessions)

> Estimates assume a 2-hour session = one focused sitting. fCC project build time is your course time and isn't counted; only the "publish to domain" overhead (~15–30 min each) is.

### Phase 0 — DNS, email, "hello world" site (3–4 sessions)

| Session | Concrete steps |
|---|---|
| 0.1 | Enable 2FA everywhere. In Cloudflare: enable DNSSEC. Email Routing → enable → `hello@` → Gmail (verify). Add DMARC `p=none`. Gmail: App Password → "Send mail as hello@" via smtp.gmail.com:587 TLS. Send a test to a friend; check headers. |
| 0.2 | `ssh -T git@github.com` → create `panunch/francispanman.com`. Add `.gitignore`, `README.md`, `public/index.html` (one-page "coming soon"; reuse the fCC countdown later), `wrangler.jsonc` (assets-only). `npm i -D wrangler` → `npx wrangler dev` → `npx wrangler deploy`. |
| 0.3 | Connect the repo in Workers & Pages → Workers Builds. Add `www` Custom Domain; apex → www Redirect Rule. Enable Web Analytics. Turn on non-production branch builds; test a Preview. |
| 0.4 (buffer) | `_headers` security baseline; `404.html`; Lighthouse check. |

**Exit criteria:** `https://francispanman.com` → 301 → `https://www.francispanman.com`; hello@ round-trips; push = deploy.

### Phase 1 — Portfolio + first projects (≈8 sessions + ~20 min per fCC project)

| Session | Steps |
|---|---|
| 1.1–1.2 | Design tokens in `site.css` (colors, type scale, spacing); header/footer via `nav.js`; Home + About. |
| 1.3–1.4 | Work page (EA/PA, GIS, bootcamps) with one artifact per section (placeholders OK); AI-Assisted Dev page. |
| 1.5 | `projects.json` + `/projects/` gallery rendered with `fetch()` (applies fCC DOM skills). Publish Quiz Game + Color Palette Generator. |
| 1.6 | **Phase 1b:** create D1 (`wrangler d1 create`), migration `0001_init.sql`, `src/worker.js` router, `/api/health`. |
| 1.7 | Turnstile widget + `/api/contact` (siteverify → D1 → `send_email` to verified Gmail) + rate-limit binding. Wire the fCC Contact Form/Contact Page here when you reach them. |
| 1.8 | `/changelog/` page from `changelog.json`; start logging every Claude Code-assisted change. |
| Rolling | Each new fCC project: copy folder → JSON entry → branch → Preview → merge. Currency Converter triggers `/api/fx`; Newsletter Signup triggers `/api/subscribe`. |

### Phase 2 — Dashboards (≈8–10 sessions)

| Session | Steps |
|---|---|
| 2.1 | Python `tools/` script: download HDX COD-AB, filter one LGU (e.g. Marikina or your home city), simplify to ADM4 GeoJSON < 2 MB. |
| 2.2–2.3 | Leaflet barangay map + NOAH 100-year flood overlay (pre-clipped); attribution footer (OSM, NOAH ODbL, PSA/NAMRIA). |
| 2.4 | Join PSA population (COD-PS via P-codes) → "population in high-hazard zones" choropleth + Chart.js summary. |
| 2.5–2.6 | Research-admin ops dashboard with synthetic data (grant deadlines, IRB renewals, manuscript pipeline) — Observable Plot timelines. |
| 2.7 | PSA OpenSTAT indicator dashboard (CSV snapshot, Chart.js). |
| 2.8 | Write-ups for each dashboard ("question → data → method → caveats"); link from Work page. |
| 2.9–2.10 (optional) | MapLibre + PMTiles on R2 version of the flood map. |

### Phase 3 — Agentic lab (≈10–12 sessions)

| Session | Steps |
|---|---|
| 3.1 | Zero Trust free plan; Access app for `lab.francispanman.com` (OTP to your email); deploy a hello-world `fp-lab` Worker behind it. |
| 3.2 | Anthropic key with a spend limit → `wrangler secret put`; AI Gateway created; minimal streaming chat page. |
| 3.3 | D1 `ai_budget` + kill switch; `/api/demo/ask` public demo on Workers AI with Turnstile + rate limit. |
| 3.4–3.5 | Daily-brief agent: Cron Trigger → gather → Claude → `send_email` to self. |
| 3.6–3.7 | `fp-mcp`: stateless `createMcpHandler`, read-only tools over `projects.json`/D1; test with MCP Inspector and Claude Desktop. |
| 3.8–3.9 | Agents SDK experiment with Durable Object memory (e.g. "research-admin assistant" that remembers deadlines). |
| 3.10 | n8n (local Docker) → lab webhook with Access service token. |
| 3.11–3.12 | Lab write-ups → public `/ai/` page + changelog entries. |

### Phase 4 — Store integration (1 session)

| Session | Steps |
|---|---|
| 4.1 | `/ventures/` page: Sicario Tactical PH and ChinaAll.store cards, outbound links with UTM tags, your role and what you built. Optional grey-cloud `sicario.` CNAME only if Sicario lacks a domain. |

**Total:** roughly 30–35 sessions (~6–8 weeks at 2 h/day) excluding fCC course time, which runs in parallel.

---

## 12. Project ideas (tied to your background)

1. **Barangay Flood-Risk Explorer:** Leaflet/MapLibre map of one city's barangays with NOAH 5/25/100-year flood layers, PSA population joins and "people in high hazard" ranking. (GIS + climate resilience)
2. **Research-Grant Deadline Tracker:** public demo with synthetic data; private version in the lab with D1, Cron reminders and `send_email` nudges 30/14/7 days out. (EA/PA)
3. **"Built with Claude Code" changelog:** every commit tagged by tool, with prompts-that-worked notes and before/after diffs. (AI-assisted dev branding)
4. **Portfolio MCP server:** `mcp.francispanman.com/mcp` lets any MCP client ask "What GIS projects has Francis shipped?" (Agents/MCP)
5. **Daily Brief Agent:** 06:00 Manila email summarising calendar, PAGASA advisories, GitHub activity and fCC progress. (Agents + ops)
6. **Surgeon-Researcher Ops Dashboard (synthetic):** manuscript pipeline Kanban (reuse your fCC Kanban), IRB renewals and conference deadlines as Observable Plot timelines. (Research admin)
7. **fCC Progress Tracker:** `/projects/` page with a progress bar, skills heat-map and "days since last ship". Doubles as the Scroll Progress Indicator showcase. (Web dev)
8. **PSA Indicators Mini-Dashboard:** regional GDP or poverty incidence over time, CSV snapshots, Chart.js small multiples. (Data analytics)
9. **Meeting-Notes-to-Actions Lab Tool:** paste notes → Claude extracts owners, dates and follow-ups → export to Notion/Google Tasks via n8n. (EA/PA + Claude API)
10. **Point-Cloud Viewer Demo:** a small LiDAR sample in the browser (potree or three.js) with a write-up of your point-cloud AI work. (Point-cloud AI)
11. **ChinaAll Sourcing Calculator:** landed-cost calculator (product + shipping + duties + FX via `/api/fx`), reusing the Currency Converter project. (Ventures + web dev)

---

## 13. Caveats and recently changed items

- **AI Gateway logs pricing changed on September 24, 2026** for newly created gateways; re-check before relying on gateway logs as your audit trail.
- **Workers Builds limits:** Free: 3,000 build minutes/month, 1 concurrent build, 20-minute timeout. Paid: 6,000 minutes/month and 6 concurrent builds.
- **Data licensing:** NOAH and OSM are ODbL (ShareAlike). The HDX COD-AB license field should be checked on the dataset page. PAGASA raw data is *not* openly licensed. BetterGov.ph mirrors are community-run.
- **Library versions:** Leaflet 2.0 is still pre-release. MapLibre v6 is ESM-only/WebGL2; pin a version after checking npm.
- **Config snippets** in this document are illustrative. Confirm field names (`run_worker_first`, `not_found_handling`, `send_email`, `ratelimits`) against current Wrangler docs before deploying.

---

## Key sources

- Cloudflare Workers best practices: https://developers.cloudflare.com/workers/best-practices/workers-best-practices/
- Migrate from Pages to Workers: https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/
- Workers limits: https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/workers/platform/limits.mdx
- Workers Previews: https://developers.cloudflare.com/workers/previews/
- Email Service pricing / limits: https://developers.cloudflare.com/email-service/platform/pricing/ , https://developers.cloudflare.com/email-service/platform/limits/
- Workers AI pricing: https://developers.cloudflare.com/workers-ai/platform/pricing/
- AI Gateway pricing: https://developers.cloudflare.com/ai-gateway/reference/pricing/
- Remote MCP server guide: https://developers.cloudflare.com/agents/guides/remote-mcp-server/
- Cloudflare Access: https://www.cloudflare.com/sase/products/access/
- Shopify domain troubleshooting (Cloudflare proxy): https://help.shopify.com/en/manual/domains/troubleshoot-issues-with-domains
- Leaflet downloads: https://leafletjs.com/download.html
- MapLibre GL JS: https://www.npmjs.com/package/maplibre-gl
- PSA OpenSTAT: https://openstat.psa.gov.ph/
- Geofabrik Philippines: https://download.geofabrik.de/asia/philippines.html
- BetterGov NOAH hazard maps (PMTiles): https://data.bettergov.ph/datasets/22
