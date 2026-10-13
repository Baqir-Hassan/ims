# Service roles — frontend plan

Owner: **frontend agent.** Touches only `frontend/`. Does not edit `backend/`.
Build against [SERVICE_API_CONTRACT.md](SERVICE_API_CONTRACT.md) for data and behaviour. **The screens must look
and read exactly like the prototype** [`prototype/index.html`](../prototype/index.html). How to run it and reach
every state is in [`prototype/README.md`](../prototype/README.md). Read "Visual fidelity" below before writing any
screen; it overrides anything else in this plan or in ARCHITECTURE.md about how screens look.

## Visual fidelity (read first)

The prototype is the visual specification, not inspiration. A screen that works but looks different from the
prototype is not done.

1. **Which document decides what.** The prototype decides layout, spacing, sizes, type, colour, icons, motion,
   wording, empty and error states, and breakpoints. The contract decides data and behaviour. The contract carries
   every field the prototype shows; if a screen seems to need data the contract lacks, stop and get the contract
   extended. Never drop or restyle an element to fit the data.
2. **Port the CSS, don't rewrite it.** Copy the prototype's whole `<style>` block into
   `app/core/service/service.css` with the same class names, values, keyframes and media queries. The only edits
   allowed:
   - rules are scoped under the `.svc` root (`.svc .btn`, …), and the `:root`/`html`/`body` rules move onto `.svc`;
   - font families point at the `next/font` variables;
   - the prototype-only controls (`.proto*`, `.demo-acc`) are left out.
3. **Reproduce the markup.** Each React component renders the same element structure and class names as the
   matching prototype component (`prototype/README.md` maps them). Treat the `htm` templates as the markup spec and
   translate them to JSX one to one, including inline `style` values where the prototype has them.
4. **Keep the legacy styles out.**
   - The service shells render in their own `<div class="svc">` root, outside `.rz` and `.entry`, and use no classes
     from `globals.css`, `redesign.css`, `console.css` or `entry.css`.
   - No Tailwind utility classes in these screens.
   - `.svc` resets what it inherits from the page: font, colour, background and box sizing.
   - ARCHITECTURE.md's "Frontend styling" section (Instrument Sans, new styling in `console.css`) describes the old
     console and doesn't apply here; F1 updates it.
5. **Same fonts.** Archivo and Martian Mono, both with their width axis, through `next/font/google`
   (`axes: ["wdth"]`). Check that `font-stretch` takes effect: table codes and kitchen headers are narrow in the
   prototype, and a font loaded without the width axis silently breaks them.
6. **Same icons.** Port the prototype's icon set (the `P` path map and `Icon`) to `app/core/service/icons.tsx`. Don't
   substitute lucide icons in these screens.
7. **Same words.** Copy every visible string verbatim: titles, sub-lines, labels, buttons, toasts, empty states and
   the messages in sheets. The exceptions:
   - the brand name comes from `BRAND.name`, which reads "Rykards" in the pilot, as in the prototype;
   - errors sent by the server are shown as sent (the contract's wording matches the prototype's).
8. **Same motion and themes.** The same keyframes and durations (rise, slide, drop, ring, flash, shake, hl), the same
   reduced-motion rule, and the same light and dark tokens. Screens follow the system theme; the kitchen display is
   always dark.
9. **Same charts.** Port the chart geometry as written: viewBoxes, bar widths, gaps, gridlines, label positions and
   the scatter's label-collision rule (`BarChart`, `HourChart`, the inventory usage chart, `MgrCosting`'s scatter).
10. **Not shipped:** the Prototype tab, the Appearance toggle, the "Prototype: …" hint lines, the demo-account chips,
    the simulation and Reset demo data.
11. **Deviations.** Only two kinds of difference are allowed, and each one goes under "Deviations" at the bottom of
    this file with its reason. Anything else that differs is a bug.
    - an accessibility fix, such as a missing label or focus state;
    - real data longer than the demo's, which must wrap or truncate the way the prototype does elsewhere.

