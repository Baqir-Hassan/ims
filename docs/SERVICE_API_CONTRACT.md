# Service roles — API contract (shared)

The backend and frontend agents both build against this file. Work split: [backend](SERVICE_BACKEND_PLAN.md) ·
[frontend](SERVICE_FRONTEND_PLAN.md). The agreed screens are the clickable prototype in
[`prototype/index.html`](../prototype/index.html) (published at https://claude.ai/artifact/WeHAmEamDt29kMrLPVcCrV).
**This file decides data and behaviour; the prototype decides how every screen looks and reads.** The contract
carries every field the prototype shows, so the frontend never has to drop or invent anything on screen. If a
screen needs data that isn't here, add it to this file instead of changing the design.

**Change rule:** neither agent changes a shape here on its own. If something has to change, edit this file in
the same change and note it under "Changelog" at the bottom so the other side picks it up.

All paths are under `/api/v1`. Errors keep the existing form `{"detail": "<message>"}`, with a plain-text message
the screen can show as it is. Where a screen must react to an error (not just show it), the message is listed
here word for word, or a machine code is given as `{"detail": "...", "code": "<code>"}`.

## 0. Decisions this contract implements

**Roles.** The pilot runs six roles. Role ids follow PILOT_PLAN §6.1 (existing ids are kept).

| Role id | Label | Signs in with | Limited to | Screens (prototype) |
|---|---|---|---|---|
| `waiter` (new, outlets) | Waiter | PIN on a venue device | one venue | Floor, Orders, Pickups |
| `cashier` (new, outlets) | Cashier | PIN on a venue device | one venue | Bills, Payments, Close register |
| `cook` (new, outlets) | Cook | PIN on a venue device | one venue + one station, or the pass | Kitchen display |
| `inventory_lead` (new, inventory) | Inventory lead | PIN on a venue device | one venue's kitchen location | Overview, Stock, Requests, Stock count, Waste |
| `fnb_store_staff` (existing) | F&B Store | email + password | the F&B Store | Overview, Dispatch, Receiving, Purchase requests, Store stock |
| `manager` (existing) | **F&B Manager** | email + password | all venues | Overview, Approvals, Sales, Purchasing, Stock, Costing, Reports, Team, Setup |

- `manager` absorbs what `fnb_manager` and `finance_staff` do today. Under the new feature
  **`roles.consolidated`** (on in the pilot profile) the role catalogue labels `manager` "F&B Manager" and hides
  `kitchen_staff`, `fnb_manager`, `finance_staff` and `headstaff`: they can't be assigned, and existing accounts in
  them must be moved to a new role before go-live (the backend ships a one-off script that lists them).
- With the feature off (full profile) nothing changes for the old roles; the four new roles exist in both profiles.

**Sign-in.** The system is reachable from anywhere, so a PIN alone never signs anyone in.
- A **shared device** (till, waiter handheld, kitchen screen, venue tablet) is set up once by the F&B Manager with
  email and password, for one venue. The server returns a device key, shown once and stored by that browser.
- On a set-up device, venue staff (`waiter`, `cashier`, `cook`, `inventory_lead`) of that venue sign in with a
  4-digit PIN. Five wrong PINs in a row lock the device until the F&B Manager unlocks it.
- `manager` and `fnb_store_staff` sign in with email and password on their own computers: a 12-hour session, or
  14 days with "Keep me signed in". They may also sign in by email on a shared device ("walk-up"); that session is
  bound to the device and gets the device's idle sign-out.
- Idle sign-out (5 minutes, kitchen screens exempt) is done by the frontend; the server bounds sessions by expiry and
  device revocation.

**Money and stock rules.**
- Waiters never take payment. They print the bill; the venue's cashier takes payment. A table is free once its order
  is fully paid (today's rule: `payment_status = 'paid'`).
- Cashiers may give a checkout discount up to `pos_settings.max_checkout_discount_percent`; above it the existing
  `pos_checkout_discount` approval applies (unchanged).
- Voiding a dish whose ticket is Preparing or Ready needs approval (new policy `pos_cooked_void`).
- Finance's separate purchase approval is replaced by **"second approver above Rs X"**: a purchase request at or
  above the threshold needs approval from two different F&B Managers.
- The venue's inventory lead confirms what arrives from the store (today only the F&B Manager can).

## Common types

```ts
type Venue = "Indoor Cafe" | "Rooftop" | "Bar";
type Role = "waiter" | "cashier" | "cook" | "inventory_lead" | "fnb_store_staff" | "manager";  // + old roles in the full profile
type SignIn = "pin" | "password";
type Money = number;                                   // rupees, 2 dp
type Iso = string;                                     // ISO 8601 with offset

interface Me {
  id: number; full_name: string; email: string | null; role: Role; role_label: string; sign_in: SignIn;
  venue: Venue | null;                                 // waiter / cashier / cook / inventory_lead
  location_id: number | null;                          // inventory_lead: the venue's kitchen stock location
  station_id: number | null; station_name: string | null;   // cook only; null = the pass (every station of the venue)
  device: DeviceRef | null;                            // set when this session is bound to a shared device
  expires_at: Iso;
}
interface DeviceRef { id: number; name: string; venue: Venue }
interface TokenResponse { access_token: string; token_type: "bearer"; expires_at: Iso; user: Me;
  pin_change_required?: boolean }                      // PIN sign-in only (see §1)
```

## 1. Sign-in, devices and PINs (core)

### Password sign-in

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/auth/login` | `{email, password, remember?: boolean}` | `TokenResponse` |
| POST | `/auth/logout` | — | 204 |
| GET | `/auth/me` | — | `Me` |
| PUT | `/auth/my-password` | `{current_password, new_password}` | 204 |

- Session length: **12 h**, or **14 days** with `remember: true`. `remember` is ignored when the request carries an
  `X-Device-Key` (walk-up sign-in on a shared device): that session lasts 12 h and is bound to the device.
- An account whose role signs in with a PIN → **403** `"Sign in with your PIN on a venue device."`
- Existing throttling stays (5 failures per email / 30 per IP per 15 minutes).
- `/auth/logout` makes the token unusable (server-side session row, see backend B2).

### Shared devices (write: `staff.devices`, which is `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/auth/devices` | `{name, venue}` | `{device: Device, device_key: string}` (201); `device_key` is only ever returned here |
| GET | `/auth/devices` | — | `Device[]` |
| POST | `/auth/devices/{id}/revoke` | — | `Device` |
| POST | `/auth/devices/{id}/unlock` | — | `Device` |

```ts
interface Device { id: number; name: string; venue: Venue; created_by_name: string; created_at: Iso;
  last_seen_at: Iso | null; revoked_at: Iso | null; locked: boolean; failed_pins: number }
```

- `name` 2–60 characters, unique among devices that aren't revoked. Revoking ends every session bound to the device.
- The frontend sets a device up by signing in with `/auth/login` (manager), calling `POST /auth/devices`, storing
  the key, then calling `/auth/logout` for the manager's token.

### On a shared device (header `X-Device-Key: <device_key>`, no bearer token)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/auth/device` | — | `{device: Device, staff: PinStaff[]}` |
| POST | `/auth/pin` | `{staff_id, pin}` | `TokenResponse` |

```ts
interface PinStaff { id: number; full_name: string; role: Role; role_label: string; title: string }  // title e.g. "BBQ station"
```

- `staff`: active PIN-role accounts of the device's venue, in role order (waiter, cashier, cook, inventory_lead).
- Unknown or revoked key → **401** `{"detail": "This device was switched off. Ask the F&B Manager to set it up again.", "code": "device_revoked"}`.
  The frontend deletes its stored key on this code.
- Wrong PIN → **401** `"Wrong PIN. 3 tries left."`; the fifth failure in a row locks the device →
  **423** `{"detail": "This device is locked after 5 wrong PINs. Ask the F&B Manager to unlock it.", "code": "device_locked"}`.
  A correct PIN resets the count.
- Staff from another venue, or a password-role account → **403** `"<name> doesn't work at <venue>."`
- PIN sessions last 12 h, are bound to the device, and end when the device is revoked.
- `pin_change_required: true` when the PIN was set or reset by the F&B Manager; that token can only call
  `PUT /auth/my-pin` until the PIN is changed (other calls → **403** `{"code": "pin_change_required"}`).

| Method | Path | Body | Returns |
|---|---|---|---|
| PUT | `/auth/my-pin` | `{current_pin, new_pin}` | `TokenResponse` (a fresh, unrestricted token) |

`new_pin`: exactly 4 digits, not four of the same digit, not `1234`/`4321`, different from `current_pin`.

### Device-bound tokens everywhere

Any request whose token is bound to a revoked device → **401** `{"code": "device_revoked"}`; a token past its
expiry or after logout → **401** `"Invalid or expired access token"` (unchanged). The frontend returns to the right
sign-in screen on both.

## 2. Staff and roles (core; write: `staff`, which is `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/staff/role-catalog` | — | `RoleInfo[]` |
| GET | `/staff` | — | `StaffRow[]` |
| POST | `/staff` | `StaffCreate` | `StaffRow` (201) |
| PATCH | `/staff/{id}` | `StaffUpdate` | `StaffRow` |

```ts
interface RoleInfo { id: Role; label: string; description: string; sign_in: SignIn;
  scope_kind: "venue" | "venue_station" | "none"; assignable: boolean }
interface StaffRow { id: number; full_name: string; email: string | null; role: Role; role_label: string;
  sign_in: SignIn; venue: Venue | null; station_id: number | null; station_name: string | null;
  active: boolean; pin_set: boolean; last_sign_in_at: Iso | null }
interface StaffCreate { full_name: string; role: Role; venue?: Venue; station_id?: number | null;
  email?: string; password?: string;     // password roles: both required (password 10+ characters); the F&B Manager
                                         // gives the person this starting password, they change it with /auth/my-password
  pin?: string }                         // PIN roles: required, 4 digits; the person must change it at first sign-in
interface StaffUpdate { full_name?: string; role?: Role; venue?: Venue; station_id?: number | null;
  active?: boolean; pin?: string /* resets the PIN, must change at next sign-in */; password?: string;
  email?: string }                       // switching a PIN account to a password role requires email + password
```

- PIN roles have no email or password. Password roles have no PIN for signing in (their approval PIN is unchanged).
- `cook`: `station_id` must be a station of `venue`; `null` means the pass.
- `assignable: false` for roles hidden by `roles.consolidated`; creating or moving someone into them → **422**.

## 3. Floor: waiters (`outlets.order` = `waiter`, `cashier`, `manager`)

Permission groups list the pilot roles; in the full profile the old roles keep their access (backend B1).

### Tables and floor plan

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/floor?venue=Rooftop` | — | `Floor` |
| PUT | `/cafe/floor/{venue}` | `{tables: TableLayout[], features: FloorFeature[]}` (`outlets.setup`) | `Floor` |

```ts
interface TableLayout { code: string; shape: "round" | "square" | "long"; x: number; y: number; width: number;
  seats: number; zone: string | null }                // x, y, width in % of the plan; centre point
interface FloorFeature { kind: "zone" | "line" | "block"; label: string; x: number; y: number; w: number; h: number }
interface Floor { venue: Venue; tables: FloorTable[]; features: FloorFeature[] }
interface FloorTable extends TableLayout { id: number; state: "free" | "seated" | "ready" | "bill";
  late: boolean; order: OrderSummary | null }         // late: a live ticket older than 12 minutes
```

`state`: `free` no unpaid order; `bill` bill printed; `ready` any ticket Ready; else `seated`.

### Orders (existing endpoints, extended)

```ts
type GuestType = "Walk-in" | "Member" | "Staff meal";
interface OrderLineCreate {             // existing fields plus:
  product_id: number | null; quantity: number; modifier_ids: number[]; note: string | null;
  open_name?: string; open_price?: number;
  seat_no?: number | null;              // 1..pax; null = the table
  hold?: boolean }                      // true: stored but not sent to the kitchen until fired
interface OrderLine { id: number; product_id: number | null; name: string; quantity: number; unit_price: Money;
  line_total: Money; modifiers: string[]; note: string | null; seat_no: number | null; held: boolean;
  station_id: number | null; station_name: string | null; ticket_id: number | null;
  ticket_status: "Placed" | "Preparing" | "Ready" | "Served" | "Cancelled" | null;
  cancelled: boolean; cancel_reason: string | null; void_pending: boolean }
interface OrderSummary { id: number; order_code: string; venue: Venue; table_code: string | null; pax: number;
  guest_type: GuestType; member_ref: string | null; member_name: string | null;
  waiter_id: number; waiter_name: string; opened_at: Iso; total: Money; balance_due: Money;
  bill_printed_at: Iso | null; discount_pending: boolean }
interface Order extends OrderSummary {
  lines: OrderLine[]; kot: OrderKot[] /* KOT contract */; print_alerts: PrintAlert[];
  subtotal: Money; /* before member_discount; checkout_discount applies after it */ member_discount: Money; checkout_discount: Money; checkout_discount_reason: string | null;
  checkout_discount_by_name: string | null; // approving person for the currently applied checkout discount; otherwise null
  tax_amount: Money; service_charge: Money;
  payments: Payment[]; paid: Money; payment_status: "unpaid" | "partial" | "paid";
  bill_printed_by_name: string | null;
  discount_request: { approval_id: number; discount_type: "percentage" | "amount"; discount_value: number; reason: string } | null }
```

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/cafe/orders` | existing body plus `pax`, `guest_type`, `member_ref`, `member_name`, `cost_centre_code`; `items` may be empty (seats the table) | `Order` (201) |
| POST | `/cafe/orders/{id}/items` | `{items: OrderLineCreate[]}` | `Order` — fires every new line not on hold (`additional`) |
| POST | `/cafe/orders/{id}/fire` | `{line_ids?: number[]}` (omitted: every held line) | `Order` |
| PATCH | `/cafe/orders/{id}/lines/{line_id}` | `{seat_no?, note?, hold?}` — only lines not yet on a ticket | `Order` |
| POST | `/cafe/orders/{id}/bill` | — | `Order` (sets `bill_printed_at`; queues a bill print if the venue has a bill printer) |
| DELETE | `/cafe/orders/{id}/bill` | — | `Order` (reopens; **409** `"Payments are already taken. Ask the cashier."` once any payment exists) |
| GET | `/cafe/orders?venue=&open=true` | — | `OrderSummary[]` (existing list, filtered to unpaid) |
| GET | `/cafe/availability?venue=Rooftop` | — | `{items: Record<string, number \| null>}` portions left per product id; `null` = not limited; products taken off the menu are absent |
| GET | `/cafe/menu?venue=Rooftop` | — | `MenuSection[]` — everything the order screen shows; cached for offline use |

```ts
interface MenuSection { section: string; station_names: string[]; items: MenuItem[] }   // in menu order
interface MenuItem { product_id: number; name: string; description: string | null; price: Money;
  station_id: number | null; station_name: string | null; modifier_groups: ModifierGroup[];
  portions_left: number | null }          // same rule as /cafe/availability
interface ModifierGroup { id: number; name: string;          // "Spice level", "Portion", "Add-ons"
  choice: "one" | "many"; required: boolean; options: { id: number; name: string; price_delta: Money }[] }
```

`OrderLineCreate.modifier_ids` takes option ids. A line missing a required group's choice → **422**
`"Choose <group name> for <item>."` Catalog products gain `description` (up to 120 characters, edited in Setup →
Menu & prices).

- `waiter_id` is the account that opened the order (`cafe_orders.created_by`); `PATCH /cafe/orders/{id}/details`
  gains `waiter_id` so the F&B Manager can hand a table over.
- After `POST /bill`, adding items → **409** `"The bill is printed. Reopen the order to add items."`
- Guests: `guest_type: "Member"` needs `member_ref` and `member_name` (pilot rule, unchanged discount logic);
  `"Staff meal"` needs `cost_centre_code` and goes through the existing department-meal path when the bill is closed.
- **Voids.** The existing line-cancel endpoints keep their shapes. When the line's ticket is Preparing or Ready they
  return **202** with the standard approval payload (policy `pos_cooked_void`), the line shows `void_pending: true`
  and the station gets a `void_pending` slip. Approved → `void` slip; rejected → `void_rejected` slip (KOT contract).

### Pickups

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/kot/tickets?venue=Rooftop&status=Ready` | — | `Ticket[]` (KOT contract; `status` filter is new) |
| PATCH | `/kot/tickets/{id}/status` | `{status: "Served"}`; pickup Undo `{status: "Ready"}` after Served | `Ticket` |

`Ready → Served` is new and allowed for `outlets.order` and `outlets.kitchen`. The order becomes Served when every
live ticket is Served and no uncancelled held line remains (it stays unpaid until the cashier closes it).
Owner-approved pickup Undo permits `Served → Ready` for `outlets.order` and `outlets.kitchen`, with the same
venue/station checks. Cancelled orders refuse Undo; an order with any recorded payment returns **409**
`"Payments are already taken. Ask the cashier."`. Undo recomputes order readiness and does not repeat stock consumption.
The kitchen may also recall a served ticket to Preparing before any payment, under the rules in section 4.

## 4. Kitchen: cooks (`outlets.kitchen` = `cook`, `manager`)

KOT contract endpoints, with these changes:

- `GET /kot/tickets` for a `cook`: the venue is the account's venue; a station cook always gets only their station
  (`station_id` in the query is ignored); a pass cook (`station_id: null`) gets every station and may filter.
- `PATCH /kot/tickets/{id}/status` transitions: `Placed→Preparing`, `Preparing→Ready`, **`Ready→Preparing`** (recall,
  new), **`Ready→Served`** (new), **`Served→Ready`** (pickup Undo before payment), and **`Served→Preparing`** (kitchen recall before payment).
  A station cook may only change their station's tickets → **403**
  `"That ticket belongs to <station>."`
- `Served→Preparing` requires `outlets.kitchen` (`cook`, `manager`, and the group's legacy roles);
  waiters/cashiers get **403**. Venue confinement is unchanged; pass cooks may recall any station in their venue.
  Cancelled tickets/orders return **409** with the existing invalid-transition/Undo wording. Any recorded payment,
  including partial, full or later reversed payments, returns **409** `"Payments are already taken. Ask the cashier."`.
  Lock the order, then the ticket, before checking payments, as for Undo. Return the same updated `Ticket`;
  reuse `Ready→Preparing` timestamp handling (`accepted_at` retained/set, `ready_at` cleared), and clear a served
  timestamp if present (the current ticket schema has none). Keep line `done` flags. Recompute order state using
  the pickup Undo helper: Served only when every live ticket is Served and no uncancelled held line remains.
  Recall never consumes/restores stock or creates a print job, slip or alert; retain status-change live topics
  and audit. Existing `Placed→Preparing` and `Ready→Preparing` remain valid; repeated `Preparing→Preparing` is invalid.
- Crossing off lines is shared between screens:

| Method | Path | Body | Returns |
|---|---|---|---|
| PATCH | `/kot/tickets/{id}/lines/{order_line_id}` | `{done: boolean}` | `Ticket` |

`TicketLine` gains `done: boolean` and `seat_no: number | null`; `Ticket` gains `waiter_name` (already present) and
`late_after_seconds: 720`. "All day" counts are computed by the frontend from the open tickets.

## 5. Till: cashiers (`outlets.pay` = `cashier`, `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/bills?venue=Rooftop` | — | `{waiting: OrderSummary[], dining: OrderSummary[]}` — waiting: bill printed and unpaid, oldest print first; dining: unpaid, bill not printed |
| GET | `/cafe/orders/{id}` | — | `Order` (existing) |
| POST | `/cafe/orders/{id}/payments` | `PaymentCreate` | `Order` |
| DELETE | `/cafe/orders/{id}/payments/{payment_id}` | — | `Order` (only while `payment_status != "paid"`, same shift; otherwise **409** `"Paid bills are refunded through the F&B Manager."`) |
| POST | `/cafe/orders/{id}/bill` | — | `Order` (a cashier may print too) |
| PUT | `/cafe/orders/{id}/discount` | existing `CheckoutDiscount` | `Order`, or **202** approval above the limit |
| POST | `/cafe/orders/{id}/charge-department` | existing | existing (staff meals; now `outlets.pay`) |
| GET | `/cafe/orders/{id}/receipt` | — | existing receipt |

```ts
interface PaymentCreate { tender: "cash" | "card" | "bank_transfer"; amount: Money;
  reference?: string;    // card approval code (optional) or transfer reference (required for bank_transfer)
  tendered?: Money;      // cash handed over; must be >= amount
  split_label?: string; client_ref?: string; captured_at?: Iso }
interface Payment { id: number; tender: "cash" | "card" | "bank_transfer"; amount: Money; reference: string | null;
  tendered: Money | null; change: Money | null; taken_by_name: string; taken_at: Iso }
```

- `amount` above `balance_due` → **422** `"That's more than the balance of Rs <x>."`
- The payment that makes `balance_due` 0 sets `payment_status: "paid"`; the table is free from that moment.
- A zero-value bill without any captured settlement remains unpaid until explicit department/staff-meal
  settlement; a captured zero settlement or refund history retains its existing paid state.
- `discount_request` is set while a checkout discount waits for approval; approval applies it (existing action).

### Register (shift close)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/registers/current?venue=Rooftop` | — | `RegisterShift` |
| POST | `/cafe/registers/close` | `RegisterCloseCreate` | `RegisterClose` (201) |
| GET | `/cafe/registers/closes?venue=&limit=20` | — | `RegisterClose[]` (cashier: own venue; manager: all) |

```ts
interface RegisterShift { venue: Venue; opened_at: Iso; opening_float: Money; payments: number;
  card_total: Money; bank_total: Money; unpaid_printed_bills: number }   // cash and expected cash are withheld (blind count)
interface RegisterCloseCreate { venue: Venue; counts: Record<"5000"|"1000"|"500"|"100"|"50"|"20"|"10", number>;
  coins: Money; note?: string }
interface RegisterClose { id: number; venue: Venue; opened_at: Iso; closed_at: Iso; closed_by_name: string;
  opening_float: Money; cash_total: Money; card_total: Money; bank_total: Money; expected_cash: Money;
  counted_cash: Money; variance: Money; note: string | null; counts: Record<string, number>; coins: Money }
```

- A variance of Rs 100 or more needs `note` → **422** `"Say what happened: the count is Rs <x> <short|over>."`
- Closing starts the next shift at once with the configured float (`pos_settings.register_float`, default 10,000).
  A setting change applies to the next shift; the current shift retains its opening float.

## 6. Venue stock: inventory leads (`inventory_lead` limited to its venue's location)

Existing endpoints (`/fnb/kitchen-requests*`, `/inventory/fnb/*` counts and enquiry, `/fnb/waste*`) keep their shapes.
`/fnb/waste` and `/fnb/waste/options` share the existing `/kitchen/waste*` handlers.
Kitchen requests may include optional `needed_by`, an ISO timestamp including timezone.
Changes:

- `inventory_lead` gets the groups `fnb.kitchen` (request, withdraw, waste) and the new **`fnb.receive`**:
  `POST /fnb/kitchen-requests/{id}/receive` now needs `fnb.receive` (= `inventory_lead` for its own location, `manager`)
  instead of `fnb.approve`. Another venue's request → **403** `"That request is for <location>."`
- Counts (`/inventory/fnb/counts*`) and stock enquiry are limited to the lead's location, as for kitchen staff today.

| Method | Path | Returns |
|---|---|---|
| GET | `/fnb/forecast?location_id=` (new) | `ForecastRow[]` — ingredients that won't last until the next delivery |
| GET | `/fnb/overview?location_id=` (new) | `LocationOverview` — the inventory lead's Overview screen |
| GET | `/inventory/fnb/enquiry?location_id=` (existing list, extended) | rows gain `storage_area`, `pack_size`, `par`, `on_hand`, `value`, `last_movement: {kind, at} \| null` |

```ts
interface LocationOverview { location_id: number; location: string; stock_value: Money; ingredients: number;
  below_par: {ingredient_id: number; name: string}[]; waste_7d: Money; used_7d: Money;
  next_count_at: Iso;                                   // from fnb_settings.count_weekday (default Sunday) at 11:00
  count_in_progress: boolean;
  usage_14d: {date: string; value: Money}[];            // consumption at cost per day, oldest first; today so far last
  used_today: Money;                                    // orders since the venue opened today, at cost
  incoming: {request_no: string; lines: number; dispatched_at: Iso; dispatched_by_name: string}[];
  recent_movements: {at: Iso; ingredient: string; unit: string; kind: "received" | "sales" | "waste" | "count" | "dispatch" | "transit_loss";
    quantity: number; reference: string}[] }           // newest first, 7 rows
interface ForecastRow { ingredient_id: number; name: string; unit: string; on_hand: number; daily_use: number;
  hours_left: number; next_delivery_at: Iso | null }
```

Ingredients gain `storage_area` (`"Walk-in chiller" | "Freezer" | "Veg rack" | "Dry store" | "Bar fridge"`, walking
order for counts) and `pack_size` (the store's pack, used to round requests); `fnb_settings` gains `count_weekday`
(0 = Monday … 6 = Sunday, default 6).

The ingredient drawer (movements, batches, par) uses the existing `GET /inventory/fnb/enquiry/{ingredient_id}`,
which already limits a venue login to its own kitchen.

`daily_use`: average daily consumption at that location over the last 14 days. `next_delivery_at`: the earliest
`needed_by` of an approved or dispatched request for that location, else the next 09:00.
Enquiry retains its existing `{locations, ingredients}` envelope; zero balances remain in the list.

## 7. F&B Store (`fnb_store_staff`)

Existing dispatch, purchase-request and goods-received endpoints keep their shapes. New:

| Method | Path | Returns |
|---|---|---|
| GET | `/fnb/batches/expiring?location_id=&days=2` | `Batch[]` — batches with stock left that expire within `days` |

```ts
interface Batch { id: number; ingredient_id: number; name: string; unit: string; batch_no: string | null;
  expiry_date: string; remaining: number; location: string }
```

## 8. F&B Manager (`manager`)

### Approvals

The Approvals screen merges three existing lists; no new endpoint:
`GET /approvals?status=Pending` (till approvals, refunds), `GET /fnb/kitchen-requests?status=Pending`,
`GET /fnb/purchase-requests?status=Pending`. Deciding uses the existing endpoints.

Expiring-batch warnings include overdue lots and dates through local today + `days`, inclusive;
only positive `remaining` is returned. Transfers preserve receipt expiry in location-specific lots.

### Purchase approval threshold (replaces the Finance step)

- `GET/PUT /fnb/settings` gains `second_approval_above: Money | null` and loses `finance_approval_required`
  (still accepted on PUT in the full profile; ignored in the pilot). The legacy
  `finance_po_approval` name remains supported in the full profile.
- `POST /fnb/purchase-requests/{id}/decision` for a request at or above the threshold: the first approval sets status
  **`Awaiting second approval`**; a second approval by a different manager raises the PO.
  Threshold money is stored to two decimals and compared in whole paisa. The same person →
  **409** `"A different F&B Manager has to give the second approval."`
- `POST /fnb/purchase-requests/{id}/finance-decision` → 404 in the pilot.

### Overview and sales

| Method | Path | Returns |
|---|---|---|
| GET | `/cafe/venues/live` | `VenueLive[]` |
| GET | `/finance/daily-summary?date=` | existing, plus `hours: {hour: number, net: Money}[]` and `checkout_discounts: Money` |
| GET | `/finance/payment-methods/transactions?date=&venue=&tender=` | existing (Transactions tab) |
| GET | `/finance/discounts?date=` | existing (Discounts & comps tab), plus checkout discounts and member discounts |
| GET | `/finance/voided-transactions?date=` | existing, plus voids after cooking |

```ts
interface VenueLive { venue: Venue; net_sales_today: Money; open_tables: number; tables: number;
  late_tickets: number; avg_ticket_seconds: number | null; waiting_bills: number }
```

### Purchasing (existing finance purchasing endpoints, unchanged unless listed)

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/finance/vendor-bills/{id}/settle-received` | — | `VendorBill` — sets the payable amount to the received value and raises a debit note for the difference |

The bill check (ordered vs received vs invoiced) uses the existing `/finance/vendor-bills/matching`.

### Reports (`/finance/purchase-reports/*` existing, `manager`)

Existing: `/summary` (store summary), `/lines` (orders, invoices and purchases by supplier, item or category),
`/supplier-weekly`, `/prices`, `/price-changes`, plus `/fnb-costing` (COGS and gross profit) and
`/finance/waste-log`. New:

| Method | Path | Returns |
|---|---|---|
| GET | `/finance/purchase-reports/dashboard?weeks=4` | `{received: Money, weekly: {week_start: string, received: Money}[], top_suppliers: {name, value}[], top_items: {name, value}[], price_rises: number, open_orders_value: Money}` |
| GET | `/finance/purchase-reports/transfers?weeks=4` | `{rows: {request_no, location, status, requested_at, asked: Money, sent: Money, received: Money, lost_in_transit: Money}[]}` |

"Production events" is not built. `GET /finance/exceptions` (existing) gains kinds `invoice_over_delivery`,
`price_rise`, `transit_shortfall`, `cooked_void`, `late_clock_in` and lowers the register variance threshold
to Rs 100. `GET /audit-logs` (existing) is the audit log.

### Team

Devices: §1. Staff: §2. Rosters and attendance: existing `/staff/shifts*` endpoints, unchanged; clock-in and
clock-out also work for PIN sessions.

### Setup

Existing: catalog product price and `active` (off the menu), `/kot/setup` (stations, section map), `/pos/settings`.
`GET /pos/setup` also returns `approval_manager_name: string | null`: the one active person eligible
under the checkout-discount approval policy. If there are zero/multiple eligible people, or the policy
is disabled, it is null and the screen uses the approved F&B Manager wording. This display field does
not select or authorize an approver. `Order.checkout_discount_by_name` identifies the actual approver
of the current discount, not this configured eligible person; own-authority edits/removal clear it.

`/pos/settings` gains `register_float: Money`; `member_discount_percent`, `max_checkout_discount_percent` and the
F&B tax setting already exist.

## 9. Live events

Existing topics (`orders`, `reservations`, `kot`) keep working. New:
- **`approvals`**: an approval request created or decided (core). The F&B Manager's queue and the screen that
  asked (a pending discount or void) refresh on it.
- **`till`**: a payment, a bill printed or reopened, a register close.
- **`devices`**: a device set up, revoked or unlocked. Device screens re-check `GET /auth/device` on it.
- **`stock`**: a kitchen request decided, dispatched or received, a goods-received note, waste or a count
  (inventory screens refresh on it).

## Changelog
- 2026-10-13 (approved by owner): added kitchen recall `Served→Preparing` with kitchen-only permission,
  venue/station checks, any-recorded-payment and cancellation guards, order-before-payment locking,
  Undo order-state recomputation, retained line flags/timestamps, no stock/print/slip/alert changes, and existing events/audit.
- 2026-10-13 (approved by owner): added pickup Undo `Served → Ready` with venue/station confinement,
  cancellation/payment guards and order-state recomputation; held lines prevent automatic order completion.

- 2026-10-13 (approved by owner): added optional `StaffUpdate.email`. Switching a PIN account to a password
  role requires email and a starting password, since PIN accounts have neither credential.

- 2026-10-13 (approved by owner): service Order.subtotal is before the separately returned member_discount
  in both profiles. Stored pricing/accounting subtotal remains net; totals and discount calculations are unchanged.

- 2026-10-13 (B5, owner-authorized F5 corrections): `/pos/setup` adds nullable `approval_manager_name`.
  It names the one active person eligible under the checkout-discount policy; zero/multiple eligible people
  or a disabled policy returns null, so the frontend uses its approved F&B Manager wording.
  `Order.checkout_discount_by_name` names the actual approver of the currently applied discount, null for
  own-authority discounts or removal. Pending/rejected requests do not replace current attribution.
  Payment responses retain legacy wrapper fields alongside the service Order fields for existing consumers.
