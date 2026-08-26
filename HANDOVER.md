# Handover: find and fix correctness bugs & logic loopholes (Flot ecosystem)

Goal: read through the whole codebase — the **dashboard** and the **merchant site repos** —
hunt for BUGS, logic loopholes, broken edge cases, and silent failures, then FIX them.
Correctness first, not security theatre. Work on branches with small, focused PRs; verify
each with a clean build before merging.

## What this is
- **flot-dashboard** (repo `Open-hubb/flot-website-dashboard`, live at `dashboard.flotme.ai`):
  Next.js (App Router) + TypeScript merchant dashboard for the Flot payment ecosystem
  (~30 small-business sites in Sierra Leone).
- Stack: Next.js App Router, Tailwind v4, Prisma + PostgreSQL (Neon dev/prod branches),
  NextAuth v4 (credentials), Vercel Blob, Zod, Recharts. Self-hosted Satoshi + Inter +
  JetBrains Mono fonts.
- The dashboard is the CMS + orders/analytics backend for separate merchant websites (each
  its own Open-hubb repo). Sites read content/menu/products from the dashboard's PUBLIC APIs
  and POST orders back; a Flot webhook marks orders paid.

## Hard rules (do not break these while fixing)
1. **Multi-tenant isolation** — every authenticated query is scoped to `session.user.id`
   (the merchant id, from `lib/auth.ts`). Never introduce a query that could return another
   merchant's data. If you find one, that's a top-priority bug to fix.
2. **Do not touch the production database** or rotate secrets. Use the DEV Neon branch only
   if needed; prefer static reasoning + `npm run build`.
3. One logical fix per PR; keep diffs minimal; don't reformat unrelated code.
4. Merchant sites must degrade gracefully — every dashboard fetch has a bundled fallback so
   the live site NEVER renders empty. Don't remove those fallbacks.

---

## PART A — Dashboard (`Open-hubb/flot-website-dashboard`)

### Repo map (key paths)
- `prisma/schema.prisma` — Merchant, Order, CustomerOrder(+Event), SiteContent, MenuContent,
  Product, CmsPage, CmsMedia, WebsiteAnalyticsEvent, InAppNotification, NotificationPrefs.
- `app/(dashboard)/**` — overview, transactions, analytics, payouts, customers,
  orders(+[id]), products, menu, cms, website-analytics, notifications, settings.
- `app/(auth)/**` — login, forgot-password, set-password.
- `app/api/public/**` — menu/[id], site-content/[id], products/[id], order, track, tracker.js.
- `app/api/cms/**` — menu, site-content, products(+[id]), media (auth-scoped editors).
- `app/api/webhooks/flot/[merchantId]` — payment webhook receiver.
- `lib/*.ts` — auth, admin-auth, crypto (safeEqual), site-content, menu-content,
  cms-validation, format, db, app-url.
- `middleware.ts`.

### Bug classes to hunt (core of the task)

1. **Data-integrity / logic loopholes**
   - Order→payment linking in `webhooks/flot/[merchantId]`: it links the *most recent*
     `CustomerOrder` with `flotRequestId: null` to an incoming payment. Trace the race where
     order-capture failed but payment succeeded → a payment could bind to the WRONG customer's
     stale pending order. Propose/implement a safer correlation (carry a reference or amount).
   - Order capture (`/api/public/order`) is fire-and-forget on the sites — if it silently
     fails the merchant loses the order but the sale still happens. Find every silent
     `catch {}` / unawaited fetch and decide: retry, surface, or log.
   - Status transitions on CustomerOrder (PENDING→PAID→PREPARING→READY→COMPLETED/CANCELLED):
     verify no invalid transition is allowed and the event log stays consistent.

2. **Null/undefined & fallback correctness**
   - CMS Zod schemas (`lib/site-content.ts`, `lib/menu-content.ts`, `cms-validation.ts`):
     partial payloads must be accepted and defaults backfilled; `withDefaults`/`withMenuDefaults`
     must never throw on old/odd stored shapes. Test empty, partial, extra-field inputs.
   - Money: Decimal fields, `Number()` coercions, currency codes ("Le"/"SLE"/"NLE"), zero vs
     null price ("Market Price"), rounding. Look for `Number(decimal)` precision loss and
     mismatched currencies between order total and Flot amount.

3. **Broken/dead flows & UX loopholes**
   - Any `href="#"`, dead links, empty-states that render nothing, buttons that silently do
     nothing, forms with no success/error state.
   - Retired/duplicate paths (legacy Sanity / `sanityStudioUrl` remnants, unused components) —
     remove safely.
   - Sidebar tab gating (`disabledTabs`, `isMenu`) — confirm the right tabs show per merchant.
   - Pagination/filters on orders/transactions — off-by-one, missing empty results, bad counts.

4. **Async / state bugs (React)**
   - Editors (menu, site-content, products): stale closures, missing `await`, race between
     debounced save and navigation, `useEffect` deps, memoization with large lists (500+ menu
     items). The live-preview postMessage handshake (origin checks, ready race).
   - Double-submit on save/checkout; loading/disabled states.

5. **API robustness**
   - Every `app/api/**` handler: bad/missing body, wrong types, unknown merchant → correct
     status codes (not 500s). CORS + OPTIONS present where sites call cross-origin.
   - `tracker.js` must skip iframe embeds; public GETs must not leak internal fields
     (`passwordHash`, `webhookPassword`, `inviteToken`, `email`).

