# Service roles — backend plan

Owner: **backend agent.** Touches only `backend/` and backend docs (`API_COVERAGE.md`, `ARCHITECTURE.md`,
`docs/FNB_DFD.md`, `docs/FNB_INVENTORY_FLOW.md`, `docs/PILOT_PLAN.md`). Does not edit `frontend/`.
Build against [SERVICE_API_CONTRACT.md](SERVICE_API_CONTRACT.md). The agreed screens are in
[`prototype/index.html`](../prototype/index.html).

Module rules from ARCHITECTURE.md still hold: each change lives in the module that owns the data; core imports no
business module; new links use registries, `optional_depends_on` or features. `lint-imports` stays green.

Ship the phases in order. **B1–B3 unblock the frontend's sign-in and role shells**, so land those first.
Every phase ends with its tests passing in both profiles (`full` and `pilot`).

## B1. Roles and permission groups

**New roles** (`Role(...)` in the owning manifest; ids final):

| Id | Module | `group` | Scope |
|---|---|---|---|
| `waiter` | fnb_outlets | staff | `scope_values = OUTLETS` (venue) |
| `cashier` | fnb_outlets | staff | venue |
| `cook` | fnb_outlets | staff | venue; station in `staff_stations` (B3) |
| `inventory_lead` | fnb_inventory | staff | venue (its kitchen location via `fnb_venue_locations`) |

- Extend `core/registry.py` `Role` with `sign_in: Literal["password", "pin"] = "password"`. The four new roles are
  `pin`. Nothing else changes for existing roles.
- **Permission groups** (fnb_outlets manifest): split today's `outlets.serve` into
  - `outlets.order` = `waiter`, `cashier`, `manager`, `headstaff`, `kitchen_staff`
  - `outlets.pay` = `cashier`, `manager`, `headstaff`, `kitchen_staff`
  - `outlets.kitchen` = `cook`, `manager`, `headstaff`, `kitchen_staff`
  - keep `outlets.serve` as the union, for routes that every service role uses (menu, routing, tables).
  Re-point each route in `routes/orders.py`, `payments.py`, `pos_tools.py`, `adjustments.py`, `charging.py`,
  `tables.py`, `kot_tickets.py` to the narrowest group. Keeping `kitchen_staff` in each group means the full profile
  behaves exactly as today.
- fnb_inventory: add `inventory_lead` wherever `kitchen_staff` appears in `KITCHEN` and `FNB_VIEW`; new group
  `fnb.receive` = `inventory_lead`, `manager`, `fnb_manager`. `POST /fnb/kitchen-requests/{id}/receive` moves from
  `fnb.approve` to `fnb.receive`, with a location check through the existing venue-scope helper.
- payroll_shifts: the new roles clock in and out like `kitchen_staff`.
- **`roles.consolidated` feature** (core, listed in `IMS_FEATURES`; add it to `profiles/pilot.env`):
  - the role catalogue labels `manager` "F&B Manager" and returns `assignable: false` for `kitchen_staff`,
    `fnb_manager`, `finance_staff` and `headstaff`; `POST/PATCH /staff` refuse them (422);
  - `manager` must be able to do everything `fnb_manager` and `finance_staff` can. Most role tuples already include
    `manager`. Grep every `require_roles(...)` tuple, `permission_groups` entry and `ApprovalPolicy` role list
    (321 `require_roles` calls today, mostly `FINANCE_ROLES`/`FINANCE_VIEW_ROLES`) for ones naming `fnb_manager`
    or `finance_staff` without `manager`, and add `manager`. Add a test that walks the registry and fails if any
    group or policy grants `fnb_manager`/`finance_staff` something `manager` lacks.
  - `scripts/consolidate_roles.py`: lists active accounts in the hidden roles, so the Manager can move them. It
    changes nothing by itself.
- `GET /staff/role-catalog` returns the contract's `RoleInfo` (`sign_in`, `scope_kind`, `assignable`, `label`).

Tests: `tests/core/test_roles_consolidated.py` (catalogue, refusal, the "manager covers everything" walk);
update every test that creates `kitchen_staff`/`fnb_manager`/`finance_staff` in the pilot profile.

## B2. Sessions, shared devices and PINs — core

**Schema** (`core/schema.py`, idempotent style):
- `staff_users`: `email` and `password_hash` become nullable (`ALTER COLUMN ... DROP NOT NULL`; `UNIQUE` still
  holds for non-null emails); add `sign_in_pin_hash TEXT`, `pin_must_change INTEGER NOT NULL DEFAULT 0`,
  `last_sign_in_at TEXT`.
