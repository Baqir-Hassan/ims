# KOT routing — frontend plan

Owner: **frontend agent.** Touches only `frontend/`. Does not edit `backend/` or `print_agent/`.
Build against [KOT_API_CONTRACT.md](KOT_API_CONTRACT.md); product rules in [KOT_ROUTING_PLAN.md](KOT_ROUTING_PLAN.md).

Until the backend lands an endpoint, develop against the contract shapes (typed fixtures or a temporary
mock). Remove any mock before finishing. All work is in `frontend/app/modules/fnb_outlets/`.

## F1. API client and types — `api.ts`, new `screens/kot-types.ts`
- Add the contract's types (`Station`, `Ticket`, `OrderKot`, `KotSetup`, ...) to `kot-types.ts`.
- Add to `outletApi`: `kotSetup`, `saveStation`, `mapSection`, `setProductOverride`, `savePrinter`,
  `testPrinter`, `createAgent`, `setAgentActive`, `kotRouting(venue)`, `kotTickets(venue, stationId?)`,
  `ticketStatus(id, status)`, `kotAlerts`.

## F2. Manifest — `manifest.ts`, `dashboard.ts`
- New screen `kitchen-stations` ("Kitchen Stations", "Stations, section mapping, printers and print
  agents.", `Printer` icon). Add it to navigation, the Food & Beverage sidebar and `roleNavigation.manager`
  only.
- Add `/^\/kot\/routing(?:\?|$)/` to `offline.cachedGets` so the till still knows unmapped items offline.
- Kitchen nav counts: count open tickets from `/kot/tickets?venue=…`, plus member-app orders as today;
  topics `["orders","kot"]`.
- `kitchenOverview`: put `/kot/alerts` rows into `attention` (unprinted KOTs, agent offline) and add `"kot"`
  to its topics.

## F3. Kitchen Stations screen — new `screens/kitchen-stations.tsx`, export from `screens/workspaces.tsx`
Venue picker (Indoor Cafe / Rooftop / Bar), then four panels in the style of `PosSetupScreen`:
1. **Stations:** table with name, mode (Paper + screen / Paper only), printer, order, active; add and edit
   forms. Show the backend's 409 message as a toast (e.g. "Move its sections first").
2. **Section mapping:** every section in `sections[venue]` with a station dropdown. Highlight unmapped rows
   with a count banner: "3 sections unmapped — those items can't be ordered".
3. **Item overrides:** search that venue's menu products (catalog products filtered by the venue's menu
   category) → choose a station; list existing overrides with a remove action.
4. **Printers & agents:**
   - Printers: name, IP, port, 80/58 mm, active, last OK / last error, and a "Test print" button.
   - Agents: "Add agent" opens a one-time dialog showing the key with a copy button and the warning "You
     won't see this key again". Show last seen, an online dot, and an active toggle.
- Refresh with `useLive(..., "kot")`. Read-only for anyone without setup rights (match how `PosSetupScreen`
  uses `canEdit`).

## F4. Till — `screens/order-builder.tsx`
- Load `kotRouting(venue)` alongside products in `useLive` (topics add `"kot"`).
- Menu tile for an item with `items[id] === null`: disabled, labelled "Not assigned to a kitchen station".
  `saveOrder` refuses carts containing such items (defensive; the backend also returns 409).
- Order list and order detail:
  - **station progress chips** from `order.kot` (e.g. "Pakistani ✓ Ready", "Chinese · Preparing",
    "Bakery · Placed");
  - **print alert banner** from `order.print_alerts`: "KOT not printed — Rooftop Chinese (printer offline)", with
    **Reprint** and, for a paper-only station, **Accept by hand** / **Mark ready** right beside it;
  - **paper-only tickets** (`station_mode === "paper"`) get **Accept** / **Mark ready** buttons that call
    `ticketStatus`;
  - "Complete" stays the existing order status call, enabled only when every live ticket is Ready.
- Orders with an empty `kot` (placed before this feature) keep today's display.

## F5. Kitchen Queue — `screens/kitchen-queue.tsx`, `screens/shared.ts`
- Tabs: **All** + one per active station of the venue (from `kotRouting(venue).stations`). Remember the
  chosen tab per device in `localStorage`, so a station tablet reopens on its own tab.
- **Station tab:** ticket cards from `kotTickets(venue, stationId)`:
  - header: large table number, KOT no, ADDITIONAL badge, urgency (reuse `orderUrgency`);
  - lines with modifiers and notes; cancelled lines struck through;
  - slip banners from `ticket.slips`: VOID PENDING (amber), VOID (red), VOID REJECTED — continue (green),
    TABLE MOVED, JOINED, CHANGED;
  - **Accept** / **Mark ready** on every ticket, paper-only stations included. When the KOT has not printed the
    amber banner carries **Reprint** and, for a paper-only station, the hand tick-off instead (Accept asks why).
- **All tab:** tickets grouped by order — one card per order with a row per station ticket.
  - Paper-only rows get Accept / Mark ready (this is the pass).
  - Screen-station rows show status only.
  - **Complete** appears on the order when every ticket is Ready (existing order status `Served`).
  - Member-app orders keep their current whole-order card in this tab, unchanged.
- Show print problems on the ticket (`print_stuck`, or a `queued` ticket older than 30 s) as an amber banner with Reprint.
- Live topics `["orders","kot"]`.

## F6. POS order tools — `screens/pos-tools.tsx`
- No new calls (slips are printed server-side). Change toasts for move/join/modify/cancel to mention the
  kitchen, e.g. "Table changed — kitchen notified".

## F7. Checks
- `npm run typecheck`, `npm run lint`, `npm test` pass. Extend `tests/modules.test.mjs` if it lists screen
  ids, so `kitchen-stations` is covered.
- Verify in the browser preview (`.claude/launch.json`) against a local backend once B1–B4 have landed:
  - set up a station, then map sections;
  - an unmapped item is blocked;
  - an order splits into station tickets;
  - per-station Accept and Ready work;
  - paper-only tickets can be ticked from the till;
  - slips appear on tickets after move, join, cancel and modify.
