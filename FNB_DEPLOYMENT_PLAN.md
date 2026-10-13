# IMS — F&B First Deployment: Modular Separation Plan

Status: Decisions confirmed · 2026-10-09
Source system: `D:\Projects\RumanzaCMS` (branch `redesign`)
Target: `D:\Projects\ims` (`backend/`, `frontend/`)

---

## 1. Decisions taken

| Topic | Decision |
|---|---|
| Codebase | `ims` is a **new modular repo**: a shared *core* plus pluggable *modules*. F&B, Finance and Members are ported first; Golf, Academy, Pro Shop etc. are ported in later as new modules. RumanzaCMS stops being the main line for anything that has moved. |
| Outlet scope | **Full outlet operations**: POS order taking and payments, kitchen queues (Indoor Cafe, Rooftop, Bar), tables, reservations, on-course orders, waste, kitchen requests, stock. |
| Members | **Members + member app.** The member API ships, limited to F&B, account and notification endpoints. The app team ships an **F&B-only app build** against it, so tee/academy endpoints are not needed. |
| Member accounts | **Full**: charges, receipts, statements, aging, month-end billing, dues/recurring charges, statement delivery (WhatsApp/SMS/email), loyalty. |
| Finance | **All four areas**: purchasing & payables, sales/Z-reports/tax (FBR/PRA), costing, general ledger. |
| Roles | Kitchen Staff, F&B Manager, F&B Store Staff, Finance Staff, Manager, Membership Staff. |
| Database | **Fresh PostgreSQL database**, with a module-aware demo seed. No migration from RumanzaCMS data. |
| Stack & hosting | **Same as RumanzaCMS**: FastAPI + Pydantic + SQLAlchemy Core/psycopg + PostgreSQL, API in Docker; React 19 / Vinext frontend on Cloudflare Workers. |
| GL & tax codes | RumanzaCMS account codes and FBR/PRA tax categories are kept **as-is**. |
| Membership applications | **Left out.** Membership Staff create members directly. |
| Product catalog | `catalog_products` lives in its own **`catalog` module**, shared by F&B now and Pro Shop later. |
| Payroll & shifts | ims gets its **own dedicated `payroll_shifts` module**, shipped in the first deployment. Both shifts **and payroll** cover **F&B staff only**, not every club employee. It is separate from `finance` and `core`. |
| Payroll workflow | The **F&B Manager** builds rosters and confirms attendance, **Finance Staff** prepare payroll runs, and the **Manager** approves them. |
| Finance extras | Fixed assets, budgets and recurring/prepaid expenses ship as part of `finance`. |
| Head Staff / Top Brass | Not deployed. The **Manager covers everything they do today**: refunds, escalations, expense approvals and executive read-only reporting. |
| Repository | `ims` gets its **own GitHub repo**. The owner creates it and makes all commits; nothing is committed on their behalf. |
| RumanzaCMS after the move | **Reference only** for the modules that have moved to ims. No new F&B, finance or member features are added there. |

---

## 2. What ships in the first deployment

### 2.1 Modules

| Module | Purpose | Main users |
|---|---|---|
| `core` | App shell, auth, staff users, roles/permissions registry, audit log, notifications outbox and dispatch, live events, approvals engine, document numbering, settings, time zone helpers, PDF/spreadsheet helpers | everyone |
| `members` | Member directory, member auth, member app base (profile, devices, notifications), member accounts (credit transactions, statements, aging, recurring charges, rebates, loyalty), receipts and payment channels, month-end billing, billing delivery, member feedback | Membership Staff, Finance, Manager |
| `finance` | Chart of accounts, journals, ledgers, financial statements, period close, vendors, requisitions/POs/GRN, vendor bills/matching/payments/credit notes, expenses, bank, petty cash, register close, budgets, tax (FBR/PRA invoices, registers, withholding, returns), revenue-by-source and payment-method reports, coupons/discounts, fixed assets | Finance Staff, Manager |
| `catalog` | Sellable items (`catalog_products`): name, category, venue, price, SKU/barcode, tax category, active flag, stock adjustments. Generic, with no F&B or Pro Shop rules inside it | Manager (menus); used by F&B screens |
| `payroll_shifts` | Shift rosters for F&B staff (Kitchen, F&B Store, F&B Manager), clock-in/attendance, pay rates, payroll runs, labour cost by outlet; payroll posts to the GL through a journal source | F&B Manager (rosters), Finance Staff (payroll), Manager |
| `fnb_outlets` | Menus (Cafe/Rooftop/Bar catalog), POS (order builder, modifiers, guest types, discount rules, happy hour, covers, split/join/move table), payments and receipts at the till, kitchen queues, tables, reservations, on-course orders, ordering hours, department/host charging of F&B bills, offline tills, consumption report | Kitchen Staff, Manager |
| `fnb_inventory` | F&B Store, kitchen requests (store → kitchen transfers), F&B purchase requests, stock enquiry by location, counts, valuation, waste, recipes/ingredients, costing (menu engineering, outlet P&L, cost targets), F&B settings | F&B Manager, F&B Store Staff, Kitchen Staff, Finance |

Dependency direction (must never be reversed):

```
core ◄── members ◄── finance ◄── fnb_inventory ◄── fnb_outlets
  ▲                     ▲  ▲          │                  │
  │                     │  │          ▼                  ▼
  ├── catalog ◄─────────┼──┼──────────┴──────────────────┘
  │                     │  └── payroll_shifts
  └─────────────────────┴── (later) golf, academy, pro_shop, facilities, ...
```

- `catalog` depends only on `core`. `fnb_inventory` (recipes) and `fnb_outlets` (menus, order lines) depend on it; Pro Shop will too.
- `payroll_shifts` depends on `core` and `finance` (to post payroll journals). It knows staff by role, not by F&B module, so widening it to all staff later is a configuration change, not a rewrite.
- `fnb_outlets` depends on `fnb_inventory` because a sale consumes recipe ingredients at the venue's kitchen.
- `finance` never imports from F&B or payroll; they *register* things with finance (see §4).

### 2.2 Explicitly **not** in the first deployment