- `staff_devices(id, name, venue, key_hash UNIQUE, created_by, created_at, last_seen_at, revoked_at, revoked_by,
  failed_pins INTEGER NOT NULL DEFAULT 0, locked_at)`.
- `staff_sessions(id TEXT PRIMARY KEY, staff_id, device_id NULL, created_at, expires_at, ended_at, kind)`; `kind` is
  `password` | `walkup` | `pin` | `pin_setup`.

**Tokens** (`core/auth.py`):
- Add `sid` (session id) and optional `dev` (device id) claims; `exp` from the session (12 h, or 14 days for a
  remembered password session). `TOKEN_LIFETIME_SECONDS` becomes the 12-hour default.
- `current_user` already re-checks the account every 30 s (cached). Extend that check to the session (not ended,
  not expired) and, when `dev` is set, the device (not revoked) → the contract's 401 with `code: "device_revoked"`.
  Keep the 30-second cache; revocation takes effect within it. `forget_staff` also clears session cache entries.
- A `pin_setup` session can only call `PUT /auth/my-pin` (dependency in `current_user`).
- Errors with a `code`: raise `HTTPException(detail={"detail": ..., "code": ...})` through one helper and make the
  app's exception handler flatten it to `{"detail": "...", "code": "..."}`, so other responses keep their shape.

**Routes** (`core/routes/system.py` for auth, new `core/routes/devices.py`):
- `POST /auth/login`: add `remember`; PIN-role accounts → 403; with `X-Device-Key` → `walkup` session bound to that
  device (the device must be valid). Throttling unchanged. Record `last_sign_in_at`.
- `POST /auth/logout`, `PUT /auth/my-password`, `PUT /auth/my-pin`.
- Devices: `POST/GET /auth/devices`, `/revoke`, `/unlock` (new permission group `staff.devices` = `manager`).
  Keys: `secrets.token_urlsafe(32)`, store a SHA-256 digest, compare with `hmac.compare_digest`, the same pattern as
  the print agent keys (`modules/fnb_outlets/routes/kot_agent.py`). Revoking ends the device's sessions.
- `GET /auth/device` and `POST /auth/pin` (dependency reading `X-Device-Key`; updates `last_seen_at`).
  - The staff list comes from a registry hook so core stays generic: a role is listable on a device when
    `Role.sign_in == "pin"` and the account's `scope` equals the device's `venue`.
  - Lockout: increment `failed_pins` on a wrong PIN; at 5 set `locked_at` and return 423; a correct PIN resets it.
    Also throttle per IP (30 per 15 minutes) with the existing `throttle` helper.
  - PIN hashing: the same password hasher as `hash_password` (slow hash; 4 digits are guessable offline, so never
    log or return hashes).
- `POST/PATCH /staff`: the contract's `StaffCreate`/`StaffUpdate` rules (PIN roles: no email/password, 4-digit PIN,
  `pin_must_change = 1`; password roles: email + password; `cook` station via the outlets hook in B3).
- Live topic `devices` (declare it in the core manifest) and `approvals` (announce from `core/services/approvals.py`
  on create and on every decision).

Tests: `tests/core/test_devices_and_pins.py`: device set-up, wrong PINs and lockout, unlock, revoke ending sessions,
walk-up sessions, PIN change gating, password sign-in refused for PIN roles, remember-me expiry, logout.

## B3. Floor: tables, seats, holds, bills — fnb_outlets

**Schema:**
- `cafe_tables`: add `shape TEXT NOT NULL DEFAULT 'square'`, `x REAL`, `y REAL`, `width REAL`, `zone TEXT`.
- `cafe_floor_features(venue PRIMARY KEY, features_json TEXT NOT NULL DEFAULT '[]')`.
- `order_lines`: add `seat_no INTEGER`, `held INTEGER NOT NULL DEFAULT 0`.
- `kot_ticket_lines`: add `done_at TEXT` (a line crossed off on the kitchen display).
- `cafe_orders`: add `bill_printed_at TEXT`, `bill_printed_by INTEGER`. The waiter is the existing `created_by`.
- `staff_stations(staff_id PRIMARY KEY REFERENCES staff_users(id), station_id INTEGER REFERENCES kitchen_stations(id))`:
  a cook's station (no row = the pass). Fill it through a core registry hook (`staff_extras`, generic: a module
  can validate and store extra fields for a role on staff create/update and add them to `StaffRow`), so core never
  imports outlets.

