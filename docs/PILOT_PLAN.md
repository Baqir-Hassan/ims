# Rumanza F&B pilot — plan

**Status (2026-10-13): profile infrastructure and service B1-B6 implemented in source, with the split
validation recorded in the phase reports.** B7 manager aggregates/reports are incomplete; B8 showcase and
combined release-flow checks remain pending. This is not production or deployment sign-off. Profiles live
in `backend/profiles/` and `frontend/profiles/`; existing pilot/upgrade checks live in
`backend/tests/release/test_pilot_profile.py`; deployment steps are in `docs/FNB_RELEASE.md`.
The member-app "Ordering Hours" screen is behind members; service venue hours remain outlet-owned.

A hosted pilot for Rumanza, branded as Rykards and named "Inventory Management System" (`IMS_BRAND=rykards` in the frontend profile), so they can evaluate us. Rumanza's own ERP keeps members,
membership verification, accounting and payroll. IMS covers:

1. F&B Store inventory
2. Stock per venue (one location per venue)
3. POS: member discount, an extra discount at checkout, stock consumption by recipe, table plans,
   department/staff meals, station KOT printing
4. F&B Manager restock approvals
5. Finance for F&B only: purchasing and vendors, F&B sales reports, costing and outlet P&L
6. F&B staff shifts and attendance (rosters, clock-in/out, attendance confirmation) — **no payroll**

```
IMS_MODULES=catalog,finance,fnb_inventory,fnb_outlets,payroll_shifts
IMS_FEATURES=finance.purchasing,finance.sales_reports,finance.controls,staff.shifts,roles.consolidated        # see §2
```

**Not in the pilot:** the members module (its code stays in the repo but is never loaded or shipped to
the browser), payroll (pay rates, payroll runs, labour cost), GL / journals / financial statements, tax
invoices, receivables, member-app ordering, e-receipts, reservations, and On Course orders.

Roles in the service pilot: **F&B Manager** (`manager`), **F&B Store** (`fnb_store_staff`), **Waiter** (`waiter`),
**Cashier** (`cashier`), **Cook** (`cook`) and **Inventory lead** (`inventory_lead`). The last four work in one
venue and use device-bound PIN sign-in; a cook is assigned a station or the venue pass. Manager spans venues. Inventory
leads confirm receipts for their own venue; store staff dispatch. See the shared service API contract.

**Backend rollout status (2026-10-13): service B1-B6 implemented in source.** `roles.consolidated` is on
in the backend pilot profile. The catalogue labels manager "F&B Manager" and marks kitchen_staff, fnb_manager,
finance_staff and headstaff unassignable; POST/PATCH staff rejects assignment to these roles. Existing legacy
accounts remain available until reassigned; run `python -m scripts.consolidate_roles` from backend to list them
without changes. The full profile retains legacy role assignment.

Managers can now provision and revoke shared devices, unlock their persisted PIN lockout and create/reset
venue staff PINs. Staff sessions expire after 12 hours; remembered personal password sessions last 14 days.
First sign-in after a PIN reset requires a PIN change. Logout and device revocation end the affected sessions.
The approved StaffUpdate email field permits moving PIN accounts to password roles with email and password.
No migration has been run against the pilot public schema. Existing staff tokens require fresh sign-in when B2
is deployed. B3 owns floor layouts, menu/modifier projections, seats, held dishes and printed bills; B4 owns
station/pass kitchen access, pickups, kitchen recall and cooked-void approval; B5 owns cashier payments and
register shifts; B6 owns inventory-lead receipt confirmation, scoped stock/forecast/batches and pilot purchase
thresholds. A venue role without a valid assignment is refused by the scope helper. B7 manager aggregates
and reporting extensions remain incomplete, and the prototype-matching B8 seed is not implemented.
Phase reports under `backend/docs/` and `SERVICE_REFERENCE_MAP.md` distinguish source, targeted testing and
remaining release gates; B8 preparation is in `backend/docs/SERVICE_B8_REPORT.md`.

