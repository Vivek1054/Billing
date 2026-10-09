# Billing Module: Research, Audit and Implementation Blueprint

Date: 2026-10-08. Scope: `/admin-settings/billing/*` and every billing-adjacent screen in the repo.
Method: four parallel read-only code audits, with every file in scope read in full. Nothing was executed except the unit tests in `tests/`, and no code was changed.

**Evidence labels** (as requested):
✅ VERIFIED WORKING · ❌ VERIFIED BROKEN · ⚠️ PARTIAL · 🟡 NOT IMPLEMENTED · 🔵 RECOMMENDED · ❓ NOT VERIFIABLE

**The most important fact in this report:** the repo contains the frontend only. There is no billing, payment, invoice, plan or card backend in the repo, and no Stripe webhook receiver. The backend is a separate, source-less `default-api` service. Anything the server does with money (re-pricing, tax, idempotency, webhook handling, authorisation) is ❓ NOT VERIFIABLE from here. Wherever the UI exists, the rule applies: **"UI exists — backend behavior not verified."**

Paths below are relative to `src/` unless they start with another folder name.

---

# 1. Billing Route Tree

```text
/admin-settings/billing/                  (index; renders Summary; NOT admin-guarded, see Bug B1)
├── summary          Summary                 adminOnly
├── usage            Usage                   adminOnly
├── plan             Plan                    adminOnly
├── resources        Licences & resources    adminOnly   (tabs: Licences | Numbers | Storage | AI)
├── purchase         Credit & payment        adminOnly   (tabs: Add Credit | Payment Methods)
├── invoices         Invoices                adminOnly
├── statement        Statement               adminOnly
├── modules          Modules & access        adminOnly
├── add-ons          Add-ons                 adminOnly   (read-only catalogue)
├── cost-centres     Cost centres            adminOnly   (reporting labels only)
└── spending  ──redirect──> usage

Billing-adjacent routes outside /billing:
/renew-plan                         forced renewal page for expired plans (PlanPendingGuard)
/pricing, /sign-up, /payment,       public acquisition funnel: plan pick, signup, first payment,
/payment-success, /phone-lines      DID selection
/admin-settings/numbers/all-numbers/add-number-new   DID purchase wizard with payment modal
/admin-settings/people/add-users                     seat purchase through "Add users"
/admin-settings/calling-rates/destinations           the rate card (linked from Usage)
/reports/call-history                                itemised calls (linked from Summary/Usage)
```

Route definitions are generated from one list, `pages/admin-settings/billing/billing-sections.ts`, shared by the sidebar and the router (`router/index.tsx:1379-1399`). A missing page component is a build error, which is good design. ✅

| Route | Page | Parent | Purpose | Reached from | Status |
|---|---|---|---|---|---|
| `/billing` (index) | Summary | Admin settings | plan, next bill, card, recent invoices | sidebar, Summary links | ❌ not guarded |
| `/billing/summary` | Summary | billing | what am I paying / when | sidebar | ⚠️ |
| `/billing/usage` | Usage | billing | allowances vs use, spend breakdown, CSV | sidebar, Summary | ⚠️ |
| `/billing/plan` | Plan | billing | plan, next billing, change/cancel/renew, seats tabs | sidebar, plan widget, search | ⚠️ |
| `/billing/resources` | Licences & resources | billing | seats, numbers, storage, AI agent cost | sidebar | ⚠️ |
| `/billing/purchase` | Credit & payment | billing | top-up, auto-recharge, low-balance alert, cards | sidebar, avatar menu "Add Funds", dialpad | ⚠️ |
| `/billing/invoices` | Invoices | billing | list, row expand, detail modal, CSV | sidebar, Summary | ⚠️ |
| `/billing/statement` | Statement | billing | ledger, wallet, meters | sidebar | ⚠️ |
| `/billing/modules` | Modules & access | billing | which plan modules exist, who can see them | sidebar | ⚠️ |
| `/billing/add-ons` | Add-ons | billing | catalogue, nothing purchasable | sidebar | 🟡 purchase |
| `/billing/cost-centres` | Cost centres | billing | reporting labels | sidebar | 🟡 no consumer |
| `/billing/spending` | n/a | billing | legacy URL | bookmarks | ✅ redirect |

**Hidden and conditional screens:** Change Plan dialog, cancel/revoke confirmations, renewal payment dialog, seat drawer, Add Card modal, Invoice detail modal, rate-details dialog, the global `UpgradePlanWidget` bar, `AccountStateBanner`, `PlanUpgradeRequired`. All are covered in section 6.

---

# 2. Complete Page / Subpage Inventory

| Page | Goal | Primary CTA | Secondary CTAs | Data / API | Provider |
|---|---|---|---|---|---|
| Summary | see cost and next charge | Change plan | Replace card, Usage, Call history, View all invoices | `current-plan-detail`, `card/list`, `billing/list`, `tenant/report/call-list` | none |
| Usage | understand overage | Export CSV | date range, hide/show breakdown | `call-list` (paged, max 2,000 rows), `current-plan-detail`, session `company_info` | none |
| Plan | manage the subscription | Change Plan / Upgrade Now / Renew Now | Cancel Subscription, Revoke Request, Download Invoice, rates | `current-plan-detail`, `plan/list`, `calculate-tax`, `plan/change/request`, `payment/plan-renew`, `upgrade-trial-plan` | Stripe |
| Licences tab | buy and remove seats | Buy seats | Remove seats, undo, assign | `user/licenses`, `user/buy-license`, `user/revoke-license`, `calculate-tax` | Stripe |
| Numbers tab | see DIDs and cost | n/a | n/a | `numbers/list` | none |
| Storage tab | buy storage | Buy Now | n/a | `get-bucket-size`, `get-prorated-cost`, `buy-extra-storage` | Stripe |
| AI tab | per-agent cost | n/a | n/a | `ai/agent/billing/list` | none |
| Credit & payment, Add Credit | add prepaid funds | Pay $X | auto-recharge, low-balance alert | `payment/add-fundV2`, `user/update-auto-recharge-settings`, `user/update-low-balance-settings` | Stripe |
| Credit & payment, Payment Methods | manage cards | Add card | Set primary, Delete | `card/list`, `card/add`, `card/set-default`, `card/delete` | Stripe |
| Invoices | find and prove a charge | open invoice | search, year/date, CSV | `billing/list` | none |
| Statement | ledger | n/a (read-only) | none | `billing/list` (all pages), `current-plan-detail`, session wallet | none |
| Modules & access | explain missing modules | n/a | checkbox filter | session features only | none |
| Add-ons | see extras | n/a (cannot buy) | rates link | session features plus hardcoded catalogue | none |
| Cost centres | label spend | Save | Add, Archive | company-default template (`settings.cost_centres`) | none |
| Renew plan | pay to regain access | Renew / Upgrade | Logout | `current-plan-detail`, `calculate-tax`, `plan-renew`, `upgrade-trial-plan` | Stripe |
| Pricing, Signup, Payment, Phone lines | acquire and onboard | Subscribe / Pay | trial | `plan/list`, `calculate-tax`, `signup`, `signup-on-trial`, `initial-plan-payment` | Stripe |
| DID purchase wizard | buy numbers | Pay | n/a | `did/billing-details`, `didw/create-order`, `did/complete-did-process` | Stripe |

**Required permission, current implementation:** every billing section requires `user_info.role === 'ADMIN'` (`hooks/rbac.tsx:58`, `router/protected-route.tsx:46-48`). The `billing.action.{view, change_plan, request_plan}` keys exist but only gate buttons inside pages that already require ADMIN (see section 17).

---

# 3. Page-by-Page Analysis