**Endpoints** (contract §3):
- `GET /cafe/floor`, `PUT /cafe/floor/{venue}` (`outlets.setup`). Seed the demo layout from the prototype's
  `TABLES`/`DECOR` (Rooftop R-01–R-10, Indoor Cafe I-01–I-08, Bar B-01–B-06).
- `POST /cafe/orders`: `items` may be empty (seats the table; status `Open`); accept `pax`, `guest_type`, `member_ref`,
  `member_name`, `cost_centre_code`.
- `fire_kots`: skip `held` lines. `POST /cafe/orders/{id}/fire` releases held lines → `additional` tickets.
  `PATCH /cafe/orders/{id}/lines/{line_id}` only for lines not on a ticket.
- `POST/DELETE /cafe/orders/{id}/bill` and the 409 on adding items to a printed bill. Announce `till`.
- **Menu for the order screen:** `catalog_products.description` (catalog); modifier groups in outlets:
  `menu_modifier_groups(id, name, choice, required, sort_order)`, `menu_modifiers.group_id`, and
  `menu_modifier_group_links(group_id, product_id NULL, venue, section NULL)` (a group applies to a product, or to a
  venue's section). Existing flat modifiers get a group "Extras" (`many`, not required) so nothing breaks.
  `GET /cafe/menu` joins catalog, stations and availability; validate required groups on order lines (422).
- `GET /cafe/availability`: portions left per product from the venue location's stock and the recipe. It goes
  through a public fnb_inventory service (outlets already depends on inventory); off-menu products are absent.
- Order responses carry the contract's `Order`/`OrderSummary` fields (`waiter_name`, `seat_no`, `held`,
  `ticket_status`, `void_pending`, `bill_printed_at`, `payments`, `balance_due`, `discount_request`).
- `PATCH /cafe/orders/{id}/details` accepts `waiter_id` (manager only).

## B4. Kitchen: station-scoped cooks, recall, pickups, cooked voids — fnb_outlets

- `GET /kot/tickets`: for `cook`, force the venue from `scope` and the station from `staff_stations`; add the
  `status` filter used by Pickups.
- `change_ticket_status` (`services/kot.py`): allow `Ready→Preparing` (recall) and `Ready→Served`. A station cook can
  only change their station's tickets (403). `sync_order_status`: order Served when every live ticket is Served;
  keep the narrow exception style from the KOT work.
- Owner-approved pickup Undo (2026-10-13): `Served→Ready` before payment, with the same venue/station checks;
  recompute order readiness without repeating stock consumption. Cancelled orders or orders with any recorded
  payment refuse Undo. Held lines prevent automatic order completion until fired.
- Owner-approved served-ticket kitchen recall (2026-10-13): `Served→Preparing` for `outlets.kitchen` only,
  with existing venue/station checks, cancellation wording and any-recorded-payment refusal (including reversal).
  Lock order, then ticket, before payment check; reuse `Ready→Preparing` timestamps and Undo order recomputation.
  Preserve `done`; no inventory consumption/restoration, print job, slip or alert; retain status live events/audit.
  Full/pilot recall tests and pre-B7 source answers are in the B4 report addendum. Wait for owner go-ahead before B7.
- `PATCH /kot/tickets/{id}/lines/{order_line_id}` `{done}` → sets/clears `kot_ticket_lines.done_at`; announce `kot`.
- **Cooked voids:** in the line-cancel paths (`remove_order_line`, `cancel_order_line_now`), when the line's ticket
  is Preparing or Ready, gate through a new `ApprovalPolicy("pos_cooked_void", ...)` (approvers: `manager`) with an
  `ApprovalAction` that applies the cancel; emit `void_pending` now, `void` on apply, `void_rejected` through the
  existing `on_closed` hook. Lines carry `void_pending` while it's open.

## B5. Till: cashiers, payments, register shifts — fnb_outlets (+ finance service)

- `routes/payments.py`: `outlets.pay`; tenders limited to `cash`/`card`/`bank_transfer` for till orders in the
  pilot (`member_account` stays behind the members extras); add `reference` (required for `bank_transfer`) and
  `tendered` (`>= amount`) columns on `order_payments`; refuse more than `balance_due`.
  `DELETE /cafe/orders/{id}/payments/{payment_id}` while unpaid and inside the current register shift.