**How to check a screen** (each phase's definition of done):
1. Run the prototype (`ims-ui-prototype`, :8765) and the app against a backend seeded with
   `scripts/seed.py --showcase` (backend B8 reproduces the prototype's demo data). In the prototype, turn the
   simulation off and reset the data.
2. For every screen and state in the phase, screenshot both at the same viewport and theme:
   - 1366×860 for everything;
   - also 768×1024 and 375×812 for sign-in, floor and till;
   - light and dark (the kitchen display is dark only).

   They must match: the same layout, elements in the same places, the same type, colour and spacing.
3. Run `app/core/service/style-probe.js` in both pages through the browser tools. It's a small script written in F1:
   for a fixed list of selectors per screen it prints `font-family`, `font-size`, `font-weight`, `font-stretch`,
   `line-height`, `color`, `background-color`, `padding`, `margin`, `gap`, `border`, `border-radius`, `width` and
   `height`. Every value must match, unless the difference is listed under "Deviations".
4. Attach the screenshot pairs to the phase's hand-off note.

Until the backend lands an endpoint, develop against the contract's shapes with typed fixtures behind the API
client. Build the fixtures from the prototype's demo data so screenshots line up. Remove every fixture before
finishing a phase. Nothing from the prototype's simulation ships.

## Ground rules

- **One app, signed in by account.** The role comes from the account, not from a role picker. `/app` is the single
  entry; `/staff` and `/admin` redirect to it. Full-profile roles that aren't service roles (membership staff and
  others) still land in today's `ManagementApp` after signing in, so the full product keeps working.
- **One shell per kind of work**, chosen by the role:

  | Layout | Roles | Device | Prototype component |
  |---|---|---|---|
  | `floor` | `waiter` | tablet / phone | `WaiterApp` |
  | `till` | `cashier` | till tablet | `CashierApp` |
  | `kitchen` | `cook` | wall or pass screen, always dark | `KitchenApp` |
  | `desk` | `inventory_lead`, `fnb_store_staff`, `manager` | laptop / desktop | `InventoryApp`, `StoreApp`, `ManagerApp` (`DeskShell`) |

  Add `layout?: "floor" | "till" | "kitchen" | "desk"` and `signIn?: "pin" | "password"` to `RoleDefinition`
  (`app/core/modules.ts`), and give the new roles in the module manifests. Roles without `layout` use `ManagementApp`.
- **Screens stay in the module that owns the data**: floor, till and kitchen screens in `app/modules/fnb_outlets`;
  inventory lead and store screens in `app/modules/fnb_inventory`; purchasing, sales and reports screens in
  `app/modules/finance` (reports that read inventory data stay in `fnb_inventory`, as today); devices and staff in
  `app/core`. Desk navigation is contributed per role through `roleNavigation`, as today, plus a new `deskGroups`
  field for the sidebar group labels ("Money", "Stock", "Reports", "People & setup").
- **Live data**: `useLive(loader, deps, topics)` (existing) with the contract's topics (`orders`, `kot`, `till`,
  `stock`, `approvals`, `devices`). No polling loops of our own.
- **Offline**: the till's existing queue keeps working; add `POST /cafe/orders/{id}/payments` and
  `POST /cafe/orders/{id}/fire` to `offline.posts`, and `/cafe/floor`, `/cafe/menu`, `/cafe/availability` to `cachedGets`.
- Both themes, phone width, keyboard focus and reduced motion, exactly as the prototype handles them.

## F1. Foundation — `app/core/service/`

- **Stylesheet** (`service.css`, loaded last in `app/layout.tsx`): the prototype's whole `<style>` block, ported as
  "Visual fidelity" §2 says. Fonts with `next/font/google`: Archivo and Martian Mono, both with `axes: ["wdth"]`,
  feeding the prototype's own `--f-sans` / `--f-mono` tokens.
- **Icons** (`icons.tsx`): the prototype's `P` path map and `Icon`, unchanged.
- **Components** (`ui.tsx`), one per prototype component with the same markup and class names: `Sheet`, `Drawer`,
  `Stepper`, `UTabs`, `Tog`, `Toasts` (with an action such as Undo), `RTable`, `BarChart`, `HourChart`, `HB`, `Level`,
  `Move`, and thin wrappers for the class-only pieces (`btn`, `chip`, `pill`, `seg`, `opt`, `kpi`, `panel`, `row`).
- **Fidelity tools:** `style-probe.js` for the side-by-side check, and a dev-only page `/app/_gallery` that renders
  every shared component in each state next to the same markup copied from the prototype.
- **Docs:** rewrite ARCHITECTURE.md's "Frontend styling" section to describe both systems: the legacy console for
  the full product's older screens, and `service.css` for the service roles. Point it to `prototype/README.md`.
- **Contract types** (`types.ts`) and API additions in `app/core/api-client.ts`:
  - send `X-Device-Key` when a device key is stored;
  - keep `code` from error bodies on `RumanzaApiError`;
  - on `401` with `code: "device_revoked"` → forget the device key and go to the start screen; on plain `401` →
    sign out to the right sign-in screen; on `403 code: "pin_change_required"` → open the PIN change.