6. **Build / typecheck / hygiene**
   - `npm run build` clean (no TS errors, no build-failing lint). Remove dead code, tighten
     obvious `any` in logic paths, confirm `.gitignore` covers `.env*`, `*.pem`, `*.pdf`,
     `.claude/`.

---

## PART B — Merchant site repos (the dashboard-wiring)

Each merchant website is a separate Open-hubb repo that reads from the dashboard's public
APIs and posts orders back. Review the **integration glue** on each: the dashboard fetch,
the fallback to bundled data, branding wiring, the live-preview `postMessage` receiver, the
analytics tracker, and (for shops) the order-capture-before-payment step.

| Repo | Stack | Type | Flot merchant id |
|---|---|---|---|
| `Open-hubb/twins-beauty-saloon` | Next.js/React | ecommerce | `5d43fac9-9f53-4892-a7f2-817987d9ea5e` |
| `Open-hubb/dove-group` | static HTML (`index.html` + `marketplace.html`) | marketing + shop | `bfecc381-1a77-4ec2-8ccc-814708a6d4b7` |
| `Open-hubb/SMJ-Esthetics` | Vite + TS (`src/main.ts`) | ecommerce | `cbeac99e-a952-48a8-92ab-62d9e5d54906` |
| `Open-hubb/mamba-point-digital-menu` | static HTML (`public/index.html`) | menu (Cape Leisure) | `25da80ad-9001-4451-b5ec-547c56f6c9d5` |
| `Open-hubb/scarlet-lounge-menu` | static HTML (inline `MENU`) | bar menu | `ca3abf78-bec4-4be6-b81c-8594d6c3c3c5` |
| `Open-hubb/wild-geese-menu` | Next.js/React | menu | `7cbba528-25a2-4164-a879-fe54b9e9eb2f` |
| `Open-hubb/country-lodge-menu` | static HTML (inline `menu`) | menu | `72aac6f3-2000-4542-ab8f-5a43bea7d4e6` |

### What to check on EACH site
1. **Data fetch + fallback** — the site fetches its menu/products/content from
   `https://dashboard.flotme.ai/api/public/<menu|products|site-content>/<merchantId>` and MUST
   fall back to the bundled data if the response is missing/empty/errors. Confirm the fallback
   actually triggers (bad JSON, 404, network fail) and never renders an empty page.
   - React sites (Twins, Wild Geese): the `useMenu` / `useProducts` / `useSiteContent` hooks —
     shared module cache, cancellation on unmount, no infinite refetch.
   - Static sites: the inline `MENU`/`menu`/`products`/`categories` object is the fallback;
     the fetch reassigns it then re-renders. Verify re-render rebinds event listeners (cart
     add-to-cart, category filters, collapsibles) to the NEW nodes — a classic loophole is
     listeners bound once to the old static cards.

2. **Branding wiring** — name/subtitle/phone/logo/currency/email applied from the dashboard
   with fallback to hardcoded. Check selectors still match the DOM; setting `textContent`
   must not wipe animated spans or break layout.

3. **Live-preview receiver** — the site listens for `postMessage` from the dashboard editor
   (`type: "menu-preview"` or `"site-content-preview"`, `source: "flot-dashboard"`), origin-
   checked to `https://dashboard.flotme.ai`, and re-renders. It also posts `preview-ready`
   to its parent when embedded. Verify: only runs in an iframe, origin check present, no
   effect on normal visits, no console errors.

4. **Analytics tracker** — `<script src="https://dashboard.flotme.ai/api/public/tracker.js?id=<merchantId>">`
   present; must NOT double-count and must skip iframe/preview embeds.

5. **Order capture (shops: Twins, Dove, SMJ)** — before opening the Flot payment, the site
   collects name/phone/address/city and POSTs the order + cart to
   `https://dashboard.flotme.ai/api/public/order`. Check: required-field validation, correct
   item mapping `{name,size,qty,price}` + `total` + `currency`, the merchantId is right, and
   capture failure does NOT block the sale (but IS surfaced/logged, not swallowed). Confirm
   the Flot checkout URL points at the right merchant slug (a bug like a bare `pay.flotme.ai/`
   existed on SMJ and was fixed — check the others).

6. **Build** — React/Vite sites must `npm run build` clean; static sites: confirm the injected
   scripts survive the build (Vite keeps inline scripts) and the HTML is valid.

7. **Cross-cutting loopholes** — hardcoded merchant ids must match the table above; retired
   editors (`public/admin.html` on the older menu sites) that write to a Blob the site no
   longer reads should be removed or redirected (they let a merchant "edit" with no effect);
   images should resolve (Blob absolute URLs, no dead `public/...` relative refs on the
   dashboard side).

---

## Workflow
- Map routes/actions/pages (and each site's integration file) once; build a checklist.
- For each confirmed bug: smallest fix, add a guard/validation as needed, verify the relevant
  build passes. Group related fixes into one PR per repo with a clear title/body.
- If a "bug" is actually intended behavior, note it in the report instead of changing it.
- When unsure whether a fix risks multi-tenant isolation or data integrity, STOP and flag it
  rather than guessing.

## Deliverable
1. A report per repo: every bug → `file:line`, what's wrong, the failure scenario, and what
   you changed (or why you left it).
2. One or more PRs per repo, each build-verified, most-impactful first.
3. A "left for follow-up" list (bigger refactors / product decisions — e.g. the webhook order-
   correlation redesign, adding rate limiting, retiring legacy admin editors).

Prefer high-confidence fixes; verify each claim before acting. Don't invent problems.