- `GET /cafe/bills` (contract §5).
- `charge-department` → `outlets.pay`.
- **Register shifts:** finance owns `cash_register_closes`; outlets calls a new public finance service
  `record_register_close(database, outlet, opened_at, closed_at, opening_float, cash, card, bank, counted, counts, coins, note, actor)`.
  - `cash_register_closes`: add `opened_at`, `closed_at`, `bank_sales`, `counts_json`, `coins`, `note`
    (`business_date` stays, taken from `closed_at`).
  - outlets keeps `register_shifts(venue PRIMARY KEY, opened_at, opening_float)` and resets it on each close.
  - `GET /cafe/registers/current` (no cash figures), `POST /cafe/registers/close` (note required at Rs 100+),
    `GET /cafe/registers/closes`.
  - `pos_settings.register_float` (default 10000) in `/pos/settings`.
- Announce `till` on payments, deletions and closes.
- Owner-authorized F5 additions: `/pos/setup.approval_manager_name` (one active eligible person, else null)
  and `Order.checkout_discount_by_name` for the current applied approval, cleared by own-authority edits/removal.

## B6. Stock: inventory leads, store, purchase threshold — fnb_inventory (+ finance)

- `fnb.receive` (B1) on receive; location checks for `inventory_lead` on requests, counts, waste and enquiry
  (reuse the existing `scope_values` venue logic used for `kitchen_staff`).
- `ingredients.storage_area`, `ingredients.pack_size`, `fnb_settings.count_weekday`; the enquiry list's extra fields;
  `GET /fnb/overview` (contract §6) in one query set per call.
- `GET /fnb/forecast` (contract §6): 14-day average use from net consumption movements; `next_delivery_at` from open
  requests. Requests accept optional `needed_by` as an ISO timestamp including timezone; undated legacy requests remain valid.
- **Batches:** add `remaining REAL`. Legacy NULL balances are bounded by present positive location stock,
  reserving known remaining and assigning the balance to newest receipts first (older receipts take inferred depletion).
  Do not invent expiry for unbatched history. Deplete first-in-first-out on dispatch, consumption and waste,
  preserve receipt expiry through requisition-specific transit into the kitchen, and restore proven cancelled-sale lots.
  `GET /fnb/batches/expiring`.
- **Purchase threshold:** `fnb_settings.second_approval_above` replaces `finance_approval_required` under
  `roles.consolidated`. `decide_purchase_request`: at or above the threshold, the first approval sets
  `Awaiting second approval`; the second uses `require_second_person` (already in core) and raises the PO. With the
  feature off, today's Finance step is unchanged.
- Announce a new `stock` topic (declare in the inventory manifest) on request decisions, dispatch, receipt, GRNs,
  waste and counts.

B6 is complete in source; full/pilot and inventory regression evidence, contract clarifications and
frontend handoff are in `../backend/docs/SERVICE_B6_REPORT.md`. Live integration and release gates remain B9.

Pre-B7 F6 extension (2026-10-13): scoped lead count submission, retained-history discard, waste cost/note,
all-ingredient daily use/references, request value/timeline/dispatch note. Interfaces and unchanged behavior:
shared contract section 6 and `../backend/docs/SERVICE_PRE_B7_ANSWERS.md`.

## B7. F&B Manager aggregates and reports

Status (2026-10-13): B7 implemented with full/pilot focused validation. Owner authorized this phase after
B8 independent work. Untaxed-only received-value settlement is the owner's selected rule; taxed invoices
remain unchanged with 422. Exact interfaces, evidence and limits: `../backend/docs/SERVICE_B7_REPORT.md`
and shared contract section 8. B8 operational histories are now implemented; current B9 evidence is recorded separately.

