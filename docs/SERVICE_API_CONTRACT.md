# Service roles â€” API contract (shared)

The backend and frontend agents both build against this file. Work split: [backend](SERVICE_BACKEND_PLAN.md) Â·
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

**Roles.** The pilot runs six roles. Role ids follow PILOT_PLAN Â§6.1 (existing ids are kept).

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
  pin_change_required?: boolean }                      // PIN sign-in only (see Â§1)
```

## 1. Sign-in, devices and PINs (core)

### Password sign-in

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/auth/login` | `{email, password, remember?: boolean}` | `TokenResponse` |
| POST | `/auth/logout` | â€” | 204 |
| GET | `/auth/me` | â€” | `Me` |
| PUT | `/auth/my-password` | `{current_password, new_password}` | 204 |

- Session length: **12 h**, or **14 days** with `remember: true`. `remember` is ignored when the request carries an
  `X-Device-Key` (walk-up sign-in on a shared device): that session lasts 12 h and is bound to the device.
- An account whose role signs in with a PIN â†’ **403** `"Sign in with your PIN on a venue device."`
- Existing throttling stays (5 failures per email / 30 per IP per 15 minutes).
- `/auth/logout` makes the token unusable (server-side session row, see backend B2).

### Shared devices (write: `staff.devices`, which is `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| POST | `/auth/devices` | `{name, venue}` | `{device: Device, device_key: string}` (201); `device_key` is only ever returned here |
| GET | `/auth/devices` | â€” | `Device[]` |
| POST | `/auth/devices/{id}/revoke` | â€” | `Device` |
| POST | `/auth/devices/{id}/unlock` | â€” | `Device` |

```ts
interface Device { id: number; name: string; venue: Venue; created_by_name: string; created_at: Iso;
  last_seen_at: Iso | null; revoked_at: Iso | null; locked: boolean; failed_pins: number }
```

- `name` 2â€“60 characters, unique among devices that aren't revoked. Revoking ends every session bound to the device.
- The frontend sets a device up by signing in with `/auth/login` (manager), calling `POST /auth/devices`, storing
  the key, then calling `/auth/logout` for the manager's token.

### On a shared device (header `X-Device-Key: <device_key>`, no bearer token)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/auth/device` | â€” | `{device: Device, staff: PinStaff[]}` |
| POST | `/auth/pin` | `{staff_id, pin}` | `TokenResponse` |

```ts
interface PinStaff { id: number; full_name: string; role: Role; role_label: string; title: string }  // title e.g. "BBQ station"
```

- `staff`: active PIN-role accounts of the device's venue, in role order (waiter, cashier, cook, inventory_lead).
- Unknown or revoked key â†’ **401** `{"detail": "This device was switched off. Ask the F&B Manager to set it up again.", "code": "device_revoked"}`.
  The frontend deletes its stored key on this code.
- Wrong PIN â†’ **401** `"Wrong PIN. 3 tries left."`; the fifth failure in a row locks the device â†’
  **423** `{"detail": "This device is locked after 5 wrong PINs. Ask the F&B Manager to unlock it.", "code": "device_locked"}`.
  A correct PIN resets the count.
- Staff from another venue, or a password-role account â†’ **403** `"<name> doesn't work at <venue>."`
- PIN sessions last 12 h, are bound to the device, and end when the device is revoked.
- `pin_change_required: true` when the PIN was set or reset by the F&B Manager; that token can only call
  `PUT /auth/my-pin` until the PIN is changed (other calls â†’ **403** `{"code": "pin_change_required"}`).

| Method | Path | Body | Returns |
|---|---|---|---|
| PUT | `/auth/my-pin` | `{current_pin, new_pin}` | `TokenResponse` (a fresh, unrestricted token) |

`new_pin`: exactly 4 digits, not four of the same digit, not `1234`/`4321`, different from `current_pin`.

### Device-bound tokens everywhere

Any request whose token is bound to a revoked device â†’ **401** `{"code": "device_revoked"}`; a token past its
expiry or after logout â†’ **401** `"Invalid or expired access token"` (unchanged). The frontend returns to the right
sign-in screen on both.

## 2. Staff and roles (core; write: `staff`, which is `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/staff/role-catalog` | â€” | `RoleInfo[]` |
| GET | `/staff` | â€” | `StaffRow[]` |
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
- `assignable: false` for roles hidden by `roles.consolidated`; creating or moving someone into them â†’ **422**.

## 3. Floor: waiters (`outlets.order` = `waiter`, `cashier`, `manager`)

Permission groups list the pilot roles; in the full profile the old roles keep their access (backend B1).

