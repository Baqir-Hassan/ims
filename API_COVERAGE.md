# API coverage

## Phase 4: F&B inventory

The API lives in `backend/app/modules/fnb_inventory`. All paths below are under `/api/v1`.

| Area | Endpoint families | Verification |
|---|---|---|
| Kitchen transfers | `/fnb/locations`, `/fnb/kitchen-requests` (list, create, suggest, decision, withdraw, dispatch, receive), `/fnb/stock-levels/{id}` | Roles, approval quantities, insufficient stock, partial dispatch, transit loss, withdrawal, par suggestions |
| Store purchasing | `/fnb/vendors`, `/fnb/purchase-requests` (list, create, suggest, decision, finance-decision), `/fnb/purchase-orders`, `/fnb/goods-received` | Toggle off/on, pending Finance approval survives toggle off, approved PO, weighted average cost, vendor bill and batch |
| Purchase history | `GET /finance/ingredient-purchases` | Accepted GRN quantities and prices; kitchen access denied; direct purchase POST unavailable |
| Stock and ledger | `/inventory/fnb/enquiry`, `/valuation`, `/valuation/align`, `/counts` (create, enter, post), `/waste` | Cost visibility, stock takes, waste, full-cycle reconciliation of 1210/1215/1220/1230 |
| Kitchen waste | `/kitchen/waste/options`, `/kitchen/waste`, `/kitchen/restock` | Venue quantities, ingredient/recipe consumption, no costs in kitchen responses |
| Recipes and costing | `/finance/ingredients` (create, update, valuation, movements), `/finance/recipes`, `/finance/menu-engineering`, `/finance/outlet-pnl`, `/finance/waste-log`; report `/export.xls` routes | Recipe margins, registered sales and labour, menu engineering, outlet P&L, spreadsheet export |
| Controls | `/fnb/settings`, `/fnb-costing`, `/fnb-costing/export.xls` (`view=outlets` or `items`), `/fnb-costing/targets/{outlet}`, `/fnb-costing/classes/{kind}/{id}` | Manager-only purchase toggle, costing targets, role restrictions, a venue login's export limited to its own outlet |
| Purchasing reports | `/finance/purchase-reports/summary` (`kind` orders, purchases or invoices; `group_by` supplier, item or category), `/lines` (the lines behind a row), `/supplier-weekly` (by week or month), `/prices`, `/price-changes`, `/filters`; each report has an `/export.xls` | Orders are approved POs, deliveries are accepted GRN lines at the price paid, invoices are bill lines (a bill raised from a delivery uses that delivery's lines; a flat bill shows as one unitemised line); date range, supplier, category and department filters; per-supplier price comparison and what was paid above the cheapest; price changes against the same supplier's previous delivery; kitchen and store logins denied |
| Ingredient categories | `category` on `POST /finance/ingredients` and `PATCH /finance/ingredients/{id}` | Spaces tidied, blank clears it, demo ingredients seeded with one; the purchasing reports group by it (no category shows as Uncategorised) |
| Approval policies | `/approval-policies/{kind}` for three `fnb_*` policy kinds | Module roles accepted, unknown role rejected, optional second-person enforcement |

Tests are in `backend/tests/fnb_inventory/`. Finance's ledger tests cover new module accounts registered after the chart cache was read. Boundary tests cover imports and fixed-route shadowing.

The frontend mounts screens through its manifest and contributes purchase cards to Finance through `panels`. Phase 5 supplies outlet sales; Phase 5b supplies payroll costs. Their full sale/payroll flows belong to those phases.

## Phase 5: F&B outlets

The backend and frontend live in `app/modules/fnb_outlets`. Paths below are under `/api/v1`; shorter entries in a row are subpaths of that endpoint family.

| Area | Endpoint families | Verification |
|---|---|---|
| Till orders and kitchen | `/cafe/orders`, `/orders/history`, `/orders/{id}`, `/orders/{id}/status`, `/orders/{id}/items` | Order/payment replay, role restrictions, completion guards, catalog stock and ingredient consumption at the mapped kitchen |
| Payments and receipts | `/cafe/members/verify/{no}`, `/cafe/orders/{id}/payments`, `/divide`, `/receipt`; shared `/member-pay/resolve`, `/receipts/email-sale` | Partial/split tenders, overpayment rejection, member limits and transaction rollback, payer-specific refunds, receipt invoice idempotency |
| Adjustments | `/cafe/orders/{id}/cancel`, `/items/{line}/comp`, `/items/{line}/cancel`, `/compensate` | Manager approval, unauthorized decisions, repeated decision protection, successive tax credits and final void |
| POS tools and setup | `/pos/setup`, `/settings`, `/guest-types`, `/modifiers`, `/discount-rules`; `/cafe/orders/{id}/details`, `/move-table`, `/join`, `/items/{line}/modify` | Modifier pricing, guest discounts, covers, table moves, same-venue joining and stock restoration |
| Tables and reservations | `/cafe/tables`, `/cafe/reservations` | Member reservation ownership, cancellation cutoff and shared role permissions |
| Host/department meals | `/cafe/orders/{id}/charge-host`, `/charge-department`, `/cost-centres`, `/consumption` | Actual host refund, Finance cost-centre setup, zero-tax department meals, expense journals and consumption totals |
| On-course and member app | `/cafe/on-course-orders`, `/cafe/on-course-orders/{id}/status`; `/member/menu`, `/ordering-hours`, `/tables`, `/table-reservations`, `/on-course-orders` | Compatible response fields, outlet inference, member isolation, immediate account charge, cancellation refund once |
| Ordering controls | `/fnb/ordering-hours` | Closed-window refusal, invalid hours, overnight services, weekend changes and the 30-minute cutoff |
| Extensions | Registered revenue/payment sources for Indoor Cafe, Rooftop, Bar and On Course; tax types `cafe_order` and `on_course_order`; sales lines, member sections, receipts, journals and live topics | Import/route boundaries, balanced journals and receipt/tax integration |

Tests are in `backend/tests/fnb_outlets/`. The frontend contributes 13 screens, kitchen overview/nav counts and the existing offline queue's order/payment routes. Manager menu screens mount CatalogManager with a fixed category. Ingredient COGS remains with inventory; sales journals do not duplicate it.

## Phase 5b: F&B payroll and shifts

The backend and frontend live in `app/modules/payroll_shifts`. All paths are under `/api/v1`. Accounting flows are in the [F&B DFD](docs/FNB_DFD.md).

| Area | Endpoint families | Verification |
|---|---|---|
| Work options and rosters | `GET /payroll/options`; `GET/POST /staff/shifts`, `PATCH /staff/shifts/{id}` | Only active F&B roles; Manager/F&B Manager writes; overlap checks; workers see their own shifts |
| Attendance | `/staff/shifts/{id}/clock-in`, `/clock-out`, `PUT /attendance`, `POST /attendance/confirm` | Self clocking, aware Pakistan timestamps, unpaid breaks, completed attendance required, Manager confirmation, payroll snapshots freeze inputs |
| Pay rates | `GET/POST /finance/pay-rates` | Finance/Manager only; hourly or salary; daily regular hours and overtime rate |
| Payroll runs | `GET/POST /finance/payroll-runs`, `GET/DELETE /{id}`, `POST /{id}/status`, `PATCH /finance/payroll-run-lines/{id}` | Confirmed hours, daily overtime across shifts, no period overlap, rates frozen, bonus/deduction limits, draft-only deletion/adjustments, Manager approval through core, explicit bank/cash payment and period locks |
| Labour report | `GET /finance/labor-cost?anchor=`; existing `/finance/outlet-pnl` | Approved/paid allocations by actual attendance outlet; drafts excluded; gross cost and revenue percentage |
| Finance extensions | Journal source and `fnb_labour` cost source | Accrual and payment journals balance; deductions remain payable; payroll payable clears on payment; costs reach outlet P&L without reverse imports |

Regression tests are in `backend/tests/payroll_shifts/`. Frontend navigation supplies a roster calendar, attendance, pay rates, payroll runs and labour-cost report. The F&B Manager handles rosters/attendance, Finance prepares/pays runs, and a different Manager approves through the existing Approvals screen. Workers can clock their own shifts. Shift/attendance lists expose no pay amounts.

## Phase 6: hardening and release rehearsal

| Area | Endpoints / checks | Release evidence |
|---|---|---|
| Empty installation | Setup status/bootstrap, staff creation, all module schema callbacks | Separate empty PostgreSQL schema; demo seeding/background jobs/gateways disabled; private token and repeat-bootstrap guards |
| Full workflow | Catalog + recipes → purchase approvals → GRN/bill → kitchen transfer → prepared sale → attendance/payroll → monthly billing → statements/receipts | One acceptance test creates its own six-role staff and business records through the API |
| Accounting reconciliation | `/finance/gl-export`, ingredient movements, `/finance/outlet-pnl` | Store 1210, transit 1215, kitchen 1230, GRN clearing 2150, AP 2100, payroll 6250/2220, COGS 5210 and member AR 1150 match underlying records |
| Member billing sources | Membership revenue source and members-owned journals | Subscription/fee/interest/credit/write-off postings, tax 2200, separate receipts and linked month-end reversals; F&B sources remain single-counted |
| Returned cheques | Receipt reversal, fee charge, membership tenders | Principal reverses; member AR contains one fee; bank expense and fee recovery post separately; fee excluded from receipt tender |
| Member API compatibility | 35 F&B/account operations under `/member-auth` and `/member` | Frozen reference parameters/request/declared-response schemas; concrete profile, menu, statement, transactions, receipts, device, preferences, pay-code and feature responses |
| Hosted configuration | `IMS_ENVIRONMENT`, private setup/signing secrets, docs/CORS/demo flags | Fail-fast staging/production guard tests; `/ready` checks DB and hides connection details on failure |
| Live checker | `python -m tools.release_check` with actual API/Worker URLs | Read-only GET/OPTIONS checks for readiness/modules/setup/docs/CORS and all Worker entry routes; positive/negative mock transport tests |
| Worker package | Frontend typecheck/lint/build/tests; Wrangler dry-run | Explicit separate test Worker config with ASSETS/IMAGES; no Worker published during local validation |

Tests are in `backend/tests/release/`; frontend deployment checks are in `frontend/tests/deploy.test.mjs`. The frozen schema fixture can be refreshed deliberately with `python -m tools.member_contracts --reference <reference-backend>`.

The actual mobile F&B-only build, browser visual/role acceptance, live Render deployment and hosted read-only checks remain external release gates. See [Render/Cloudflare runbook](docs/FNB_RELEASE.md) and [architecture](ARCHITECTURE.md).


## Station KOT routing (backend)

Staff routes below are under `/api/v1`. Their shared shapes remain in [KOT API contract](docs/KOT_API_CONTRACT.md). Frontend implementation is separate.

| Methods | Paths | Permission / verification |
|---|---|---|
| GET | `/kot/setup` | `outlets.setup`; full setup, boolean active fields, agent keys excluded |
| POST / PUT | `/kot/stations`, `/kot/stations/{id}` | `outlets.setup`; venue/name uniqueness, mapped station deactivation refused |
| PUT | `/kot/section-map`, `/kot/product-overrides/{product_id}` | `outlets.setup`; same-venue menu validation, null unmaps/removes |
| POST / PUT | `/kot/printers`, `/kot/printers/{id}` | `outlets.setup`; host, port, 80/58 mm, status |
| POST | `/kot/printers/{id}/test` | `outlets.setup`; 202 structured test job |
| POST / PUT | `/kot/agents`, `/kot/agents/{id}` | `outlets.setup`; key shown once, only digest stored, active toggle |
| GET | `/kot/routing?venue=...` | `outlets.serve`; override precedence, active stations, null for unmapped products |
| GET / PATCH | `/kot/tickets?venue=...&station_id=...&include_done=false`, `/kot/tickets/{id}/status` | Reads: `outlets.serve`; status writes: `outlets.kitchen`; per-station Placed -> Preparing -> Ready, order/ingredient synchronization |
| POST | `/kot/tickets/{id}/reprint` | `outlets.serve`; retries a KOT that never printed, or sends a COPY of one that did; Placed/Preparing tickets only |
| GET | `/kot/alerts` | `outlets.serve`; failed or 30-second unprinted jobs, 60-second offline active agents |
| GET | `/kot/agent/jobs?limit=10` | `X-Agent-Key`; atomically claims jobs with 60-second leases, attempts and COPY flag |
| POST | `/kot/agent/jobs/{id}/printed`, `/kot/agent/jobs/{id}/failed` | `X-Agent-Key`; current claimant/lease required, idempotent confirmation, connect/send retry distinction |
| GET | `/dashboard/attention` -> `kot_unprinted` | Registered core attention list using the same unprinted rows |

Existing till create/add-items reject unmapped lines with the contract's 409 message. Order responses include `kot`, `print_alerts`, item `station_id`/`station_name`; Drafts and legacy orders have empty ticket summaries. Replay does not fire again. Ticketed orders refuse whole-order Preparing/Ready and cannot be Served until every live ticket is Ready. Adjustments issue void pending/void/rejected, changed, moved and joined slips only to affected unserved stations; comps print nothing. Joins retain ticket ownership on the surviving order.

Acceptance coverage: `backend/tests/fnb_outlets/test_kot.py`, updated outlet and fresh-install tests, and `backend/print_agent/tests/test_agent.py` (both paper widths, COPY/additional/slip layouts, fake TCP delivery and transport failure stages). The agent protocol returns claimed jobs as `{id, payload, lease_until, attempts}[]`; `payload` is structured print data, and acknowledgements return `{printed: true}` or `{queued: true}`. Failed jobs remain queued with a bounded retry delay; staff projections show `print_status: failed` while an error remains.

## Rumanza F&B pilot (profiles/pilot.env)

Modules `catalog, finance, fnb_inventory, fnb_outlets, payroll_shifts`; features `finance.purchasing, finance.sales_reports, finance.controls, staff.shifts, roles.consolidated`.

| Area | Endpoints | Verification |
|---|---|---|
| Member order (checked in the club's member system) | `POST /cafe/orders` with `member_ref`, `guest_type: "Member"` | Membership number stored, member discount per line, no `member_id`; member-account tender 422; `/cafe/members/verify` 404 |
| Checkout discount | `PUT /cafe/orders/{id}/discount` | Reason required; up to `max_checkout_discount_percent` (POS settings) applied, above it 202 for approval (`pos_checkout_discount`) |
| POS settings | `PUT /pos/settings` (`max_checkout_discount_percent`), `PUT /pos/fnb-tax` | F&B tax rate through the tax-rate approval, from today or later |
| Linking pilot orders to members (members enabled) | `GET /cafe/member-refs`, `POST /cafe/member-refs/link` | Matched/unmatched membership numbers; matched orders get `member_id` |
| Switched off in the pilot | members, member app, `/finance/payroll-runs`, `/finance/pay-rates`, `/finance/labor-cost`, `/gl/*`, `/assets/*`, `/tax/*`, `/finance/coupons*`, `/finance/collections`, `/cafe/reservations` | 404 (`tests/release/test_pilot_profile.py`) |
| Stock per venue | `GET /fnb/locations` | Indoor Cafe Kitchen (also On Course), Rooftop Kitchen, Bar (GL 1240) |
| Pilot to full product | same database, `profiles/full.env` | Deferred member FKs and `collection_activity` created, pilot orders linked, payroll and ledger back |

## Service roles — B1

`backend/tests/core/test_roles_consolidated.py` runs against the full and pilot profiles: the flat role catalogue,
consolidated-role assignment refusal, Manager coverage of permission groups, approval policies and route role
dependencies, waiter/cashier/cook writes, venue isolation, quantity-only inventory lead receipts, self clock-in/out, password
login refusal for PIN roles and the read-only legacy-account report. Registry tests cover consolidation opt-in.
Device/PIN sessions and cook stations are implemented in B2/B3 below; B4-B6 cover kitchen, till and stock.
B7 is implemented; prototype-matching B8 remains incomplete, as tracked in `docs/SERVICE_BACKEND_PLAN.md`.


## Service authentication — B2

All paths are under `/api/v1`. `tests/core/test_devices_and_pins.py` exercises both full and pilot profiles.

| Methods | Paths | Contract coverage |
|---|---|---|
| POST / GET | `/auth/login`, `/auth/me` | Staff `Me`, remembered expiry, device-bound walk-up, PIN-role password refusal |
| POST | `/auth/logout` | Session ends; token becomes unusable |
| PUT | `/auth/my-password`, `/auth/my-pin` | Credential verification; strong new secrets; fresh unrestricted PIN token |
| POST / GET | `/auth/devices` | Manager only; creation returns one-time key; public lists exclude key/hash |
| POST | `/auth/devices/{id}/revoke`, `/auth/devices/{id}/unlock` | Session revocation, durable PIN lock/reset, `devices` event |
| GET / POST | `/auth/device`, `/auth/pin` | Key required; venue/role roster; concurrent failures and restart-safe lockout |
| GET / POST / PATCH | `/staff`, `/staff/{id}` | StaffRow; nullable PIN credentials; mode/venue validation; approved email transition |
| Existing writes | Core approvals | `approvals` signals on creation, decisions, withdrawal and scheduled processing |

Security coverage includes first-PIN-change gating, exact device/PIN error codes, another venue/password role
refused for PIN login, old tokens invalid after logout/reset, device revocation after the 30-second cache expires,
and credential/key/hash exclusion from responses and audit details. CORS accepts `X-Device-Key`. No B3 station
assignment or floor/menu/order extensions are included in B2.

B2 verification (2026-10-13): all 17 service-auth cases passed in each of full and pilot. The initial core folder
had 129 passed and two approval-PIN fixture collisions; both corrected cases passed in the targeted rerun.
Migration and SQL error redaction tests each passed separately. Release 22, inventory 16 and payroll 13 passed;
all three import contracts remain green. Detailed split-run evidence and phase boundaries are recorded in
`backend/docs/SERVICE_REFERENCE_MAP.md`. B9 full-suite sign-off remains pending.


## Service floor, menu and orders — B3 (2026-10-13)

All paths below are under `/api/v1` and are implemented in the outlets module.

| Method | Path | Behaviour |
|---|---|---|
| GET / PUT | `/cafe/floor`, `/cafe/floor/{venue}` | Layout, features, unpaid-order state and ticket lateness; setup-only transactional layout replacement preserves occupied tables and table ids |
| GET | `/cafe/menu`, `/cafe/availability` | Venue menu, descriptions, station routing, applicable modifier groups and recipe-based portions |
| POST | `/cafe/orders`, `/cafe/orders/{id}/items` | Empty seating, guest/cost-centre details, seats and holds; printed bills reject new items |
| POST | `/cafe/orders/{id}/fire` | Selective or all-held release; additional tickets, no duplicate firing |
| PATCH | `/cafe/orders/{id}/lines/{line_id}` | Seat, note and hold edits before ticket creation |
| POST / DELETE | `/cafe/orders/{id}/bill` | Print marker/optional durable print job, reopen only before any payment, `till` notifications |
| GET | `/cafe/orders?venue=&open=true` | Unpaid `OrderSummary[]` with waiter, balance and bill/discount state; legacy unfiltered lists retain expanded orders |
| PATCH / PUT | `/cafe/orders/{id}/details` | Covers/guest type and manager-only handover; PUT remains for legacy clients |
| POST / PATCH | `/staff`, `/staff/{id}` | Cook station assignment through the role-owned hook; identity and device roster show the assigned station or pass |
| POST / PATCH | `/catalog/products`, `/catalog/products/{id}` | Description, nullable and at most 120 characters |

Order detail responses add the service `lines`, totals, paid/balance, waiter, payment projections and discount
request. Stored `partially_paid` is projected as the contract's `partial`; legacy item/amount fields remain.
Ticket lines expose seats and `done: false` until B4 supplies the crossing-off endpoint. B5 still owns payment
reference/tendered writes. B3 verification is detailed in `backend/docs/SERVICE_B3_REPORT.md`.

Owner-approved B3 subtotal clarification: both profiles return subtotal before the separately reported
member discount, as the prototype displays it. The stored net subtotal, price calculations, tax, service
and accounting are unchanged. Checkout discounts apply after the member discount.

Existing cash/card/member payment and host/department settlement paths reject live held lines until
fired. This prevents an order being paid while some of its dishes can no longer be sent to the kitchen;
no B5 payment-write fields, tender rules or register endpoints are introduced by this guard.

## Service backend B4 — kitchen (2026-10-13)

- GET /api/v1/kot/tickets accepts status and include_done. Cook venue/station are server-confined; pass cooks may filter the venue's stations.
- PATCH /api/v1/kot/tickets/{id}/status supports Placed→Preparing→Ready, Ready→Preparing recall, Ready→Served pickup and owner-approved Served→Ready Undo before any payment; cancelled orders refuse Undo. Delivery/Undo permit outlets.order and outlets.kitchen; other kitchen transitions require outlets.kitchen.
- Owner-approved PATCH /api/v1/kot/tickets/{id}/status Served→Preparing: kitchen group only, existing venue/station checks, cancellation and any-payment (including reversed) 409 guards under order-then-ticket locks. Same ticket/timestamps/done flags, Undo state recomputation, no stock/print/slip/alert creation, existing kot/orders events and audit. Full/pilot tests and source-based staff-meal/cost-centre/list answers: backend/docs/SERVICE_B4_REPORT.md addendum.
- PATCH /api/v1/kot/tickets/{id}/lines/{order_line_id} stores/clears done_at for live lines, with kitchen and station checks, and announces kot.
- Ticket reprint and unprinted alerts also confine station cooks. Shared print-agent alerts remain visible.
- Existing item DELETE/cancel and whole-order cancel routes use pos_cooked_void for Preparing/Ready dishes. Manager approves; pending/rejected/withdrawn/failed/applied states retain KOT slips and order projections. Request and pending slip commit atomically.
- Pickup completion waits for every live ticket and no held dishes. Recall/Undo do not repeat ingredient consumption or restore cooked stock. B4 source and split test evidence: backend/docs/SERVICE_B4_REPORT.md.

## Service backend B5 — till (2026-10-13)

- GET /cafe/bills: cashier/pay group, venue confinement, oldest printed unpaid bills and dining summaries.
- POST /cafe/orders/{id}/payments: reference/tendered/change fields; contract overpayment 422, transfer
  reference validation; idempotent captured replay and incompatible/removed-reference conflicts.
- DELETE /cafe/orders/{id}/payments/{payment_id}: unpaid positive till payments, current shift only;
  audit and reversed history retained. Paid bills use Manager refunds.
- GET /cafe/registers/current, POST /cafe/registers/close, GET /cafe/registers/closes: blind current totals,
  valid denomination counts/coins, variance note Rs 100+, atomic next shift and Finance-owned close/history.
- PUT /pos/settings and GET /pos/setup: register_float; nullable policy-derived approval_manager_name.
- Expanded Order: nullable checkout_discount_by_name tied to the currently applied approval.
- Existing department/host settlement announces till/orders and preserves expense/member/tax ownership.

Source and split targeted validation evidence: backend/docs/SERVICE_B5_REPORT.md. This does not establish
that the live server has loaded the source or that the public database has been migrated.


B6 adds scoped GET /fnb/overview, /fnb/forecast and /fnb/batches/expiring; extended enquiry/ingredient/settings fields; /fnb/waste aliases; FIFO batch provenance; pilot second-manager purchase decisions; transactional stock events. See backend/docs/SERVICE_B6_REPORT.md for contracts and validation.

## B8 independent seed boundary (2026-10-13)

`python -m scripts.seed --showcase --now <timezone-aware-ISO>` creates the independent prototype foundation
on a fresh development database: exact static data/opening stock, service staff and all six owner-authorized
devices. No-flag usage retains older showcase; `--no-showcase` selects basic demo. New mode disables inherited
legacy demos, refuses populated unmarked/deployed targets, commits as one transaction and prints only new
keys after commit. Matching reruns preserve credentials, revoked devices, times and identities. Secrets
are absent from metadata/reports. Demo-only hashed PIN 1357 waives first change; ordinary creation/reset
still requires it.

`backend/docs/SERVICE_B8_REPORT.md` records focused full/pilot checks, actual upgrade retention evidence and
pending scope. B7 endpoints are implemented; operational showcase histories and dependent fixture validation remain pending; empty-recipe unlimited
ordering and exact foundation modifier scoping across upgrade need decisions. No pilot/public seed,
deployment or B9 sign-off. Exact command/guards in that report apply only to fresh development targets.

Pre-B7 F6 additions (scoped lead count submission/discard, waste cost/note, ingredient use/cost, request
timeline and movement references): `backend/docs/SERVICE_PRE_B7_ANSWERS.md`, shared contract section 6.
Approved discard retains history without stock or journal changes.

B7 continuation (2026-10-13): manager venue/payment-hour, discount/void and purchase/transfer reports,
untaxed received-value settlement and owner-provided exception feeds are implemented. Full/pilot evidence
and limitations: `backend/docs/SERVICE_B7_REPORT.md`; shared contract section 8 records exact semantics.
No deployment, public migration/seed or B9 acceptance is inferred from these checks.


## Service continuation: F7, Manager, B8-B9 (2026-10-13)

Receipt read /finance/goods-received now includes Store, strictly F&B Store department only; accepted receipt
value is additive. Inventory enquiry exposes explicit usual supplier/perishable/FIFO batch; purchase
request/PO exposes aware delivery time; expiring lots expose midnight expiry status. Manager daily order
covers/tax/unpaid counts and payment actor/venue/table remain owner-contributed RevenueSource metadata.
Costing adds scoped complete product/venue series and posted count variance, entered recipe percentage
overhead cards, and explicit-empty-recipe unlimited ordering. Supplier fill metrics derive actual accepted
PO line ratios; roster PATCH edits times/location with all existing guards.

New workbook GETs: /finance/purchase-reports/dashboard/export.xls, /finance/purchase-reports/transfers/
export.xls, /inventory/fnb/waste/export.xls, /inventory/fnb/store/export.xls, /finance/exceptions/export.xls,
/audit-logs/export.xls. Permissions and bounded report scopes are in SERVICE_API_CONTRACT.md.

Service contract table route presence/authentication is exercised live in both disposable profiles by
`test_every_contract_table_endpoint_is_loaded_and_auth_guarded`; existing B1-B7 workflow/error tests plus
`test_service_continuation.py` cover success and refusal paths. Operational showcase tests cover whole-owner
rollback, key-once CLI/reruns, linked histories, parsed workbook content and actual foundation pilot-to-full
retention. Exact complete suite counts and coverage limits are in backend/docs/SERVICE_B9_REPORT.md;
no browser, physical printer or deployment acceptance is implied.

Final continuation interface/evidence: backend/docs/SERVICE_FRONTEND_HANDOFF.md, SERVICE_B9_COVERAGE.md and SERVICE_B9_REPORT.md. No frontend ownership or deployment acceptance is implied by the automated backend checks.