Waiters order, hold/fire dishes, collect ready tickets and print bills; cashiers take venue payments, give
within-limit checkout discounts and count/close registers. Station cooks can change only their station's
tickets; pass cooks see every station in their venue. Kitchen-only Served-to-Preparing recall and shared
Served-to-Ready pickup Undo require no recorded payment, including reversed payments. Inventory leads
request, count and waste their venue stock and confirm its dispatched receipts; store staff dispatch and
receive vendor deliveries. Managers provision staff/devices and decide approvals across venues.

In the consolidated pilot, `second_approval_above` applies at or above the configured amount and requires
two different eligible managers; the old separate `/finance-decision` purchase step returns 404. Below the
threshold (or with a null threshold), one eligible manager can convert the request. The full profile keeps
its existing optional Finance approval switch. Service source completion does not establish live deployment
or an uninterrupted B9 suite result.

## 1. Members becomes optional (core + finance + outlets)

The implemented optional-module boundary lets the pilot start without members. Finance and outlets declare
members as optional and load its extras only when enabled; inventory has no members dependency. The following
rules describe that composition rather than an outstanding dependency-removal task.

**Core** (`backend/app/core/registry.py`, `loader.py`; `frontend/app/core/modules.ts`, `registry.ts`):
- `optional_depends_on` / `optionalDependsOn`: these control load order when the module is present and
  never block startup.
- `when_enabled={"members": Extras(routers, member_routers, ensure_schema, seed_demo)}`: extras loaded only
  when that module is enabled. `registry.enabled(name)` for guarded code paths.
- A `receivable_sources` registry, so finance never reads the `members` table.
- **Frontend component slots:** members contributes `member-picker`, `scan-member-code` and
  `email-receipt`. Finance and outlets render the slot, which is empty without members. This replaces the
  direct `../../members/...` imports, which would otherwise ship members code in the pilot bundle.
- Loader tests for optional dependencies and `when_enabled`. Update ARCHITECTURE.md and the import contracts.

**F&B inventory:** remove `members` from `depends_on` (unused).

**Finance** (`depends_on=("core",)`, `optional_depends_on=("members",)`):
- Dashboard, receivables aging, dues list and balance sheet: read `receivable_sources` instead of
  `members` (members registers its own source).
- Collections endpoints, `receipts_bank.py`, `collection_activity` and every FK to `members` (including
  `tax_invoices.member_id` and `MEMBER_FOREIGN_KEYS`): moved to `when_enabled["members"]`. FKs are added
  with guarded `DO` blocks, as for golf in E11.
- `services/tax.py` `_member()`: only when members is enabled.
- `services/documents.py`: read `active_registry().club_details` directly, with no members import.
- Frontend: `receipts-bank.tsx` (imports `MemberPicker`) moves into members, which contributes it.

**F&B outlets** (`depends_on=("core","finance","catalog","fnb_inventory")`, `optional_depends_on=("members",)`):
- `schema.py`: the FKs from orders, payments, reservations and on-course orders to `members` move to
  `when_enabled["members"]`. The columns stay nullable.
- Member-app routers, member sections and the app receipt source register only with members.
- Without members: the `member_account` tender, host-member charging, `/cafe/members/verify` and
  `issue_receipt` are unavailable (422 / not mounted). Members imports move inside the functions that use them.

## 2. Feature switches (scope only)

Trimming finance and outlets is about **scope, not secrecy**, so a runtime switch is enough.

- Core: modules may group routers, screens and navigation under named **features**. A new env
  `IMS_FEATURES` lists the enabled features; when it is unset, default features are on (roles.consolidated requires explicit selection) (the full product is
  unchanged).
- The backend mounts only enabled feature routers. The frontend registry hides screens and navigation of
  disabled features (the code may stay in the bundle).