### Tables and floor plan

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/floor?venue=Rooftop` | â€” | `Floor` |
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
| POST | `/cafe/orders/{id}/items` | `{items: OrderLineCreate[]}` | `Order` â€” fires every new line not on hold (`additional`) |
| POST | `/cafe/orders/{id}/fire` | `{line_ids?: number[]}` (omitted: every held line) | `Order` |
| PATCH | `/cafe/orders/{id}/lines/{line_id}` | `{seat_no?, note?, hold?}` â€” only lines not yet on a ticket | `Order` |
| POST | `/cafe/orders/{id}/bill` | â€” | `Order` (sets `bill_printed_at`; queues a bill print if the venue has a bill printer) |
| DELETE | `/cafe/orders/{id}/bill` | â€” | `Order` (reopens; **409** `"Payments are already taken. Ask the cashier."` once any payment exists) |
| GET | `/cafe/orders?venue=&open=true` | â€” | `OrderSummary[]` (existing list, filtered to unpaid) |
| GET | `/cafe/availability?venue=Rooftop` | â€” | `{items: Record<string, number \| null>}` portions left per product id; `null` = not limited; products taken off the menu are absent |
| GET | `/cafe/menu?venue=Rooftop` | â€” | `MenuSection[]` â€” everything the order screen shows; cached for offline use |

```ts
interface MenuSection { section: string; station_names: string[]; items: MenuItem[] }   // in menu order
interface MenuItem { product_id: number; name: string; description: string | null; price: Money;
  station_id: number | null; station_name: string | null; modifier_groups: ModifierGroup[];
  portions_left: number | null }          // same rule as /cafe/availability
interface ModifierGroup { id: number; name: string;          // "Spice level", "Portion", "Add-ons"
  choice: "one" | "many"; required: boolean; options: { id: number; name: string; price_delta: Money }[] }
```

`OrderLineCreate.modifier_ids` takes option ids. A line missing a required group's choice â†’ **422**
`"Choose <group name> for <item>."` Catalog products gain `description` (up to 120 characters, edited in Setup â†’
Menu & prices).

- `waiter_id` is the account that opened the order (`cafe_orders.created_by`); `PATCH /cafe/orders/{id}/details`
  gains `waiter_id` so the F&B Manager can hand a table over.
- After `POST /bill`, adding items â†’ **409** `"The bill is printed. Reopen the order to add items."`
- Guests: `guest_type: "Member"` needs `member_ref` and `member_name` (pilot rule, unchanged discount logic);
  `"Staff meal"` needs `cost_centre_code` and goes through the existing department-meal path when the bill is closed.
- **Voids.** The existing line-cancel endpoints keep their shapes. When the line's ticket is Preparing or Ready they
  return **202** with the standard approval payload (policy `pos_cooked_void`), the line shows `void_pending: true`
  and the station gets a `void_pending` slip. Approved â†’ `void` slip; rejected â†’ `void_rejected` slip (KOT contract).

### Pickups

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/kot/tickets?venue=Rooftop&status=Ready` | â€” | `Ticket[]` (KOT contract; `status` filter is new) |
| PATCH | `/kot/tickets/{id}/status` | `{status: "Served"}`; pickup Undo `{status: "Ready"}` after Served | `Ticket` |

`Ready â†’ Served` is new and allowed for `outlets.order` and `outlets.kitchen`. The order becomes Served when every
live ticket is Served and no uncancelled held line remains (it stays unpaid until the cashier closes it).
Owner-approved pickup Undo permits `Served â†’ Ready` for `outlets.order` and `outlets.kitchen`, with the same
venue/station checks. Cancelled orders refuse Undo; an order with any recorded payment returns **409**
`"Payments are already taken. Ask the cashier."`. Undo recomputes order readiness and does not repeat stock consumption.
The kitchen may also recall a served ticket to Preparing before any payment, under the rules in section 4.

## 4. Kitchen: cooks (`outlets.kitchen` = `cook`, `manager`)

KOT contract endpoints, with these changes:

- `GET /kot/tickets` for a `cook`: the venue is the account's venue; a station cook always gets only their station
  (`station_id` in the query is ignored); a pass cook (`station_id: null`) gets every station and may filter.
- `PATCH /kot/tickets/{id}/status` transitions: `Placedâ†’Preparing`, `Preparingâ†’Ready`, **`Readyâ†’Preparing`** (recall,
  new), **`Readyâ†’Served`** (new), **`Servedâ†’Ready`** (pickup Undo before payment), and **`Servedâ†’Preparing`** (kitchen recall before payment).
  A station cook may only change their station's tickets â†’ **403**
  `"That ticket belongs to <station>."`
