# Project Overview

## 1. Project Summary
This is a single-file (`index.html`) front-end prototype of an **"MCM Admin Settings"** panel — an admin dashboard shell (top bar, left sidebar navigation, settings hub grid) built around one fully-realized area: **Billing**. It simulates the billing module of a business-communications / call-center SaaS product (voice, messaging, numbers, teams, etc. — see sidebar categories), built specifically to match the page inventory and UX recommendations in `billing-audit-2026-10-08.md`.

There is no backend, no build tooling, and no framework: everything (markup, CSS, and JavaScript) lives in this one ~3,500-line HTML document and runs entirely client-side. All money, plan, seat, invoice, and payment behavior is a **frontend simulation** — there is no `fetch()`/API layer anywhere, no real payment provider, and no server. The app is explicit about this in its own UI copy ("This demo does not send a real message", "Secure checkout · reference `idem_…`. Pressing Pay twice never charges twice", etc.).

**Primary user workflow:**
1. Land on the Settings hub (a grid of category cards: Captain, Channels, My Account, People, Numbers, Social, Canned Messages, Templates, Integration, Broadcasting).
2. Open **Billing** from the sidebar accordion (or a deep link), landing on **Summary** — the billing "command center."
3. Navigate between 12 fully-built billing sub-pages (see §6) using the sidebar, in-page tabs, or hash URLs.
4. Take real (simulated) billing actions: change plan, buy/remove seats, buy storage, add credit, manage cards, pay an overdue invoice, edit tax/GSTIN details, manage cost centres — each with its own review → confirm → simulated-processing → success flow.
5. Use the floating **Demo controls** panel to reset data, load a different account scenario (trial, past due, suspended, canceled, empty), switch role (to see permission-gated views), or inject a simulated payment failure.