**Finance features**
| Feature | Screens |
|---|---|
| `finance.purchasing` ✅ | Vendors, Vendor Directory, Vendor Bills, Payment History, Vendor Documents, Vendor Scorecard, Purchase Orders, New Order, Orders in Queue, Order History, PO Templates, Blanket & Recurring Orders, Purchase Requisitions (list/new), Goods Received, Bill Inbox, New Bill, Bill Matching, Exception Queue, Credit & Debit Notes, Approval Queue |
| `finance.sales_reports` ✅ | Revenue by Source and Revenue Transactions (F&B sources only), Daily Sales (Z-Report), Outlet Transactions, Payment Methods (Cash, Card, Bank Transfer), Discounts & Comps, Discounts Used, Voided Transactions, Refund Queue |
| `finance.gl` ❌ | General Ledger, Ledgers, Sales Tax & GL Export, P&L Statement, Balance Sheet, Cash Flow, Period Close, Fixed Assets, Forecast, Custom Report |
| `finance.tax` ❌ | Tax & Compliance (tax invoices) |
| `finance.receivables` ❌ | Accounts Receivable, Collections, Member Account / In App payments |
| `finance.expenses` ❌ | Expenses, Expense Log, Recurring Expenses, Prepaid & Deposits, Staff Expense Claims, Expense Reports, Contract Register, Budgets |
| `finance.controls` ✅ | Finance dashboard (F&B purchasing and sales tiles only; receivable tiles hidden without a source), Register Close, Audit Log, Exceptions |
| `finance.cash` ❌ | Cash & Bank, Petty Cash |
| `finance.coupons` ❌ | Create Discounts & Coupons, Coupons Used |

- Costing (recipes, ingredients, menu engineering, outlet P&L, waste) belongs to `fnb_inventory` and stays.
  Without payroll, the outlet P&L has no labour line.
- The till's **tax % and service charge** come from finance tax settings (`rate_percent("fnb")`). Keep a
  small "F&B tax rate" setting in POS setup, so the pilot can set it without the tax screens.
- Journal sources keep registering; with `finance.gl` off they are simply never shown.

A feature can carry registrations as well as routers and screens (approval policies/actions, journal and
cost sources, GL accounts), using the same `Extras` shape as `when_enabled`, so a disabled feature leaves
no trace in the approval editor or reports.

**Shifts without payroll — `payroll_shifts`**
| Feature | Backend | Screens |
|---|---|---|
| `staff.shifts` ✅ | `/payroll/options`, `/staff/shifts` (list/create/update), clock-in/out, attendance edit/confirm | F&B Shift Rosters, F&B Attendance |
| `staff.payroll` ❌ | `/finance/pay-rates`, `/finance/payroll-runs*`, `/finance/payroll-run-lines/*`, `/finance/labor-cost`; journal source, GL accounts 6250/2220/2230, `fnb_labour` cost source, `payroll_run` approval policy/action | Staff Pay Rates, Payroll Runs, Labour Cost |

- Split `payroll_shifts/routes.py` into `routes_shifts.py` and `routes_payroll.py`, and move the payroll
  registrations into the `staff.payroll` feature in `manifest.py`. Do the same with the frontend manifest's
  screens and role navigation.
- Without `fnb_labour`, the outlet P&L has no labour line. Costing must handle "no cost sources".
- Work locations in `scope.py` (`DEPARTMENTS`) follow the per-venue stock change (§3): Indoor Cafe →
  Indoor Cafe Kitchen, Bar → Bar, Rooftop → Rooftop Kitchen, F&B Store → F&B Store. Drop On Course.
- The demo seed creates a week of rosters and attendance, but no payroll run when `staff.payroll` is off.
- Staff in scope include waiter, cashier, cook, inventory lead, F&B Store and F&B Manager. The F&B Manager builds
  rosters and confirms attendance; staff clock themselves in and out.

**Outlet features:** `outlets.reservations` ❌, `outlets.on_course` ❌. Tables (`cafe-tables`),
department/staff meals and KOT stations stay on.

