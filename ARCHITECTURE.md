# IMS module architecture

IMS builds an F&B deployment from core plus members, finance, catalog, F&B inventory, F&B outlets and payroll/shifts. RumanzaCMS is a read-only source reference. The owner creates the repository and makes all commits.

## Composition and boundaries

Backend composition belongs in `backend/app/main.py`. The loader discovers manifests, always enables core, orders dependencies, and rejects a missing dependency before serving requests. `IMS_MODULES` selects modules and `IMS_FEATURES` selects module features (unset: everything); a deployment names a profile in `profiles/` (`full`, `pilot`) holding both lists. Each module owns its routes, contracts, tables, schema changes, seed, and registered contributions.

A module may use another **optionally** (`optional_depends_on`): it only orders loading, and the module's `when_enabled[name]` extras (routes, schema steps, registrations; the same fields as a manifest) apply only when that module is on. **Features** (`features[name]`) are extras a deployment can switch off. Code that imports an optional module lives in those extras or in a guarded branch (`active_registry().enabled(...)`), so it is never imported while that module is off. A module's schema fingerprint includes the optional modules and features applied to it, so switching one on later re-runs exactly the affected modules' schema steps (deferred foreign keys are added `NOT VALID`, then validated, with `core.db.ensure_foreign_key`).

| Module | Allowed dependencies | Owns |
|---|---|---|
| core | none | Auth/staff, audit, numbering, approvals, outbox, live events, loader and registries |
| catalog | core | Sellable products and stock adjustments |
| members | core | Member identities/accounts, receipts, billing, app account endpoints |
| finance | core; optional members | GL engine, purchasing/payables, tax, financial reports |
| fnb_inventory | core, finance, catalog | Ingredients, recipes, transfers, waste, inventory costing |
| fnb_outlets | core, finance, catalog, fnb_inventory; optional members | POS, kitchens, reservations, on-course orders, outlet app endpoints |
| payroll_shifts | core, finance | F&B staff rosters/attendance (`staff.shifts`), pay rates, payroll and labour costs (`staff.payroll`) |

Features: finance `finance.purchasing`, `.sales_reports`, `.controls`, `.receivables`, `.expenses`, `.cash`, `.coupons`, `.gl` (ledger, assets, GL export, journal approval), `.tax` (tax master and invoices, the tax-documents task); outlets `outlets.reservations`, `outlets.on_course`; payroll `staff.shifts`, `staff.payroll`. Finance features that share routers only switch screens off.

`backend/.importlinter` and `tools/import_contracts.py` enforce these imports. Consumers use public services and registered sources rather than directly reading another domain's tables. Shared records such as staff, catalog items, member identity and finance-owned purchasing lines retain the explicitly agreed relationships.

Core never imports business modules. Finance never imports outlets, inventory or payroll. Members never imports finance or outlets. Finance and outlets import members only inside their members extras or guarded branches; without members, finance reads amounts owed from registered receivable sources (none) and outlets offer no member accounts, member app or member checkout (a member checked elsewhere is recorded by membership number, `cafe_orders.member_ref`, and linked to the member record by `POST /cafe/member-refs/link` once members is enabled). Dependent modules own cross-module FKs; nullable booking IDs remain unconstrained until Golf installs its constraints.

## Extension points

The registry accepts receivable sources (what customers owe, without finance reading member tables), journal sources (E1), revenue/payment sources (E2), GRN handlers (E3), tax source types/background tasks (E4), approval policies/actions (E6), permissions (E7), dashboard widgets/attention lists (E9), GL accounts/document prefixes/live topics (E10), labour cost sources (E12), member sections/ledger rows (E13–E14), delivery documents, receipt loaders and club details (E15), and push-device resolution (E16). Account charges/refunds use the members API (E5). Member routes contributed by modules retain their original paths (E8).

E6 approval actions may register `on_closed(payload, requester, outcome)` for terminal Rejected, Withdrawn and Failed outcomes, including scheduled failures. Core dispatches this generic callback through the registry and imports no business modules. Outlets uses it to notify kitchens when a pending paid void is not applied; ordinary apply failures that return the request to Pending retain the pending workflow.

