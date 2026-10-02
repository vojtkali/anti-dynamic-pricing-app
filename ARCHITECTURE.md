# Architecture

> **Status: DRAFT proposal, waiting for maintainer approval.** No code exists yet.
> This file describes the planned design. Once approved, it will be updated as each
> milestone lands, and it will describe what is actually built.

## Contents

1. [Goals and non-goals](#1-goals-and-non-goals)
2. [Ground rules that shape the design](#2-ground-rules-that-shape-the-design)
3. [System overview](#3-system-overview)
4. [Tech stack](#4-tech-stack)
5. [Plugin model (verticals and sites)](#5-plugin-model-verticals-and-sites)
6. [Data model](#6-data-model)
7. [Privacy pipeline](#7-privacy-pipeline)
8. [Detection: typical price, outliers, segmentation, personalization](#8-detection)
9. [Clean check](#9-clean-check)
10. [Alerts and dashboard](#10-alerts-and-dashboard)
11. [API sketch](#11-api-sketch)
12. [Folder structure](#12-folder-structure)
13. [Testing strategy](#13-testing-strategy)
14. [Milestones M1–M5](#14-milestones)
15. [Risk register (legal and technical)](#15-risk-register)
16. [Open questions](#16-open-questions)

---

## 1. Goals and non-goals

**Goals (MVP, flights vertical)**

- A browser extension (Chrome and Firefox, Manifest V3) that reads the flight prices a
  user already sees and records them with coarse context.
- An overlay that shows: what other people were shown for the same flight, price
  history, and an honest verdict with a sample size and a confidence level.
- A *clean check* that compares the user's normal session with a clean session,
  to test for personalization.
- A backend that stores minimized, crowdsourced observations and computes typical
  ranges, outliers and segment differences.
- Price-drop alerts and a dashboard (watched items, history, estimated savings).

**Non-goals (MVP)**

- Price *prediction* ("buy now / wait"). We show history and percentiles; we do not
  forecast. Forecasting needs much more data and can easily mislead users.
- Booking, affiliate links, or any revenue from retailers. (Affiliate income would
  create a conflict of interest with honest verdicts.)
- Mobile apps. An extension cannot see prices inside airline apps.
- Checkout pages. These pages contain passenger names and payment forms. MVP plugins
  read only search-result and offer-detail pages.

### A note on what we expect to find

Flights have the strongest *dynamic* pricing (prices move with time and seat
inventory), but published research has found little evidence of *personalized*
(per-user) flight pricing. An example is Vissers et al., *Crying Wolf? On the Price
Discrimination of Online Airline Tickets* (HotPETs 2014). Price differences that are
easy to observe usually come from:

1. **Time:** the price changed between two searches.
2. **Point of sale (POS):** a different country site, language or currency.
3. **Caching:** metasearch sites show cached prices that are out of date.
4. **Product differences:** a different fare brand, bags included or not, or a
   different number of passengers.

So the detection model must separate these effects *before* it says "personalized".
Even when it finds no personalization, the product is useful. "Same flight, 9% cheaper
on the German point of sale" and "this is in the cheapest 20% of prices seen" are
actionable. Airlines are moving to NDC "continuous pricing" and offer engines, so
personalization is becoming technically easier, and monitoring it is still worthwhile.

The McDonald's story that motivated this project describes **zone pricing** (a price
per store or area, based on local willingness to pay). It does not describe
per-person pricing. The model below treats *segmentation* (zone, device, point of
sale) and *personalization* (you, specifically) as different findings, and it labels
them differently.

---

## 2. Ground rules that shape the design

| Rule | Consequence in the design |
|---|---|
| **The project never sends a request to a retailer that the user did not start.** | No server-side scraping. No background polling of airline sites. The clean check is one page load in a visible tab, started by the user. |
| **Collect the minimum.** | Observations hold only whitelisted fields. No URLs, no account IDs, no IP addresses stored, and no persistent user ID. |
| **Minimize data on the device before upload.** | Rounding, field stripping and token generation happen in the extension. The server validates strictly and rejects unknown fields. |
| **Sharing is opt-in.** | The default is local-only mode. Uploading needs explicit consent during onboarding, and the user can withdraw it at any time. |
| **Verdicts are honest.** | Every verdict shows *n*, the time window and a confidence level. "Likely personalized" is only possible with clean-check evidence (see §8.5). A property-based test enforces this rule. |
| **One maintainer.** | One language (TypeScript), one database (Postgres, which also runs the job queue), one `docker compose up`, and no Redis, Kafka or Kubernetes. |

---

## 3. System overview

```
┌──────────────────────────── User's browser ─────────────────────────────┐
│                                                                          │
│  Supported site tab (e.g. Kayak results)                                 │
│  ┌──────────────────────────────┐    ┌───────────────────────────────┐   │
│  │ Content script               │    │ Overlay (Shadow DOM, Preact)  │   │
│  │  • site plugin: detect()     │───▶│  verdict · peers · history    │   │
│  │  • extract() → offers+prices │    │  [Run clean check] [Watch]    │   │
│  └──────────────┬───────────────┘    └──────────────▲────────────────┘   │
│                 │ messages                          │                    │
│  ┌──────────────▼──────────────────────────────────┴─────────────────┐   │
│  │ Background (service worker / event page)                          │   │
│  │  • local store (IndexedDB): own observations, watches, settings   │   │
│  │  • verdict engine (@adp/core): same code as the server            │   │
│  │  • minimizer → upload queue (only if opted in; batched, jittered) │   │
│  │  • clean-check orchestrator (container / incognito tab)           │   │
│  │  • alarms → alert evaluation → browser notifications              │   │
│  └──────────────┬───────────────────────────────────▲────────────────┘   │
│                 │                                   │                    │
│  Dashboard (extension page): watches, history charts, savings            │
└─────────────────┼───────────────────────────────────┼────────────────────┘
                  │ POST /v1/observations             │ GET summaries
                  │ POST /v1/clean-checks             │ (aggregates only)
┌─────────────────▼───────────────────────────────────┴────────────────────┐
│ API (Fastify, Node)                                                      │
│  • strict validation (shared Zod schemas)                                │
│  • IP → coarse region (offline GeoIP DB), then the IP is dropped         │
│  • FX normalization to EUR (ECB reference rates)                         │
│  • aggregation: typical range, outliers, segment tests, k-anonymity      │
│  • in-memory rate limiting (IP never persisted)                          │
├──────────────────────────────────────────────────────────────────────────┤
│ Worker (same codebase, pg-boss on Postgres)                              │
│  • daily ECB FX sync • daily rollups • retention/deletion • site reports │
├──────────────────────────────────────────────────────────────────────────┤
│ PostgreSQL 17                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

**Main flows**

1. **Observe.** The user opens a supported results page. The plugin extracts the
   visible offers (the cheapest N plus any offer the user expands). Each offer is
   stored locally. If sharing is on, a minimized copy is queued for upload.
2. **Explain.** The background asks the API for the aggregate summary of the offer
   (or of its route and date). It then runs the verdict engine *locally* and renders
   the overlay. The user's own price is never sent in the summary request, so a
   lookup reveals only that someone viewed this flight.
3. **Clean check.** The user clicks *Run clean check*. The extension opens the same
   search in a clean context, captures the price, and re-reads the normal tab (A/B/A).
   It then shows the result. If the user opted in, it uploads a minimized result.
4. **Watch.** The user watches an offer or a route and date. Alarms check the
   aggregates every few hours. If no fresh crowd data exists, the extension shows a
   *re-check* reminder that opens the page when the user clicks it.

**Why there are no accounts in the MVP:** watches, history and savings live on the
user's device. The server never learns who watches what, and there is no personal
data to protect, export or delete on the server. The dashboard is an extension page.
(This is decision Q1 in §16.)

---

## 4. Tech stack

| Area | Choice | Why |
|---|---|---|
| Language | **TypeScript (strict)** everywhere | One language. Domain types and validation are shared by the extension and the API. |
| Monorepo | **pnpm workspaces** (no Turborepo at first) | Simple and fast. `pnpm -r` is enough for 3–4 packages. |
| Extension framework | **WXT** (Vite-based) | One codebase builds MV3 for Chrome and Firefox. It handles manifest differences, has HMR, and has Shadow-DOM UI helpers. It is actively maintained. |
| Extension UI | **Preact** + plain CSS in Shadow DOM | The overlay is injected into third-party pages, so size and style isolation matter. Preact is about 4 KB and has a React-compatible API. |
| Charts | **uPlot** | Small (about 50 KB) and fast for time series. |
| Local storage | **Dexie** (IndexedDB) | Gives indexed queries over the user's own history. `chrome.storage` is used only for small settings. |
| Validation | **Zod** | One schema serves the extension, the API request validation and the OpenAPI docs. |
| API | **Node LTS + Fastify** with `fastify-type-provider-zod` | Mature and fast. It has first-party plugins (rate limiting, Helmet, Swagger). NestJS is too heavy for one maintainer. Hono is fine but has a smaller plugin ecosystem. |
| Database | **PostgreSQL 17** | `percentile_cont`, window functions, JSONB for vertical-specific attributes, and BRIN indexes on time. TimescaleDB is not needed at this scale. |
| DB access | **Drizzle ORM** + drizzle-kit migrations | TypeScript-first and close to SQL. Raw SQL is easy for statistics queries. |
| Jobs | **pg-boss** | A job queue and scheduler that run on Postgres, so there is no Redis. |
| FX rates | **ECB euro reference rates** (daily, public) | Free and authoritative, and a good fit for EU users. |
| Geo | **DB-IP Lite** country/city database (CC BY 4.0), read offline | No third-party API call, and the IP never leaves our server. Needs attribution. |
| Tests | **Vitest**, **fast-check** (property tests), **happy-dom** (plugin fixtures), **Playwright** (extension E2E, later) | |
| Lint/format | **Biome** | One fast tool instead of ESLint + Prettier. |
| Local dev | **Docker Compose** (Postgres, API, worker) | `docker compose up` and `pnpm dev`. |
| CI | **GitHub Actions** | Lint, typecheck, test (with a Postgres service) and build the extension zips. |

No telemetry, analytics or third-party error reporting in the extension. Server errors
are logged locally without IP addresses.

---

## 5. Plugin model (verticals and sites)

There are two levels of plugin:

- **Vertical definition** (in `packages/core`, pure TypeScript, shared with the server).
  It answers: *what is a product in this vertical, and when are two observations
  comparable?*
- **Site plugin** (in the extension, uses the DOM). It answers: *how do I read
  products and prices from this site?*

```ts
// packages/core — shared by the extension and the API
interface VerticalDefinition<A> {
  id: 'flights' | 'hotels' | 'ecommerce' | 'food-delivery' | 'ride-hailing';
  attributesSchema: z.ZodType<A>;          // strict whitelist of item attributes
  itemKey(a: A): string;                   // exact comparable product
  groupKey(a: A): string;                  // coarser group (e.g. route + date)
  consumptionDate(a: A): string | null;    // departure / check-in; drives retention
  comparability: {                         // detection parameters per vertical
    peerWindowMinutes: number;             // flights: 360
    minAbsDiffEur: number;                 // flights: 2
    minRelDiff: number;                    // flights: 0.02
    sanityRangeEur: [number, number];      // flights: [1, 50_000]
  };
}

// apps/extension — one per site (or per platform; Kayak + momondo share one)
interface SitePlugin<A> {
  id: string;                              // 'kayak'
  version: string;                         // bumped when selectors change
  vertical: VerticalDefinition<A>['id'];
  matches: string[];                       // drives manifest host_permissions
  detect(doc: Document, url: URL): PageKind | null;   // 'results' | 'offer' | null
  extract(ctx: ExtractContext): ExtractedOffer<A>[];  // must be pure DOM reading
  sessionState(doc: Document): { loggedIn: boolean | null; memberPrice: boolean };
  pointOfSale(url: URL, doc: Document): { country?: string; currency: string; locale: string };
  cleanCheck?: { buildUrl(offer: ExtractedOffer<A>): string | null };
}
```

**Flights item keys** (deterministic, built in core, contain no personal data):

```
groupKey: fl1|PRG-LHR|2026-11-01|RT:2026-11-08|Y|1A0C0I
itemKey:  fl1|PRG-LHR|2026-11-01|RT:2026-11-08|Y|1A0C0I|OK620@0710+OK621@1840|BASIC
          └ version │ route  │ out date │ return  │cabin│ passengers │ segments              │ fare brand
```

Passenger mix and fare brand are part of the key. A price for 2 adults, or a fare that
includes bags, is never compared with a 1-adult basic fare.

**Plugin rules (checked in code review):** plugins read only price and itinerary
elements. They never read form inputs and never run on checkout or passenger-details
pages. They never call `fetch`. Host permissions are the union of the plugin `matches`
(never `<all_urls>`).

**Proposed first sites** (Q2 in §16): **Kayak + momondo** (one plugin, shared
platform) and **Google Flights**. Both put the search in the URL, so the clean check
can reproduce it. **Skyscanner** and **Kiwi.com** come next. Direct airline sites come
later (see risk R3).

---

## 6. Data model

### 6.1 Server (PostgreSQL)

```sql
-- Sites and their plugin versions
sites (
  id            smallserial PK,
  slug          text UNIQUE,            -- 'kayak', 'google-flights'
  vertical      text NOT NULL,
  display_name  text NOT NULL
)

-- One row per comparable product (a flight offer)
items (
  id               bigserial PK,
  vertical         text NOT NULL,
  item_key         text UNIQUE NOT NULL, -- see §5
  group_key        text NOT NULL,        -- route + date + cabin + passengers
  consumption_date date,                 -- departure; drives retention
  attributes       jsonb NOT NULL,       -- validated by vertical schema
  first_seen_at    timestamptz NOT NULL
)  -- index: (group_key), (consumption_date)

-- Crowdsourced price observations (minimized)
observations (
  id                 bigserial PK,
  item_id            bigint FK → items,
  site_id            smallint FK → sites,
  observed_at        timestamptz NOT NULL,  -- rounded to 5 min on the device
  stage              text NOT NULL,         -- 'results' | 'offer'
  price_minor        bigint NOT NULL,       -- integer minor units, as displayed
  currency           char(3) NOT NULL,      -- ISO 4217
  price_eur_minor    bigint NOT NULL,       -- normalized at ingest
  fx_date            date NOT NULL,         -- ECB rate date used
  -- context (all coarse)
  pos_country        char(2),               -- site point of sale (.de, /cz/)
  region_country     char(2),               -- from IP at ingest, or user override, or NULL
  region_subdiv      text,                  -- ISO 3166-2, NULL unless k-anon allows
  device_class       text NOT NULL,         -- 'desktop' | 'mobile' | 'tablet'
  browser_family     text NOT NULL,         -- 'chrome' | 'firefox' | ...
  os_family          text NOT NULL,         -- 'windows' | 'macos' | 'linux' | 'android' | 'ios'
  logged_in          boolean,               -- NULL = unknown
  member_price       boolean NOT NULL DEFAULT false,
  session_kind       text NOT NULL,         -- 'normal' | 'clean'
  repeat_view_bucket smallint NOT NULL,     -- 0, 1, 2 (2–3), 3 (4+) views of this group in 7 days
  days_to_consumption integer,              -- derived
  -- quality and abuse control
  dedup_token        bytea NOT NULL,        -- see §7; unlinkable across items and days
  plugin_version     text NOT NULL,
  ext_version        text NOT NULL,
  is_outlier         boolean NOT NULL DEFAULT false,  -- set by cleaning (§8.1)
  UNIQUE (item_id, dedup_token, stage, session_kind)
)  -- indexes: (item_id, observed_at), BRIN(observed_at)

-- Clean-check results (A/B/A, see §9)
clean_checks (
  id                bigserial PK,
  item_id           bigint FK → items,
  site_id           smallint FK → sites,
  checked_at        timestamptz NOT NULL,   -- rounded
  currency          char(3) NOT NULL,
  normal_minor      bigint NOT NULL,        -- A1
  clean_minor       bigint NOT NULL,        -- B
  normal2_minor     bigint,                 -- A2 (re-read), NULL if unavailable
  method            text NOT NULL,          -- 'firefox_container' | 'chrome_incognito'
  outcome           text NOT NULL,          -- 'same' | 'normal_higher' | 'normal_lower' | 'drift' | 'failed'
  delta_rel         real,
  logged_in         boolean,
  region_country    char(2),
  device_class      text NOT NULL,
  dedup_token       bytea NOT NULL
)

fx_rates (rate_date date, currency char(3), per_eur numeric(18,8), PK(rate_date, currency))

-- Rollups (built by the worker; feed history charts and site reports)
item_daily_stats  (item_id, day, n, contributors, min, p25, p50, p75, max)        -- EUR minor units
group_daily_stats (group_key, day, n, cheapest_p50, p25, p75)                     -- route-date curve
site_reports      (site_id, period, dimension, finding, effect_rel, n, p_value, method_version)
```

There is **no users table**: the MVP has no accounts (Q1). If we add accounts later
(for email alerts or cross-device sync), they will live in a separate schema with
their own retention policy, and they will never be joined to observations.

### 6.2 Device (extension, IndexedDB)

```
local_observations  full-detail copy of what the user saw (exact time, URL path, all offers)
watches             { id, kind: 'item'|'group', key, label, targetEur?, mode: 'below_target'|'below_typical', createdAt, lastNotifiedAt }
bookings            { itemKey, paidMinor, currency, bookedAt }  ← user marks "I booked this"; used for savings
clean_check_runs    { itemKey, A1, B, A2, outcome, method, at }
settings            { sharing: boolean, regionMode: 'auto'|'override'|'withheld', regionOverride?, installSecret }
upload_queue        minimized observations waiting for a jittered batch upload
```

The user can export and delete all of this. Local data stays on the device unless the
user turns on sharing, and then only the minimized form leaves the device.

---

## 7. Privacy pipeline

```
local observation (full detail)
   │  minimize (on device)
   ├─ keep only whitelisted fields (Zod .strict(); unknown fields = error)
   ├─ drop URL, query string, page title, any free text
   ├─ round timestamp to 5 min
   ├─ dedup_token = SHA-256(installSecret ‖ itemKey ‖ UTC date)
   ├─ coarse device/browser/OS family; logged-in boolean; repeat-view bucket
   ├─ region: not sent (server derives a coarse value) | user override | 'withheld'
   ▼
upload queue → batch upload every 10–30 min (random jitter); no batch ID stored
   ▼
API ingest
   ├─ strict validation; reject unknown or out-of-range values
   ├─ IP → country (+ subdivision) via offline DB, then the IP is discarded
   │    (the IP is not logged by the app; the reverse proxy is configured with no IP logging)
   ├─ FX normalize; mark outliers
   ▼
query time
   └─ k-anonymity: any segment value with < 5 contributors is merged into its
      parent (subdivision → country → 'other') or hidden
```

- **Why server-side IP geolocation?** Sites price by the location *they* see, which is
  the IP address (or the VPN exit). Locating the upload request is the closest proxy,
  and it needs no location permission. Users can override the region or withhold it.
- **The dedup token** lets one person count once per flight per day without any ID
  that links their observations across flights or days.
- **GDPR posture:** we treat uploaded observations as personal data, to be safe.
  Rare combinations could, in theory, single someone out, and the IP is processed
  briefly at ingest. Lawful basis: consent (Art. 6(1)(a)) for sharing. Summary lookups
  are part of the service the user asked for. The server cannot link records to a
  person, so Art. 11 applies to access and erasure requests. Local data is fully
  user-controlled. We will provide `PRIVACY.md`, a record of processing, a short DPIA,
  and EU-only hosting with no non-EU processors.
- **Retention:** raw observations are deleted 90 days after the departure date and
  kept only as daily rollups. Raw clean checks are deleted after 12 months and kept
  only in site reports.
- **Store declarations:** Chrome Web Store *Limited Use* disclosures, and Firefox's
  required `data_collection_permissions` manifest key (mandatory for new Firefox
  extensions since November 2025).

---

## 8. Detection

The verdict has several parts. The overlay shows a headline plus the evidence behind it.

```ts
interface Verdict {
  headline: 'not_enough_data' | 'good_deal' | 'fair' | 'above_typical' | 'likely_personalized';
  peers:   { status: 'insufficient' | 'below' | 'typical' | 'above'; n: number; contributors: number;
             windowMinutes: number; medianEur: number; p25: number; p75: number;
             percentileRank: number; confidence: 'low' | 'medium' | 'high' };
  history: { status: 'insufficient' | 'low' | 'typical' | 'high'; percentile: number; days: number; n: number };
  segments: Array<{ dimension: 'pos' | 'region' | 'device' | 'login'; finding: string;
                    effectRel: number; n: number; pAdj: number }>;
  personalization: { status: 'not_tested' | 'no_difference' | 'difference_observed' | 'likely_personalized' | 'inconclusive';
                     checks: number };
}
```

### 8.1 Data cleaning (before any statistics)

- Hard sanity range per vertical. For flights, 1–50,000 EUR per booking; values
  outside are rejected.
- Robust outlier flag on the comparable set (same item, stage and window):
  modified z-score `z = 0.6745·(x − median)/MAD`. If `|z| > 3.5`, the observation is
  flagged (Iglewicz–Hoaglin). Flagged observations are excluded from typical ranges
  and history. This protects against parse errors and against a minority poisoning
  the data (the median tolerates up to 50% contamination).
- If `MAD = 0` (common: everyone saw the same price), the z-score is undefined. We
  then flag only values outside the practical-significance floor (§8.2).

### 8.2 Practical significance

A difference counts only if it is at least **max(2 %, €2)** (configured per vertical).
Cent-level noise, rounding and FX differences are never reported.

### 8.3 Peer comparison: "what are others shown right now?"

- **Comparable set:** same `item_key` and same `stage`, normal sessions, not flagged,
  within ±6 h of the user's observation, excluding the user's own dedup token.
- **Minimum evidence:** 3 or more distinct contributors, or the status is `insufficient`.
- **Statistic:** the percentile rank of the user's price in the set, and the difference
  from the median.
- **`above`:** percentile rank ≥ 90 and the floor in §8.2 is met. Wording: *"You were
  shown €12 (8 %) more than 9 out of 10 people who looked at this flight in the last
  6 hours."* `below` is the mirror case.
- **Confidence from contributors:** 3–9 = low, 10–29 = medium, 30+ = high. It drops
  one level if the prices in the window show a strong time trend (Spearman |ρ| > 0.5
  against time). In that case the difference may be timing.
- **Caching caveat:** for metasearch `results`-stage prices, the overlay states that
  the prices can be cached.

### 8.4 History: "is now a good time?"

- Distribution of the same item's cleaned daily medians over the last 14 days, falling
  back to the group (route and date, cheapest offer).
- `low` ≤ P20, `high` ≥ P80, plus the floor. A line explains that fares on most routes
  rise in the last weeks before departure, so "high" may be normal when the departure
  is close.
- No forecasting in the MVP.

### 8.5 Segmentation: "do prices differ by point of sale, region, device or login?"

- For one item in one window, split the comparable set by a dimension (POS country,
  region country, device class, logged-in). Each group needs 5 or more contributors.
- **Test:** Mann–Whitney U (two groups) or Kruskal–Wallis (more than two, then pairwise
  Mann–Whitney). The effect is the difference of group medians. Benjamini–Hochberg
  correction runs across all dimensions tested. A finding needs adjusted p < 0.01
  **and** an effect that meets the floor.
- The worker aggregates findings per site and month into `site_reports`. Example:
  *"On site X, the same flights were shown ~6 % higher on the CZ point of sale than on
  the DE point of sale (412 observations, 37 flights)."*
- This is labeled **segmentation**, never personalization. It is still actionable:
  "buy on the DE point of sale". For EU flights it is also legally relevant:
  Regulation (EC) No 1008/2008, Art. 23(2), forbids fare discrimination based on the
  customer's nationality or place of residence.

### 8.6 Personalization: "is the price different for *me*?"

Passive crowd data can never prove personalization, because too many explanations
remain (see §1). Only the controlled clean check (§9) can produce this verdict.

- One clean check produces a factual statement:
  `difference_observed` ("your normal session showed €14 more than a clean session at
  the same moment"), `no_difference`, or `inconclusive` (the price moved between the
  reads).
- **`likely_personalized` requires** the floor to be met in the same direction in
  **either** 2 or more of the user's own clean checks on this item (24 h apart or less)
  **or** 1 check plus site-level evidence. Site-level evidence means 20 or more
  conclusive checks on the site in the last 30 days, a sign test p < 0.01 (normal
  session higher), and a median effect that meets the floor.
- **An invariant enforced by property-based tests:** the verdict engine never returns
  `likely_personalized` without clean-check evidence that meets these rules. No
  combination of peer, history or segment data can produce it.
- The overlay links the relevant law. In the EU, traders must disclose prices that
  were personalized by automated decision-making (Consumer Rights Directive
  Art. 6(1)(ea), added by the Omnibus Directive 2019/2161). A *documented* undisclosed
  difference is something the user can report to their national consumer authority.

### 8.7 Headline priority

`likely_personalized` › `above_typical` (peers `above`, or history `high` when peers
are insufficient) › `good_deal` (peers `below` or history `low`) › `fair` ›
`not_enough_data`. The evidence panel always shows n, the window and the confidence,
and it says what the verdict does *not* claim.

---

## 9. Clean check

**What we will not do:** a background `fetch` to the airline's search API without
cookies. That is automated access. Bot protection (Akamai, DataDome and others) blocks
it, it needs a reverse-engineered private API for each site, and it probably breaks the
site's terms. See R1.

**What we will do:** an A/B/A comparison in a real, visible tab that the user starts:

1. **A1:** the price already captured in the user's normal tab.
2. **B:** open the same search (`plugin.cleanCheck.buildUrl`) in a clean context:
   - **Firefox:** a temporary **container** (`contextualIdentities`). It has its own
     cookies, storage and cache. The container is deleted afterwards.
   - **Chrome:** a new **incognito window**. The user must enable *Allow in Incognito*
     once, and the extension explains why. If incognito access is not allowed, the
     check is unavailable (no hidden workaround). Data captured in incognito is used
     only for the check, and it is not stored in local history.
   - The same plugin extracts the price, and the same offer is matched by `item_key`.
3. **A2:** re-read the normal tab. If the page cannot be re-read, reload it once,
   only with user consent.
4. **Outcome:** if |A1 − A2| is below the floor and |A − B| meets the floor, the result
   is a difference. If A1 ≠ A2, the price is moving, so the result is `inconclusive`.

**What the check controls for and what it does not:** it holds constant the IP and
location (same network), the time (seconds apart, checked by A/B/A) and the device. It
isolates cookies, login, browsing history and site storage. It does **not** isolate
browser fingerprinting, because the browser is the same. The UI states this limit.

**Requirement:** a plugin without a reproducible search URL (many airline sites use
POST forms) declares `cleanCheck` as unsupported, and the overlay says so.

---

## 10. Alerts and dashboard

**Watches** are stored only on the device. A watch can target a specific offer
(`item_key`) or a route and date (`group_key`, cheapest offer). It can trigger
*below €X* or *below typical* (below P25 of the 14-day history and the floor).

**Evaluation:** a `chrome.alarms` timer runs hourly. Each watch is checked about every
6 h with jitter, through one batched summary request. A browser notification shows when
the condition is met. There is no email and no server-side subscription.

**Honest limit:** alerts depend on crowd coverage, because we never poll airlines. If
a watch has no new observations for 3 days, the user gets a **re-check reminder**. A
click opens the search page, the plugin captures the price, and the watch is evaluated.
That is one normal page view by the user.

**Dashboard** (an extension page, Preact + uPlot):
- Watched items with the current status, typical range and a sparkline.
- An item page with a history chart (own observations, crowd median and P25–P75 band)
  and the clean-check log.
- **Savings** (labeled as estimates): for items the user marks as booked, the
  difference between the price paid and (a) the first price the user was shown and
  (b) the crowd median at the time of booking.
- Settings: sharing on or off, region mode, export (JSON), and delete all local data.

A small **public site** with aggregate route curves and site reports is planned after
the MVP. It only reads the public API.

---

## 11. API sketch

All routes are under `/v1`. They are versioned, documented with OpenAPI at `/docs`,
and validated with the shared Zod schemas.

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/observations` | Batch upload (≤ 50) of minimized observations. Opt-in only. |
| `POST` | `/clean-checks` | Upload one minimized clean-check result. |
| `POST` | `/summaries` | Batch lookup of aggregates for up to 50 item or group keys. The response contains statistics only, never raw rows. |
| `GET` | `/items/{itemKey}/history?days=30` | Daily rollup for the charts. |
| `GET` | `/groups/{groupKey}/history?days=60` | Route-date curve. |
| `GET` | `/sites/{slug}/report` | Segmentation and clean-check findings per site. |
| `GET` | `/health` | Liveness and DB check. |

There are no raw-data export endpoints (see R4). Rate limiting is in memory and per
IP. The IP is never persisted.

---

## 12. Folder structure

```
anti-dynamic-pricing-app/
├── apps/
│   ├── extension/                     # WXT, MV3, Chrome + Firefox
│   │   ├── entrypoints/
│   │   │   ├── background.ts          # storage, sync, alarms, clean-check orchestration
│   │   │   ├── content.ts             # loads the matching site plugin, mounts the overlay
│   │   │   ├── popup/                 # quick status + toggles
│   │   │   ├── dashboard/             # watches, history, savings (extension page)
│   │   │   └── onboarding/            # consent + permissions explanation
│   │   ├── src/
│   │   │   ├── plugins/
│   │   │   │   ├── registry.ts
│   │   │   │   ├── kayak/             # index.ts, selectors.ts, __fixtures__/, kayak.test.ts
│   │   │   │   └── google-flights/
│   │   │   ├── overlay/               # Shadow-DOM Preact components
│   │   │   ├── clean-check/           # firefox-container.ts, chrome-incognito.ts, protocol.ts
│   │   │   ├── storage/               # Dexie schema + repositories
│   │   │   ├── sync/                  # minimizer.ts, upload-queue.ts, api-client.ts
│   │   │   └── alerts/
│   │   └── wxt.config.ts
│   └── api/                           # Fastify API + worker
│       ├── src/
│       │   ├── server.ts / worker.ts
│       │   ├── routes/                # observations, clean-checks, summaries, items, sites, health
│       │   ├── services/              # aggregation, geo, fx, k-anon, site-reports
│       │   ├── db/                    # drizzle schema.ts, migrations/
│       │   └── jobs/                  # fx-sync, rollups, retention, site-reports
│       └── test/                      # integration tests against Postgres
├── packages/
│   └── core/                          # pure TS: no DOM, no Node APIs
│       └── src/
│           ├── money/                 # price parsing, ISO 4217 minor units, FX math
│           ├── stats/                 # quantiles, MAD, rank tests, BH correction
│           ├── verdict/               # verdict engine (§8)
│           ├── verticals/flights/     # schema, item/group keys, parameters
│           └── schemas/               # API contracts (Zod)
├── docs/
│   ├── adding-a-site-plugin.md
│   ├── privacy-design.md              # DPIA notes, data inventory
│   ├── risk-register.md
│   └── adr/                           # architecture decision records
├── docker-compose.yml
├── .github/workflows/ci.yml
├── README.md  CONTRIBUTING.md  ARCHITECTURE.md  PRIVACY.md  LICENSE
├── biome.json  tsconfig.base.json  pnpm-workspace.yaml  package.json
```

---

## 13. Testing strategy

| Layer | What | How |
|---|---|---|
| `core/money` | Price parsing across locales (`1 234,56 Kč`, `€1.234,56`, `$1,234.56`, `1.234 kr`, `¥12,000`, NBSP / narrow NBSP, ranges, "from €"), resolving an ambiguous `$` through the POS, minor units | Table-driven Vitest |
| `core/stats` | Quantiles, MAD, modified z, Mann–Whitney / Kruskal–Wallis, BH, sign test | Known reference values (checked against R/SciPy outputs) |
| `core/verdict` | Scenario tests, plus **invariants**: never `likely_personalized` without clean-check evidence; never a peer verdict with fewer than 3 contributors; the floor is respected | Vitest + fast-check |
| Plugins | Extraction against saved page fixtures | happy-dom. Fixtures are *minimized* (structure and prices only, no third-party scripts or images) |
| Minimizer | No URL, text or non-whitelisted field can leak | Property tests over random inputs |
| API | Ingest validation, dedup, k-anonymity, aggregation SQL, retention | Vitest against real Postgres (Docker locally, service container in CI) |
| Extension E2E | Overlay renders on a local fixture page; clean check on a mock site | Playwright with the built extension (M4–M5) |

---

## 14. Milestones

Each milestone ends with: a summary, run instructions, manual test steps, the next
steps, and updates to README, CONTRIBUTING and this file.

**M1 — Foundations and core logic** (nothing user-facing yet)
- Monorepo, TypeScript strict, Biome, Vitest, CI, Docker Compose (Postgres).
- `@adp/core`: money and price parsing, flights vertical (schema, keys), the stats
  library, and verdict engine v1 (peers, history, headline), with full tests and
  property-based invariants.
- README (with the project description), CONTRIBUTING, LICENSE, PRIVACY draft,
  risk register.
- *Done when:* `pnpm install && pnpm test` passes locally and in CI.

**M2 — Backend API**
- Drizzle schema and migrations. `POST /observations` with strict validation,
  GeoIP coarsening, FX (ECB sync job), dedup and rate limiting.
- `POST /summaries`, history endpoints, k-anonymity, outlier flagging, daily rollups,
  the retention job, and OpenAPI docs.
- A seed script that generates synthetic observations, so the API can be tested
  without real users.
- *Done when:* `docker compose up` gives a working API with docs, integration tests
  pass, and the seeded data produces sensible summaries.

**M3 — Extension v1 (observe and explain)**
- WXT setup (Chrome and Firefox builds), plugin registry, the **Kayak/momondo** plugin
  and the **Google Flights** plugin with fixtures.
- Local storage, onboarding with consent (local-only by default), the minimizer and
  the upload queue.
- The overlay with verdict, peers and a history sparkline. Popup.
- *Done when:* loading the unpacked extension and searching flights on a supported
  site shows the overlay, observations appear in the local DB (and in the API if
  opted in), and fixture tests pass.

**M4 — Clean check and segmentation**
- The A/B/A protocol: Firefox containers and Chrome incognito, the permission UX, and
  `POST /clean-checks`.
- Segmentation tests in the API, the site-report job, and personalization verdict
  rules with invariants.
- Evidence panel UI ("what this does and does not show").
- *Done when:* a clean check runs end to end on both browsers against a local mock
  site with controllable personalization, and against one real site manually.

**M5 — Watches, alerts, dashboard, release readiness**
- Watches (item and route-date), alarms, notifications and re-check reminders.
- The dashboard: watches, history charts (uPlot), savings, export and delete.
- A third plugin (Skyscanner or Kiwi.com), Playwright E2E, store listings, the final
  privacy policy, and the Firefox `data_collection_permissions` key.
- *Done when:* signed builds are ready for the Chrome Web Store and AMO submission,
  and docs are complete.

**After the MVP (candidates):** the public stats site, hotels vertical, drip-pricing
detection (results price vs. final price), optional accounts for email alerts and
sync, Safari (iOS) extension, and anti-Sybil tokens (Privacy Pass style).

---

## 15. Risk register

| # | Risk | Severity | Mitigation / safer alternative |
|---|---|---|---|
| R1 | **Clean check as an automated request** (background fetch without cookies) breaks site terms and triggers bot protection | High | Use a visible tab started by the user in a container or incognito window, one page load per check (§9). No private-API reverse engineering. |
| R2 | **Polling airline sites for alerts** is scraping | High | Not done. Alerts use crowd data and re-check reminders (§10). Free flight-data APIs are scarce: the Amadeus Self-Service APIs closed in July 2026. Licensed data is possible later, but it costs money. |
| R3 | **Airline terms and litigation.** Some carriers actively sue screen scrapers (for example Ryanair; CJEU C-30/14 *Ryanair v PR Aviation* confirmed that terms can restrict screen scraping when the database right does not apply) | Medium | Read only pages the user loads, in the user's own browser. Start with metasearch sites, and add direct airline plugins one at a time after reviewing each site's terms. Get legal review before public launch. |
| R4 | **EU sui generis database right:** publishing a substantial part of a site's fare data | Medium | Aggregates only. No raw or bulk export per site. Rate-limited reads. Any open dataset is published as aggregates. |
| R5 | **GDPR:** IP processing, re-identification from rare combinations | Medium | §7: on-device minimization, transient IP use, coarse fields, k-anonymity, retention limits, DPIA, EU hosting, consent. Treat the data as personal data. |
| R6 | **Wrongly accusing a company** of personalized pricing (reputational or legal harm) | Medium | Strict evidence rules (§8.6), factual wording ("in 12 of 40 checks…"), confidence and n always shown, legal review before publishing site reports that name companies. |
| R7 | **Data poisoning or Sybil uploads** (by competitors or retailers) | Medium | Robust statistics, per-item and per-day dedup tokens, IP rate limits, anomaly detection on bursts. Later: privacy-preserving tokens. |
| R8 | **Fragile parsers** (obfuscated or changing DOM, especially Google Flights) | Medium | Versioned plugins, fixture tests, a parser confidence score, an automatic "plugin broken" state (stop reporting, never show wrong data), and a contributor guide. Reading JSON from page network traffic was considered and rejected for the MVP (more invasive). |
| R9 | **Store review** (permissions, data use) | Low–Med | Host permissions only for supported sites. Single-purpose description. Clear disclosures. Firefox `data_collection_permissions`. |
| R10 | **Sparse data early on:** verdicts often say "not enough data" | Product risk | Show it honestly. Show the user's own history from day one. Group-level (route-date) fallbacks. Seed communities (EU travel forums). |

*This register is engineering risk analysis, not legal advice. Before public launch,
a lawyer should review R1, R3, R4, R5 and R6.*

---

## 16. Open questions

These questions block decisions and are waiting for the maintainer. Each answer will be
recorded as an ADR in `docs/adr/`, and this section will then be removed.

1. **Q1 – Accounts:** no accounts in the MVP (local watches, an extension-page
   dashboard, browser notifications), or accounts with email alerts and a hosted web
   dashboard? *Recommended: no accounts.*
2. **Q2 – First sites:** Kayak/momondo + Google Flights, then Skyscanner or Kiwi.com?
3. **Q3 – License:** AGPL-3.0 (copyleft; hosted forks must share changes) or
   MIT/Apache-2.0 (maximum adoption)? *Recommended: AGPL-3.0 for code; any published
   aggregate data under ODbL.*