## 3.1 Summary
**Layout:** an optional alert banner, then the "Your plan" card, a 4-tile strip (licences, credit, next bill, this month's calling), "How it gets paid" (card), "Where the charges came from", and "What you have paid" (last 5 invoices).
**Components:** `AdminPage`, `SettingCard`, `Tile`, Skeleton, alert from `lib/billing-alerts.ts`.
**Interactive:** six `<Link><Button>` navigation controls. ✅ they navigate. ❌ nested interactive elements (a button inside a link), so two tab stops per control.
**Data:** all four queries are real API calls. ✅ In the Vercel demo mode only the plan call is mocked, so the page looks nearly empty there. ❓
**Findings:**
- ❌ A card-list failure reads as `cards = []`, so the page shows the amber "No payment method saved" alert and "No card saved". This is a false alarm on a money page (`summary:99`).
- ❌ "This month's calling" shows `$0.00` on error and when `call_stats` is absent (`summary:114-126, 346`; `spend-breakdown.ts:170`). The file's own rule is that unknown is never zero.
- ⚠️ With no primary card it falls back to `list[0]` yet says "This is the one that gets charged".
- ⚠️ The invoice list amount is `tax_detail.total_amount ?? total_amount` with no "incl. tax" note.
- ⚠️ The alert uses `role="status"` for stop conditions. `role="alert"` is the correct role.
- ⚠️ "Credit balance" reads `plan.credits` here and `company_info.amount` on Statement and dialpad. Two sources for one number. ❓ which is authoritative.

## 3.2 Usage
**Layout:** header with Export CSV, a date filter, "This period", an allowances table (calls, SMS), a spend breakdown (by person, by destination, by direction), and a link to call history.
**Findings:**
- ✅ Loading, error, empty and truncation states ("covers X of Y") are all handled.
- ❌ The destination table's share is computed against the per-person total (`usage:369, 561-566`), so shares can exceed or miss 100% when charged calls have no extension.
- ❌ CSV formula injection: the label is `contact_name`, which an external caller can control (`usage:371-396`). Labels starting with `= + - @` are not neutralised.
- ❌ Export stays enabled on error or an empty period, and the filename can read `usage-undefined-to-undefined.csv`.
- ❌ A band of 99.5-99.99% used rounds to 100 and is shown red as "over" while the overage is 0 (`billing-usage.ts:64,78`).
- ⚠️ Overage cost uses hardcoded rates from `plan-catalogue.ts`, which ❓ may differ from what the server charges. The page claims figures "match your invoice", which is ❓ unproven.
- ⚠️ `UsageBar` has no `role="progressbar"` or aria values. Tables have no `<caption>` or `scope`.

## 3.3 Plan (`plan/index.tsx`, 966 lines)
**Layout:** header with Cancel Subscription or Revoke Request; a banner "plan changes at next renewal"; left column (Current Plan, Next Billing, AI allowances, usage table, comparison table, requested plan); right column tabs (License, DID, Storage, AI).
**Lifecycle the code recognises:** `plan_status === 'EXPIRED'` drives logic. `is_trial` is `'Y'` or `'N'`. A pending change lives in `plan_info.plan_requests` with `action_type` UPGRADE, DOWNGRADE or CANCEL.
**Findings:**
- ❌ **Stale data bug:** seven call sites invalidate `['getMyPlanDetails']`, but the hook registers `['useGetMyPlanDetails']` (`hooks/common.ts:175`). The invalidation matches nothing, so seat and plan numbers stay stale after buy, revoke, upgrade and cancel until a reload. Only the renewal/payment path refreshes (4 s delayed refetch). Call sites: `change-plan.tsx:216,258`, `plan/index.tsx:202,215`, `license-management:155,197`, `add-users/index.tsx:101`.
- ❌ Status badge fallback is green "NA" for `SUSPENDED`, `ON_HOLD`, `CANCELLED` (`RequestedPlanStatusMap` only knows P, C, A, ACTIVE, D, E, EXPIRED, R, U, S).
- ❌ Price displays `$undefined/month/user` when data is missing; "Last billing" prints `|| 0`.
- ❌ Revoke Request button's spinner watches the wrong mutation flag (`plan/index.tsx:356`).
- ⚠️ Cancel Subscription is not gated by trial or expired state.
- ⚠️ Missing `key` on `TabsTrigger`; the tab is read from `?tab=` once and never written back.
- ⚠️ Missing expiry date reads "0 days left", so Renew Now appears at ≤3 days. Unknown is treated as zero.
- ⚠️ Annual vs monthly is only a label (`plan_duration === 12`). Durations 3 and 6 are labelled "month".

## 3.4 Change Plan dialog (`plan/change-plan.tsx`)
- ✅ Prices come from the plan list, unpriced plans are disabled with an inline warning, scheduled change vs immediate upgrade is offered with a radio group.
- ❌ **Wrong upgrade/downgrade label across billing cycles:** it compares the selected-period price with the current-period price (`change-plan.tsx:73-75`). A yearly price always looks "higher" than a monthly one, so toggling Annual makes the same plan an "Upgrade", and the reverse labels a higher tier "Downgrade".
- ❌ If `request_plan` permission is false, a **Downgrade** button goes straight to payment at full price (`:465-475`).
- ⚠️ Marketing copy is hardcoded ("Upgraded Savings", the same eight bullets for every plan).
- ⚠️ No price breakdown or proration preview before paying; only "Pay $X". 🟡
- ⚠️ Only Monthly and Annual can be selected, though maps contain 3 and 6.
- ⚠️ Fixed-width plan cards (`w-1/3`) break on narrow screens; close controls are `div onClick`.

## 3.5 Licences tab (`plan/license-management`)
- Seat model fields: `total_licenses`, `used_licenses`, `free_licenses`, `revoked_licenses`, `free_revoked_licenses`, `payable_licenses`.
- ✅ Counter bounded by the plan cap, Remove-seats mode, per-seat undo, bulk removal of idle seats, revoke deferred to the next billing date.
- ❌ Restore button for an **unassigned revoked** seat is hidden unless "Remove seats" mode is on (`:340-347` vs `:362`).
- ⚠️ A hardcoded cap of **50** is used when the plan limit has not loaded.
- ⚠️ `charge_amount: getTaxes?.total_amount || 0` is sent if the quote failed, and Pay is not disabled while the quote is missing (`:203, 725-738`).
- ⚠️ "Buy seats" is blocked for pending downgrade (message wrongly says "upgrade your plan") but not for EXPIRED or pending CANCEL.
- ⚠️ The idle-seat list relies on the server sorting unassigned rows last and a paging trick (`:115-149`). ❓
- 🟡 No pending-invite or deactivated-user count anywhere.

## 3.6 Storage tab
✅ Sends GB only (no amount) to the buy endpoint, which is the safest payload shape in the app. ⚠️ If the prorated-cost call returns nothing, the button shows the full price while the server may charge the prorated one. ❌ `console.log` of the payment method (`storage:68`). 🟡 No error state (failure reads "No data available"). Inline styles block dark mode.

## 3.7 Numbers and AI tabs
`dids-list` and `agent-costing` are read-only tables. ✅ Missing values show "Not available yet". 🟡 no filters or date range. The response parser tries seven shapes, so the real shape is ❓.

## 3.8 Credit & payment, Add Credit (`purchase/top-up.tsx`)
- Fixed amounts only: $20, 50, 100, 150, 200 (no custom amount). 🟡 No balance is shown on this page.
- ✅ Saved-card vs new-card paths, 3DS handled.
- ⚠️ After 3DS success there is no confirmation call; the page assumes a webhook credited the balance and navigates to Invoices, where the entry may not exist yet (the DID flow does it better).
- ❌ `console.log` of the add-fund response (`:67`), unguarded `JSON.parse` of the stored alert setting.
- ❌ Amount tiles are `div onClick` with no role, keyboard support or `aria-pressed`.
- Auto-recharge: ❌ turning it OFF posts immediately with no rollback on failure; 🟡 no precondition that a card exists; 🟡 no check that refill covers threshold. It is gated on `billing.action.view`, which is a view permission, not an auto-recharge flag.
- Low-balance alert: ⚠️ no maximum, "Infinity" edge, 🟡 no channel or recipient choice. Typo: "Receive an notification".

## 3.9 Payment Methods (`purchase/manage-cards.tsx`, `add-card-modal.tsx`)
- ✅ Add, set primary (with confirmation), delete (with confirmation). Primary card cannot be deleted from the UI. Empty state exists.
- ❌ No expiry month, no expired or expiring badge (`cardExpiresSoon` exists and is unused here). Copy says "you can update … cards" but no edit exists.
- ❌ Brand is uppercased without a null guard; a missing brand throws.
- ❌ Card actions are `div onClick` icons with no role, label or keyboard access.
- ⚠️ Cards are saved by `createPaymentMethod` plus attach, not a SetupIntent (no `confirmCardSetup`), so off-session auto-recharge may fail SCA. This matters particularly for Indian-issued cards. ❓ server may create the SetupIntent.
- ❌ The cardholder name is required but never sent anywhere (not even Stripe `billing_details`).
- ❌ Missing `return` after the "Stripe was not working" alert (`add-card-modal.tsx:39`).
- ❌ Saved-card list shows a hardcoded Visa logo for every brand; radios are `display:none` so not keyboard-focusable; the "Save this card" checkbox uses `value=` instead of `checked=` and desyncs after reset.
- 🟡 No guard against deleting the last card while auto-recharge or a plan depends on it.

## 3.10 Invoices
- Columns: date, invoice no. (`bill_no`), amount, description, paid with, status, action. Filters: invoice-number search, year, date range.
- ❌ Refunded and pending invoices cannot be opened as invoices; only `completed` opens the detail, others show a "why it failed" dialog.
- ❌ CSV exports only the loaded page (not all filtered rows) and has no formula-injection guard; `invoice/constants.ts` has a second CSV helper that does not even quote commas.
- ❌ The PDF is a raster screenshot (`dom-to-image` → `jsPDF`), always named `invoice.pdf`, with page breaks that can slice rows. The same page says "A formal PDF invoice is not available yet".
- ❌ The invoice modal lacks: supplier tax ID, customer tax ID, company name, due date, currency code, SAC/HSN, place of supply, CGST/SGST/IGST split.
- ⚠️ Line tax is recomputed in the browser (`perSeatTax × seats`) and may differ by cents from the server total shown in the list.
- ⚠️ "Processing fee" is derived as `total − subtotal − tax`, so rounding looks like a fee.
- 🟡 No status filter, no retry-payment for failed invoices, no email-invoice, no credit notes. The `DELETE /api/payment/delete/:id` wrapper exists with no caller (correctly not exposed in the UI; see Security).
- `getInvoiceDetails` (`/api/payment/info/:uuid`) is also unused; the modal shows list-row data.

## 3.11 Statement
- ✅ Error and empty states; read-only.
- ❌ "Charged to date" is labelled "this plan period" but the query has no date filter (all-time total). Missing amounts count as 0 in totals but "Not available yet" in rows.
- ❌ Table is capped at 12 rows with no pagination or export, though it claims "every charge".
- ⚠️ Totals are summed with float addition without `roundMoney`; refunded or cancelled rows appear but are in no total.
- ⚠️ Meter thresholds 75/90 here vs 80 on Usage and in `allowance-meter`.
- ⚠️ The Licences meter uses `last_billing.total_license` as "used", which may differ from the seats actually in use. ❓

## 3.12 Resources (tab host)
❌ Tab buttons have no tab roles or arrow-key support; tab state is not in the URL. ⚠️ Expiry uses session `plan_status === 'EXPIRED'` only. ⚠️ All four tab modules are imported eagerly.

## 3.13 Modules & access
✅ Honest, read-only, accessible checkbox. ❌ It can report Billing as "Hidden / Not provisioned" while the admin sees the Billing menu (billing access depends only on role). ❌ Its non-admin footnote branch is unreachable because the page is admin-only.

## 3.14 Add-ons
✅ Honestly labelled "coming soon", no fake purchase button. ⚠️ Prices (`$20` for international, `$45` AI voice) are hardcoded client data ("set by the product owner"); two catalogues (`ADD_ONS` and `PLAN_ADD_ONS`) use different ids and sets. 🟡 No "contact sales" action.

## 3.15 Cost centres
✅ Validation works, unit-tested. 🟡 Nothing reads `settings.cost_centres`. The allocation and splitting functions in `lib/cost-centres.ts` have no UI. ❌ A blank row added by mistake cannot be removed (archive only) and blocks Save. ❌ No `onError`; a background refetch can silently overwrite unsaved edits. ⚠️ Stored inside the "Company Default" user template, so another screen saving that template can erase it.

## 3.16 Renew plan (`/renew-plan`)
✅ Navigation trap (logout/redirect semantics), uses tax quote, correct responsive layout. ❌ Button reads **"Pay $0"** while the quote is missing (`renew-plan.tsx:389`). ⚠️ Two sources for trial status. ⚠️ The forced redirect depends on a client flag set by matching the server message text "no longer active" (`login/index.tsx:170`), which is brittle. A session that expires mid-use is only bannered, not forced here.

## 3.17 Signup, pricing, onboarding
- ❌ **High:** the Pricing "Annually" toggle is cosmetic. Checkout hardcodes `plan_duration = 1` (`signup/index.tsx:140`; `pricing:310-312`). A visitor sees the annual price and is charged monthly.
- ❌ **High:** on the payment step, Pay is live while the tax quote is loading or failed; button reads "Pay $0", sends `amount: 0` and an empty or stale `tax_calculation_id` (`signup/payment.tsx:215-220,358-361`).
- ❌ Pricing's compare table is hardcoded mock data (Basic/Pro/Enterprise at "$20/$30" each) and contradicts the live cards. Dead "International Top-Up" link, dead mobile hamburger, non-responsive grid.
- ❌ The 3DS fallback path reads `data.data.data.result.payment_intent_id` (wrong depth, always undefined).
- ❌ Trial display shows real subtotal and tax with a hardcoded $0 total. A failed tax call prints `$0`.
- ❌ Placeholder phones `(111) 111-1111`, `(000) 123 4567`; the brand string "Acepeak" leaks into Teloz copy.
- ❌ The "no DID available" path on phone-lines is a dead end (button disabled forever).
- ⚠️ An auth token travels in router state to `/phone-lines`. ⚠️ Retry of a failed brand-new signup may re-call `signup` with the same email (idempotency ❓).
- 🟡 No tax ID, GST or business-vs-individual question; the address is capped at 50 characters.

## 3.18 DID purchase wizard
- ❌ The Pay button lacks `type="button"` inside the form, so it also triggers `handleSubmit` and a resolver for the wrong step (`add-number-new/index.tsx:592-600`).
- ❌ Checkout shows and charges **no tax** (commented out, `step-two.tsx:35-36`), calls `setAmount` during render, and shows `$NaN` or `$undefined` if billing details have not loaded.
- ❌ Mislabels: "Monthly DID Cost" shows the prorated total; "Prorated Charge for a DID" shows the total for all DIDs. The real recurring price is never shown. 🟡
- ✅ The 3DS **failure** path calls `did/complete-did-process` with `payment_succeeded:false` to release the reservation. This is the best failure handling in the app.
- ❌ The client reports `payment_succeeded: true/false` to the server. If the server trusts it without re-reading the PaymentIntent from Stripe, an unpaid order could be marked paid. ❓
- ❌ A 75 ms `setInterval` forces `body.pointerEvents = 'auto'` while the modal is open (a hack that defeats focus-lock). `did_type` is sent as an object. `console.log` of order data.

## 3.19 Company record (billing address and tax)
No GST, VAT, PAN or tax ID field exists. 🟡 Editing the company address saves to a console-only record; the upsert to the real billing record is staff-only and its failure is swallowed, so **the edited address may never reach invoices** (invoice reads `user.company_info.address`). ❌ Country, state and city changes use `setValue` without `shouldDirty`, so changing only those likely reports "Nothing has been changed" and saves nothing (by React Hook Form's documented behaviour; not executed). No validation of postal code on this screen, unlike signup.

## 3.20 Global billing surfaces
- `UpgradePlanWidget` (fixed bottom bar, every authenticated page): ❌ invalid class `max-w-[calc(100%-80px))]`; ❌ shows "Plan Expired!" on the last day before expiry; ❌ shows "0 day(s)" when expiry is missing; ❌ close control is `<a href="javascript:void(0)">` with no name; ❌ white on `#e17100` contrast ≈3.2:1; ❌ dead `ChangePlan` import and an unconditional `current-plan-detail` request on every page; "Upgrade Now" sends non-admins to an admin-only page.
- `AccountStateBanner`: ❌ contradicts `billing-alerts` (see 26/27); knows `EXPIRED/SUSPENDED/ON_HOLD/CANCELLED` while `billing-alerts` knows `S/D/DISABLED/E/EXPIRED`. Action goes to Summary, not to the card page. Shown to all admin-settings users, including non-admins who cannot open the target page.
- `DialpadBalance`: dead (usage commented out). If revived it would show "0.00 USD" for unknown balance.
- `PlanUpgradeRequired`: ✅ works; ignores its `featureKey` and never says which feature is missing.

---

# 4. Component Inventory

| Group | Components |
|---|---|
| Shells | `AdminPage`, `SettingCard`, `Tile`, `TableManager`, Tabs, `DateDropdown`, `CustomSelect`, `AlertConfirm` |
| Billing pages | Summary, Usage (`AllowanceTable`, `SpendTable`, `UsageBar`), Plan, `ChangePlan`, `PlanComparison`, `PlanUsageTable`, `RateDetails`, `LicenseManagement`, `Storage`, `DidsList`, `AgentCosting`, `TopUp`, `AmountSection`, `AutoPurchase`, `LowBalanceAlert`, `ManageCards`, `AddCardModal`, `Invoice`, `InvoiceDetails`, `InvoiceLines`, Statement (`Meter`), Modules, Add-ons (`AddOnCard`), Cost centres |
| Payment | `PaymentScreen` (Stripe Elements: card number, expiry, CVC; New Card / Saved Cards tabs; 3DS), payment dialogs |
| Global | `UpgradePlanWidget`, `AccountStateBanner`, `PlanUpgradeRequired`, `PlanPendingGuard`, avatar-menu "Add Funds", `GlobalSearch` entries |
| Helpers (`lib/`) | `billing-money`, `billing-alerts`, `billing-usage`, `allowance-meter`, `cost-centres`, `spend-breakdown`, `plan-catalogue`, `plan-allowance` (orphaned), `addons`, `destination-rates`, `invite-role`, `role-permission-defaults` |

Approximately 30 billing page or dialog components and 12 helper libraries. This is a hand count from the audits, not a tool-generated figure.

---

# 5. Button & Interaction Inventory (condensed)

Element → behaviour → API → error state. Full per-element detail is in the four audit reports; the table below lists the consequential ones.

| Element | Behaviour | API | Missing |
|---|---|---|---|
| Change plan / Upgrade / Downgrade card button | scheduled (`REQUESTED`) or immediate | `plan/change/request`, `calculate-tax`, `payment/charge`, `upgrade-trial-plan` | proration preview, cycle-aware label |
| Cancel Subscription | end-of-cycle cancel request | `plan/change/request` (`action_type:'CANCEL'`) | reason capture, immediate cancel, trial/expired gating |
| Revoke Request | cancels pending change | GET `plan/cancel/request/:uuid` | correct loading flag |
| Renew Now | tax quote then charge | `calculate-tax(RENEWAL)`, `payment/plan-renew` | quote-error UI |
| Buy seats / Remove / Undo / Assign | see 3.5 | `user/buy-license`, `revoke-license`, `licenses` | disable on missing quote |
| Buy storage | prorated charge | `get-prorated-cost`, `buy-extra-storage` | error state |
| Pay (top-up) | charge new/saved card | `add-fundV2` | `onError`, balance refresh |
| Auto-recharge switch / Submit | save settings | `update-auto-recharge-settings` | rollback on failure |
| Low-balance switch / Submit | save settings | `update-low-balance-settings` | max, channel |
| Add card / Set primary / Delete | card management | `card/add|set-default|delete` | `onError`, edit, expiry |
| Invoice eye / expand / CSV | detail / lines / export page | `billing/list` | retry, email, full export |
| Invoice Download | screenshot PDF | none (client) | real PDF |
| Statement | none | none | pagination, export |
| Usage Export CSV | CSV of breakdown | none (client) | injection guard, disable on error |
| Cost centre Add / Archive / Save | edit labels | company-default template upsert | remove, `onError`, unsaved-warning |
| Widget "Upgrade Now" | navigate | none | non-admin handling |

---

# 6. Modal / Popup / Drawer Inventory

| Modal | Trigger | Fields | API | Success / error | Escape / outside click | Mobile |
|---|---|---|---|---|---|---|
| Change Plan | Plan → Change Plan | cycle switch, plan cards | list, change/request | toast, refetch (key bug) | closes | ❌ `w-1/3` cards |
| Plan change confirm | card button | radio (end of cycle / immediate) | change/request | toast | n/a | ⚠️ |
| Payment (upgrade/renew) | confirm / Renew Now | Stripe card, name, save | charge, plan-renew, upgrade-trial-plan | toast; 3DS | blocked | ⚠️ `w-2/5` |
| Cancel subscription confirm | Cancel | none | change/request | toast | closes | ✅ |
| Revoke / Cancel request confirm | buttons | none | cancel/request | toast | closes | ✅ |
| Seat payment | Pay (seats) | Stripe | buy-license | toast, 3DS | blocked | ⚠️ |
| Revoke seat confirm | trash/undo | none | revoke-license | toast | closes | ✅ |
| Storage payment | Buy Now | Stripe | buy-extra-storage | toast, 3DS | closes on outside click | ⚠️ |
| Rate details | button on Plan | none | get-rate | list | closes | ✅ |
| Add Card | Add card | name, Stripe Elements | card/add | toast, refetch | blocked | ❌ close is a `div` |
| Delete / Set primary confirm | row icons | none | card/delete, set-default | toast | closes | ✅ |
| Invoice detail | eye (completed only) | none | none | PDF download | blocked | ⚠️ 1200 px wide |
| "Why this failed" | eye (non-completed) | none | none | n/a | closes | ✅ |
| DID payment modal | wizard Pay | Stripe | create-order, complete-did-process | toast; 3DS | blocked, hack | ❌ `w-1/2` |
| Signup success / failure popups | payment end | none | none | route changes | n/a | ⚠️ close is `div` |
| Customise plan | pricing | enquiry form | customize-plan | toast | closes | ❓ |
| `UpgradePlanWidget` bar | auto | none | `current-plan-detail` | n/a | draggable, `javascript:` close | ❌ |

No billing dialog has a tested loading, error and retry set. Most rely on a global axios toast and have no `onError`.

---

# 7. Subscription Research

**States the code actually knows:** trial (`is_trial 'Y'`), active, expired (`EXPIRED`), plan-payment-pending (login flag), pending plan request (UPGRADE, DOWNGRADE, CANCEL). Backend model enum: `ACTIVE/EXPIRED/INACTIVE` (campaign-api company mirror). UI maps additionally mention `SUSPENDED`, `ON_HOLD`, `CANCELLED`, `S`, `D`, `E`, `DISABLED`; which the backend really emits is ❓.

**Missing states:** past-due, grace period, paused, pending cancellation as a status (it is inferred from a pending request), reactivation, scheduled-change cancelled, incomplete payment. 🟡

**Recommended lifecycle (🔵):**
```text
Trial ──convert──> Active ──renewal OK──> Active
  │                  │ renewal fails
  │ trial ends       v
  v             Past due (retry 1/2/3, banner, emails)
Expired              │ grace ends             │ payment fixed
                     v                        v
                 Suspended ──30d──> Canceled  Active
Active ──cancel at period end──> Active (cancelling) ──period ends──> Canceled
Canceled ──reactivate──> Active (new cycle, charge now)
```
One status vocabulary must be defined by the backend and shared by the banner, `billing-alerts`, the Plan badge and the widget. Today they use four.

---

# 8. Plan Research

Client catalogue (`lib/plan-catalogue.ts`, pinned by tests, **not** necessarily what is charged):

| Plan | $/seat/month | $/seat/year | Domestic minutes | SMS | Numbers | Overage |
|---|---|---|---|---|---|---|
| Starter | 18 | 162 | 1,000 | 100 | 1 | $0.02/min, $0.04/SMS |
| Professional | 30 | 270 | unlimited | 500 | 1 | $0.04/SMS |
| Enterprise | 42 | 378 | unlimited | 1,000 | 1 | $0.04/SMS |

Annual = 75% of twelve months for each (25% saving). ❓ Real prices come from `GET /api/plan/list` and `calculate-tax`; nothing verifies the two agree.
Upgrade/downgrade today: end-of-cycle (`REQUESTED`) or immediate (`IMMEDIATE`, same cycle only). 🟡 Per-plan "who should use it" copy, plan-specific feature lists, restrictions on downgrade (usage over the lower limit).

---

# 9. Seat / Member Research

Fields: total, used, free, revoked, free-revoked, payable. Add Users computes `available = min(free + freeRevoked, total − used)`, `extraUnits = max(0, newUsers − available)`; extra seats trigger payment (`amount` from the tax quote, `tax_calculation_id`). A 402 from the server also opens the payment step.

**Expected behaviour table (🔵 recommended; the code covers only some rows):**

| Scenario (10-seat plan) | Recommended billing | Current code |
|---|---|---|
| 10 → 8 members | seats stay 10 until you reduce seats; idle seats remain billed | ✅ idle seats shown, can be scheduled for removal at renewal |
| 8 → add 1 / 2 | use free seats, no charge | ✅ |
| 8 → add 5 | 2 free + 3 bought, prorated to renewal | ✅ client; proration server ❓ |
| 10 → remove 2 (delete user) | seats stay billed unless revoked | ⚠️ per the UI text |
| 10 → deactivate 2 | seat remains consumed unless explicitly released | 🟡 no deactivated state |
| Downgrade to 8 seats | schedule at renewal, block if 9+ assigned | ⚠️ revoke only works on unassigned or via the user |
| Upgrade to 20 | prorated charge now | ✅ buy seats |
| 2 members on a different plan | needs per-seat plan tiers | 🟡 not supported (single plan per company) |
| Member removed before renewal | seat billed to period end, free next | ⚠️ UI says so, server ❓ |
| Member added after billing date | prorated from add date | ✅ prorated quote, server ❓ |
| Pending invitation | should reserve a seat (🔵) | ❓ presumably consumes a seat on creation; no pending state shown |

Bugs: three different seat-limit sources (`licenses_limit` with 0=unlimited and fallback 50, `plan_info.licenses`, `plan.licenses`); a company at its plan limit with idle seats may be blocked from assigning them in Add Users (edge, ❓); the Add Users sidebar cost is unprorated and untaxed, while the actual charge comes from the order summary.

---

# 10. Payment Research

- **Provider:** Stripe only. Publishable key comes per tenant from org metadata (`mainSiteInfo.stripe_publish_key`), not from `VITE_STRIPE_PUBLISHABLE_KEY` (that env var is unused). ✅ per-tenant keys. ❌ No key → `<Elements>` missing → `useStripe` throws and the Credit & payment page crashes with no graceful message.
- **PCI:** ✅ raw card data never touches app code (Elements → `createPaymentMethod` → only `pm_…` id sent).
- **Flows:** new card, saved card, 3DS via `confirmCardPayment`. ❓ Several pages pass a field named `payment_intent_id` as the "client secret"; `confirmCardPayment` needs a client secret. Only `client_secret` pages (Plan, change-plan) are clearly correct.
- **No payment method → add → default → pay → fail → replace:** add ✅, default ✅, pay ✅, fail ⚠️ (toast only), replace ⚠️ (add new, set primary, delete old; no guided flow). 🟡 Expired-card handling, update-card, wallets, PayPal (env var exists, no code).
- **Currency:** USD hardcoded (`$`, `en-US`). 🟡 INR.

---

# 11. Invoice Research

Lifecycle recommended (🔵): `draft → open → paid | failed | void`, then `refunded/partially refunded` through credit notes. Today the list shows server statuses `completed`, `failed`, `cancel`, `refunded`, `processing`, `pending` as pills.
Missing: a real PDF generated server-side, invoice numbering policy (server `bill_no`, format ❓), email delivery, credit notes, retention policy, a retry/pay-now action, separate Receipt vs Tax Invoice, all-rows export. Refund display is read-only. Delete endpoint: see section 24.

---

# 12. Tax / GST Research

- Tax is computed **server-side** via `POST /api/plan/calculate-tax` (returns `tax_calculation_id`, `tax_percentage`, `tax_location`); the UI only displays it. A `tax_calculation_id` strongly suggests Stripe Tax. ❓
- 🟡 **No** GST/GSTIN/PAN/HSN/SAC, no customer tax ID, no business-vs-individual selector, no tax-inclusive/exclusive indicator, no reverse charge. (`vat_number`/`tax_id` exist only in DID regulatory identity forms, which is KYC, not billing.)
- ❌ DID checkout shows no tax; top-up tax treatment ❓.
- 🔵 **India UX/data requirements:** collect GSTIN (15-char, validated against the state code and PAN pattern; optional for B2C), legal business name, billing state (determines CGST+SGST vs IGST by place of supply), SAC for telecom or software services, a sequential invoice number series, invoice date, supplier GSTIN and address on every tax invoice, and INR with GST rate shown. Also plan for RBI e-mandate and additional-factor-authentication rules for recurring card charges. Auto-recharge off-session on an Indian card will likely fail without a mandate (🔵 test early; server ❓).

---

# 13. Coupon / Credit Research

- Coupons: 🟡 no entry field anywhere; only read-only display of `promo_discount`/`promo_applied` on invoices. No validity, usage-limit, plan restriction, first-time or recurring logic in the UI.
- Credits: prepaid wallet exists (top-up, auto-recharge, low-balance alert). ❌ The balance is not shown on the top-up page, and the Summary, Statement and dialpad read three different fields (`plan.credits`, `company_info.amount`, `company_info.credits`). 🟡 Credit history, expiry, manual and promotional credits, credit-note refunds. The client `wallet-update-webhook` in campaign-api is unauthenticated (see 24).

---

# 14. Upgrade / Downgrade Research

Current flow: Change Plan → choose cycle → pick plan → (downgrade) confirm for end of cycle, or (upgrade) choose end-of-cycle or immediate → tax quote → payment → refresh. Trial upgrades go straight to payment.
Gaps: ❌ cycle-blind classification; ❌ downgrade charges full price when `request_plan` is off; 🟡 proration preview; 🟡 feature-loss and over-limit warnings; 🟡 monthly↔annual rules; 🟡 downgrade blocked while usage exceeds the target plan; stale plan data after the change (key bug).
🔵 Recommended screens: compare → review (old vs new, effective date, proration credit/charge, tax, total) → pay → confirmation with effective date and an "undo scheduled change" link.

---

# 15. Cancellation Research

Present: end-of-cycle cancel request, revoke, and a banner showing the date. Missing: 🟡 reason capture and win-back, immediate cancel, final invoice, data-retention notice, reactivation flow, gating for trial and expired states, and confirmation of what is lost. The server effect is ❓.

---

# 16. Failed Payment / Dunning Research

UI: ✅ inline Stripe errors; 3DS failure message; ⚠️ failed server charge shows only a global toast; invoice list shows the raw failure text. 🟡 Everything else: retry schedule, retry button, banners for "payment failed" (the alert exists in `billing-alerts` but depends on a `payment failed` input; Summary passes the card state, not a failure), grace period, emails, suspension, auto-recharge failure notice. Session status is not refreshed after payment.
🔵 Recommended schedule: retry at +1, +3, +5 and +7 days (Stripe Smart Retries), email admin at each failure, in-app red banner with "Update payment method" going straight to the card page, restrict adding seats and numbers on day 7, suspend on day 14, cancel on day 30, with all state changes driven by webhooks.

---

# 17. Role & Permission Matrix

| Feature | Super Admin | Admin | Manager | Agent | Current implementation |
|---|---|---|---|---|---|
| View billing pages | ✅ | ✅ | ❌ | ❌ | `role === 'ADMIN'` only; index route open to anyone who reaches admin-settings (B1) |
| View plans | ✅ | ✅ | ❌ | ❌ | same |
| Upgrade / downgrade | ✅ | ✅ | ❌ | ❌ | also `billing.action.change_plan/request_plan` |
| Cancel | ✅ | ✅ | ❌ | ❌ | not separately permissioned |
| Manage seats | ✅ | ✅ | ❌ | ❌ | not separately permissioned |
| Add/remove payment method | ✅ | ✅ | ❌ | ❌ | not separately permissioned |
| View/download invoices | ✅ | ✅ | ❌ | ❌ | not separately permissioned |
| Change billing info | ✅ | ✅ | ❌ | ❌ | staff-only upsert for billing record |
| Coupons / credits | n/a | n/a | n/a | n/a | not implemented |

- "Super Admin" is only a name alias for the admin tier; the real roles are `ADMIN`, `MANAGER`, `AGENT`, `USER` plus custom roles. (The 14 roles in `demo-mode.ts` are mock data.)
- `lib/role-permission-defaults.ts` grants billing only to `company_admin`, and its own header admits "No enforcement: the platform does not read these permissions".
- ❌ A custom role with `billing.action.view` sees "Add Funds", clicks, and is bounced by `adminOnly`.
- 🔵 **Recommended:** separate `billing.view`, `billing.invoices.download`, `billing.payment_methods.manage`, `billing.plan.change`, `billing.seats.manage`, `billing.cancel` (owner only). Add a read-only "Finance" role for invoices and statements. Enforce all of them on the server, not only in the route guard.

---

# 18. API Research

Full route inventory (method, URL, callers, fields) is in the table below. All calls use `apiClient` with a Bearer token from `localStorage['ucaas-public-token']` and an `X-ORG-ID` header (branding org, not company). `limit` is clamped to 200. There is no 402/403 handling in the axios interceptor, no idempotency key, and 401 clears the token.

| Purpose | Method | URL |
|---|---|---|
| Plan details | POST | `/api/user/current-plan-detail` |
| Plan list | GET | `/api/plan/list` |
| Tax quote | POST | `/api/plan/calculate-tax` |
| Plan change/cancel request | POST | `/api/plan/change/request` |
| Cancel request | GET | `/api/plan/cancel/request/:uuid` |
| Trial upgrade / charge / renew | POST | `/api/payment/upgrade-trial-plan`, `/api/payment/charge`, `/api/payment/plan-renew` |
| Signup payment | POST | `/api/signup`, `/api/signup-on-trial`, `/api/payment/initial-plan-payment` |
| Top-up | POST | `/api/payment/add-fundV2` |
| Auto-recharge / low balance | POST | `/api/user/update-auto-recharge-settings`, `/api/user/update-low-balance-settings` |
| Cards | POST | `/api/card/list|add|delete|set-default` |
| Invoices | POST | `/api/billing/list` |
| Invoice detail / delete | GET / DELETE | `/api/payment/info/:uuid`, `/api/payment/delete/:id` (no caller) |
| Seats | POST | `/api/user/buy-license`, `/api/user/licenses`, `/api/user/revoke-license` |
| Storage | POST | `/api/plan/get-bucket-size`, `/get-prorated-cost`, `/buy-extra-storage` |
| DID | POST | `/api/did/billing-details`, `/api/didw/create-order`, `/api/did/complete-did-process` |
| Rates | POST | `/api/plan/get-rate`, `/api/plan/get-sms-rate` |
| Agent cost / numbers | POST | `/api/ai/agent/billing/list`, `/api/numbers/list` |
| Usage rows | POST | `/api/tenant/report/call-list` |

**Client-computed money sent to the server (server must re-derive; ❓ unverified):** top-up `charge_amount`; auto-recharge `refill_amount`, `threshold_amount`; plan charge `amount`; seat `charge_amount` (`|| 0` fallback); signup `payment.amount` and `is_trial`; DID `amount` and `payment_succeeded`. Safest shape in the app: storage purchase (GB only).

---

# 19. Database Research

The repo has no billing schema. The only server-side artifact is a read mirror of the company record (`campaign-api .../CompaniesModel.js`): `plan_uuid`, `plan_duration`, `plan_start_date`, `plan_expiration_date`, `is_trial` (Boolean there, but the frontend compares to `'Y'/'N'`), `plan_status` (`ACTIVE/EXPIRED/INACTIVE`), `amount`, `licenses`, `call_duration(_used)`, `sms(_used)`, `plan_features`, `stripe_token`, `payment_verified`, `purchase_plan_detail`. There is no Plan, Invoice, Card or Payment model.

🔵 Recommended entities and relations:

```text
Organization 1─* Company 1─1 Subscription ─* SubscriptionItem (plan seat line, storage line, add-on line)
Subscription *─1 Plan ─* PlanPrice (monthly|annual|…, currency)
Company 1─* Seat (state: assigned|free|pending_invite|scheduled_removal) ─0..1 User
Company 1─* PaymentMethod (stripe_pm_id, brand, last4, exp_month, exp_year, is_default)
Company 1─* Invoice 1─* InvoiceItem ; Invoice 1─* Transaction ─* Refund
Company 1─* CreditLedger (type, amount, balance_after, expires_at, source)
Coupon 1─* CouponRedemption ─ Company
Company 1─1 BillingProfile (legal name, address, country, state, GSTIN/VAT, contact email)
SubscriptionChange (type, from, to, effective_at, status)   ← scheduled up/downgrade/cancel
WebhookEvent (stripe_event_id UNIQUE, type, payload, processed_at)  ← idempotency
```
Money is stored as integer minor units plus currency code. Append-only ledger for credits and transactions.

---

# 20. Payment Provider Research

**Provider:** Stripe. PayPal: env var only, no code. Razorpay: no references. 🟡

```text
Browser (Stripe Elements → pm_id)
   → default-api (create/confirm PaymentIntent from SERVER price; return client_secret if 3DS)
   → Stripe
   → Webhook → default-api (verify signature, dedupe by event id)
   → DB (invoice, transaction, subscription, credit ledger)
   → UI refetch (and socket event)
```
No webhook receiver exists in this repo, so signature verification, event coverage and idempotency are ❓. The campaign-api `wallet-update-webhook` is unauthenticated and unsigned and can push a fake balance event to any socket (UI spoof only).
🔵 Required webhook events: `payment_intent.succeeded|payment_failed|requires_action`, `invoice.paid|payment_failed|finalized|upcoming`, `customer.subscription.updated|deleted|trial_will_end`, `charge.refunded`, `payment_method.attached|detached|automatically_updated`, `setup_intent.succeeded`, `customer.updated`, `charge.dispute.created`.
Use SetupIntent to save cards. For Indian cards also plan for e-mandates.

---

# 21. User Journeys (condensed)

For each: action → UI → API → backend/provider → DB → UI update → notification. "Today" notes what the code does.

1. **New subscription:** Pricing → Sign-up (address, tax pre-check) → Payment (tax quote, card, 3DS) → success popup → Phone lines. ❌ Annual not carried through; ❌ Pay live without a quote.
2. **Upgrade:** Plan → Change Plan → choose → (immediate) quote → charge → refetch. ❌ stale data; ❌ cycle-blind label.
3. **Downgrade:** choose lower plan → confirm "end of cycle" → `change/request` → banner "plan changes at renewal". ✅ (server ❓)
4. **Cancel:** Cancel → confirm → `change/request(CANCEL)` → banner. 🟡 reason, retention, final invoice.
5. **Reactivate:** Revoke Request while pending. 🟡 after the period ends.
6. **Add member:** Add Users → seats computed → payment if over → create. ⚠️ amount client-sent; ❌ stale `paymentCalculation` can carry an old amount/id (`order-summary.tsx:29-36`).
7. **Remove member:** delete user → seat freed, still billed → "Remove seats" later. ⚠️
8. **Deactivate member:** 🟡 not modelled.
9. **Change billing cycle:** only via Change Plan with the same plan. ⚠️
10. **Add payment method:** Add card → Elements → `card/add`. ⚠️ no SetupIntent.
11. **Replace expired card:** 🟡 no expiry warning on the card list; Summary shows an alert only if `cardExpiresSoon` fires.
12. **Failed payment:** toast only. 🟡 dunning.
13. **Download invoice:** eye → modal → screenshot PDF. ❌ not a real PDF.
14. **Update billing info:** Company record; ❌ may not reach invoices; no tax ID.
15. **Apply coupon:** 🟡.
16. **Use credit:** automatic server-side at call time (❓); balance not visible on top-up.
17. **Reach seat limit:** counter capped; message "reached maximum". ⚠️
18. **Exceed usage limit:** Usage page shows overage using client rates. ⚠️
19. **Subscription expires:** banner, widget, forced `/renew-plan` only on login. ⚠️
20. **Renewal:** manual "Renew Now" payment; automatic renewal is presented as "Auto-renews on {date}" with no toggle. ❓ server auto-charges.

---

# 22. UI/UX Audit

- ✅ Strong: one shared route list; honest "coming soon" labelling; "Not available yet" instead of zero in most helpers; money helper tests; responsive tables with horizontal scroll on Usage and Comparison.
- ❌ Weak: inconsistent date formats across pages (`15 September 2026`, `D MMM YYYY`, `YYYY-MM-DD HH:mm`); `$` formatting varies (`$20` vs `$20.00` vs `20.00 USD`); fixed-width dialogs; seven of ten sidebar entries share one icon; inconsistent meter thresholds (75/90 vs 80); different "licences used" meanings; `Monthly` labels on annual plans; mixed Tailwind palette typos (`border-grey-*`, `text-grey-*`); broken `max-w-[calc(100%-80px))]`.
- ⚠️ Dark mode: `mcm-page.css` remaps `bg-white`, `text-gray-*` and `border-gray-*`, but no remap was found for `amber/red/green-50` banners and pills; Storage uses inline hex; widget, dialpad and Stripe elements are not themed. ❓ not rendered.
- 🔵 Keep the Teloz orange/cream language; fix with tokens, not a redesign.

# 23. Accessibility Audit

❌ `div onClick` controls with no role, label or keyboard support: amount tiles, license ±, plan cards, card actions, invoice eye, modal close icons, success popup close, seat icons, widget icon. ❌ `display:none` radios for saved cards. ❌ Resources tabs lack tab roles. ❌ Nested `<a><button>`. ❌ `javascript:` anchor. ❌ Progress bars without roles. ❌ Tables without caption or scope. ❌ Colour-only status on several tags. ❌ White on `#e17100` ≈3.2:1. ❌ Alert uses `role="status"`. ❌ Tiny 8-9 px text in dialpad. ⚠️ Hardcoded `id="1"` checkbox label. ⚠️ Cost-centre fields all labelled "Code/Name" without row context.

# 24. Security Audit

| Item | Finding | Status |
|---|---|---|
| Admin route guard | index route `/admin-settings/billing` renders Summary without `adminOnly` | ❌ |
| Authorisation | security audit (2026-08-29): "no authorisation middleware at all", permission tree "decoration"; billing enforcement ❓ | ❓ |
| Tenant scoping | `BillingController` filter injection was reported (returned all companies' records); fixed by a denylist; cross-tenant rejection "not yet verified" by the audit itself | ⚠️ |
| Client-trusted amounts | see section 18 | ❓ high risk |
| Payment-complete flag | DID flow sends `payment_succeeded` from the browser | ❓ |
| Delete invoice | `DELETE /api/payment/delete/:id` defined, unused; server existence and authorisation ❓. 🔵 remove, or admin-gate and soft-delete | ⚠️ |
| Idempotency | none on any charge | 🟡 |
| Token storage | JWT in localStorage; XSS-exposed by design | ⚠️ |
| Card data | never in app code (Stripe Elements) | ✅ |
| Console leaks | Stripe PaymentMethod and order data logged (`storage:68`, `top-up:67`, `payment-modal:118`, `signup/payment:109,222`, `phone-lines:71` logs the whole user object, `signup/index:180,226`) | ❌ low/med |
| CSV injection | Usage, Invoices exports | ❌ |
| Token in router state | access token passed to `/phone-lines` through history state | ⚠️ |
| Webhooks | campaign-api `wallet-update-webhook` unauthenticated; Stripe webhook verification ❓ | ❌ / ❓ |
| `RequireAdmin` (campaign-api) | falls back to client-supplied `x-user-role` header | ❌ |
| Secrets | none found in tracked files (names only in `.env.example`); `VITE_*` values ship in the public bundle | ✅ |
| Demo mode | active on any `*.vercel.app` preview unless `VITE_DEMO_MODE=false`; production hosts cannot enable it | ✅ |

# 25. Industry Comparison

Caveat: I did not browse the web for this report. The right-hand columns reflect general, widely known product patterns (Stripe Billing, Twilio, Zendesk, HubSpot, Slack, Genesys Cloud, Dialpad), not freshly verified documentation. Treat as 🔵 recommendations.

| Feature | Current project | Industry standard | Recommended |
|---|---|---|---|
| Subscription state model | trial/active/expired + request | trial, active, past_due, unpaid, canceled, paused | adopt Stripe-style statuses |
| Proration preview | none | shown before confirming | upcoming-invoice preview |
| Seat model | free/used/revoked | assigned/unassigned, pending invite, scheduled change | add pending and deactivated |
| Payment methods | add/default/delete | + update, expiry alerts, SetupIntent, wallets | SetupIntent + expiry alerts |
| Invoices | list + screenshot PDF | hosted PDF, email, credit notes, tax IDs | server PDF + email |
| Tax IDs | none | collected at checkout, shown on invoice | GSTIN/VAT with validation |
| Dunning | none | smart retries, emails, grace, suspension | webhook-driven |
| Coupons | none | promo codes with rules | Stripe coupons |
| Credits | prepaid wallet | credit ledger with history | ledger + history view |
| Usage billing | per-call overage with client rates | metered, near-real-time, spend caps | server meter + alert thresholds |
| Roles | one admin | billing-admin / finance read-only | granular billing roles |
| Cancellation | request only | reason, retention offer, end-of-period | add reason and offer |

# 26. Missing Features

🟡 Past-due/grace/suspended handling; dunning and retries; coupons; credit history; tax IDs and GST; real PDF invoices; credit notes; refunds UI; email invoice; retry-pay for failed invoices; update/expiring card handling; SetupIntent; reactivation; scheduled-change preview; plan-change proration preview; per-seat plan tiers; pending-invite and deactivated seat states; add-on purchase; cost-centre allocation consumer; balance display on top-up; billing-specific roles; INR/multi-currency; PayPal; payment-method guard before enabling auto-recharge; audit log of billing actions.

# 27. Bugs

**Critical**
- B1 Index route unguarded (`router/index.tsx:1385`).
- B2 Annual toggle not carried to checkout (`signup/index.tsx:140`, `pricing:226-233,310-312`).
- B3 Signup Pay live with no tax quote; sends `amount:0` (`signup/payment.tsx:215-220,358-361`).
- B4 Wrong query key leaves plan/seat state stale (7 sites).
- B5 Downgrade charges full price when `request_plan` is off (`change-plan.tsx:465-475`).
- B6 Upgrade/downgrade label ignores billing cycle (`change-plan.tsx:73-75`).
- B7 DID checkout: no tax, `$NaN`, Pay without `type="button"`, setState in render.
- B8 Company address edit may not reach invoices; location fields may not mark the form dirty.

**High**
- B9 False "No payment method" alert on card-list error (`summary:99`).
- B10 `$0.00` for unknown on Summary, Usage, dialpad, widget.
- B11 Statement "to date" is all-time; capped at 12 rows.
- B12 Banner vs alert contradiction on EXPIRED; four status vocabularies.
- B13 3DS secret field naming; wrong-depth fallback.
- B14 Seat `charge_amount: 0` fallback; renew page "Pay $0"; stale `paymentCalculation` in Add Users.
- B15 Cards: no expiry/expired indicator, hardcoded Visa logo, cardholder name discarded, no `return` after Stripe error.
- B16 Invoice: refunded/pending not openable; screenshot PDF; incomplete tax invoice; single-page CSV.

**Medium / low**
- Usage share denominator; CSV injection; 99.5% rounding to "over"; past-dated trial "ends soon"; unrestorable unassigned revoked seats; hardcoded 50-seat cap; plan widget defects; non-admin dead-end links; cost-centre blank-row trap; `knownNumber('  ')` returns 0; `formatBillingDate` accepts 31 Feb; same-day comment contradicts code (`billing-money.ts:140`); `PlanDurationMap`/`durationMap`/`RequestedPlanDurationMap` disagree (quarter = 3 vs 4; `/user/undefined`); toast typos; mock Pricing table; placeholder phones; "Acepeak" leak; missing React keys; `console.log` leaks. (Tests exist only for the `lib/` helpers; no billing page or component has a test.)
- Orphan code: `plan-allowance.ts`, `prorate()`, `licenceQuote()`, `utils.ts` legacy proration helpers (truncate cents), `DialpadBalance`, `willBecomeAdminByDefault`, `getInvoiceDetails`, `deleteInvoice`, `CALL_RATE_DETAIL`.

# 28. Recommended Improvements (🔵)

1. Guard the index route; consolidate billing permissions.
2. Make the **server authoritative for all money**: accept only IDs and quantities, re-derive amounts and tax, and ignore client `is_trial`, `amount` and `payment_succeeded`.
3. One shared subscription status enum and one status→UI mapping module used by banner, alert, badge, widget.
4. Fix the query key and add a single `invalidateBilling()` helper.
5. Cycle-aware change-plan comparison (compare per-month equivalents) with a proration preview.
6. SetupIntent for saving cards; expiry badges; `onError` and rollback on every mutation.
7. Server-generated PDF invoices with tax IDs; credit notes.
8. One money format and one date format across billing.
9. Remove `console.log`s; sanitise CSV cells.
10. Replace `div onClick` controls with buttons; add tab roles; fix contrast.

# 29. Complete Implementation Blueprint

Format per page: components → data → API → DB → business logic → validation → states → permissions → notifications.

**Overview / Summary**
- Components: status card (status chip, plan, price, cycle, next charge, seats used/total), alert slot, payment method card, recent invoices, outstanding balance.
- Data/API: `GET /billing/overview` (single call returning subscription, seats, balance, default PM, last 5 invoices, alerts) instead of four.
- DB: Subscription, Seat, PaymentMethod, Invoice, CreditLedger.
- Logic: alert priority (suspended > past due > no PM > card expiring > trial ending > low balance).
- States: loading skeleton, partial error per card (never show zero or a false alert on error), empty.
- Permissions: `billing.view`. Notify: banners from webhook-driven status.

**Plans / Change plan**
- Components: plan cards, cycle toggle, comparison, review step (old/new, effective date, proration, tax, total), confirmation.
- API: `GET /plans`, `POST /subscription/preview-change` → `{effective_at, proration, tax, total, warnings[]}`, `POST /subscription/change` (plan_id, cycle, timing only).
- Logic: compare normalised per-month price; downgrade blocked if seats/usage exceed target; scheduled change creates `SubscriptionChange`; cancel scheduled change.
- Permissions: `billing.plan.change`. Notify: email on schedule and on effect.

**Subscription**
- API: `POST /subscription/cancel` (reason, timing), `/reactivate`, `/pause` (optional), `/renew-now`.
- Logic: lifecycle in section 7; grace and suspension timers driven by Stripe events; final invoice.

**Seats**
- Components: seat summary (total/assigned/free/pending/scheduled removal), table, buy/remove drawers with quote.
- API: `POST /seats/quote`, `/seats/purchase` (count only), `/seats/release`, `/seats/restore`.
- Logic: proration = remaining days of cycle × per-seat price, rounded once; release effective at period end; pending invite reserves a seat; deactivate keeps seat until release.
- Validation: count ≥ 1, ≤ plan cap; block when expired or past due.

**Payment methods**
- API: `POST /payment-methods/setup-intent`, list, set default, delete (blocked if last and active subscription/auto-recharge).
- UI: brand icons, expiry badge, "Update card", billing address, failed-card state with "Fix now".

**Credit & top-up**
- Show balance and ledger; custom amount with min/max set by server; top-up creates PaymentIntent server-side; confirm on webhook; auto-recharge requires a default PM and refill ≥ threshold.

**Invoices / transactions**
- API: `GET /invoices?status&from&to&q`, `GET /invoices/:id/pdf`, `POST /invoices/:id/email`, `POST /invoices/:id/retry`.
- Statuses: draft/open/paid/failed/void/refunded; credit notes; filters by status; export all.
- Immutable once finalised. No delete endpoint.

**Billing information and tax**
- Fields: legal name, billing email, phone, address lines, country, state, city, postal code (country-specific validation), tax ID type and number (GSTIN: 15 chars, state code matches billing state, PAN pattern), business vs individual.
- API: `GET/PUT /billing-profile`; the profile feeds Stripe Customer and tax calculation. Invoices snapshot the profile at issue time.

**Coupons**
- `POST /coupons/validate`, `POST /coupons/apply`; rules: percent/fixed, once/forever/repeating, max redemptions, expiry, plan restriction, first-time only, error messages per rule.

**Dunning / status**
- Webhook consumer, retry schedule, notification service, banner states, access restrictions, admin override.

**Usage and statement**
- Server-side metering endpoint returning allowances, used, overage and cost with the same rates the invoice uses; statement paginated and period-scoped with export.

**Cross-cutting:** error boundary around Stripe Elements with a friendly "payments not configured" state; a shared `useBillingMutation` that handles `onError`, 402/403, idempotency keys and invalidation; one status/format module.

# 30. Prioritized Roadmap

| Pri | Item | Page | Backend / DB | Complexity |
|---|---|---|---|---|
| **P0** | Server re-derives all money; drop client `amount`, `is_trial`, `payment_succeeded` | all | pricing service, PaymentIntent from IDs | M |
| P0 | Stripe webhook receiver with signature check and event dedupe | n/a | WebhookEvent table | M |
| P0 | Guard billing index route; enforce billing authz server-side | router, API | middleware | S/M |
| P0 | Idempotency keys on every charge | all payments | header handling | S |
| P0 | Fix annual-cycle carry-through and disable Pay without a quote | pricing, signup, renew, seats | none | S |
| P0 | Fix query-key invalidation | plan, seats, users | none | S |
| P0 | Downgrade/upgrade classification and `request_plan` fallthrough | change plan | none | S |
| P0 | Single status vocabulary and past_due/grace/suspended states | banner, alert, plan | Subscription.status | M |
| P1 | SetupIntent card saving; expiry badges; SCA | cards | Stripe | M |
| P1 | Dunning (retries, emails, banner, restrictions) | global | jobs, notifications | L |
| P1 | Server PDF invoices, tax IDs, GST fields and tax-invoice layout | invoices, company | BillingProfile, invoice service | L |
| P1 | Proration preview for plan and seat changes | change plan, seats | preview endpoint | M |
| P1 | Fix DID checkout (type, tax, NaN, labels) | numbers | none | S |
| P1 | Fix Summary false alerts and unknown-as-zero values | summary, usage, widget | none | S |
| P1 | Billing-specific roles and permissions | roles | permission keys | M |
| P2 | Cancellation reasons, reactivation, final invoice | plan | SubscriptionChange | M |
| P2 | Credit ledger and balance on top-up page | purchase | CreditLedger | M |
| P2 | Coupons | checkout, plan | Coupon, CouponRedemption | M |
| P2 | A11y pass (buttons, tab roles, contrast, progress roles) | all | none | M |
| P2 | Statement pagination/export; all-rows CSV with injection guard | statement, invoices, usage | none | S |
| P2 | Remove `console.log`s; consolidate formats | all | none | S |
| P3 | Credit notes and refund UI | invoices | Refund | M |
| P3 | PayPal, multi-currency/INR, wallets | payments | Stripe config | L |
| P3 | Per-seat plan tiers, add-on purchase, cost-centre allocation | seats, add-ons | schema | L |
| P3 | Dark-mode token cleanup | all | none | M |

# 31. Master Implementation Checklist

**Routes:** ☐ guard index ☐ keep `spending` redirect ☐ add deep-linkable tabs (`?tab=`)
**Pages / subpages:** ☐ Overview ☐ Plans ☐ Subscription ☐ Seats ☐ Payment methods ☐ Credit ☐ Invoices ☐ Transactions/Statement ☐ Billing info and tax ☐ Coupons ☐ Usage ☐ Renew
**Components:** ☐ `StatusChip` ☐ `MoneyText` (single format) ☐ `ProrationPreview` ☐ `SeatSummary` ☐ `CardBrandIcon` ☐ `ErrorBoundary` for Stripe
**Buttons:** ☐ all `button` elements with labels ☐ no nested link/button ☐ loading and disabled while quote missing
**Forms:** ☐ billing profile ☐ GSTIN/VAT ☐ coupon ☐ top-up amount ☐ cancel reason
**Modals:** ☐ shared `BillingDialog` (Escape, focus trap, responsive width)
**APIs:** ☐ overview ☐ preview-change ☐ change ☐ cancel/reactivate ☐ seats quote/purchase/release ☐ setup-intent ☐ invoices pdf/email/retry ☐ billing-profile ☐ coupons
**Database:** ☐ entities in section 19 ☐ integer money ☐ append-only ledgers
**Billing logic:** ☐ proration (single implementation) ☐ cycle-aware comparison ☐ tax from server only
**Subscription / Plans / Seats:** ☐ lifecycle enum ☐ scheduled changes ☐ pending/deactivated seats
**Payments / Invoices / Taxes / Coupons / Credits:** ☐ SetupIntent ☐ SCA ☐ server PDF ☐ GST fields ☐ coupon rules ☐ ledger
**Permissions:** ☐ billing keys ☐ server enforcement ☐ Finance read-only role
**Notifications:** ☐ payment failed ☐ upcoming renewal ☐ card expiring ☐ plan changed ☐ cancelled ☐ low balance
**Error handling:** ☐ `onError` everywhere ☐ 402/403 interceptor ☐ no false alerts on error
**Accessibility / Responsive / Light-dark:** ☐ section 23 list ☐ no fixed-width dialogs ☐ tokens for status colours
**Security:** ☐ idempotency ☐ remove `console.log` ☐ CSV sanitise ☐ remove delete-invoice route ☐ sign the wallet webhook ☐ drop header-role fallback
**Payment provider / Webhooks:** ☐ events in section 20 ☐ signature verification ☐ event dedupe

# 32. Final Billing Readiness Score

Rationale: the UI is broad and the helper libraries are well tested (about 520 passing assertions across 11 suites, which I ran). However, there are no tests for any billing screen, several money-path defects (annual cycle, quote-less Pay, cycle-blind change plan, stale data), no dunning, no real invoice PDFs, no tax ID or GST support, a guard gap on the index route, and a server side that cannot be verified from this repo.

```text
TOTAL BILLING ROUTES: 12 under /admin-settings/billing (10 sections + index + 1 redirect), plus ~7 adjacent
TOTAL BILLING PAGES: 10 sections + renew-plan + pricing/signup funnel (4) + DID wizard
TOTAL SUBPAGES: 8 (Resources 4 tabs, Credit & payment 2 tabs, plus Plan tabs overlap Resources)
TOTAL COMPONENTS: ~30 billing components + 12 lib modules (hand count, approximate)
TOTAL INTERACTIVE ELEMENTS: roughly 150 (approximate; hand count from the four audits)

Finding tallies (approximate, from this report's lists):
VERIFIED WORKING: ~35 notable behaviours
BROKEN: ~55
PARTIAL: ~45
NOT IMPLEMENTED: ~30
NOT VERIFIABLE: ~25 (all server-side)

CRITICAL ISSUES: 8
HIGH PRIORITY: 8
MEDIUM PRIORITY: ~20
LOW PRIORITY: ~25

OVERALL BILLING SCORE: 38/100

PRODUCTION READY: NO
  Conditions to reach WITH CONDITIONS: complete every P0 item, confirm the server-side
  behaviours marked ❓ against the live default-api, and add end-to-end tests for the
  payment, renewal and seat flows.
```

*Counts marked approximate are judgement-based tallies, not machine-generated. The per-finding evidence lives in the sections above.*