Till orders snapshot their kitchen station on each line. Station tickets drive Preparing/Ready while the order still owns Served/Cancelled, ingredients, money and customer notices. Member-app/on-course queues retain their existing lifecycle. KOT setup, tickets, slips and job changes publish the registered `kot` topic; order lifecycle changes also publish `orders`. Queue/list projections batch their database reads.

`backend/print_agent/` is a separate LAN process (kept out of the API image), authenticated with a one-time high-entropy key whose SHA-256 digest is stored in IMS. It claims structured print jobs using row locks and 60-second leases, renders ESC/POS, and sends TCP to centrally configured printers. Connect failures retry without COPY; send failures and expired claims retry with COPY. Backend retries and the agent both back off; unprinted jobs and offline agents remain visible in alerts. Printer configuration is refreshed at claim time so a corrected address can recover queued jobs. Composite-key table registrations happen on module load as well as schema setup, including restarts that skip migration.

Examples: Finance receives goods by calling the registered ingredient handler. Outlets consumes or restores recipe ingredients through inventory. Inventory's P&L reads registered sale lines and payroll cost allocations. Finance calls each journal source once; the members source excludes F&B charges because outlets already posts them.

Membership account journals include subscriptions, fees, interest, rebates, adjustments, write-offs and linked month-end reversals. Debit/credit entries use member AR 1150, membership revenue 4600, tax liability 2200, and generic expense 6000 for write-offs. Receipt journals remain separate. Returned cheques reverse principal and record bank charges as expense 6200; fee recovery is a separate members-owned charge, so AR is not debited twice. Tax invoice issuance documents the already posted member charge; it does not post AR/tax a second time. The membership revenue source reports billed fees and their reversals; receipts contribute collected tenders without turning collections into a second sale.

Frontend `app/registry.ts` combines core with `modules.generated.ts`. The prebuild generator reads shell `IMS_MODULES`; undeployed module code is excluded. Manifests contribute lazy screens, metadata, role navigation, dashboard loaders, live topics and extension panels. Finance's approval workspace reads inventory panels through core's registry context.

## Database and lifecycle

PostgreSQL is mandatory. Startup executes idempotent schema callbacks in dependency order, maintains one fingerprint per module in `schema_state`, then seeds settings/registered accounts/policies. Demo data is test-only; manual demo seeding is explicit. Tests isolate themselves in disposable schemas; the release smoke uses an additional empty schema with no demo rows and performs Manager bootstrap through the public setup API.

One API process activates one registry. Background tasks and live listeners stop on lifespan exit. Database transactions cover each money/stock mutation; approvals and replay guards preserve the original workflow controls.

## Release topology and checks

Render's free Docker service is the manual-testing API; PostgreSQL remains external and dedicated to IMS. Cloudflare hosts the Vinext Worker. `NEXT_PUBLIC_API_URL` is embedded at build time and must match that API's `/api/v1` URL. `render.yaml` creates the API configuration; `frontend/wrangler.jsonc` names the separate test Worker.

`IMS_ENVIRONMENT=staging|production` rejects weak/placeholder secrets, open setup, demo seeding, exposed API docs and invalid CORS origins. `/health` is liveness; `/ready` checks a real database connection. Render probes `/ready`.

Release acceptance lives in `backend/tests/release`: empty-install workflow, reconciliation to inventory/AP/member AR/payroll, reference member schemas, live deployment checks and configuration guards. See [F&B DFD](docs/FNB_DFD.md), [release runbook](docs/FNB_RELEASE.md) and [API coverage](API_COVERAGE.md).

## Frontend composition

`scripts/generate-modules.mjs` reads `IMS_MODULES` and `IMS_FEATURES` at build time and applies the same optional-module extras (`optionalDependsOn`, `whenEnabled`) and features as the API. Screens of switched-off extras are withheld: navigation, sidebar and role entries naming them are dropped, so a manifest keeps one navigation tree. A module lends components to others by name (`slots`, rendered with `<Slot name=…/>`), so finance and the till never import members code; a test fails if anything outside `app/modules/members` imports it, keeping it out of builds without members.

## Frontend styling