## 3. Stock per venue — `fnb_inventory`

Replace the shared "Cafe & Bar Kitchen" with one location per venue:

| Location | Kind | GL account | Supplies |
|---|---|---|---|
| F&B Store | fnb_store | 1210 | — (buys from vendors) |
| In transit | fnb_transit | 1215 | — |
| Indoor Cafe Kitchen | kitchen | 1220 | Indoor Cafe |
| Rooftop Kitchen | kitchen | 1230 | Rooftop |
| Bar | kitchen | 1240 (new) | Bar |

- Update `LOCATIONS`, the GL account list and the `fnb_venue_locations` mapping in `fnb_inventory/schema.py`.
- Existing (non-pilot) databases: rename "Cafe & Bar Kitchen" to "Indoor Cafe Kitchen" and keep its
  stock. Bar starts empty and requests stock from the store. On Course keeps using the Indoor Cafe Kitchen.
- Kitchen requests, counts, waste and stock enquiry automatically list the new location. Check the Bar's
  screens and the F&B Manager approval queue.
- Update `docs/FNB_INVENTORY_FLOW.md` and `docs/FNB_DFD.md`.

## 4. POS for the pilot — `fnb_outlets`

**Members (verified in the ERP, no lookup in IMS)**
- Customer type "Member": the cashier checks the member in the ERP, then enters the **membership number**
  and name. IMS stores them on the order (`cafe_orders.member_ref`, new) and does no verification.
- Service order creation accepts `guest_type="Member"` with membership number and name. This records
  identity checked outside IMS in the pilot; backend source does not prove frontend/browser acceptance.
- **Member discount:** one % for all members, set by the Manager in POS setup on the existing "Member"
  guest type (`pos_guest_types.discount_percent`). As today, each line gets the single best of the
  guest-type % and any matching discount rule (happy hour etc.); they don't stack.
- Members pay at the till (cash / card / bank transfer) like guests. No member-account tender and no
  e-receipts.

**Extra discount at checkout** (new)
- The backend already stores an order-level discount (`discount_type` percentage/amount, `discount_value`),
  but only at order creation, and the till never sends it.
- Add `PUT /cafe/orders/{id}/discount {discount_type, discount_value, reason}` for unpaid orders (it
  recalculates totals), and a checkout control on the till. It applies after the member/line discounts,
  before service charge and tax.
- Show it on the receipt and in "Discounts Used".
- **Controls:**
  - a reason is always required;
  - any cashier may give up to a Manager-set limit (`pos_settings.max_checkout_discount_percent`, new;
    an amount discount is converted to a % of the subtotal for the check);
  - above the limit, the request goes through core approvals under a new policy
    `pos_checkout_discount` (Manager / Head staff), with an `ApprovalAction` that applies the discount
    once approved;
  - the limit is set in POS setup.

**Also**
- Tables stay; reservations and On Course are hidden (§2).
- Department and staff meals stay.
- Station KOT printing per `KOT_ROUTING_PLAN.md`. The Bar is now its own stock location; its KOT stations
  are unaffected.

## 5. Deployment, seed and tests
- `render.yaml` and a pilot `wrangler.jsonc` environment with the pilot `IMS_MODULES` / `IMS_FEATURES`;
  the frontend generator reads both at build time.
- The demo seed works without members and payroll (no member rows, no payroll run; rosters and
  attendance are seeded).
- Existing backend checks in `tests/release/test_pilot_profile.py` cover disabled optional surfaces,
  member-ref discounts, legacy-actor sales/KOT/stock, shift creation and same-schema member/full upgrade.
  They do not yet prove B8's device/PIN waiter/cook/cashier flow, blind register close, correct lead receipt
  or complete two-manager purchasing flow.