- **Storage** (`session-storage.ts`): the session becomes `{token, user: Me, expires_at, kind: "password" | "pin" | "walkup"}`;
  the device as `{id, name, venue, key}` under its own key. A password session survives a restart; PIN and walk-up
  sessions are kept in `sessionStorage` so closing the browser ends them.

## F2. Sign-in — `app/core/service/sign-in/` (prototype: `Entry`, `AccountForm`, `SignIn`, `PinPad`)

- **Start screen** when there's no device and no session: "How is this computer used?" → **My own computer**
  (email + password, "Keep me signed in for 14 days" → `remember`) or **A shared device at a venue**.
- **Device set-up**: manager email + password (`/auth/login`) → venue and name → `POST /auth/devices` → store the
  key → `/auth/logout` for the manager token → PIN screen. Errors as in the prototype ("Only the F&B Manager can set
  up a device.").
- **PIN screen** on a set-up device: `GET /auth/device` lists the venue's staff grouped by role; the clock, date and
  device name on the dark side panel; tap a name → `PinPad` (keypad and keyboard digits) → `POST /auth/pin`. Show
  "Wrong PIN. N tries left." under the dots with the shake; on 423 show the locked state with the message.
  "F&B Manager or F&B Store? Sign in with email" opens the walk-up form (`/auth/login` with the device key).
  "Set this device up again" forgets the key after a confirmation sheet.
- **First PIN change**: when `pin_change_required`, a sheet asks for a new 4-digit PIN twice → `PUT /auth/my-pin`.
- **Idle sign-out** on shared devices: 5 minutes without pointer or key input → back to the PIN screen with the
  toast "Signed out after 5 minutes without use." Not for `cook`. Not for password sessions on own computers.
- **Session end**: at `expires_at` the shell signs out with "Your session ended. Sign in again."
- Desk shells show "Signed in until …" under the name for password sessions.

## F3. Floor — waiter (`fnb_outlets/screens/floor/`; prototype `WaiterApp`, `Floor`, `OrderScreen`, `ItemSheet`, `LineSheet`, `OpenOrders`, `Pickups`, `SeatSheet`)

- Shell: top bar (venue, service, clock, **Ready** pickups button with count, name / sign out), left rail on
  tablet, bottom bar on phones (Floor, Orders, Pickups).
- **Floor** from `GET /cafe/floor`: plan view (tables by `x/y/width/shape`, chairs, features) and list view
  (default under 700 px); legend counts; "All tables / Mine"; Food ready pulses; LATE flag; another waiter's table
  shows their initials. Free table → seat sheet (guests, Walk-in / Member with membership no. and name / Staff meal
  with cost centre) → `POST /cafe/orders` with empty `items` → order screen.
- **Order screen**: sections rail (chips under 1100 px), menu search with `/`, item tiles (name, description, price, station,
  "Options", "N left" / "Sold out" / "Off menu") from `/cafe/menu`, items needing options open the item sheet (required groups, add-ons with prices,
  kitchen-note chips incl. allergies, seat, quantity). The check keeps unsent lines on the device (a draft per order
  in storage) until **Send to kitchen** posts them with `seat_no` and `hold` → toast "Sent to BBQ and Pakistani ·
  KOT-…". Sent lines group by ticket with station status; held lines show "Held" and a **Fire** action
  (`/cafe/orders/{id}/fire`). Tapping a sent line offers **Void** with reasons; a 202 shows "Waiting for the F&B
  Manager" on the line. **Print bill** → confirm sheet → `POST /bill`; a printed bill locks the order with
  "The cashier takes payment and frees the table" and **Reopen order** (`DELETE /bill`, disabled once paid).
  Phone: "Review & send" bar opens the check full-screen.
- **Open orders**: cards per order with station progress pills; Mine / Everyone.
- **Pickups**: `GET /kot/tickets?status=Ready` for the venue, mine first, waiting timer (amber after 2 min, red after
  4); **Served to the table** → `PATCH status Served` with an Undo toast. Toast when one of my tickets turns Ready.

## F4. Kitchen display (`fnb_outlets/screens/kitchen-queue.tsx` rework; prototype `KitchenApp`, `Ticket`, `PassView`, `AllDay`)

- Always dark. Top bar: station name (or "The pass"), Open / Late / Oldest / Avg ticket figures, clock, All day,
  Recall, chime on/off, sign out.
- Station cook: their station only (from `Me.station_id`), no station tabs. Pass cook: "All stations" orders view
  plus a tab per station.
- Tickets as paper slips: header colour by age (under 8 min, 8–12 amber, 12+ red), table, KOT no., waiter, covers,
  ADDITIONAL / COOKING stamps, slips (VOID, VOID PENDING, VOID REJECTED, MOVED), lines with modifiers (heat and size in
  bold), notes, allergy notes in red; tap a line to cross it off (`PATCH .../lines/{id}`); **Start** / **Ready** with
  an Undo toast after Ready. Strict oldest-first order, no reshuffling.
- **All day** side panel (counts of open lines, by modifier). **Recall** panel: recently bumped tickets → Bring back
  (`Ready→Preparing`).
- Pass view: one card per order with a row per station; paper-only stations get Start / Ready there; **Send out**
  when every station is Ready.
- Chime on new Placed tickets (Web Audio, after the first tap on the screen). Keep the screen awake
  (`navigator.wakeLock`, ignore refusal).

## F5. Till — cashier (`fnb_outlets/screens/till/`; prototype `CashierApp`, `Settle`, `DiscountSheet`, `Receipt`, `CashPayments`, `CloseRegister`)

- Shell like the floor's (rail: Bills, Payments, Close register).
- **Bills**: `GET /cafe/bills`; waiting (oldest print first, waiting timer) and still dining; toast when a waiter
  prints a bill (topic `till`).
- **Settle**: the itemised bill and totals beside a payment panel: "Balance due", tender (Cash / Card / Bank
  transfer), amount (split 2/3/4 ways helpers), cash handed over with quick amounts and the change line, card
  approval code (optional), transfer reference (required), payments taken with remove (`DELETE` while unpaid),
  **Add discount** (percent or amount, reason chips; within the limit applies, above it shows "Ask Ali to approve"
  and then the pending notice until `approvals` resolves), **Print bill** when not printed, **Close as staff meal**
  for zero bills (`charge-department`). When the balance reaches zero: the receipt sheet (thermal-receipt look) with
  Print again and Next bill.
- **Payments**: this shift's totals by tender and the list.
- **Close register**: blind count by note (Rs 5,000 … 10) and coins; card and transfer totals; the variance appears
  once counting starts; note required at Rs 100+; previous closes.

## F6. Venue stock — inventory lead (`fnb_inventory/screens/venue/`; prototype `InventoryApp` and its screens)

- Desk shell with the location locked ("Your location · Rooftop Kitchen").
- **Overview** (`/fnb/overview` and `/fnb/forecast`): delivery-on-its-way banner → Receive; Stock on hand, Below par,
  Waste 7 days, Next stock count; "Won't last until the next delivery" with Request these; ingredients used, last 14
  days, with today live; recent movements.
- **Stock**: search, storage-area chips, Below par; level bars against par; row → drawer (`enquiry`): on hand, par,
  days of cover, movements; Request more / Log waste; multi-select → request.
- **Requests**: list (Open / Received / All) and status pills; composer suggested from par, rounded to pack size,
  needed-by choice, note → `POST /fnb/kitchen-requests`; drawer with the four-step timeline, managers' notes, trimmed
  quantities, and **Receive** (received quantity per line, reason chips when short) → `/receive`.
- **Stock count**: start, area tabs with progress, blind by default (Show expected), steppers, review sheet with the
  variance, submit → existing counts endpoints.
- **Waste**: quick log (ingredient search, quantity, reason chips, note, cost preview), by reason, the week's log.

## F7. F&B Store (`fnb_inventory/screens/store/`; prototype `StoreApp` and its screens)

- **Overview**: delivery-due banner, Store value / To dispatch / Deliveries due / Below reorder, Pick and send,
  Use first (`/fnb/batches/expiring`).
- **Dispatch**: ready to pick, waiting for the F&B Manager, sent recently; pick-list drawer with tick-off rows and
  send quantities capped at stock; Send.
- **Receiving**: expected deliveries and goods received; receive drawer per line (accepted, rejected with reason,
  unit price with the price-change flag, use-by date for perishables) → existing goods-received endpoint.
- **Purchase requests**: list; composer grouped by supplier from reorder levels; one request per supplier.
- **Store stock**: levels (read-only), average cost, batch and use-by.

## F8. F&B Manager (prototype `ManagerApp` and the `Mgr*` screens)

- Sidebar: Overview · Approvals | Money: Sales, Purchasing | Stock: Stock, Costing | Reports | People & setup:
  Team, Setup. Counts on Approvals and Purchasing.
- **Overview**: Net sales today, Waiting for you, Food cost 7 days, Stock below par; Waiting for you (top five, Review
  opens the approval drawer); Venues right now (`/cafe/venues/live`); Sales by hour; Deliveries and bills.
- **Approvals**: one list merged from the three sources in the contract, tabs Everything / Kitchen requests /
  Purchases / Till, oldest first; drawer per kind (kitchen: approve quantities against store stock; purchase: lines,
  value, the second-approver note; till: details); Reject needs a note; Decided recently.
- **Sales**: Daily sales (Z-report) by venue, by hour, how guests paid, register closes; Transactions; Discounts &
  comps; Voids & refunds.
- **Purchasing**: Invoices to pay (check against order and receipt; Approve for payment, **Pay what arrived**
  (`settle-received`), Dispute, Record payment), Purchase orders (drawer, cancel), Goods received, Payment history,
  Credit & debit notes, Suppliers (owed, open orders, on time, delivered in full); New purchase order sheet.
- **Stock**: all locations matrix with below-par highlighting and the par/reorder drawer; Waste across kitchens.
- **Costing**: food cost per venue against an editable target; menu engineering (popularity against margin chart
  with labels that never overlap, and the table with what to do).
- **Reports**: the hub with the left list (Dashboard; COGS and gross profit, Waste report; Price report, Price change
  report; Orders by supplier / item / category, Open orders; Invoices by item, Purchases by item / supplier, Supplier
  by week; Store summary, Transfers; Exceptions, Audit log), period filter (This week / 4 weeks / 8 weeks) where it
  applies, filter box, sortable columns, Export (existing `.xls` endpoints through `apiDownload`). No Production events.
  Reuse the existing report screens' data code where it fits, restyled.
- **Team**: Staff & access (shared devices with Switch off, who-sees-what matrix, people with how they sign in and an
  on/off toggle; Add person: name, role cards, venue, station for cooks, work email + password for office roles,
  starting PIN for venue roles); Rosters & attendance (week grid, shift sheet, confirm day).
- **Setup**: Menu & prices (price, cost %, on the menu), Kitchen stations (mode, section → station), Till & approval
  rules (member discount, checkout discount limit, F&B tax, staff meals need approval, second approver above, register
  float).

Each of F2–F8 ends with the side-by-side check of its screens ("How to check a screen" above). The states to check
are those in the phase's bullets, plus every sheet and drawer open, every empty state, and each list's loading and
error state.

## F9. Checks

- **Visual sign-off:** a final pass of every screen and state against the prototype, at all three widths and both
  themes, with the style probe clean. The screenshot pairs for every screen go in the hand-off note.

- `npm run typecheck`, `npm run lint`, `npm test` for the full and pilot builds. Extend `tests/modules.test.mjs` so it
  checks each new role's navigation and that pilot builds don't offer the hidden roles.
- Browser check per role against a local backend (`.claude/launch.json`), in both themes and at phone width for
  the floor and till:
  - set up a device, sign in with a PIN, wrong PIN count, lock, unlock from Team, switch off a device;
  - own-computer sign-in with and without "keep me signed in";
  - seat → order with seats, options and a held dish → send → kitchen Start/Ready → pickup → print bill → split
    payment with change → receipt → table free;
  - discount above the limit → approval → applied on the till;
  - void after cooking → approval and slips;
  - kitchen request → approve with a trim → dispatch → receive short;
  - purchase request above the threshold → two managers → PO → receive with a reject → invoice check → pay what arrived;
  - register close with a variance and note.
- Remove `/staff` and `/admin` role pickers from the pilot build once `/app` covers everything.

## Order of work

F1 → F2 (needs backend B1–B2) → F3, F4, F5 in parallel (B3–B5) → F6, F7 (B6) → F8 (B7) → F9.
Rough size: F1 3 days, F2 3, F3 4, F4 2, F5 3, F6 3, F7 2, F8 6, F9 2 — about 28 developer days.

## Deviations

Intentional differences from the prototype, each with its reason (see "Visual fidelity" §11).

- F2 accessibility: the PIN roster tablist has named tabs, `aria-selected`, one Tab stop, and ArrowLeft/ArrowRight/Home/End navigation. The prototype supplies the tablist but omits its tab semantics and keyboard navigation. Classes, wording and visual styles are unchanged.

- F2 additional auth states: on 2026-10-13 the owner approved the layouts and wording shown in `frontend/docs/service-f2/auth-states.html` (wrong PIN, locked device, first-PIN change and its server error, and device-reset confirmation at all required sizes/themes). These required behaviors have no matching rendered states in the frozen prototype; the implementation reuses its Sheet, auth-form, error and shake pieces. Owner response: Approve shown states. The prototype and API contract remain unchanged.

- F3 additional required states: on 2026-10-13 the owner approved the layouts and wording in `frontend/docs/service-f3/required-states.html`: loading/error retry, persisted held dishes with Fire, offline queue feedback, and pending void approval (36 screenshots across all required sizes/themes). The frozen prototype has no rendered equivalents; the implementation reuses its existing boot/empty, group headers, line rows, pills, buttons and Sheet. Owner response: Approve shown F3 states. The prototype and API contract remain unchanged.

- F4 accessibility: the kitchen pass station tablist has named tabs, aria-selected, one Tab stop and ArrowLeft/ArrowRight/Home/End navigation. The prototype omits tab semantics and keyboard navigation. Classes, wording and visual styles are unchanged.

- F4 additional required states: on 2026-10-13 the owner approved the layouts and wording in frontend/docs/service-f4/required-states.html: loading/error retry and the reason sheet for accepting an unprinted paper ticket, including server rejection (five desktop dark screenshots). The frozen prototype has no rendered equivalents; the implementation reuses its boot/empty, Sheet, field, button and toast pieces. Owner response: Approve shown F4 states. The prototype and API contract remain unchanged.
- F5 accessibility: payment tender radios use one Tab stop and ArrowLeft/ArrowRight/ArrowUp/ArrowDown navigation, and amount/reference/discount/count fields have associated labels. Classes, wording and screen styles are unchanged.

- F5 additional required states: on 2026-10-13 the owner approved the layouts and wording in frontend/docs/service-f5/required-states.html: loading/error retry, rejected payment/discount/register requests preserving entries, receipt loading/retry after server settlement, saved offline payments and staff-meal approval waiting. Owner response: Approve shown F5 states. The implementation reuses boot/empty, Sheet, pills, fields, buttons and toasts. Final payment closes on the server per the contract, and browser receipt printing is used because no till-receipt print-job endpoint exists.

- F5 staff-meal totals: on 2026-10-13 the owner decided the frontend matches the backend instead of the backend adding `staff_meal_discount`. The till shows only the fields the contract returns (`subtotal`, `member_discount`, `checkout_discount`, `tax_amount`, `total`), so a staff meal no longer shows the prototype's "Staff meal" reduction row on the bill or the receipt. The frontend never assumes a staff meal is free and never reconstructs a reduction. The prototype and API contract remain unchanged.

- F3/F5 staff-meal department: on 2026-10-13 the owner decided the frontend matches the backend instead of the backend adding `cost_centre_code` to order summaries. The department is read only from full order details; when none is returned the label reads "Staff meal" without the " · department" suffix (prototype: "Staff meal · OPS"), and the till's "Close as staff meal" button drops the suffix the same way. No placeholder such as "—" or "null" is ever shown.

- F6 additional required states (pending owner approval; the layouts reuse the prototype's boot/empty, drawer and toast pieces, which have no rendered equivalents there): the lead's screen loading and "Try again" when the first load fails, the ingredient drawer's movements loading and "Try again", and the server's own message as a warning toast when a request, receipt, count or waste write is refused (the drawer, quantities and chosen reasons stay as entered). Checked by behaviour in `scripts/check-service-inventory-behavior.mjs`.

- F6 movement kinds (pending owner approval): the contract's `dispatch` and `transit_loss` movements are not in the prototype's movement map. They use the same row with the truck icon and "Dispatched" and the alert icon and "Lost in transit". The prototype's four kinds are unchanged.

- F6 wording from data: the prototype names the demo F&B Manager ("Ali Raza approves or trims quantities."). No endpoint gives an inventory lead the manager's name, so the line reads "The F&B Manager approves or trims quantities." and the request toast "REQ-0420 sent to the F&B Manager for approval." (the generic fallback already used on the till).

- F6 data the server does not send: an ingredient with no par shows "No par set" in place of the level bar; a request with no `needed_by` shows "—" (the prototype's REQ-0402 has a needed-by with no day, which is the same case). Needed-by wording is derived from the stored time: Today / Tomorrow / Yesterday, then "9:00 delivery", "before service" or "afternoon run" for 9:00, 11:00 and 15:00, else the clock time. The "Needed by" choices store today 11:00, tomorrow 9:00 and tomorrow 15:00.