- `GET /cafe/venues/live` (outlets).
- `GET /finance/daily-summary`: add `hours` (net by hour from payments' revenue rows) and `checkout_discounts`.
- `/finance/discounts` and `/finance/voided-transactions`: include checkout discounts, member discounts and cooked voids.
- `POST /finance/vendor-bills/{id}/settle-received` (finance): payable = received value from the matched GRN;
  insert a `credit_debit_notes` row with `note_type = 'debit'`; audit `finance.bill_settled_received`.
- `GET /finance/purchase-reports/dashboard` and `/transfers` (inventory's `purchase_reports.py`).
- `GET /finance/exceptions`: new kinds per the contract; threshold Rs 100.
- Nothing for "Production events".

## B8. Seed, profiles, docs

Status (2026-10-13 continuation): foundation and operational histories implemented through real owners.
Contradictory prototype totals remain documented evidence rather than fabricated financial rows. Empty
recipe availability and foundation modifier scoping are resolved. Final validation/cleanup status is
recorded in the B9 report; exact pixel/financial reproduction is not claimed.
See `../backend/docs/SERVICE_B8_REPORT.md` and `../backend/docs/SERVICE_B8_SHOWCASE_MAP.md`.

- **Demo data must reproduce the prototype's**, because the frontend checks every screen side by side against it:
  the same venues, tables (codes, seats, shapes, positions, zones and floor features from `TABLES`/`DECOR`), stations
  and section mapping (`STATIONS`), menu items with prices, descriptions and modifier groups (`MENU_RAW`, `MODS`),
  ingredients with units, costs, storage areas, pack sizes, par levels and stock per location (`ING_RAW`,
  `LOC_STOCK`, `LOW`), suppliers (`VENDORS`), and the sample orders, tickets, requests, purchase orders, invoices,
  approvals and register closes from the prototype's `seed()`, `seedOffice()`, `seedCashier()` and `seedReports()`,
  with times relative to "now" as the prototype does. Add a `--showcase` flag to `scripts/seed.py` for it.
- `scripts/seed.py` / module seeds: one account per new role per venue matching the prototype (Hamza, Areeba,
  Danish waiters; Nimra, Fahad, Saad cashiers; Naveed BBQ, Imran Pakistani, Adeel Chinese, Saima pass, Rafay, Tariq,
  Bilal cooks; Faisal, Mehwish, Kamran inventory leads), demo PIN `1357` with `pin_must_change = 0` in the demo seed
  only, the demo floor layout, and all six prototype devices (owner direction), whose new keys print once after commit.
- `profiles/pilot.env` already selects `roles.consolidated` alongside the four existing pilot features;
  preserve it without duplication. Release checks in `tests/release/test_pilot_profile.py`: the old
  roles are not assignable; the Finance purchase step returns 404; a full pilot flow runs with the new roles
  (device set-up → PIN sign-in → order → tickets → pickup → bill → payment → register close; kitchen request →
  approval → dispatch → receipt by the inventory lead; purchase above the threshold needs two managers).
- Extend `test_pilot_upgrades_to_the_full_product_on_the_same_database` in
  `tests/release/test_pilot_profile.py` (there is no `test_pilot_upgrade.py` in this checkout): restore
  the full legacy surface and retain pilot service data. The actual operational foundation upgrade now checks scoped modifier links, all six devices, identities and remaining batches.
- Update `API_COVERAGE.md`, `ARCHITECTURE.md` (Role `sign_in`, sessions/devices in core, `staff_extras` hook,
  `roles.consolidated`), `docs/FNB_DFD.md`, `docs/FNB_INVENTORY_FLOW.md` (inventory lead confirms receipts;
  threshold instead of Finance), `docs/PILOT_PLAN.md` roles list.

## B9. Checks

Status: B9 automated backend complete on frozen source (full339/pilot338; owned test database cleaned).
Automated backend evidence: `../backend/docs/SERVICE_B9_REPORT.md`; endpoint/error mapping:
`../backend/docs/SERVICE_B9_COVERAGE.md`. Browser, physical printer/device and deployment acceptance
remain separate gates. Exact additive frontend fields and remaining decisions:
`../backend/docs/SERVICE_FRONTEND_HANDOFF.md`.

- `lint-imports`; the full suite in both profiles (run module folders as parallel processes, see the preview notes).
- Every contract endpoint has a test, including each listed error message and `code`.
- Security tests: a PIN without a device key is refused; a revoked device's token stops working within the cache
  window; walk-up sessions end on revocation; lockout survives restarts (it is stored, not in memory).

## Estimate

About 20–27 developer days: B1 5, B2 3, B3 3, B4 2, B5 2, B6 3, B7 2, B8–B9 3–5 (most of it the tests that use the
old role names). The roles work in B1 carries the most risk.

## Integration continuation — 14 October 2026

The owner authorized the remaining cross-layer work. Source fixes and synchronized interfaces are in
SERVICE_INTEGRATION_REPORT.md and SERVICE_API_CONTRACT.md. Previous missing-report-field and export
statements are historical; current source implements them. Tests and browser/visual acceptance remain
skipped by owner instruction. This note does not mark the outstanding acceptance gates complete.