- B8 must extend those actual tests on disposable schemas: device setup -> PIN sign-in -> order -> kitchen
  tickets -> pickup -> printed bill -> cashier payment -> register close; lead request -> approval -> store
  dispatch -> own-venue receipt; one/two-manager thresholds and full legacy compatibility; retained service
  records after full upgrade. Optional members/payroll routes remain disabled in the pilot.
- B9 requires a clean full suite in both profiles and passing import boundaries. Earlier targeted/split
  phase results do not satisfy that gate.
- Frontend: typecheck/lint/test for the full and pilot builds. A bundle test fails if members code is in
  the pilot build. Browser check of the pilot navigation for each role.

## 6. Adding modules later

The pilot is **configuration over one codebase**, not a fork. Nothing is deleted. Turning a module or
feature on later means changing `IMS_MODULES` / `IMS_FEATURES`, rebuilding the frontend and restarting
the API on the **same database**. These rules keep that route open.

### 6.1 Rules for the pilot work
- **Unset enables defaults.** With `IMS_MODULES` / `IMS_FEATURES` unset, all modules and default features run exactly as
  today. CI runs both the full profile and the pilot profile, so the members-only and payroll paths don't
  rot while switched off.
- **Deployment profiles.** `backend/profiles/pilot.env` and `backend/profiles/full.env` hold the
  `IMS_MODULES` / `IMS_FEATURES` pairs. `render.yaml`, `wrangler.jsonc` and the release checks read a
  profile instead of hard-coding lists. Adding a module = editing one profile.
- **One way to plug in.** New links between modules use `depends_on`, `optional_depends_on` +
  `when_enabled`, features, or the existing registries (E1–E16). Never a direct import of an optional
  module at load time, and never a direct read of its tables.
- **No new hard-coded lists.** Venue → stock location goes through the `fnb_venue_locations` table, and
  roster locations through `payroll_shifts/scope.py`. New code does not add more fixed venue or location
  lists.
- **Role ids stay as in RumanzaCMS** (`membership_staff`, `frontdesk_staff`, `headstaff`, ...), so later
  modules' permissions work unchanged.

### 6.2 Deferred database steps must run when a module is switched on later
The implemented `_fingerprint()` in `backend/app/core/db.py` includes module code and applied optional
extras/features. This allows deferred `when_enabled["members"]` steps (FKs, `collection_activity`) to run
when the same pilot database is upgraded.
- A module's fingerprint includes the sorted names of its `when_enabled` modules that are
  enabled, and its features that are enabled. Turning members (or `staff.payroll`) on then re-runs exactly
  the modules affected.
- Deferred FKs are added `NOT VALID` and then validated, so existing pilot rows can't block startup.
  Their columns are only ever null or valid in the pilot.

### 6.3 Pilot data carries forward
| Turned on later | What happens to pilot data |
|---|---|
| `members` | A one-off, Manager-run **link step** matches `cafe_orders.member_ref` to `members.membership_no`, sets `member_id`, and lists anything unmatched for review. Pilot orders then show in member history. The "Member" guest type stays; tier-based guest types can be added. |
| `staff.payroll` | Pilot rosters and confirmed attendance are already the data payroll needs. Add pay rates and run payroll from that date; labour cost then appears in the outlet P&L. |
| `finance.gl` | GL journals are built from source records when read (`finance/services/journals.py`, `journal_sources`), so the ledger covers the pilot period from day one. Only manual entries and period close start at that point. |
| `finance.tax` | Tax invoices are issued from that date. Pilot sales already store their tax amounts. |
| `finance.receivables`, `finance.expenses`, `finance.cash`, `finance.coupons`, `outlets.reservations`, `outlets.on_course` | Switch on; no data step. |
| E-receipts | Return with members (they are owned there). |

### 6.4 Proof: an upgrade test
The existing test is `test_pilot_upgrades_to_the_full_product_on_the_same_database` in
`backend/tests/release/test_pilot_profile.py`; the planned separate filename does not exist. B8 must extend
that test to preserve service staff/devices/orders/stock/payment/close identities across the upgrade.
The intended broader proof is:
1. Boot the pilot profile on a fresh schema and run a short pilot flow (purchase, sale with `member_ref`,
   roster/attendance).