Tee sheet and bookings, starter/kiosk/gate/QR check-in, range, carts, caddies and caddie pay, academy/coaches/simulator, pro shop and club fitting, tournaments/competitions/eScoring/handicaps/federation, facilities, club TV, golf cards, vouchers, club (corporate) accounts and customer invoices, marketing campaigns, membership application pipeline, maintenance, shifts/payroll for non-F&B staff, product (non-F&B) inventory, access control hardware.

> Membership *applications* (lead → interview → approval → fee) are left out: Membership Staff create members directly.

### 2.3 Roles and navigation (first deployment)

Role ids stay **exactly** as in RumanzaCMS (`kitchen_staff`, `fnb_manager`, `fnb_store_staff`, `finance_staff`, `manager`, `membership_staff`) so later modules can reference them unchanged. Roles not deployed (`headstaff`, `frontdesk_staff`, `top_brass`, …) may still appear in permission tuples; they simply have no users until their module ships.

| Role | Screens (navigation ids carried over) |
|---|---|
| Kitchen Staff (one login per venue; each opens only its own venue's till and kitchen screens, and its records) | overview, order-builder, rooftop-order, bar-order, cafe-kitchen, rooftop-kitchen, bar-kitchen, cafe (on-course), reservations, order-history, kitchen-waste, kitchen-requests, fnb-stock, consumption, fnb-costing |
| F&B Manager | overview, fnb-manager-queue, fnb-stock, kitchen-requests, kitchen-waste, fnb-costing, fnb-settings, approvals, shift rosters & attendance (F&B staff) |
| F&B Store Staff | overview, fnb-store, fnb-stock, fnb-settings (read) |
| Finance Staff | overview, finance dashboard/revenue (F&B + membership sources only), daily Z-reports (Indoor Cafe, Rooftop, Bar, On Course), payment methods, discounts/coupons, vendors, POs, requisitions, GRN, bills, credit notes, approval queue, expenses, kitchen costing (recipes, ingredients, menu engineering, outlet P&L, waste), margins, voided sales, receivables, collections, bank, register close, petty cash, budgets, P&L, balance sheet, cash flow, period close, general ledger, fixed assets, tax, receipts, billing delivery, month-end, member finance screens, approvals, consumption, fnb-costing, pay rates, payroll runs, labour cost |
| Membership Staff | overview, member directory/members, member accounts, member billing setup, month-end, billing delivery, receipts, member feedback, approvals, membership renewals |
| Manager | everything above + staff, pricing/menus (cafe/rooftop/bar menu admin, cafe tables), pos-setup, fnb-hours, fnb settings toggle, notification outbox. Also everything Head Staff and Top Brass do today: refunds, escalations, expense approvals, executive read-only reports (all-source revenue, audit trail). |

Head Staff and Top Brass duties move to the Manager by **adding `manager` to the permission groups** that currently list `headstaff`/`top_brass` (refunds, escalations, expenses, executive reports), not by renaming roles. When those roles ship later, they are added back to the same groups and nothing else changes.

---

## 3. Target repository layout

### 3.1 Backend (`ims/backend`)

```
backend/
  app/
    main.py                 # builds the app from enabled modules (no business code)
    core/
      db.py                 # engine, ?-placeholder shim, connection(), per-module schema state
      auth.py               # tokens, current_user, require_roles, member auth helpers
      registry.py           # Module dataclass + extension-point registries (see §4)
      loader.py             # IMS_MODULES, manifest discovery, dependency checks and order
      permissions.py        # role catalogue assembled from module manifests
      services/             # operations (audit, fetch_*), messaging, notification_delivery,
                            # live_events, approvals, gl numbering (next_number), pdf,
                            # spreadsheet, throttle, cache, clock (club_now / CLUB_TIME)
      routes/               # system, staff, configuration, approvals, live, dashboard shell
      schema.py             # staff_users, audit_logs, notification_outbox, approval_*,
                            # doc_numbering, pricing/settings, schema_state
      manifest.py
    modules/
      members/   { manifest.py, schema.py, seed.py, contracts/, services/, routes/ }
      finance/   { ... }
      catalog/   { ... }
      payroll_shifts/ { ... }
      fnb_inventory/ { ... }
      fnb_outlets/   { ... }
  scripts/seed.py           # seeds enabled modules in dependency order
  tests/
    core/ members/ finance/ catalog/ payroll_shifts/ fnb_inventory/ fnb_outlets/ boundaries/
  .importlinter             # enforces the dependency direction in §2.1
  tools/import_contracts.py # custom contract: a module's imports must match its manifest's depends_on
```

Each module's `manifest.py` exposes one object:

```python
MODULE = Module(
    name="fnb_inventory",
    depends_on=("core", "members", "finance", "catalog"),
    routers=[transfers.router, purchasing.router, stock.router, costing.router, waste.router],
    member_routers=[],
    ensure_schema=schema.ensure,          # idempotent DDL, run in dependency order
    seed_demo=seed.seed,                  # used only by scripts/seed.py
    roles={"fnb_manager": ..., "fnb_store_staff": ...},
    gl_accounts=[("1210", "Inventory – F&B Store", "asset"), ...],
    journal_sources=[fnb_stock_journals, fnb_cogs_journals],
    revenue_sources=[],
    grn_line_handlers={"ingredient": receive_ingredient},
    approval_policies=[...],
    approval_actions=[...],
    doc_numbering={"kitchen_requisition": "KR-", "fnb_purchase_request": "FPR-", "ingredient_count": "IC-"},
    background_tasks=[],
    live_topics=[],
)
```

`main.py` reads `IMS_MODULES` (default: all modules present in the image), checks every `depends_on` is enabled, then registers routers, schema, roles and extension points in dependency order. A missing dependency is a startup error, not a runtime 500.

### 3.2 Frontend (`ims/frontend`)

```
frontend/app/
  core/       api-client.ts, live-updates.ts, session-storage.ts, roles registry,
              shell (shell.tsx ManagementApp, role-app, shell-parts, console-parts, screen-parts, ui,
              motion, chart-utils, login-panel, offline), approvals, staff
  modules/
    members/       manifest.ts + screens (member-records, member-account, member-finance,
                   month-end, billing-delivery, receipts, ...)
    finance/       manifest.ts + screens (general-ledger, finance-ledgers, finance-ar,
                   fixed-assets, tax, finance screens split out of modules.tsx)
    catalog/       manifest.ts + screens (CatalogManager: Cafe/Rooftop/Bar menu admin)
    payroll_shifts/ manifest.ts + screens (rosters, attendance, pay rates, payroll runs,
                   labour cost)
    fnb_inventory/ manifest.ts + screens (fnb-store, kitchen-waste, costing screens)
    fnb_outlets/   manifest.ts + screens (OrderBuilder, KitchenQueue, CafeTables,
                   Reservations, Cafe/on-course, OrderHistory, pos-tools, consumption)
  modules.generated.ts   # imports the manifests of enabled modules
  registry.ts            # composition root: core + enabled manifests -> registry (core never imports modules)
  staff/ admin/          # route entries: <RoleApp> with the registry
```

A frontend `manifest.ts` declares: screen ids → `{ meta (title/eyebrow/icon), component (lazy) }`, sidebar group entries, per-role navigation contributions, dashboard tiles/nav-count fetchers, and live topics. The shell builds `SCREEN_META`, the `switch(moduleId)` and `ROLE_PROFILES.navigation` from the enabled manifests instead of the hand-written lists in `app/modules.tsx` and `app/roles.ts`.

Enabled modules are chosen at build time (`IMS_MODULES` → `modules.generated.ts`, written by a small prebuild script), so code for undeployed modules is not shipped to the browser.

---

## 4. Extension points (what makes later modules "drop-in")

These are the places where RumanzaCMS currently hard-codes knowledge of every domain. Each becomes a registry in `core/registry.py` (backend) or a manifest field (frontend). **This is the heart of the plan**; the rest is mostly moving files.

| # | Today (RumanzaCMS) | In ims |
|---|---|---|
| E1 | `build_journals()` in `api/routes/finance_ar.py:334` derives every journal (pro shop, academy, golf cards, vouchers, no-shows, cafe tax, stock…) in one function | `finance` owns the GL engine and calls every registered **journal source** `(start, end, add)`. F&B registers sales tax, COGS, stock-movement and department-meal sources; members registers receipts/suspense/cheques. Golf adds its own later. |
| E2 | `finance_reports.py` revenue-by-source, daily Z-reports and payment-method screens list golf/academy/pro shop | **Revenue sources** registry: `{key, label, outlet(s), sql/query fn}`. First deployment registers F&B outlets and membership only. Frontend finance screens render tiles from `/finance/revenue/sources`. |
| E3 | `finance_purchasing.py` GRN calls `kitchen_stock.receive_ingredient` *and* `inventory.receive_into_stock` directly | **GRN line handlers** by line kind (`ingredient`, later `product`). Finance owns PO/GRN/bills; F&B registers the `ingredient` handler. Pro shop registers `product` later. |
| E4 | `services/tax.py` loop imports `tee_billing`, `tee_bookings`, `club_controls.send_renewal_alerts`, `approvals.apply_scheduled`, `billing_delivery.run_once` | **Background tasks** registry, each module contributes `(name, interval_env, fn)`. Tax invoice **source types** (`cafe_order`, `on_course_order`, later `booking`) are registered with their pricing/line resolver. |
| E5 | Charging a member account is spread across `credit.apply_member_credit`, `charging.py` (imports `tee_extras`), `members.py` | `members` exposes one **account charge API** (`charge_member(source_type, source_id, lines, …)`, `refund`, `points`). Every module calls it; members never imports them. Host/department charging is split: F&B part ships in `fnb_outlets`, tee part later in golf. |
| E6 | Approval kinds (comps, voids, write-offs, F&B PO finance approval, journals) are known inside `services/approvals.py` and its callers | **Approval** registries: policies `{kind, label, default approver roles, second person, threshold}` (seeded into `approval_policies`) and actions `{action, policy kind, label, apply fn}`. |
| E7 | Role tuples and `ROLE_PROFILES` hard-coded (`services/fnb.py`, `app/roles.ts`) | Roles and permission groups declared per module manifest; shell nav built from enabled manifests. |
| E8 | `member_portal.py` / `member_app.py` mix tee times, academy lessons, F&B orders, statements | Member API split: `members` (me, photo, devices, notifications, statement, receipts, credit transactions, feedback, password), `fnb_outlets` (menu, on-course orders, tables, table reservations, ordering hours, pay-code at till). Tee/academy endpoints move with their modules later. URLs and payloads stay identical, so the app team's **F&B-only build** works against ims now, and the full app works once golf/academy modules are added. A `GET /member/features` endpoint lists the enabled member features so the app can hide what isn't there. |
| E9 | Dashboard (`dashboard.py`, `app/dashboard.tsx`) and nav counts know every domain | Backend: summary **widgets** and **attention lists** per module (`/dashboard/summary`, `/dashboard/attention`). Frontend: per-role dashboard loaders and nav-count fetchers per manifest (a later module's loader for the same role wins). |
| E10 | `doc_numbering` rows, `gl_accounts` seed, live topics, `stock_locations.kind` values | Contributed via manifests and seeded idempotently by core/finance. |
| E11 | FKs `cafe_orders.linked_booking_id` and `on_course_orders.linked_booking_id` → `bookings` | Columns kept (nullable) **without** the FK. The golf module adds the constraint when it is installed, using a `DO` block that checks `pg_constraint` first (PostgreSQL has no `ADD CONSTRAINT IF NOT EXISTS`). |
| E12 | Payroll and labour cost live inside `finance_operations.py`, read hours straight from `staff_shifts`, and are **not posted to the GL** | `payroll_shifts` registers a **journal source** for approved/paid payroll runs and a **cost source** for labour cost by outlet/department. Outlet P&L (`fnb_inventory`) and finance reports read cost sources from the registry instead of querying payroll tables. |

Added while porting members (Phase 2), so members never reads other modules' tables:

| # | Today (RumanzaCMS) | In ims |
|---|---|---|
| E13 | Member detail, the member record (360 view), its timeline and the app's `/member/statement` query bookings, F&B orders, handicaps, golf cards, vouchers, club accounts and marketing directly | **Member sections** `{view, key, load(member_id, show_money)}`: modules add sections to the `statement`, `detail`, `record`, `record_finance` and `history` views. Keys of modules not deployed stay in the payload, empty, so the app's parsing doesn't change. |
| E14 | The member directory joins handicap records, open customer invoices and golf-card credit; the member ledger reads customer invoices | **Member directory fields** and **member ledger sources** contributed by golf and club accounts. |
| E15 | Billing delivery and documents know tax invoices and customer invoices; document headers read finance's tax settings; till e-receipts know every sale table | **Delivery documents** `{doc_type, describe, render}`, one **club details** provider (finance), and **sale receipt sources** by e-receipt `source`. |
| E16 | Push delivery reads `member_devices` from core | A **push device resolver** (members). |

Rule enforced by `import-linter` in CI: `core` imports nothing from `modules`; a module imports only from `core` and the modules it lists in `depends_on`.

---

## 5. Source → target mapping

Line counts are from RumanzaCMS today; they size the porting effort.

### 5.1 Backend

| RumanzaCMS file | Target | Notes |
|---|---|---|
| `db.py` (878) | `core/db.py` | Replace the monolithic `_initialize_database_body` with per-module `ensure_schema`; `schema_state` keyed by module name so each module migrates independently. |
| `auth.py`, `contracts/auth.py`, `contracts/staff.py`, `routes/staff.py`, `system.py`, `configuration.py`, `approvals.py`, `live.py` | `core` | Remove golf-only settings from `configuration`. Staff shifts move out of `routes/staff.py` into `payroll_shifts`. Escalations stay in core, handled by the Manager. |
| `services/operations.py`, `messaging.py`, `notification_delivery.py`, `live_events.py`, `approvals.py`, `pdf.py`, `spreadsheet.py`, `throttle.py`, `cache.py`, `gl.next_number` | `core/services` | `club_now` moves from `services/checkin.py` into `core/services/clock.py` (needed by `fnb_hours`). |
| `routes/members.py`, `member_records.py`, `member_accounts.py`, `member_finance.py`, `member_receipts.py`, `month_end.py`, `billing_delivery.py`, `member_auth.py` + member parts of `member_portal.py`/`member_app.py` | `members` | `members.py` drops `orders` import; `month_end` keeps recurring charges, rebates, interest, letters. |
| `services/member_account.py`, `member_records.py`, `member_receipts.py`, `month_end.py`, `billing_delivery.py`, `credit.py`, `documents.py`, `invoices.py`*, `receipts.py` | `members/services` | *`invoices`/`documents` only the parts used by statements and receipts; customer invoices wait for Club Accounts. |
| `database/member_account_schema.py`, `receipts_schema.py`, `delivery_schema.py`, `member_app_schema.py` (minus `club_events`), `members` + `member_credit_transactions` + `member_notes` + `member_nominees` from `schema.py`, `households`, `member_documents` from `platform_schema.py` | `members/schema.py` | |
| `routes/general_ledger.py`, `finance_ar.py` (GL + ledgers part), `finance.py`, `finance_operations.py`*, `finance_purchasing.py`, `finance_reports.py`, `finance_coupons.py`, `tax.py`, `assets.py` | `finance` | *Ingredient/recipe/waste/costing endpoints in `finance_operations.py` move to `fnb_inventory`. `finance_ar.py` direct-debit & customer-invoice parts wait for Club Accounts. |
| `services/gl.py`, `tax.py`, `stock_valuation.py` (ledger reconciliation only) | `finance/services` | `stock_valuation` stops importing `finance_ar`; it asks the GL engine for balances. |
| `database/ledger_schema.py` (GL, numbering, fixed assets, `stock_locations`), `tax_schema.py`, `charging_schema.py` (`cost_centres`), finance tables from `schema.py` (vendors … discount_coupons) | `finance/schema.py` | `product_stock`, `stock_requisitions`, `stock_counts` (product inventory) stay behind for Pro Shop. Payroll tables go to `payroll_shifts`. |
| `routes/catalog.py` (127), `catalog_products`, `stock_adjustments` (schema.py) | `catalog` | Generic: venue/category are data, not code. F&B menus are just categories `Cafe Menu`, `Rooftop Menu`, `Bar Menu`. |
| Payroll endpoints in `finance_operations.py` (`/finance/pay-rates`, `/finance/payroll-runs*`, labour cost), shift endpoints in `routes/staff.py`, `staff_shifts`, `staff_pay_rates`, `payroll_runs`, `payroll_run_lines` | `payroll_shifts` | Used as a **starting point only**; this is ims's own dedicated module. Additions: rosters per F&B location (Cafe & Bar Kitchen, Rooftop Kitchen, F&B Store), clock-in/out, attendance status, overtime rules, GL posting (E12). Scope is restricted to staff in F&B roles by a configurable role list, so widening it later is configuration only. |
| Bank side of `member_receipts.py` (bank matching, statement-line match, gateway settlements, payment-channel GL/bank mapping) and `gateway_settlements` | `finance` | Members keeps taking payments, PDCs, collections, verify, deposit, cancel, dishonour and the gateway checkout; finance adds the bank-side endpoints under the same URLs and the foreign keys to `bank_accounts` / `bank_statement_lines` / `gateway_settlements`. |
| Renewals in `club_controls.py` (`/membership/expiring*`) | `members` | The reminder lead time moves from the golf policy to `membership_settings` (`GET/PUT /membership/renewal-settings`). |
| `routes/fnb_transfers.py`, `fnb_purchasing.py`, `fnb_inventory.py`, `kitchen.py`, ingredient/recipe/costing parts of `finance_operations.py`, F&B parts of `club_controls.py` (settings, cost targets) | `fnb_inventory` | `fnb_purchasing` calls finance's PO/GRN service, not a route module function. |
| `services/fnb.py`, `kitchen_stock.py`, F&B parts of `club_controls.py` (`FNB_CLASS_SQL`, costing) | `fnb_inventory/services` | |
| `database/fnb_inventory_schema.py`, `ingredients`, `ingredient_purchases`, `recipes`, `recipe_ingredients`, `waste_log` (schema.py), `ingredient_movements` (platform_schema.py), `fnb_cost_targets` (controls_schema.py), `stock_batches` (if used by ingredients) | `fnb_inventory/schema.py` | |
| `routes/cafe.py` (775), `pos_tools.py`, F&B parts of `charging.py`, `digital.py` (`_cafe_lines`, `_cafe_tender`, till pay-code), `fnb_hours` parts of `club_controls.py` | `fnb_outlets` | `cafe.py` is over the 400-line guideline — split into `orders.py`, `payments.py`, `tables.py`, `reservations.py`, `on_course.py`. |
| `services/orders.py`, `pos.py`, `fnb_hours.py` | `fnb_outlets/services` | |
| `cafe_orders`, `order_lines`, `order_payments`, `cafe_tables`, `cafe_reservations`, `on_course_orders` (from schema.py), `pos_schema.py`, `fnb_ordering_hours` | `fnb_outlets/schema.py` | `e_receipts` moved to `members` (every sale module issues them; all depend on members). | `order_lines.product_id` references `catalog_products` in the `catalog` module (allowed: `fnb_outlets` depends on `catalog`). |
| `database/demo_seed.py` (1119) | `seed.py` per module | Keep `_seed_staff`, `_seed_members`, F&B parts of `_seed_configuration`/`_seed_operations`/`_seed_report_samples`, `_seed_fnb_store`. |
| `tests/test_api.py` (5318, 107 tests) | `tests/<module>/` | Port only tests whose endpoints exist in ims; keep names for traceability. |

### 5.2 Frontend

| RumanzaCMS file | Target |
|---|---|
| `page.tsx` (shell, nav, notifications), `staff/`, `admin/`, `login-panel.tsx`, `api-client.ts`, `live-updates.ts`, `session-storage.ts`, `offline.tsx`, `shell-parts.tsx`, `console-parts.tsx`, `screen-parts.tsx`, `motion.tsx`, `chart-utils.ts`, `qr.tsx`, `globals.css`, `redesign.css`, `approvals.tsx`, `roles.ts` (as registry) | `core` |
| `member-records.tsx`, `member-account.tsx`, `member-finance.tsx`, `month-end.tsx`, `billing-delivery.tsx`, `receipts.tsx`, `member-code.tsx`; `Members*`, `MembershipFees`, `RevenueDues`, renewals from `modules.tsx` | `members` |
| `general-ledger.tsx`, `finance-ledgers.tsx`, `finance-ar.tsx` (GL parts), `fixed-assets.tsx`, `tax.tsx`; every `Finance*` component from `modules.tsx:5418-9680` except payroll and pro shop | `finance` |
| `fnb-store.tsx`, `kitchen-waste.tsx`, `FinanceKitchen`, `FinanceRecipes`, `FinanceIngredients`, `FinanceMenuEngineering`, `FinanceOutletPnl`, `FinanceWaste`, `FnbCostingScreen` | `fnb_inventory` |
| `OrderBuilder`, `KitchenQueue`, `CafeTables`, `Reservations`, `Cafe`, `OrderHistory`, `pos-tools.tsx`, `consumption.tsx`, `FnbHoursScreen` | `fnb_outlets` |
| `CatalogManager` (`modules.tsx:882`) | `catalog` |
| `FinancePayRates`, `FinancePayrollRuns`, `FinanceLaborCost` (`modules.tsx:9100-9213`), shift parts of `Staff` | `payroll_shifts` (plus new roster and attendance screens) |
| `dashboard.tsx` `RoleDashboard` | core shell + widgets contributed by modules |

`api-client.ts` (`rumanzaApi`, 600+ lines of methods) is split the same way: `core` keeps `apiRequest`/`apiGet`/`apiSend`/offline queue; each module owns its typed calls.

---

## 6. Database

- **Fresh database.** `initialize_database()` runs each enabled module's `ensure_schema` in dependency order and stores one fingerprint per module in `schema_state(module, fingerprint, applied_at)`. Adding a module later only runs that module's DDL.
- DDL stays in the existing idempotent style (`CREATE TABLE IF NOT EXISTS`, guarded `ALTER`) so code ported from RumanzaCMS later fits without conversion.
- Cross-module FKs are owned by the **dependent** module (E11).
- Chart of accounts: finance seeds the base chart (cash, bank, AR 1150, AP 2100/2150, tax 2200/2210, suspense 2400, equity, generic expense); modules add theirs (`fnb_inventory`: 1210, 1215, 1220, 1230, 5210, 5220, 5290, 6150, 6160; `fnb_outlets`: F&B revenue accounts; `payroll_shifts`: salaries & wages expense and payroll payable). Codes are kept as in RumanzaCMS; only payroll needs new accounts because RumanzaCMS never posted payroll. Golf revenue (4100) etc. arrive with golf.
- Demo data: `python -m scripts.seed --date … --members N` seeds enabled modules only, including a week of F&B rosters and one draft payroll run. Demo accounts for the six roles (`manager@`, `kitchen.cafe@` / `kitchen.rooftop@` / `kitchen.bar@` (one Kitchen Staff login per venue), `fnbmanager@`, `fnbstore@`, `finance@`, `membership@`, password `demo123`). `RUMANZA_SEED_DEMO_DATA` stays test-only.
- Production first start: empty DB, one-time Manager setup with `RUMANZA_SETUP_TOKEN` (unchanged flow).

---

## 7. Work plan

Each phase ends with the backend test suite green and the frontend building.

**Phase 0 — Skeleton (ims repo)**
1. The owner creates the ims GitHub repo and makes **all commits**; work is left as uncommitted changes for review. Add backend/frontend tooling copied from RumanzaCMS (`requirements.txt`, `package.json`, Vite/Vinext/wrangler config, Dockerfiles, compose, CI workflow).
2. Write `core/registry.py` (Module + registries E1–E10) and `main.py` module loader with `IMS_MODULES` and dependency checks.
3. Add `import-linter` contract and a "boots with only `core`" test.

**Phase 1 — Core**
1. Port `db.py` (per-module schema state), auth, staff, audit, notifications, approvals engine (with kinds registry), live events, configuration, clock.
2. Frontend core: shell with manifest-driven nav/screens, login, api client, session, shared UI parts, staff screen.
3. Tests: auth, staff, roles registry, approvals engine, module loader.

**Phase 2 — Members**
1. Port member tables, account charge API (E5), receipts, statements, month-end, billing delivery, loyalty, member auth.
2. Split member app endpoints (E8): members part.
3. Frontend member screens; Membership Staff nav.
4. Tests: member CRUD, charges/refunds, receipts + allocation, month-end preview/run/reversal, member token isolation.

**Phase 3 — Finance**
1. Port GL engine with journal-source registry (E1); move members' receipt journals into a members source.
2. Port purchasing/payables with GRN line handlers (E3), tax with source-type registry and background-task registry (E4), revenue sources (E2), reports, fixed assets, bank/petty cash/register close/budgets/period close.
3. Frontend finance screens split out of `modules.tsx`.
4. Tests: balanced journals, trial balance, PO→GRN→bill→payment, tax invoice numbering/credit notes, period close lock.

*Phase 3 notes (2026-10-10).*
- Revenue reports and their screens read only registered sources. The per-source and per-outlet screens from RumanzaCMS (golf, F&B, pro shop and so on) became one `finance-revenue-source` screen and one `finance-daily-outlet` screen, with labels from the API.
- The bank side of member receipts lives in finance: `receipts_bank.py` routes and the "Bank Matching & Online" screen. Verifying a suspense receipt against a statement line is `POST /member-receipts/{id}/verify-with-bank-line`.
- Members' `/member-receipts/{receipt_id}` routes take integer ids only, so finance's fixed paths under that prefix are reachable. `tests/boundaries/test_route_shadowing.py` fails if any module's fixed route is caught by an earlier parameter route.
- Deferred to the module that owns them:
  - pro shop margins, stock valuation and returns credit notes;
  - kitchen costing screens (Phase 4);
  - F&B Store purchase requests in the finance approval queue (completed in Phase 4 through module panels);
  - payroll screens (Phase 5b).
- Phase 6 resolves the missing membership revenue source: members now registers billed fees/subscriptions and their reversals, plus receipt tenders. Members-owned account journals also bring those charges into AR/revenue/tax without duplicating F&B or receipt postings.

**Phase 3b — Catalog**
1. Port `catalog_products`, stock adjustments and `/catalog/products*` endpoints into the `catalog` module; port `CatalogManager`.
2. Tests: product CRUD, barcode lookup, category/venue filtering.

Phase 3b only needs Phase 1, so it can run any time after it. It must be done before Phases 4 and 5.

*Phase 3b notes (2026-10-10).*
- `catalog` depends on `core` only. Its table keeps RumanzaCMS's base columns (sku … image_url, plus `purchase_price` and `brand`). Columns that only one kind of product needs (`fnb_class`, `base_uom`, `avg_cost`, `max_level`, `perishable`) are left to the module that needs them, added with guarded `ALTER`s as RumanzaCMS did.
- Access is set by permission groups, not role tuples: `catalog.view`, `catalog.barcode` and `catalog.manage` (Manager). RumanzaCMS's hard-coded "Pro Shop staff manage only Pro Shop products" rule is now a `catalog.manage:<Category>` group. Pro Shop declares `"catalog.manage:Pro Shop": (...)` and also adds its roles to `catalog.view`. Catalog never names a category. A category-limited role also can't move a product into another category (RumanzaCMS allowed this).
- Other modules move stock through `catalog.services.products.adjust_stock(database, product_id, delta, reason, actor_id)`, which logs a `stock_adjustments` row. They don't update `catalog_products.stock` directly.
- Frontend: `CatalogManager` (`modules/catalog/screens/catalog-manager.tsx`) takes `category`/`title`/`venue`, plus `brands` in place of the old `isProShop` switch. Phase 5's menu screens (`order-builder`, `rooftop-order`, `bar-order` for the Manager) mount it with a fixed category. Catalog's own `catalog` screen ("Products & Menus", Manager) covers every category. Sidebar slot `SIDEBAR_ORDER.catalog`.
- Catalog seeds no demo data. The Cafe/Rooftop/Bar menu items from RumanzaCMS's demo seed belong to the first F&B module that needs them: `fnb_inventory` (recipes) in Phase 4.
- Kitchen staff and the F&B Manager need `catalog.view`/`catalog.barcode` from their own modules' manifests (`kitchen_staff` is already listed as in RumanzaCMS; `fnb_manager` is not).

**Phase 4 — F&B inventory**
1. Port store/kitchen transfer workflow, F&B purchasing (via finance services), stock enquiry/valuation/counts, waste, recipes/ingredients/costing, settings toggle; register journal sources, GL accounts, GRN `ingredient` handler, approval kinds, doc numbering.
2. Frontend: F&B Store, Manager queue, Kitchen requests, F&B stock, settings, costing screens.
3. Tests: everything listed in Phase 7 of `RumanzaCMS/docs/FNB_INVENTORY_PLAN.md` (permissions per step, dispatch blocked when short, transit loss, toggle on/off, weighted average cost, 1210–1230 reconcile to valuation).

*Phase 4 notes (2026-10-10).*
- Backend: `fnb_inventory` owns transfers, purchasing, ingredient stock/valuation/counts, kitchen waste, recipes and costing. `consume_sale` / `restore_sale` are available for Phase 5; costing reads registered sales-line sources and labour cost sources, so it does not import outlets or payroll.
- Frontend: the module provides the Store, Manager queue, Kitchen Requests, stock enquiry/valuation/counts/waste, settings, kitchen waste, Food & Beverage costing, recipes, ingredients, menu engineering and outlet P&L screens. It contributes the three F&B roles, their navigation and inventory dashboards. Recipe/ingredient/value-waste edits stay with Finance and the Manager; the F&B Manager has read-only costing screens. Store staff see stock values and waste but do not get the ledger valuation tab.
- Finance's approval queue renders `FrontendModule.panels` contributions through core's `usePanels`. Inventory contributes `Awaiting Finance` purchase cards. A request stays there if the Finance-approval toggle is subsequently disabled.
- Workflow policies `fnb_kitchen_request`, `fnb_purchase_request` and `fnb_purchase_finance` are registered with core. Dedicated workflow endpoints remain the decision entry points; policies restrict approvers and can require a second person. The Finance second-person check compares against the F&B approver. Workflow stages remain required according to the F&B toggle; generic policy amount thresholds do not bypass them. Core's policy editor accepts module-contributed roles.
- Compatible `GET /finance/ingredient-purchases` is read-only history derived from accepted GRN lines, including vendor, price, batch and expiry. There is no direct purchase POST. F&B locations come from `/fnb/locations` (the undeployed product-inventory API is not required).
- Finance invalidates its cached chart after registering module accounts, so enabling F&B after Finance has already been used lets opening stock journals find the new accounts.
- Source UI components were ported from RumanzaCMS into inventory-owned screens with shared core helpers. Sales and labour figures become populated when Phase 5 and Phase 5b contribute their sources.
- Verification: the full backend run passed 90 tests; all 18 tests in the final affected-area run also passed (core approvals, module-account cache regression and F&B inventory). Frontend typecheck, build, all 9 frontend tests and all 3 import-boundary contracts passed. Lint has only the 3 existing image warnings. The browser tool failed to initialize, so visual verification remains for the next browser-capable session.

**Phase 5 — F&B outlets**
1. Split `cafe.py`; port POS tools, catalog (F&B categories), tables, reservations, on-course orders, ordering hours, F&B charging, till pay-code/e-receipts, offline tills.
2. Register tax source types (`cafe_order`, `on_course_order`), revenue sources (Indoor Cafe, Rooftop, Bar, On Course), sales/COGS journal sources, live topics (`orders`, `reservations`).
3. Member app F&B endpoints (same URLs).
4. Frontend: order builders, kitchen queues, tables, reservations, on-course, order history, POS setup, consumption.
5. Tests: order lifecycle, payments/split/comp/void with approvals, ingredient consumption by venue, member charge, department meals, tax invoice per receipt, offline replay idempotency.

*Phase 5 notes (2026-10-10).*
- Backend: `fnb_outlets` owns till orders/payments/adjustments, POS tools/setup, tables/reservations, on-course orders, ordering hours, host/department charging and consumption. The compatible member-app F&B routes are enabled with the module.
- Finance reads registered F&B revenue/payment sources, itemised sales, tax source types (`cafe_order`, `on_course_order`) and journals. Members reads registered outlet sections and receipt loaders. Ingredient consumption and COGS stay in inventory; outlets do not duplicate the COGS posting.
- Kitchen staff get the three tills and kitchens, on-course, reservations and history. Manager menu screens reuse CatalogManager with fixed Cafe/Rooftop/Bar categories, as agreed in Phase 3b. POS setup, ordering hours and table administration are Manager screens. Finance and the F&B Manager get consumption and history. The frontend contributes live topics, kitchen dashboard/nav counts and the existing offline order/payment queue routes.
- Paid till voids, paid-item cancellation, comps, compensation and department meals use core approval actions. Replay guards lock orders/payments and retain the reference queue's `client_ref` and `captured_at`. Member refunds go back to the actual tender payer, including a guest's host. Invoiced reductions create incremental credit notes, with a final void crediting only what remains.
- Shared APIs extended for outlets: catalog stock-adjustment actors may be null for autonomous member-app orders; inventory exposes `reassign_sale` for bill joins. Prepared orders must be settled separately so joins cannot lose their stock-restoration state. Finance serializes tax issuance per source across receipts and its background job.
- Till orders include a minimal checkout identity, so Kitchen Staff do not need access to full member records.
- Codegraph located the reference files in RumanzaCMS; its nested backend checkout was indexed separately. Endpoint coverage is in `API_COVERAGE.md`. No source business files were changed in RumanzaCMS.
- Verification: the final full backend run passed all 112 tests, including 21 outlet tests. The affected member-payment regression passed again after the final checkout response was narrowed. Frontend typecheck, build, Worker artifact validation and all 9 frontend tests passed; all 3 import-boundary contracts passed. Lint has zero errors and 5 image warnings (3 existing, 2 receipt images). Browser visual QA and the app team's F&B-only build contract check remain for release validation. Phase 5b payroll and deployment were not started.

**Phase 5b — Payroll & shifts**
1. Build `payroll_shifts` from the RumanzaCMS payroll/shift code: rosters per F&B location, clock-in/out and attendance, pay rates, payroll runs (draft → approved → paid, with the approvals engine), overtime.
2. Register the payroll journal source and labour-cost source (E12); outlet P&L shows labour cost per outlet.
3. Frontend: roster calendar (F&B Manager), attendance, pay rates and payroll runs (Finance), labour cost report.
4. Tests: only F&B-role staff can be rostered/paid, hours from attendance, payroll journal balances, labour cost appears in outlet P&L.

Phase 5b needs Phases 1, 2 and 3 (`payroll_shifts` depends on `finance`, which depends on `members`), so it can run alongside Phases 4 and 5. Its outlet P&L step (step 2) and the matching test need Phase 4, because outlet P&L lives in `fnb_inventory`.

*Phase 5b notes (2026-10-10).*
- `payroll_shifts` now owns F&B rosters, self clock-in/out, confirmed attendance, hourly/salary rates, payroll snapshots, daily overtime, bonuses/deductions and draft → approved → paid runs. Only active Kitchen, F&B Store and F&B Manager staff are eligible.
- Finance prepares and pays runs; a different Manager approves through core's approvals engine. Pending approvals freeze drafts; approved/paid payroll freezes attendance and rate snapshots. Closed accounting periods block payroll changes and payments.
- Accounts 6250 Salaries & wages, 2220 Payroll payable and 2230 Payroll deductions payable support balanced accrual/payment journals. The registered `fnb_labour` cost source supplies gross labour by attendance day and outlet to inventory's P&L. The module depends only on core and finance and can load without inventory/outlets.
- Frontend screens cover the weekly roster calendar, attendance, pay rates, payroll runs and labour-cost report. Source files were located with CodeGraph in RumanzaCMS. RumanzaCMS business files were unchanged.
- Verification: frontend typecheck, production build, Worker artifact validation and all 9 frontend tests passed. Lint has zero errors and 5 existing image warnings; payroll-specific lint passed. Backend boundary/loader checks passed all 15 tests, and all 3 import-boundary contracts passed.
- The full backend run passed 69 tests before laptop sleep closed the database connection during the purchasing approval test. Per the requested retry scope, only that failed test and the previously failing payroll attendance-hours/rate-snapshot test were rerun: both passed (2 passed, 1 existing TestClient deprecation warning). The interrupted full suite was not completed; other payroll regression cases and browser visual QA remain unverified. Phase 6 deployment has not started.

**Phase 6 — Hardening & release**
1. End-to-end smoke on a fresh DB: setup Manager → create staff → menu + recipes → purchase → GRN → kitchen request → sale → rosters → payroll run → month-end → statements → GL/trial balance ties out.
2. Member app contract check with the app team's **F&B-only build** (same paths and response shapes as RumanzaCMS for F&B/account endpoints).
3. Update `API_COVERAGE.md`, `ARCHITECTURE.md` (module rules), DFDs for the F&B scope.
4. Rehearse manual testing on a Render free-tier Docker API + dedicated PostgreSQL, with the frontend on a Cloudflare Worker (`NEXT_PUBLIC_API_URL` pointing at the ims API), exact `RUMANZA_ALLOWED_ORIGINS`, independent secrets and `RUMANZA_DOCS_ENABLED=false`. Provision the final production hosting after rehearsal acceptance.


*Phase 6 notes (2026-10-10).*
- CodeGraph located the release, member-app, member-account, revenue and journal implementations in IMS and the nested RumanzaCMS backend. RumanzaCMS business files and data remain unchanged.
- Added a fresh-install API smoke covering private Manager setup, all six staff roles, menu/recipes, approved purchasing, GRN/vendor bill, kitchen transfer, member-account sale/receipt, attendance, approved/paid payroll, month-end statement/collection and GL balances. Reissuing tax invoices must leave the ledger unchanged; reversing billing must reverse its AR/revenue/tax.
- The smoke exposed missing members-owned billing journals and revenue reporting. Members now registers fees/subscriptions, interest, credits and their linked reversals without duplicating outlet charges, receipts or tax issuance. Returned-cheque fees post to AR once; the bank expense remains separate from the receipt principal reversal.
- Added a frozen reference contract for 35 member-auth/member API operations and concrete response checks. The app team's actual F&B-only build remains a separate acceptance gate.
- Added `ARCHITECTURE.md`, F&B DFDs, API coverage, a release runbook, read-only live checker, staging configuration validation and PostgreSQL readiness. Render's blueprint supplies independent signing/setup secrets and disables demo/open setup/docs; the explicit Cloudflare config and predeploy guard require the actual HTTPS API build URL.
- Local verification: the 125-test baseline passed; the final 143-test run completed with 127 passed and 17 PostgreSQL connection errors (including teardown). The failed-only retry passed all 17 cached checks. These are passing results across runs, rather than a single uninterrupted final full-suite pass. After PostgreSQL recovered, all three targeted checks passed: fresh-install smoke, affected receipt API workflow and cheque-fee journal/tender regression. The current suite collects 144 tests, all with passing results across the full run and targeted retries. All 17 release configuration/contract/live-check unit tests passed. All three import boundaries and frontend typecheck/build/Worker validation passed; all 11 frontend tests passed. Lint has zero errors and five existing image warnings. Wrangler dry-run passed, and built landing/staff/admin routes returned HTTP 200 HTML.
- Hosting is Render free tier for manual testing and Cloudflare Workers for the frontend, as requested. Repository/account/service IDs, live URLs, database credentials and app-build details remain blank at the owner's request. No repository initialization, commits, pushes or live deployment were performed.
- Phase 6 remains open for the live API/Worker smoke, container startup, browser visual QA and actual mobile-build acceptance. Docker CLI is unavailable, and browser automation failed to initialize because of its Windows sandbox helper. See `docs/FNB_RELEASE.md` for the exact remaining checks.

Phases 4 and 5 both need Phase 3's registries and Phase 3b (`catalog`). Phase 5 depends on Phase 4, because a sale consumes recipe ingredients through `fnb_inventory`. The two can overlap only once Phase 4's ingredient-consumption service exists; the rest of Phase 4 can carry on alongside Phase 5.

---

## 8. Adding a module later (checklist)

Example: porting **Golf / tee sheet** from RumanzaCMS.

1. Create `backend/app/modules/golf/` with `manifest.py` (`depends_on=("core","members","finance")`).
2. Move routes/services/schema from RumanzaCMS; replace direct cross-domain calls with the registries (journal sources, revenue sources, tax source type `booking`, account charges, background tasks, approval kinds).
3. In golf's `ensure_schema`, add the `linked_booking_id` FKs onto `cafe_orders`/`on_course_orders` (E11).
4. Add its member API router(s) (tee times, bookings, guests) under the same URLs as today.
5. Create `frontend/app/modules/golf/manifest.ts`; move screens; contribute role navigation for `frontdesk_staff`, `headstaff`, etc.
6. Add its seed and tests; add `golf` to `IMS_MODULES`; deploy. Existing modules are not edited.

If step 2 ever requires editing `finance`, `members` or an F&B module, that is a missing extension point: add the registry in the owning module instead of a direct import.

---

## 9. Risks

| Risk | Mitigation |
|---|---|
| `build_journals()` and finance reports are the most entangled code; mis-splitting breaks the ledger | Port GL with a golden test: run RumanzaCMS and ims on the same seeded F&B/member activity and compare trial balances account by account. |
| `modules.tsx` (9,701 lines) shares many local helpers (`text`, `money`, `Table`, `Modal`, …) | Move helpers to `core/ui` first, then lift components file by file; keep visual design unchanged. |
| Member app depends on exact URLs and payloads | Contract tests on the F&B/account member endpoints; no URL changes. |
| Two codebases during transition | RumanzaCMS is reference-only for moved modules (§1); bug fixes go to ims first. |
| Offline tills replay into a new backend | Keep the offline queue format and idempotency keys identical. |
