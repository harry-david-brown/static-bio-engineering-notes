# static.bio — Engineering Notes (public receipts)

This repo is the public “receipts” companion to **static.bio**. The product repo is private/monetized, but the engineering constraints and decisions are real and intentionally documented.

- Product: https://static.bio
- Repo: private (monetized)

---

## 0) Non-negotiables (the contract)

Public profile pages must be:

- **Pure HTML + CSS** (no client-side JavaScript, no hydration)
- **No cookies** on public pages
- **No third-party scripts/origins** on public pages (no `<script>` at all; no external JS)
- **Small**: HTML+CSS gzipped ≤ **50KB** (current production output ~**1.88KB** gzipped)
- **Fast**: sub-100ms class TTFB targets (real browsers ~**20ms**, test harness ~**98ms**)

These aren’t “guidelines”—the build should fail if the contract is broken.

---

## 1) System shape (two apps, different constraints)

static.bio is intentionally split into two surfaces with different runtime behavior:

1) **Public profiles** — `/{username}`
   - Static/ISR-style HTML served from edge/CDN
   - Pure HTML output, inline CSS, system fonts, **no JS**

2) **Dashboard/admin** — `/dashboard/**`
   - Next.js + React + Tailwind + auth
   - Used by creators to manage content, domains, and billing

The public surface is designed to remain framework-independent at runtime.

---

## 2) Architecture (high level)

```
mermaid
flowchart LR
  U[Visitor] --> E[Edge/CDN Cache]
  E -->|cache miss| O[Origin]
  O --> R[(Postgres)]
  O --> S[(Object Storage)]
  O --> Q[Jobs/Queue]
  O --> OBS[Logs/Tracing]
```

Notes:
- The dashboard is a conventional web app.
- The public profile path is treated like a static artifact pipeline: fetch minimal data, render deterministic HTML, cache hard.

---

## 3) Routing strategy (how “zero JS” is enforced)

**Key routing decision:** `proxy.ts` rewrites `/{username}` → `/api/profile/{username}` specifically so Next.js doesn’t attach client JS to a “page.”

This keeps public profiles:
- outside the Next.js page rendering pipeline that tends to inject runtime JS/hydration
- served as HTML returned directly from a route handler
- consistent with the “static artifact” model

---

## 4) Rendering model (static-style behavior)

Profiles are treated as “baked” artifacts: generated and then served from edge/CDN paths like static content. The intended model is static/ISR + on-demand invalidation (save → invalidate → re-bake).

There’s also an explicit “framework apocalypse” escape hatch: a planned/export path that can emit static HTML files for all profiles and host them anywhere.

---

## 5) Performance receipts

**Observed performance (production):**
- Real browsers: ~20ms TTFB (HTTP/2 reuse, Brotli, proximity to Edge)
- Test suite: ~98ms TTFB (includes test runner network latency)
- Cold start vs warm refresh: ~103ms median cold, ~90ms min warm in the test harness
- Size: ~1.88KB gzipped HTML+CSS (budget is ≤ 50KB)

**What makes it fast/small (by design):**
- API route returns HTML directly → no React runtime, no hydration, no client bundles.
- Inline CSS and system fonts → no extra CSS/font requests.
- Minimal markup + hand-rolled templates → tiny payload.

---

## 6) Data & queries (boring on purpose)

- Postgres, accessed with raw SQL (Edge-compatible via `@vercel/postgres`).
- Public profile fetch is optimized as one round-trip (profile + links via JSON aggregation), keeping DB time ~10–20ms.
- Indexes exist on username, link ordering, and hostname lookups to keep reads predictable.

---

## 7) Privacy model (default-off analytics)

Default behavior: no tracking.

Analytics is a per-profile opt-in toggle:
- Off: links render as direct `<a href="https://…">` (no tracking endpoint).
- On: links render as `/r/{linkId}` redirects that increment counters and then 302—still no cookies and no third-party analytics.

---

## 8) Security posture (pragmatic)

- Public responses include basic hardening headers (e.g., clickjacking + MIME sniffing).
- Public pages have no scripts, reducing XSS surface area by construction.
- Dashboard routes are auth-protected; server actions benefit from built-in CSRF protections.

---

## 9) Tests enforce the contract

The guarantees are enforced with CI tests:
- Performance budget checks (TTFB + size + “no scripts” + “no third-party origins”).
- Cold-vs-warm measurements to keep regressions visible.
- Privacy checks (analytics off/on semantics without cookies/3rd parties).
- Production integration tests hit the live URL to verify real deployment behavior.

---

## 10) Current status + what’s next

As of Dec 2025, static.bio is production-ready with core functionality implemented (public rendering, dashboard CRUD, opt-in analytics, custom domains, hardening/tests/docs).

**Near-term roadmap items:**
- Billing/plan enforcement (Stripe)
- Monitoring/observability upgrades
- Optional static export / CDN cache strategy improvements