2. Restart with the full profile on the **same schema**.
3. Startup succeeds; the deferred FKs and `collection_activity` exist; the member link step attaches the
   pilot order; payroll runs from the pilot attendance; the GL includes the pilot sales; the existing full
   release smoke passes.

### 6.5 Remaining RumanzaCMS modules
These are ported later as new IMS modules (source: `D:\Projects\RumanzaCMS\rumanza-backend\app\api\routes`
and the matching `app/` screens), following `FNB_DEPLOYMENT_PLAN.md` §2 and the extension points E1–E16.

| Future module | RumanzaCMS sources | Depends on | Plugs in through |
|---|---|---|---|
| `golf` (tee sheet & bookings) | `bookings`, `tee_sheet`, `tee_setup`, `tee_assignments`, `tee_extras`, `courses`, `checkin`, `visitors`, `member_guests`, `range`, `carts`, `caddies` | core, members, finance; optional fnb_outlets | Revenue/journal sources, booking FKs on outlet orders (E11, via `when_enabled["golf"]` in outlets), member sections, host charging (tee part) |
| `academy` | `academy`, `academy_shop`, `simulator` | core, members, finance | Revenue/journal sources, member sections, labour cost sources |
| `pro_shop` (+ product inventory) | `pro_shop`, `inventory`, `invoices` | core, finance, catalog; optional members | GRN `product` handler (E3), stock locations of a product kind, revenue sources |
| `competitions` | `competitions`, `tournaments`, `escoring`, `score_entry`, `handicaps`, `member_competitions` | core, members; golf | Member directory fields (E14), member sections |
| `golf_cards`, `vouchers` | `golf_cards`, `vouchers` | core, members, finance | Journal/revenue sources, member ledger sources |
| `club_accounts` | `club_accounts`, `invoices` | core, finance; optional members | `receivable_sources` (new, §1), member ledger sources |
| `facilities`, `club_events`, `club_tv`, `marketing`, `maintenance` | same-named files | core (+ members where needed) | Dashboard widgets, permissions, live topics |

The new `receivable_sources` registry, `optional_depends_on` / `when_enabled` and features introduced
for the pilot are the same tools these modules use. The pilot work adds no throwaway mechanisms.

## 7. Order of work
1. Core: optional deps, `when_enabled`, `receivable_sources`, features + `IMS_FEATURES`, frontend slots,
   the fingerprint fix (§6.2), deployment profiles (§6.1).
2. Members decoupling in finance and outlets; drop `members` from inventory.
3. Feature grouping of finance, outlet and shifts/payroll screens, routes and registrations.
4. Stock per venue (and the matching roster work locations).
5. POS: member customer type + guest type, member ref, checkout discount, F&B tax setting.
6. Deployment config, seed, pilot tests, the pilot → full upgrade test (§6.4), the member link step, docs.

## B8 preparation and seed boundary (2026-10-13)

The current `python -m scripts.seed` still runs the older showcase by default; `--no-showcase` requests
basic demo data, and an explicit service `--showcase` flag is not implemented. Do not run these commands
on pilot/public data. B8 inspection found incomplete B7 reporting and prototype examples without complete
backend relationships; `backend/docs/SERVICE_B8_REPORT.md` and `SERVICE_B8_SHOWCASE_MAP.md` record the
prerequisite, mapping, pending decisions and intended rerun/security checks. No service showcase, generated
device keys, disposable-schema workflow run, production migration or B9 sign-off is claimed by this preparation.

Normal staff PIN creation/reset still requires a PIN change. The B8 requirement to waive that only for
explicitly demo-seeded accounts is pending implementation. Device creation already stores only a secure
digest and returns its key once; a future showcase must print newly created keys once after commit, never
rotate/reveal existing keys on rerun, and keep credentials out of reports and saved evidence.