Two styling systems share the app. Roles without a service layout keep today's `ManagementApp` and legacy console. Service-role shells render in their own `.svc` root outside `.rz` and `.entry`.

`frontend/app/layout.tsx` loads `legacy.css` first, which imports the four console stylesheets in their existing order inside the `legacy` cascade layer. The unlayered service stylesheet loads last. This gives the `.svc` reset priority over even specific legacy selectors without changing the console selectors or their order:

| File | Owns |
|---|---|
| `globals.css` | Tailwind import and older module classes (`module-card`, `button`, `data-table`, `form-grid`, ...). Do not add new rules here. Global scrollbar selectors exclude `.svc` and its descendants. |
| `redesign.css` | Console tokens (light and dark, on `.rz` and `.entry`) and the legacy sidebar, top bar and dashboard. |
| `console.css` | Older module classes mapped to console tokens inside `.rz`. New legacy module-screen styling goes here, scoped under `.rz`. |
| `entry.css` | Legacy workspace chooser, role picker and sign-in (`.entry`). |
| `app/core/service/service.css` | Service roles: the frozen prototype's styles, scoped under `.svc`, loaded last. No Tailwind utilities or legacy classes in service screens. |

The legacy console uses Instrument Sans and IBM Plex Mono. Service roles use Archivo and Martian Mono through `next/font/google`, both with `axes: ["wdth"]`; the prototype's `font-stretch` values must work. `.svc` restores browser defaults for its descendants before applying the ported CSS, retains SVG presentation attributes, and owns its fonts, colours, background and box sizing. Its default resets reproduce the frozen prototype's quirks-mode table line height and two-line row spacing in the app's standards-mode document. The prototype remains unchanged. `build/windows-fonts.ts` completes vinext 0.0.50's Windows font-cache URL rewrite, using its existing asset middleware and build output. Service screens follow the system theme independently of the legacy `html[data-theme]` preference; kitchen displays stay dark.

For service screens, [prototype/README.md](prototype/README.md) and `prototype/index.html` decide layout, spacing, typography, colours, icons, motion, wording and breakpoints. Port the CSS and markup without redesigning them, following `docs/SERVICE_FRONTEND_PLAN.md` (Visual fidelity). Only the documented accessibility and longer-real-data exceptions are permitted. Use the dev-only `/app/_gallery` and `app/core/service/style-probe.js` to compare every shared component against the prototype.

The build selects its brand through `IMS_BRAND`, else the profile's `IMS_BRAND` (pilot: Rykards, full: Rumanza), generated in `app/brand.generated.ts` from `app/core/brands.json`. Service screens take the brand name from `BRAND.name`.

## Service backend roles (B1, 2026-10-12)

The service rollout follows `docs/SERVICE_BACKEND_PLAN.md` and the shared `docs/SERVICE_API_CONTRACT.md`.
Outlets owns `waiter`, `cashier` and `cook`; inventory owns `inventory_lead`. All four declare `Role.sign_in="pin"`
and venue scope; cook also declares `scope_kind="venue_station"` for the catalogue. Core owns the presentation
and assignment rules. `GET /staff/role-catalog` returns the contract's flat `RoleInfo[]` in both profiles.

Core declares `roles.consolidated` as an explicitly selected feature (`Extras.enabled_by_default=False`). Pilot
selects it; full's empty feature list enables default features and keeps consolidation off. Pilot labels manager
"F&B Manager" and prohibits assigning kitchen_staff, fnb_manager, finance_staff and headstaff. Existing accounts
are retained for migration; `python -m scripts.consolidate_roles` lists active legacy accounts without changing them.

Ordering, payment and kitchen writes use `outlets.order`, `outlets.pay` and `outlets.kitchen`. Common venue screens
use `outlets.serve`. Venue checks still use the role registry's scope values. Inventory leads receive dispatched
requests only at their venue's kitchen via `fnb.receive` (quantities only; costs require `fnb.values`), and enter requests, waste and counts with the existing
location checks. The new roles can be rostered and clock their own attendance. Manager also covers the F&B
manager's self-clock and roster eligibility. Password login rejects PIN roles with the service contract's 403.