## 2. Main Features
- **Settings hub**: searchable grid of admin setting categories with deep links; only Billing is fully wired, the rest are inert placeholders (unchanged from the base prototype).
- **Collapsible sidebar accordion** with icon set (inline SVG sprite), grouped into labeled sections (Workspace / Account & People / Billing & Compliance), its own search filter, and a collapse-to-icon-rail mode.
- **A complete Billing module** (12 pages — see §6) covering subscription management, usage/overage, licences & resources, credit & payment, invoices, a statement/ledger, modules & access, an honest read-only add-ons catalogue, billing/tax details, and cost centres.
- **Role-based access**: four roles (Admin, Finance, Manager, Agent) with a real permission matrix; non-Admin/Finance roles see a "you don't have access to Billing" screen; individual buttons disable with a tooltip explaining which permission is missing.
- **One shared subscription-status vocabulary** (`trialing`, `active`, `past_due`, `suspended`, `canceled`, `expired`, `none`) driving the Summary alert banner, the Plan page badge, and page content consistently — no contradicting statuses between screens.
- **A real (simulated) checkout flow**: idempotency key per checkout, saved-card or new-card selection, optional credit-balance application, simulated 3-D Secure step, simulated decline/failure handling, and a success screen that links to the generated invoice.
- **Demo controls panel** (floating button, bottom-left): reset data, load a named scenario, switch role, inject a simulated payment failure, add synthetic usage.
- **Deep linking** via URL hash (`#billing/<page>`) for every billing page, including a legacy `#billing/spending` → Usage redirect.
- **Live clock**, **Recently Used** tracking, sidebar/admin search — unchanged from the base prototype (see §11 for what's still a placeholder).

## 3. Application Structure
The document has **two independent `<script>` blocks**:
1. An unnamed `<script>` (~1,100 lines) — the base prototype: clock, sidebar accordion/search/collapse, settings-hub search, Recently Used, and the *original* `showHub()` / `showBillingPage()` / `routeFromHash()` functions.
2. `<script id="bx-app">` (~1,900 lines, wrapped in its own IIFE) — the entire Billing module. On boot, it **monkey-patches** `window.showBillingPage` and `window.showHub` to redirect into its own router, and permanently hides the original `#billingView`/`#usageView`/`#cardGrid` containers whenever it takes over the screen. This means the two scripts cooperate rather than compete: the base script still owns the sidebar/hub chrome, and `bx-app` owns everything under Billing.

There are also **two `<style>` blocks**: the base design-system stylesheet (CSS custom properties, topbar/sidebar/hub/card tokens) and `<style id="bx-style">` (billing-specific component classes: `.bx-modal`, `.bx-btn`, `.bx-plan`, `.bx-alert`, `.bx-pill`, `.bx-chip`, `.bx-switch`, `.bx-form`, etc.), which reuses the base stylesheet's CSS variables (`var(--text)`, `var(--line-strong)`, `var(--blue-dark)`, …) so the two blend into one visual system.

`.content` (inside `.main`) holds, as direct children: `#cardGrid` (hub), `#noResults`, `#billingView`/`#usageView` (now permanently hidden once `bx-app` boots), and — appended at runtime by `bx-app`'s `buildShell()` — `#bxView`, the root of the entire billing module.

## 4. The Billing Module — Architecture

### 4.1 Mental model
```
Sidebar click / hash change
        │
        ▼
window.showBillingPage(label)  ──(monkey-patched)──▶  go(pageKey(label))
        │
        ▼
go(key, tab?)
  · hides #billingView / #usageView / #cardGrid / .main-head
  · shows #bxView, marks the matching sidebar sub-item active,
    opens the Billing accordion section if needed
  · writes history.replaceState('#billing/' + slugified page label)
  · calls render()
        │
        ▼
render()
  · permission check: can('view')? if not → access-denied screen
  · writes a small uppercase "BILLING" eyebrow label + the page title (PAGE_TITLE[key] || PAGES[key]) + blurb (PAGE_BLURB[key])
  · fills the header's right-hand action slot from PAGE_HEADACT[key] (e.g. Summary's "Show my real billing" toggle)
  · (on non-Summary pages) prepends the single highest-priority alert banner
  · calls PAGE_RENDER[key](main) → HTML string → main.innerHTML
  · wires tabs (PAGE_TABS[key]) and any post-render hook (PAGE_AFTER[key])
  · refreshes topbar chrome (wallet balance, notification-bell count)
```
Every click inside `#bxView` is handled by **one delegated listener** registered once in `buildShell()`: it looks at `data-act="<name>"` on the clicked element and calls `ACT['<name>'](element, event)`. Inputs/selects use `data-chg`/`data-inp` the same way. This is why new interactions are added by (a) rendering a button/input with the right `data-act`/`data-chg`/`data-inp` attribute, and (b) registering a matching `ACT[...]` function — there is no per-page `addEventListener` wiring to remember.

`PAGE_TITLE` and `PAGE_HEADACT` are small per-page override registries (most pages just fall back to their `PAGES` label and no header action). Only `summary` currently sets both: title `"Billing summary"` and a header action button that flips `ui.sumMode` between `'example'` and `'real'`.

### 4.2 State model & persistence
A single global object `S` holds the entire account: subscription (`S.sub`), seats/members (`S.members`), cards (`S.cards`), wallet/credit (`S.wallet`), invoices (`S.invoices`), ledger transactions (`S.txns`), add-ons held (`S.addons`), cost centres (`S.costCentres`), billing/tax profile (`S.billing`), activity log (`S.activity`, capped at 200), notification outbox (`S.outbox`, capped at 80), role (`S.role`), and a small `S.sim`/`S.idem` block for demo controls and idempotency tracking.

- **Persistence**: `saveState()` writes `JSON.stringify(S)` to `localStorage['mcm-billing-v2']` after every mutation. `loadState()` reads it back **only if `o.v === 2`** — a schema-version guard, so an incompatible/old cached shape is discarded and a fresh `newState()` is generated instead of crashing on load.
- **Cross-tab sync**: a `storage` event listener reloads `S` and re-renders if another tab changes the same `localStorage` key.
- **Money** is stored as **integer cents** throughout (`S.wallet.balance`, every `amount`/`unit`/`total` field) — never floats — then formatted for display only at render time via `money(cents)`.

### 4.3 Routing
`bx-app` installs its own `hashchange` listener and its own `fromHash()`, independent of (but compatible with) the base script's router:
- `#billing` → opens the sidebar's Billing accordion section without navigating.
- `#billing/<slug>` → slugifies every `PAGES` label the same way (`lowercase, non-letters → '-'`) and matches; unknown slugs fall back to just opening the accordion.
- `#billing/spending` is special-cased to redirect to Usage (the audit's documented legacy URL).
- Called once on boot (so a direct deep link / reload lands on the right page) and again on every `hashchange`.

### 4.4 Role & permission model
| Role | Can do |
|---|---|
| **Admin** | Everything: view, change plan, cancel subscription, manage seats, manage payment methods, view/download invoices, edit billing info, apply coupons, manage credit, manage add-ons, manage cost centres, issue refunds. |
| **Finance** | Read + pay: view, manage payment methods, view/download invoices, edit billing info, apply coupons, manage credit, manage cost centres — **cannot** change the plan, cancel, or manage seats. |
| **Manager** | No billing permissions (`[]`) — sees the access-denied screen. |
| **Agent** | No billing permissions (`[]`) — sees the access-denied screen. |

`can(perm)` checks the current role's permission array; `blockedReason(perm)` additionally explains *why* an action is blocked (wrong role, account past-due/suspended, or no active subscription) so buttons can show a disabled state with a real tooltip instead of silently doing nothing. Switchable live from Demo controls to exercise every role's view.

### 4.5 Alert system (one priority order, shared everywhere)
`alerts()` returns a priority-ordered list consumed by the Summary page banner and the single highest-priority banner shown atop every other billing page:
**suspended → past-due → canceled/expired/no-plan → trial ending → no payment method / card expired / card expiring soon → cancellation scheduled / plan-change scheduled → auto-recharge failed → credit running low.**
Each alert carries a `tone` (`red`/`amber`/`blue`) mapped to the correct ARIA role (`alert` for red, `status` otherwise) and its own primary action button.

### 4.6 Checkout / payment simulation (`checkout()`)
Every money-moving action (plan change, buy seats, buy storage, add credit, pay an overdue invoice) goes through one shared `checkout()` modal flow:
1. **Review** — line items, discount/coupon line, tax line(s), optional "apply credit balance" checkbox, due-today total.
2. **Payment method** — pick a saved card or add a new one inline; disabled for expired cards.
3. **Pay** → `processing` state (button shows a spinner, modal locks against Escape/outside-click) → `attemptCharge()` simulates the gateway (can randomly/deliberately fail via Demo controls → "simulated failure").
4. **3-D Secure step** (`requires_action`) — an explicit "Authenticate this payment" screen with **Complete** / **Fail authentication** buttons, clearly labeled "(simulated)".
5. **Success** — a checkmark screen with the amount, invoice/receipt number, and a note that it was "emailed to …"; **View invoice** jumps straight to the Invoices page detail view.
6. **Idempotency**: a `idem_<random>` key is generated per checkout and reused if `pay()` is called again (e.g., a double-click) — mirrors the audit's P0 recommendation ("idempotency keys on every charge") and is called out in the UI copy itself.

## 5. UI/UX
**Shared visual language**: one modal system (`openModal`/`confirmBox`/`qtyModal`), one button component (`btn()` → `.bx-btn[.primary|.danger|.sm|.link]`), one status-chip component (`chip()`/`statusChip()`), one table helper (`table()`, with built-in empty-state and horizontal scroll), one quote/line-item table renderer (`quoteTable()`), consistent `kpi()` tiles, and a `card()` wrapper reused by every page.

**Modals** (`openModal`): `role="dialog" aria-modal="true" aria-labelledby`, a focus trap (Tab/Shift+Tab cycle, implemented by hand), Escape-to-close, click-outside-to-close (unless `sticky`, used during an in-flight payment), focus restored to the triggering element on close, and a modal **stack** (nested dialogs close innermost-first).

**Accessibility**: switches are real `<input type="checkbox">` behind a styled `.bx-switch`; radios for payment-method selection are real, focusable `<input type="radio">` (not `display:none` as the audit found in the real product); tabs use `role` conventions are approximated with plain buttons + `aria-selected`-style active classes; progress/usage bars carry `role="progressbar"` with `aria-valuenow/min/max`; status is never color-only (every chip has a text label); a dedicated access-denied screen is shown (not just a blocked click) for roles without `view`.

**Responsive**: `@media(max-width:900px)` collapses the plan/resource grids to one column; `@media(max-width:640px)` adjusts modal padding and the demo panel width; `@media print` makes the invoice-detail view printable (an invoice "PDF" is produced via the browser's native print-to-PDF, not a fake screenshot library — and the UI says so).

## 6. The 12 Billing Pages

| # | Sidebar label | `PAGES` key | Purpose | Notable interactions |
|---|---|---|---|---|
| 1 | Summary | `summary` | Billing command center: current plan, 4-tile KPI row, credit/failed-payment status, add-ons on the plan, payment details | Two tabs (Current billing / Invoice history) and a header "Show my real billing" ⇄ "Show an example" toggle — see §7 |
| 2 | Usage | `usage` | Allowances vs. actual use, overage, spend breakdown, call list | Date range, Export CSV, "hide/show breakdown" |
| 3 | Plan | `plan` | Manage the subscription itself | Cancel, Revoke request, Renew, promo code |
| 4 | Plans & pricing | `pricing` | Compare plans / Monthly vs Annual | **Change Plan** wizard: compare → review (proration/tax/total) → pay → confirmation |
| 5 | Licences & resources | `resources` | Seats, numbers, storage, AI agents | Tabs: Licences / Numbers / Storage / AI — buy/remove seats, buy storage, invite |
| 6 | Credit & payment | `credit` | Prepaid credit + saved cards | Tabs: Add Credit / Auto top-up / Payment Methods / Credit history |
| 7 | Invoices | `invoices` | Find and prove a charge | Search, status/date filters, detail modal, Download (print), CSV export |
| 8 | Statement | `statement` | Running ledger | Date range, CSV export |
| 9 | Modules & access | `modules` | Which plan modules are on/off | Read-only, checkbox filter |
| 10 | Add-ons | `addons` | Extras catalogue | **Read-only** — "Contact sales" only, no working purchase flow (see §9) |
| 11 | Billing details | `details` | Legal name, address, tax ID (GSTIN/VAT/EIN/ABN) used on invoices | Country-aware tax-ID validation (e.g., 15-char GSTIN checked against billing state) |
| 12 | Cost centres | `costcentres` | Reporting labels for spend | Add / Archive / Restore / Save with validation; a real "where the seat cost lands" allocation report by member assignment |

`#bxItem "Billing details"` is **not** in the static sidebar HTML — `bx-app` inserts it at boot, immediately before "Cost centres", because it's a page the base prototype's markup never anticipated.

## 7. Core User Flows

**Summary: example vs. real billing (and its two tabs):**
Summary opens in **example mode** by default — a fixed, fully-worked sample account (`EXAMPLE_SUMMARY`: "Contact centre Premium", 8 seats, a full add-ons list, a 23-row invoice history) shown behind a yellow "Example only" notice, so the page is never empty on a fresh demo account. The header's **"Show my real billing"** button (`ACT['sum.toggle']`) flips `ui.sumMode` to `'real'`, which switches every card to the live simulated account (`S`) — current plan/KPIs from `plan()`/`seatInfo()`/`quoteNext()`, add-ons from `S.addons`, payment details from `defaultCard()` — and the button relabels itself "Show an example". This toggle is independent of, and doesn't affect, any other billing page (Plan, Resources, etc. always show live `S` data). Within either mode, the page has two tabs (`PAGE_TABS.summary`): **Current billing** (plan card, 4-tile KPI row, credit/failed-payment status line, "Add-ons on the plan" table, "Payment details") and **Invoice history** (the full invoice table — example rows don't open; real rows are the live `S.invoices`, newest first).

**Change plan (upgrade/downgrade):**
Plans & pricing → pick Monthly/Annual → "Choose" a plan card → **Review** step (current vs. new plan, effective date, proration or "applies at renewal" for a downgrade, tax, total) → **Continue to payment** → `checkout()` → success → Summary/Plan reflect the new plan immediately (or a "Plan change scheduled" alert if it's deferred to renewal). An in-flight scheduled change can be undone from the Summary alert ("Cancel the change").

**Buy seats:**
Licences & resources → Licences tab → **Buy seats** → quantity stepper with a live prorated quote → `checkout()` → seats added, member table updates, Summary's seat KPI updates.

**Buy storage:**
Licences & resources → Storage tab → **Buy Now** → GB stepper with live prorated cost → `checkout()` → extra GB added to the allowance.

**Add credit / auto top-up:**
Credit & payment → Add Credit tab → pick a preset or custom amount ($10–$500) → **Add $X** → `checkout()` (tax-free; prepaid credit is taxed when it's *used*, not when it's added) → wallet balance updates everywhere (topbar wallet, Summary). Auto top-up and the low-balance alert are separate forms with their own validation (e.g., refill amount must exceed the threshold; auto top-up can't be enabled without a valid, non-expired default card).

**Manage payment methods:**
Credit & payment → Payment Methods tab → **Add card** (inline form, no real card data leaves the browser) / **Make default** / **Edit** / **Remove** (blocked with an explanation if it's the only card and something depends on it — auto top-up or an active renewal).

**Cancel / reactivate:**
Plan page → **Cancel Subscription** → reason + confirm → subscription flips to "ends on `<date>`" (data preserved, still usable until then) → **Revoke Request** undoes it. A fully `canceled` account shows **Reactivate** instead.

**Pay an overdue invoice (dunning):**
A `past_due`/`suspended` account shows a persistent red Summary alert → **Pay now** → `checkout()` against that specific invoice → on success the subscription returns to `active` and an email notification is logged to the outbox.

**View / "download" an invoice:**
Invoices → search/filter → open a row → detail modal (supplier GSTIN/SAC, line items, tax breakdown, payment method) → **Download** triggers the browser's print dialog over a `.bx-printable` view (never claims to be a real server-generated PDF).

**Cost centres:**
Cost centres → edit codes/names/ledger references inline → **Add cost centre** / **Archive** / **Restore** → **Save changes** (disabled until something is dirty; blocked while any row fails validation) — plus a read-only "Where the seat cost lands" report driven by each member's cost-centre assignment.

## 8. Important Components / Functions

| Name | Role |
|---|---|
| `boot()` | IIFE that wires the whole module on load: builds the shell, patches `showBillingPage`/`showHub`, injects the "Billing details" sidebar item, wires the wallet/notification-bell chrome, starts the hash router |
| `go(key, tab)` / `render()` | Navigation + the single re-render entry point for the current page |
| `PAGE_RENDER[key]` / `PAGE_TABS[key]` / `PAGE_AFTER[key]` | Per-page render function, tab definitions, and post-render hook registries |
| `ACT['action.name']` | Delegated handler registry for every `data-act` button/control across all pages |
| `S` / `loadState()` / `saveState()` / `newState(scenario)` | The whole account's state, its versioned persistence, and scenario generator |
| `checkout(opts)` | Shared review → pay → 3DS → success modal flow for every money-moving action |
| `openModal(opts)` / `confirmBox(opts)` / `qtyModal(opts)` | Modal primitives (focus trap, stack, Escape) |
| `buildQuote(lines, opts)` / `proratedQuote(key, desc, cents)` | Pricing engine: subtotal, discount, tax lines, total, built from integer-cent line items |
| `alerts()` / `alertHtml(a)` | The shared, priority-ordered banner system |
| `can(perm)` / `blockedReason(perm)` | Permission checks and the human-readable reason a button is disabled |
| `money(cents)` / `fd(iso)` / `esc(s)` / `csvCell(v)` | The one money formatter, one date formatter, HTML-escaper, and CSV-injection-safe cell formatter used everywhere |
| `activity(kind, desc)` / `sendEmail(subject, body)` | Append-only audit log and notification outbox |
| `buildDemo()` / `SCENARIOS` | The floating Demo controls panel and its named account scenarios |

## 9. Dependencies & External Resources
**None at runtime.** No CDN scripts, no CSS/JS libraries, no web fonts, no API endpoints, no `fetch()`/`XMLHttpRequest` anywhere in the file. Icons are an inline SVG `<symbol>` sprite (sidebar/hub categories) plus small hand-written inline `<svg>` elements (KPI tiles, alerts). *(During development, `jsdom` was installed temporarily via `npm` purely to headlessly exercise the page's JavaScript for debugging and was removed afterward — it is not part of the shipped app and leaves no trace in `index.html`.)*

## 10. Current State
**Fully implemented**: all 12 billing pages with real, distinct content (not a shared generic shell); the full change-plan/cancel/renew lifecycle; seat and storage purchase with proration; credit top-up, auto-recharge, and low-balance alerts; payment-method management with expiry badges; invoice search/filter/detail/CSV/print; a statement ledger; an honest read-only modules page; an honest read-only add-ons catalogue; country-aware tax/GSTIN validation; cost centres with a real spend-allocation report; role-based access with four roles; one shared alert/status system; hash deep-linking (including the legacy `spending` redirect) for every page; a demo-scenario/role-switching control panel.

**Still placeholder** (outside Billing, unchanged from the base prototype): every non-Billing hub category and sidebar item (Captain, Channels, My Account, People, Numbers, Phone System, Integration, Social, Canned Messages, Templates, Broadcasting, 10DLC Compliance) — clicking them just returns to the hub; top-bar nav tabs other than Settings; top-bar global search, notification-bell contents beyond an unread-count dot, and theme toggle.

## 11. Known Limitations (by design)
- **Everything is a frontend simulation.** No real Stripe/payment provider, no real server, no real PDF generation (uses the browser's print dialog), no real email delivery (logged to an in-memory `S.outbox` instead). The UI says so wherever it matters, per the explicit "don't fake backend functionality" requirement this module was built to.
- **Add-ons purchasing is intentionally not implemented** — the page shows a catalogue with "Contact sales" buttons only; there is no working buy/remove flow, matching the audit's documented product truth for this page.
- **Single-device state**: `localStorage` only; no real multi-user/multi-session backend, though the `storage` event listener keeps multiple tabs *on the same browser* in sync.
- **The base prototype's non-Billing areas remain placeholders** — see §10.

## 12. Future Development Guidelines
- **Two script blocks, two routers.** If you need to touch navigation, know which one you're in: the base script's `showHub()`/`showBillingPage()`/`routeFromHash()` own the hub/sidebar chrome; `bx-app`'s `go()`/`render()`/`fromHash()` own everything under Billing (and monkey-patch the base functions on boot). Don't re-wire sidebar clicks in the base script expecting them to reach Billing pages directly — they're intercepted.
- **Add a new billing page** by: adding an entry to `PAGES` (and `PAGE_BLURB`), writing `PAGE_RENDER.<key> = () => ...`, optionally `PAGE_TABS.<key>` / `PAGE_AFTER.<key>`, adding a sidebar `.sub-item` (or injecting one at boot like "Billing details" does), and confirming its slug doesn't collide with an existing one in `fromHash()`.
- **Add a new interactive control** by rendering it with a `data-act="your.key"` (or `data-chg`/`data-inp`) attribute and registering `ACT['your.key'] = (el, e) => { ... }` — do not attach a one-off `addEventListener` inside a page's render function; it won't survive the next re-render (the whole page's `innerHTML` is replaced on every `render()` call).
- **Any new money-moving action** should go through `checkout()` (for consistency: idempotency, 3DS step, success screen) rather than a bespoke payment UI.
- **Keep the state schema change-safe**: if you add/rename a field on `S`, either keep `loadState()`'s `v === 2` guard meaningful (bump to `v === 3` and handle migration, or accept that older cached `localStorage` data will be discarded and a fresh scenario generated — which is what currently happens for genuinely incompatible data).
- **Preserve `localStorage['mcm-billing-v2']`** as the single source of truth for billing state; don't reintroduce the old `mcm-sample-card` key or a separate payment-method store — it no longer exists.
- Keep using **integer cents** for all money; format only via `money()`. Never introduce a raw float dollar amount into state.
- This project deliberately has **zero runtime dependencies** — don't add a CDN script, font, or npm-built asset without confirming with the user first.

## 13. Change-Safety Notes
After any change, manually re-check (no automated tests exist in this project):
- **Every sidebar billing sub-item** still opens its page with real content (not a blank `#bxMain` or a stale previous page) — click through all 12.
- **Hash routing**: reload directly on `#billing/summary`, `#billing/pricing`, `#billing/resources`, and the legacy `#billing/spending` (should land on Usage).
- **Role switching** (via Demo controls): confirm Manager/Agent see the access-denied screen, and Finance can view/pay/manage cards but cannot change the plan or manage seats.
- **Checkout flow**: run at least one real flow (e.g., buy seats) through review → pay → simulated-failure retry → success, and confirm the resulting invoice appears correctly on the Invoices page.
- **Alert banner priority**: a suspended/past-due account should always show that alert above anything else (trial-ending, low-balance, etc.) on every billing page, not just Summary.
- **Add-ons page**: confirm it still has no working buy/remove action — only "Contact sales" — if this page is ever touched again.
- **State persistence**: after any action, reload the page and confirm the change survived (read from `localStorage['mcm-billing-v2']`); if you changed `S`'s shape, confirm old cached data doesn't crash `render()` (it should just reset via the `v` guard).
- **Responsive**: resize to ~900px and ~640px and re-check the plan-comparison grid, resource tabs, and any open modal.