- `Servedâ†’Preparing` requires `outlets.kitchen` (`cook`, `manager`, and the group's legacy roles);
  waiters/cashiers get **403**. Venue confinement is unchanged; pass cooks may recall any station in their venue.
  Cancelled tickets/orders return **409** with the existing invalid-transition/Undo wording. Any recorded payment,
  including partial, full or later reversed payments, returns **409** `"Payments are already taken. Ask the cashier."`.
  Lock the order, then the ticket, before checking payments, as for Undo. Return the same updated `Ticket`;
  reuse `Readyâ†’Preparing` timestamp handling (`accepted_at` retained/set, `ready_at` cleared), and clear a served
  timestamp if present (the current ticket schema has none). Keep line `done` flags. Recompute order state using
  the pickup Undo helper: Served only when every live ticket is Served and no uncancelled held line remains.
  Recall never consumes/restores stock or creates a print job, slip or alert; retain status-change live topics
  and audit. Existing `Placedâ†’Preparing` and `Readyâ†’Preparing` remain valid; repeated `Preparingâ†’Preparing` is invalid.
- Crossing off lines is shared between screens:

| Method | Path | Body | Returns |
|---|---|---|---|
| PATCH | `/kot/tickets/{id}/lines/{order_line_id}` | `{done: boolean}` | `Ticket` |

`TicketLine` gains `done: boolean` and `seat_no: number | null`; `Ticket` gains `waiter_name` (already present) and
`late_after_seconds: 720`. "All day" counts are computed by the frontend from the open tickets.

## 5. Till: cashiers (`outlets.pay` = `cashier`, `manager`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/bills?venue=Rooftop` | â€” | `{waiting: OrderSummary[], dining: OrderSummary[]}` â€” waiting: bill printed and unpaid, oldest print first; dining: unpaid, bill not printed |
| GET | `/cafe/orders/{id}` | â€” | `Order` (existing) |
| POST | `/cafe/orders/{id}/payments` | `PaymentCreate` | `Order` |
| DELETE | `/cafe/orders/{id}/payments/{payment_id}` | â€” | `Order` (only while `payment_status != "paid"`, same shift; otherwise **409** `"Paid bills are refunded through the F&B Manager."`) |
| POST | `/cafe/orders/{id}/bill` | â€” | `Order` (a cashier may print too) |
| PUT | `/cafe/orders/{id}/discount` | existing `CheckoutDiscount` | `Order`, or **202** approval above the limit |
| POST | `/cafe/orders/{id}/charge-department` | existing | existing (staff meals; now `outlets.pay`) |
| GET | `/cafe/orders/{id}/receipt` | â€” | existing receipt |

```ts
interface PaymentCreate { tender: "cash" | "card" | "bank_transfer"; amount: Money;
  reference?: string;    // card approval code (optional) or transfer reference (required for bank_transfer)
  tendered?: Money;      // cash handed over; must be >= amount
  split_label?: string; client_ref?: string; captured_at?: Iso }
interface Payment { id: number; tender: "cash" | "card" | "bank_transfer"; amount: Money; reference: string | null;
  tendered: Money | null; change: Money | null; taken_by_name: string; taken_at: Iso }
```

- `amount` above `balance_due` â†’ **422** `"That's more than the balance of Rs <x>."`
- The payment that makes `balance_due` 0 sets `payment_status: "paid"`; the table is free from that moment.
- A zero-value bill without any captured settlement remains unpaid until explicit department/staff-meal
  settlement; a captured zero settlement or refund history retains its existing paid state.
- `discount_request` is set while a checkout discount waits for approval; approval applies it (existing action).

### Register (shift close)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/cafe/registers/current?venue=Rooftop` | â€” | `RegisterShift` |
| POST | `/cafe/registers/close` | `RegisterCloseCreate` | `RegisterClose` (201) |
| GET | `/cafe/registers/closes?venue=&limit=20` | â€” | `RegisterClose[]` (cashier: own venue; manager: all) |

```ts
interface RegisterShift { venue: Venue; opened_at: Iso; opening_float: Money; payments: number;
  card_total: Money; bank_total: Money; unpaid_printed_bills: number }   // cash and expected cash are withheld (blind count)
interface RegisterCloseCreate { venue: Venue; counts: Record<"5000"|"1000"|"500"|"100"|"50"|"20"|"10", number>;
  coins: Money; note?: string }
interface RegisterClose { id: number; venue: Venue; opened_at: Iso; closed_at: Iso; closed_by_name: string;
  opening_float: Money; cash_total: Money; card_total: Money; bank_total: Money; expected_cash: Money;
  counted_cash: Money; variance: Money; note: string | null; counts: Record<string, number>; coins: Money }
```

- A variance of Rs 100 or more needs `note` â†’ **422** `"Say what happened: the count is Rs <x> <short|over>."`
- Closing starts the next shift at once with the configured float (`pos_settings.register_float`, default 10,000).
  A setting change applies to the next shift; the current shift retains its opening float.

## 6. Venue stock: inventory leads (`inventory_lead` limited to its venue's location)

Existing endpoints (`/fnb/kitchen-requests*`, `/inventory/fnb/*` counts and enquiry, `/fnb/waste*`) keep their shapes.
`/fnb/waste` and `/fnb/waste/options` share the existing `/kitchen/waste*` handlers.
Kitchen requests may include optional `needed_by`, an ISO timestamp including timezone.
Changes:

- `inventory_lead` gets the groups `fnb.kitchen` (request, withdraw, waste) and the new **`fnb.receive`**:
  `POST /fnb/kitchen-requests/{id}/receive` now needs `fnb.receive` (= `inventory_lead` for its own location, `manager`)
  instead of `fnb.approve`. Another venue's request â†’ **403** `"That request is for <location>."`
- Counts (`/inventory/fnb/counts*`) and stock enquiry are limited to the lead's location, as for kitchen staff today.

| Method | Path | Returns |
|---|---|---|
| GET | `/fnb/forecast?location_id=` (new) | `ForecastRow[]` â€” ingredients that won't last until the next delivery |
| GET | `/fnb/overview?location_id=` (new) | `LocationOverview` â€” the inventory lead's Overview screen |
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
(0 = Monday â€¦ 6 = Sunday, default 6).

The ingredient drawer (movements, batches, par) uses the existing `GET /inventory/fnb/enquiry/{ingredient_id}`,
which already limits a venue login to its own kitchen.

`daily_use`: average daily consumption at that location over the last 14 days. `next_delivery_at`: the earliest
`needed_by` of an approved or dispatched request for that location, else the next 09:00.
Enquiry retains its existing `{locations, ingredients}` envelope; zero balances remain in the list.

### F6 confirmed stock interfaces (2026-10-13)

All the lead's reads/writes below are confined to its own kitchen location; an explicit other location
returns 403 using the existing venue-scope message. General `fnb.values`, valuation and purchasing
permissions are not granted to the lead. Full-profile kitchen staff retain their quantity-only view.

**Counts.** Start `POST /inventory/fnb/counts` with `{location_id, note?}`, save
`PUT /inventory/fnb/counts/{id}/counts` with `{counts: Record<ingredientId, number | null>}`, and submit
`POST /inventory/fnb/counts/{id}/post` are available to `inventory_lead` at its location. Save/post retain
normal count adjustments, batch handling and the balanced stock gain/loss journal. Quantities must be finite;
negative counts return 422 `"Counts can't be negative"`. There is at most one open count per location;
a second start returns 409 `"A stock take is already open for this location"`. Closed writes return
409 `"This stock take is closed"`.

Owner-approved discard: `DELETE /inventory/fnb/counts/{id}` returns the updated count with status
`Discarded`. It accepts only `Counting`, retains lines/history, changes no stock or journal, audits the
action, announces `stock`, and permits a new count. Permission is `fnb.count` (the lead at its own location;
existing authorized counters retain access). `Posted`/`Discarded` return the closed-write 409 above.
Count statuses are `Counting`, `Posted`, `Discarded`.

Count list/detail fields include `id`, `number`, `location_id`, `location`, `status`, `started_at`,
`started_by_name`, `posted_at`, `posted_by_name`, `items` (number of lines), `counted` (entered lines), and
`variance_value` (signed rupees). Detail/write responses also include `lines[]` with `ingredient_id`,
`name`, `unit`, nullable `storage_area`, `pack_size`, `system_qty`, nullable `counted_qty`, `variance`,
`unit_cost`, and `variance_value`. An uncounted line has null variance. Values are available to the scoped
lead and existing value roles. Counts retain the existing frozen-system snapshot adjustment rule; activity
between start and posting is not frozen. List responses do not need to include lines.

**Waste.** `/fnb/waste*` and `/kitchen/waste*` share handlers. The lead's log and POST response add `cost`
(rupees at the recorded cost) and nullable `note`, keeping `reason` separate. They retain `id`, `created_at`,
`item_name`, nullable `ingredient_id`/`product_id`, `quantity`, `unit`, `logged_by_name`, `venue`.
Legacy combined reasons are split only when they start with a known reason followed by `": "`; arbitrary
historic Finance text is retained. Existing legacy kitchen readers still receive no cost field.

`GET /fnb/waste/options` returns `{ingredients, menu_items, reasons, venues, venue}`. Ingredients contain
`id`, `name`, `unit`, `stock_qty`, `par_level`, `reorder_level`, `needs_restock`, plus `avg_cost` for the lead
and existing value roles. Reasons are exactly `Expired`, `Spoiled`, `Dropped / spilled`, `Overcooked / burnt`,
`Returned by guest`, `Over-prepared`, `Other`. POST body is `{ingredient_id, quantity, reason, note?, venue?}`
(or `product_id` instead of ingredient id). A lead may omit venue, using its own. Kitchen waste retains the
existing ability to take stock negative; exceeding on-hand has no server rejection in this change.

**Enquiry.** Ingredient list rows retain `id` and add alias `ingredient_id`, numeric `reorder_level`,
and `daily_use` for every ingredient including zero stock/zero use. Scoped lead rows include `avg_cost`
even at zero balance. Drawer `ingredient.daily_use` uses the same rolling 14-day net sale/sale-reversal
average as forecast. Non-risk ingredients remain in enquiry. Negative net use is preserved, rather than
inventing consumption; forecast includes only positive daily use.

Drawer movements retain raw kinds (`opening`, `receipt`, `transfer_in`, `transfer_out`, `sale`,
`sale_reversal`, `waste`, `transit_loss`, `count`, `revaluation`), `created_at`, signed `change_qty`,
`by_name` and nullable `reference`. Reference resolves a request number, count number, recorded waste reason
or order code through the sale owner's registry provider; otherwise recorded note, or null. No order
reference is invented when the selling module/record is unavailable. Overview still uses its normalized
kinds. Both quantity fields are signed from the selected location: incoming positive, outgoing negative;
count may have either sign. A kitchen receives `transfer_in`; request dispatch and transit loss occur at
the store/transit locations and do not appear as outgoing kitchen movements.

**Requests.** Lead list/get default to its own location; `kitchen` query is optional. Statuses are
`Requested`, `Approved`, `Dispatched`, `Received`, `Received with variance`, `Rejected`, `Withdrawn`;
the last four are terminal. Existing request fields include `id`, `number`, `kitchen_location_id`, `kitchen`,
`status`, `needed_by`, `remarks`, `requested_by_name`, `requested_at`, `approved_by_name`, `approved_at`,
`decision_note`, `dispatched_by_name`, `dispatched_at`, `received_by_name`, `received_at`, and `lines`.
Responses add nullable aliases `decided_by_name`/`decided_at`, nullable `dispatch_note`, and `value` (requested
quantity times unit cost, rounded to rupees, for the lead and value roles). Each line includes `id`,
`ingredient_id`, `name`, `unit`, `requested_qty`, nullable `approved_qty`/`dispatched_qty`/`received_qty`,
nullable `variance_reason`, and numeric `unit_cost`. Before dispatch, unit cost uses current ingredient
average; after dispatch it uses the recorded dispatch cost. These fields do not expose store quantities.

Create still requires `{kitchen_location_id, needed_by?, remarks?, lines: [{ingredient_id, quantity}]}`;
quantities must be positive but need not be multiples of pack size (rounding is a frontend suggestion).
Dispatch accepts optional `{dispatched_quantities?: Record<lineId, number>, note?: string | null}`;
`note` (up to 300 characters) is stored as `dispatch_note`. Receipt accepts optional
`{received_quantities: Record<lineId, number>, variance_reasons: Record<lineId, string>}` and returns the
updated request. A short line without reason returns 422 `"Say why less <ingredient> arrived than was sent"`.
Withdrawal is permitted only while Requested, by the original requester or manager; otherwise 409
`"This request is already <lowercase status>"`, or 403 `"Only the person who asked can withdraw this request"`.

## 7. F&B Store (`fnb_store_staff`)

Existing dispatch, purchase-request and goods-received endpoints keep their shapes. New:

| Method | Path | Returns |
|---|---|---|
| GET | `/fnb/batches/expiring?location_id=&days=2` | `Batch[]` â€” batches with stock left that expire within `days` |

```ts
interface Batch { id: number; ingredient_id: number; name: string; unit: string; batch_no: string | null;
  expiry_date: string; remaining: number; location: string }
```


### F7 source confirmations and additive support (2026-10-13 continuation)

- Store may read `GET /finance/goods-received`; its response is confined to `department = "F&B Store"`.
  This grants no Finance invoice/vendor/payment writes. Finance readers keep all existing receipt access.
  Header `total_received_value` retains its legacy **arrived** quantity value for compatibility; new
  `accepted_received_value: Money` subtracts rejected quantity. `lines[].quantity_received` is **all arrived**;
  accepted = arrived - rejected. Fully rejected lines send arrived = rejected > 0, accepted = 0. Sending
  a zero `quantity_received` fails the existing positive-arrival schema validation (422 detail array).
  Positive arrived quantity with rejected greater than arrived returns 422 `"More rejected than arrived"`.
  Null `batch_no` remains null; the server does not invent a batch number. Receipt POST adds top-level
  `grn_code: string`, preserving `grn.grn_code`, `grn`, and `bill`.
- `/inventory/fnb/enquiry` retains storage area/pack size and adds nullable `vendor_id`, `vendor_name`,
  `perishable: boolean | null`, and `next_batch: {batch_no: string | null, expiry_date: string | null} | null`.
  Supplier means explicitly configured **usual supplier**, not an inferred or exclusive supplier. Ingredient
  POST/PATCH accept optional `vendor_id` (null clears) and `perishable` (null = unknown), with existing
  `fnb.costing.manage`/`fnb.costing.edit` permissions. An absent/inactive supplied vendor returns 404
  `"Active vendor not found"`. Existing ingredients retain null metadata until configured.
  FIFO is received_on then id, matching depletion; next_batch uses positive remaining at the selected
  location, defaulting to Store for an unfiltered club-wide enquiry, and the scoped kitchen for a lead.
- Dispatch already supports optional `dispatched_quantities` keyed by request **line id** and `note` (300
  characters); omitted keys use approved quantities. It returns the updated request and dispatch_note/actor/time.
  Above-approved/negative/non-finite quantities return 422 `"Dispatch between 0 and <approved> <unit> of <name>"`;
  insufficient stock is 409 `"Not enough in the F&B Store: ..."`; zero on every line is 422
  `"Dispatch at least one item"`. Store list without kitchen/status is valid across venues, descending id,
  up to 300 requests; existing seven status strings and actor/quantity fields remain as section 6.
- Purchase request POST accepts optional aware `expected_delivery_at: Iso | null`; it is retained on the
  approved converted PO. `needed_by_date` remains supported independently. PO list adds that field and
  nullable perishable on ingredient-linked lines, retaining unit, ingredient_id, quantities, costs,
  request_number and estimated_total. Undated legacy delivery times stay null; no midnight time is invented.
  Purchase request fields/statuses remain Pending, Awaiting Finance (full only), Awaiting second approval
  (consolidated only), Converted and Rejected, with decision_note, decided_by_name/decided_at.
  Created request is returned with number; vendor list has id/name/category; one request per vendor is valid.
- **Owner-approved expiry:** expiry_date is the first unusable date, starting at **00:00 Asia/Karachi**.
  `/fnb/batches/expiring` adds `expires_at: Iso`, `expired: boolean`, `days_left: number` and includes past-due
  positive lots. Date today is expired; tomorrow starts expiry tomorrow. This defines date/report semantics;
  existing FIFO consumption does not automatically block expired lots. No unbatched stock gains an invented
  expiry. Store location id is valid; existing ingredient/location/batch/remaining fields stay unchanged.

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
  Threshold money is stored to two decimals and compared in whole paisa. The same person â†’
  **409** `"A different F&B Manager has to give the second approval."`
- `POST /fnb/purchase-requests/{id}/finance-decision` â†’ 404 in the pilot.

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
| POST | `/finance/vendor-bills/{id}/settle-received` | â€” | `VendorBill` â€” sets the payable amount to the received value and raises a debit note for the difference |

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

### B7 implementation confirmations (2026-10-13)

The manager reports above are implemented in full and pilot; existing Finance report envelopes remain.
`/cafe/venues/live` reports all three service venues, including empty ones. Today's sales use captured
payment rows (department meals excluded), with the registered revenue source's tax/service-inclusive
amounts. Open tables are distinct unpaid/noncancelled table codes; waiting bills are printed unpaid orders.
Late tickets are live Placed/Preparing/Ready tickets at least 720 seconds old, matching KOT. Average ticket
seconds uses creation to ready for Ready/Served tickets made ready on today's Asia/Karachi date; no samples
returns null. Existing `outlets.reports` access applies; venue PIN roles cannot read club aggregates.

`/finance/daily-summary` retains its outlet-row array. Each row adds `hours` (24 entries, hour 0-23, payment
amounts on the selected local day) and `checkout_discounts` (live orders opened that day). `date` is accepted
alongside legacy `business_date`; date wins when both are supplied. Export keeps its old shape. Hours and
order totals have different date bases and must not be added together. F&B gross adds back member/checkout
discounts from stored net amount; net does not subtract those discounts twice. `/finance/discounts` adds
`Member discount` and `Checkout discount` categories and accepts `date` alongside start/end dates.
Payment transactions also accept `date` and optional display venue, preserving required `tender`/old filters.

`/finance/voided-transactions` keeps its array and adds `kind: "void" | "cooked_void"` and nullable `line_id`
for outlet events. It accepts optional `date`. Applied line cancellations include recorded line value and
reason; whole cancellations retain their order value. A prior line cancellation remains a separate event
when the remaining order is cancelled. New outlet cancellations record time and cooking provenance before
cancelling tickets, including served food. Historical records use recorded audit time/ticket provenance,
then the legacy order update timestamp when no dedicated event time exists. Approval rules are unchanged.

`/finance/vendor-bills/{id}/settle-received`: manager/Finance permission, no body, returns expanded bill.
**Owner decision: untaxed invoices only.** Tax amount nonzero returns 422
`"Received-value settlement currently supports untaxed invoices only"`; nothing changes. Only unpaid
Pending/Approved bills without previous payments or notes may be settled. A matched GRN must belong to the
supplier; absent direct match, exactly one linked PO with exactly one GRN is required. Unmatched/ambiguous,
paid/disputed, existing-note/payment, supplier mismatch, or receipt value above invoice returns plain 409.
Accepted value is arrived quantity less rejected quantity, times actual receipt unit cost, rounded to paisa.
Bill amount becomes that value and status Approved. A positive difference creates one lowercase `debit`
note tied by `settlement_note_id`; it documents the already-applied reduction and is not applied again to
balance or GL. Existing legacy Credit/Debit behavior remains. Optional bill metadata:
`received_settled_at`, `settlement_original_amount`, `settlement_note_id`. Equal value creates no zero note;
fully rejected goods may settle to zero. Stock/GRN/line history stays unchanged. Bill, note and
`finance.bill_settled_received` audit commit atomically; concurrent/repeated calls return the same settlement,
even after payment, without additional mutations. Payments lock the bill and cannot exceed a settled balance.

Dashboard/transfers accept `weeks` 1-52 (default 4), using that many Monday-based calendar weeks including
the partial current week. Dashboard accepted F&B Store deliveries use receipt dates; weekly zeros retained,
top five suppliers/items use actual values; price rises reuse same-supplier delivery comparisons. Open-order
value is remaining estimated value of Approved F&B Store POs, independent of period. Transfers use request
creation dates; asked uses requested quantities/current average until dispatch, then recorded unit cost;
sent/received use recorded quantities/cost. Lost value is nonnegative and exists only after confirmed receipt.

Exception feed retains legacy categories and adds the five named B7 kinds. Cash variance includes either
sign at Rs 100. Invoice-over-delivery compares current invoice/accepted matched GRN value and omits settled
or cancelled/voided/paid bills. Price rise means at least 4% versus the same supplier's previous delivery.
Transit shortfall uses recorded receipt time and actual missing value. Cooked voids require applied
cancellations; late clock-in means actual clock-in later than scheduled start. Attendance provider exists
only with `staff.shifts`, including pilot without payroll. Finance consumes generic owner registry providers;
no unavailable module data is invented. Production events remains outside scope.

### Team

Devices: Â§1. Staff: Â§2. Rosters and attendance: existing `/staff/shifts*` endpoints, unchanged; clock-in and
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
- 2026-10-14 (owner-authorized remaining integration work): completed linked invoice comparisons,
  period supplier reliability, preceding actual receipt-price change, exception presentation,
  Karachi Store expiry mapping, F&B-scoped Open orders export and location stock summary.
  Definitions and compatibility rules are in the integration completion section below. Tests skipped by owner.
- 2026-10-13 (owner-authorized B7): implemented venue-live aggregates, per-outlet hourly/checkout daily
  data, member/checkout discounts and cooked voids, purchase dashboard/transfers and exception providers;
  retained existing report envelopes/date parameters and added compatible date aliases.
- 2026-10-13 (owner decision): received-value settlement supports untaxed invoices only, with one atomic
  supplier debit note, unchanged stock, preserved legacy note semantics, idempotency and locked payment cap.
- 2026-10-13 (owner-authorized pre-B7 F6): added scoped lead count posting/values, count history names and
  totals, waste cost/average cost and separate note, all-ingredient daily use, movement document references,
  request decision aliases/value and optional dispatch note. Preserved legacy role masks, stock signs,
  request states, snapshot count adjustments and negative kitchen-waste rule.
- 2026-10-13 (approved by owner): added `DELETE /inventory/fnb/counts/{id}` for Counting → Discarded,
  retaining count history without inventory/journal changes, scoped permissions, closed-write refusal,
  audit and stock event.
- 2026-10-13 (approved by owner): added kitchen recall `Servedâ†’Preparing` with kitchen-only permission,
  venue/station checks, any-recorded-payment and cancellation guards, order-before-payment locking,
  Undo order-state recomputation, retained line flags/timestamps, no stock/print/slip/alert changes, and existing events/audit.
- 2026-10-13 (approved by owner): added pickup Undo `Served â†’ Ready` with venue/station confinement,
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


### Manager additive fields and owner recipe decisions (2026-10-13 continuation)

- Daily-summary rows add `covers: number | null`, `tax: Money | null`, `open_count: number | null` from the
  owning sales provider. F&B covers/tax use noncancelled orders created in the selected Karachi calendar day;
  open_count counts those still unpaid (including partial). They are order-based, unlike payment hours.
  A provider lacking metadata yields null rather than invented attribution. Empty rows return zero.
- Tender transactions add nullable `outlet` key, display `venue`, `table_code`, `cashier_id`, `cashier_name`.
  Cashier is the recorded payment actor, never the current Manager; unavailable owner metadata remains null.
- `/fnb-costing` accepts optional `outlet` display venue; invalid venue is 422 `"Choose an F&B outlet"`;
  a scoped user selecting another is 403 `"Choose your own outlet"`. Existing combined items are retained.
  New `product_venue_items[] {venue, product_id: number | null, name, kind, sold, sales, cost}` contains every
  grouped product/venue series without the legacy top-50 cutoff. `count_variance: Money` is the signed sum
  of posted count line adjustments in the selected period/location; `counts[]` has id/location_id/location/
  posted_at/variance_value. Discarded/open counts are excluded. Count variance is separate from recipe costs.
- Recipe POST accepts `overhead_percent: number` (default 0, finite, 0..1000) alongside ingredient quantities.
  The owner selected percentage of ingredient cost for the **whole recipe**, not a fixed implicit 8%.
  Cards retain ingredient-only plate_cost/margin fields and add ingredient_cost, overhead_percent,
  overhead_amount, total_plate_cost, margin_after_overhead, margin_percent_after_overhead. These are current
  recipe estimates; entered overhead creates no inventory quantity or ledger movement and does not restate
  historical ingredient movement costs. Explicit empty recipes are allowed and have unlimited order stock;
  products with no recipe retain the catalog stock gate. No arbitrary stock quantity is seeded.
- Vendor scorecard adds nullable `fill_rate_percent` (mean capped accepted/ordered ratio per delivered PO
  line; mixed measurement units are never added), `delivered_in_full_percent` (all lines received in full per
  delivered PO), and `delivered_orders`. No deliveries means null percentages; rejected quantities do not
  count as filled. Existing on_time_percent/spend/quality fields retain their meanings.
- `PATCH /staff/shifts/{id}` adds optional aware start_time/end_time and department alongside existing
  status. Empty/null changes return 422; shifts must use an F&B work location and last up to 24 hours.
  Scheduled overlap is 409 `"This employee already has an overlapping shift"`; any attendance refuses edits
  with existing 409 `"A shift with attendance cannot be cancelled or rescheduled"`. Payroll freeze checks
  remain. Permission `payroll.roster`, including manager; shift presets remain a product decision.

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/finance/purchase-reports/dashboard/export.xls` | weeks (1..52, default 4) | XML Excel workbook; fnb.costing.view |
| GET | `/finance/purchase-reports/transfers/export.xls` | same weeks | XML Excel workbook; fnb.costing.view |
| GET | `/inventory/fnb/waste/export.xls` | from/to calendar dates | XML Excel workbook; fnb.values |
| GET | `/inventory/fnb/store/export.xls` | — | XML Excel workbook; fnb.values |
| GET | `/finance/exceptions/export.xls` | — | XML Excel workbook; existing Finance view roles |
| GET | `/audit-logs/export.xls` | limit (1..500, default 500) | XML Excel workbook; manager/headstaff/top_brass/finance_staff |

Exports preserve the existing workbook format and bounded source report scope: waste is its existing top-30
item report; exceptions is its existing bounded feed; audit is the selected latest-row limit. These are
report exports, not an unbounded archival dump. Browser download acceptance is a separate gate.

2026-10-13 continuation changelog: F7 receipt-list permission and accepted-value projection, explicit supplier/
FIFO batch/perishable metadata and aware delivery timestamps; owner-defined midnight expiry and empty
recipe behavior; Manager sales attribution/totals, product/venue/count projections, entered percentage
recipe overhead, derived supplier fill, editable shift times and six missing workbook exports. Shapes are
additive; legacy arrived receipt total and ingredient-only recipe cost remain compatible. Untaxed-only
settlement, six devices, retained count discard and blocked direct New PO decisions remain unchanged.

Manager void attribution continuation: `/finance/voided-transactions` retains all existing fields and adds
`order_id`, `venue`, `table_code`, `status: "Cancelled"`, nullable `requested_by`/`requested_by_name`,
`approved_by`/`approved_by_name`, `approval_id`/`approval_status`. Applied/Authorised approval provenance wins;
otherwise the recorded cancellation audit actor is the requester. Missing historic provenance is null.
These are applied cancellation events, not invented cash refund transactions or pending approval cards.
The Finance view permission is unchanged.

Owner expiry confirmation: expiry warnings begin at **00:00 Asia/Karachi on the stated date**. Expired
positive lots remain visible in the warning feed. The owner explicitly retained existing dispatch and
recipe-consumption stock behavior; expiry does not reject stock or impose a new availability gate.

Costing scope also applies to `/fnb-costing/export.xls?outlet=<display venue>` with the same 422/403 refusals. Both costing read/export routes use `fnb.costing`; existing `view=outlets|items` and workbook columns remain compatible.

2026-10-13 final backend changelog: B8 operational owning-workflow showcase, atomic rollback/source-hashed
reruns, retained pilot-to-full modifier/device/batch/history identities; B9 frozen full/pilot automated
validation and cleanup complete. Evidence: ../backend/docs/SERVICE_B9_REPORT.md. Frontend handoff and
remaining decisions: ../backend/docs/SERVICE_FRONTEND_HANDOFF.md. Browser/printer/deployment acceptance
remains separate, and contradictory prototype totals are documented in the showcase map.

## Integration completion — 14 October 2026

These are additive read fields/operations; approval rules, journals and stock writes are unchanged.

- `GET /finance/purchase-orders/export.xls` accepts optional `department` alongside the existing
  status/vendor/date filters. Omitted filters add no SQL condition. Manager Open orders uses
  `status_filter=Approved&department=F%26B%20Store`, matching its JSON list. Existing clients that
  omit department retain the full permitted scope.
- Invoice item summary rows add nullable `received_value: Money` and `difference: Money`.
  The period selects invoices by their bill date (else Karachi created date), not unrelated
  receipts in the same period. Comparison uses the explicit matched GRN, or exactly one receipt
  of the linked PO, from the same vendor. More than one active bill sharing that receipt is
  ambiguous and remains null: no proportional allocation is invented. The receipt's actual
  accepted quantity times actual unit cost is grouped by ingredient identity (else normalized
  item name) and counted once per bill/item even with repeated invoice lines. Missing item
  matches/unitemised bills stay null. Difference is invoiced amount minus accepted receipt value.
- Purchase supplier summary rows add nullable `on_time_percent`, `delivered_in_full_percent`,
  `fill_rate_percent`, and `delivered_orders`. The cohort is distinct POs with receipts selected
  by the requested receipt period and existing filters, including fully rejected receipts.
  Quantities include all receipts up to the period end, never later receipts. On-time is the
  latest receipt time against the aware delivery timestamp, or its Karachi date against the
  delivery date; undated POs are excluded from that denominator. Delivered in full is the
  percentage of cohort POs with every ordered line accepted in full. Fill rate is the mean
  capped accepted/ordered ratio of positive-quantity PO lines. Zero-denominator percentages
  remain null. These period metrics do not replace lifetime supplier scorecards.
- Price report rows add nullable `change_percent`: latest actual accepted-delivery unit cost
  minus its preceding actual accepted-delivery unit cost, divided by the preceding cost.
  Uses history up to the selected period end and the supplier filter when supplied. With fewer
  than two eligible receipts or zero preceding cost it remains null. It is not supplier spread.
- Exception rows retain kind/detail/amount/created_at and add `title`, `area`, `severity`.
  Cash variance and invoice-over-delivery use `bad` / Check now; discount, expense, cooked void,
  transit shortfall, price rise and late clock-in use `warn` / Review. Unknown provider kinds
  default to Review; a provider's declared presentation wins. Title defaults to the recorded
  detail. Priority sorts before reverse creation time. This changes no approval policy.
- Existing summary/price/exception workbooks preserve their legacy columns and append these
  new values. Unknown comparisons are empty cells, not fabricated zeros.
- `GET /inventory/fnb/location-summary` and `/location-summary/export.xls` require `fnb.values`.
  They return a last-seven-days operational quantity bridge at **current average ingredient cost**:
  `from`, `to`, `valuation_basis: "current_average_cost"`, `rows`, `totals`. Rows carry `key`,
  `location`, `opening`, `incoming`, `sent`, `used`, `waste`, `count_adjustment`,
  `other_adjustment`, `closing`. Only active Store/kitchen locations are shown, confined to
  the own kitchen if applicable; transit is excluded. Sale reversals reduce used value.
  Opening is closing minus the signed recorded quantity movements over the interval, all
  consistently priced at current costs. Other quantity adjustments remain separate from
  counts; pure revaluations do not become quantity flows. This is not historical GL valuation.
  The original Store stock export is retained for existing clients. Manager Reports uses the
  new location-summary workbook, which matches its table.
- Supplier by week exposes the existing `kind=purchases|orders|invoices` control. This week
  displays a minimum of two calendar buckets, matching the prototype's comparison layout.
- Store expiry uses the existing aware `expires_at`, falling back only to the specified date
  at `00:00+05:00`. Current/older expiry is labelled Expired; tomorrow is Expires tomorrow.
  Stock batch dates render in Karachi even when the browser uses another timezone. No expiry
  warning blocks consumption or dispatch.