B2 PIN provisioning, devices and server sessions and B3 cook station assignment are still pending. The new roles
are authorization groundwork; service sign-in and the complete service workflow are not yet ready for release.


## Service authentication (B2, 2026-10-13)

Core owns `staff_devices` and `staff_sessions`. Device keys are random 32-byte URL-safe tokens; only their
SHA-256 digests are stored. Active device names are unique without case differences. Devices belong to a
venue drawn from enabled PIN-role scope values, rather than a business-module import in core. Core declares
`staff.devices` (manager), `X-Device-Key` in CORS, and live topics `devices` and `approvals`.

Staff sessions have opaque ids, staff/device ownership, expiry, end timestamps and a kind: password, walkup,
pin or pin_setup. Signed tokens carry `sid` and optionally `dev`; expiry comes from the session. Password
sessions last 12 hours, or 14 days when remembered. Walk-up and PIN sessions last 12 hours. Member tokens
keep their existing 8-hour protocol and response shape. Staff tokens issued before B2 have no session id and
require a new sign-in; this is a code migration note, not a migration run on the pilot database.

Account and session checks use the existing 30-second cache window. Every check also compares expiry;
revoked device sessions return the contract's `device_revoked` error. Local logout, revocation and staff
credential changes evict the corresponding cache entries. Other API processes observe those changes within
30 seconds. Administrative password/PIN resets end the staff member's sessions. A personal password change
ends other sessions and retains the authenticated current session; a personal PIN change ends the old
sessions and returns a fresh unrestricted device-bound token.

PIN accounts have nullable email/password credentials and a separate slow-hashed `sign_in_pin_hash`, which
never replaces the approval PIN. Manager-issued/reset PINs require a change on next sign-in. A pin_setup token
can call only `PUT /auth/my-pin`. Five wrong PINs lock the device durably; the device row is locked while checking
and updating attempts. Wrong-PIN errors are raised after committing the failure counter. The IP limiter is
additional protection; persistent device counters survive restarts. No keys, PINs or hashes enter audit details
or public projections. SQLAlchemy hides bound parameters in database error messages as well.

The keyed `Module.staff_context` registry hook supplies role-owned identity details and shared-device roster
order. Inventory projects the venue's kitchen location; outlets supplies service titles (cook is the pass until
B3 adds stations). Staff rows and staff `Me` use the shared service contract. Legacy `scope` is accepted as a
staff request alias for existing full-profile integrations; password-based venue accounts retain reassignment
behavior. Non-null station assignment remains B3 and is refused rather than silently ignored.

The owner approved `StaffUpdate.email` so moving a PIN account to a password role can supply both required
credentials. The contract Changelog records it. B3 floor/order extensions and B4 station enforcement are pending.


## Service floor and staff extensions (B3, 2026-10-13)

`Module.staff_extras` maps a role to a `StaffExtras` provider. Core invokes `save(database, staff, fields,
previous)` after the account write, in the same transaction, including the previous role's provider during
role changes so it can remove its own assignment. Identity projections merge `read(database, staff)` with
`staff_context`. Providers validate and store only their module's fields; core imports no business module.
Outlets owns `staff_stations`; cook assignments require an active station at their venue, and null means the
pass. Changing venue must keep a valid assignment or explicitly select the pass. Station administration
cannot move or deactivate a station until its cooks are moved. Station changes terminate old sessions.

Outlets owns floor layout/features, line seats/holds and bill markers. Layout saves retain table ids and
refuse to remove occupied tables. Demo floors reproduce the prototype's visible layouts and preserve later
setup edits. Held items reserve catalog stock and count toward the bill but neither produce a KOT nor consume
ingredients until released. Fire reuses the existing routing snapshot and additional-ticket transaction.

Catalog owns product descriptions. Outlets owns modifier groups/options/links and validates the same
applicable groups it returns in `/cafe/menu`; old modifiers are migrated to optional many-choice Extras.
Inventory owns recipe availability through a public service: latest recipe, summed demand per ingredient,
minimum whole portions at the kitchen serving the venue, zero for shortages, null when no positive recipe
usage limits the item. Off-menu items are absent. This is an enquiry, preserving existing stock-write rules.

Printing a bill records the actor/time and publishes `orders` and `till` on commit. An internal
`cafe_bill_printers` venue-to-printer mapping enables durable bill jobs through the existing queue/agent;
without it the bill marker still works. This phase adds no shared-contract printer-setup endpoint. The LAN
renderer prints amounts/total only for `type: bill`, preserving price-free KOTs. Repeated print requests do
not enqueue duplicate jobs; any recorded payment prevents reopening. Joining into a printed bill is refused.

Owner-approved B3 subtotal clarification: both profiles return subtotal before the separately reported
member discount, as the prototype displays it. The stored net subtotal, price calculations, tax, service
and accounting are unchanged. Checkout discounts apply after the member discount.

Existing cash/card/member payment and host/department settlement paths reject live held lines until
fired. This prevents an order being paid while some of its dishes can no longer be sent to the kitchen;
no B5 payment-write fields, tender rules or register endpoints are introduced by this guard.

## Service kitchen B4 (2026-10-13)

Outlets owns station confinement, ticket completion and cooked-void policy. Ticket reads force a cook's
assigned venue/station; pass cooks retain their venue-wide view. Status/done/reprint writes re-check scope
from the database, lock the order before the ticket, and announce kot/orders after successful changes.
Kitchen transitions require outlets.kitchen; pickups and owner-approved Undo also permit outlets.order.
Held lines prevent automatic Served completion. Undo refuses cancelled orders or orders with any recorded
payment and recomputes order readiness while existing inventory deduction markers prevent repeated consumption.

The generic approval gate accepts an optional existing database transaction. Outlets uses it to commit
approval requests and pending slips together; core imports no business module. pos_cooked_void defaults
to Manager approval for Preparing/Ready line voids, including DELETE, and whole-bill cancellation cannot
bypass that requirement. Approved voids reuse existing pricing/refunds/inventory adapters; cooked ingredient
usage stays recorded. Closure hooks send continue-cooking slips. Cooked-void outcome slips survive a ticket
being served while its decision is pending; ordinary served-station adjustment behavior is preserved.

## Service till B5 (2026-10-13)

Outlets owns register_shifts and venue-confined cashier endpoints; Finance owns cash_register_closes.
Finance's public record_register_close receives an existing transaction and precomputed takings; it
imports no outlet tables. History uses Finance's public register_closes projection. No new GL accounts
or duplicate sales journals are introduced. Older day-based Finance close routes remain intact.

Payment/reversal/refund operations lock the order before the venue shift. Closing locks only the shift,
computes captured net tender amounts inside its server-time half-open window, records the close, and
starts the next shift in one transaction with audit/events. Collection timestamps follow the shift lock.
Offline captured_at is retained as metadata and cannot backdate collection into a closed shift.
Register initialization starts a new window at first use; pre-B5 history is not silently assigned to it.

Current register responses withhold cash/expected cash for blind counting. Note counts require exact
denominations and non-negative whole numbers; the variance threshold is Rs 100. Cash/card/bank receipts
use existing tax/revenue adapters; department meals keep existing expense ownership. Removed unpaid
payments retain reversed history and replay references so a retry cannot re-collect them.

F5 setup approval_manager_name is an informational policy projection: exactly one active eligible person
or null. checkout_discount_approved_by is an atomic current-discount identity on the order; expanded
Order joins checkout_discount_by_name and clears it for edits/removal without approval. Historical
records without attribution stay null. Neither field changes permission checks or approval authority.

B5 zero settlement: reconciliation leaves a free bill without any captured settlement unpaid until
explicit department settlement. Existing captured zero settlements/refund histories retain paid status.


B6 inventory owns batch quantity/provenance and keeps weighted-average costing/journals. Finance retains PO, GRN and vendor-bill services. Pilot second-manager decisions serialize on the requisition row; ingredient mutations take ingredient locks before location/batch locks. Generic core service_openings projects outlets-owned opening hours to inventory without an import dependency or cross-table read. The inventory manifest owns the stock live topic. See backend/docs/SERVICE_B6_REPORT.md.
