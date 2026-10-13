# Station KOT routing — plan

Each item on a till order goes to the kitchen station that cooks it (e.g. Rooftop → Pakistani, Chinese,
Bakery) and prints there as a KOT. Everything lives in `fnb_outlets`; module boundaries do not change,
apart from one small generic hook in core approvals (§5).

Implementation is split for two agents working in parallel: [API contract](KOT_API_CONTRACT.md) (shared),
[backend plan](KOT_BACKEND_PLAN.md), [frontend plan](KOT_FRONTEND_PLAN.md).

Scope: **till (POS) orders only.** Member-app and on-course orders keep today's whole-order venue queue.

## 1. Setup (Manager / Head staff, `outlets.setup`)

New "Kitchen stations" tab in POS setup.

| Thing | Detail |
|---|---|
| Stations | Per venue (Indoor Cafe, Rooftop, Bar). Add, rename, order, deactivate. Mode: **Paper + screen** or **Paper only**. One printer each (printers may be shared). |
| Section → station | For each venue, every menu section maps to one of that venue's stations. Unmapped sections are highlighted. |
| Item override | Any single product can be sent to a different station than its section's. |
| Printers | Name, IP, port (default 9100), paper 80 mm, active, last error. "Test print" button. |
| Print agents | Register the club PC → key shown once (stored hashed). Shows last check-in. |

Day one: the update creates **one default station per venue** ("Cafe Kitchen", "Rooftop Kitchen", "Bar")
with every existing section mapped to it, so tills keep working; the Manager then splits stations out.

### Tables (all `fnb_outlets`)
- `kitchen_stations(id, venue, name, mode, printer_id, sort_order, active, UNIQUE(venue,name))`
- `kitchen_station_sections(venue, section, station_id, PRIMARY KEY(venue,section))`
- `kitchen_station_products(product_id PRIMARY KEY, station_id)`
- `kot_printers(id, name, host, port, paper_mm, active, last_ok_at, last_error)`
- `kot_print_agents(id, name, key_hash, active, last_seen_at)`
- `kot_tickets(id, kot_no, order_id, station_id, kind, status, created_by, created_at, accepted_at, ready_at)`
  — `kind`: new | additional; `status`: Placed | Preparing | Ready | Served | Cancelled
- `kot_ticket_lines(ticket_id, order_line_id, quantity)`
- `kot_slips(id, kind, order_id, station_id, payload_json, created_by, created_at)`
  — `kind`: void_pending | void | void_rejected | moved | joined | changed
- `kot_print_jobs(id, ticket_id | slip_id, printer_id, payload_json, status, attempts, lease_until, printed_at, last_error)`
- `order_lines.station_id` — the station snapshot taken when the item is ordered.

## 2. Routing

- Station for an item = its product override, else its (venue, section) mapping.
- **Unmapped items block the order.** The till's menu carries each item's station, so unmapped items are
  greyed out ("Not assigned to a kitchen station") even when the till is offline. The API also refuses them
  (409, naming the section and venue).
- Remapping later does not move items already ordered (`order_lines.station_id` is kept).
- One helper, `fire_kots(order_id, kind)`, tickets every live line that isn't on a ticket yet, one ticket
  per station, and queues a print job per ticket. Called from:
  - order create (unless saved as Draft) → kind `new`;
  - Draft → Open/Placed → kind `new`;
  - add items → kind `additional` (printed **ADDITIONAL**, new items only, relevant stations only).
- KOT numbers come from core numbering (`kot` → `KOT-`).

## 3. Status per station

- Every ticket has the same two steps, **Accept** and **Mark ready**, whoever taps them:
  - **Paper + screen** stations tick their own tickets on their tab of the venue Kitchen Queue
    ("All · Pakistani · Chinese · Bakery").
  - **Paper only** tickets are ticked from the Kitchen Queue's **All** view (head chef / pass) or from the
    order card on the till.
- The order's status follows its tickets: the first accepted ticket → order **Preparing** (ingredients
  deducted from the venue kitchen exactly as today); every live ticket Ready → order **Ready** (existing
  notice sent). **Complete / Served** stays a single whole-order action.
- An add-on ticket may move a Ready order back to Preparing (a narrow exception to `check_transition`).
- The till's order card shows each station's progress and any print problem.

## 4. Slips (same print pipeline, retry and alerts as KOTs)

| Slip | Trigger | Goes to |
|---|---|---|
| **VOID PENDING APPROVAL** | Waiter asks to cancel a paid line/bill (approval required) | Stations holding the affected items |
| **VOID** | Unpaid line removed or cancelled, unpaid order cancelled, or a pending void approved | Same |
| **VOID REJECTED — continue** | The pending void is rejected, withdrawn or fails | Same |
| **TABLE MOVED** `K-1234 · R4 → R7` | Move table | Stations with unfinished tickets for the order |
| **JOINED** `K-1234 (R4) → K-5678 (R7) — serve to R7` | Join orders | Stations holding the merged-away order's tickets |
| **CHANGED** `1x Karahi — now: No onions, Extra spicy` | Modifiers/note edited after the KOT printed | That item's station |

- Comps print nothing (the item is still served, just free).
- Stations whose ticket is already Served get no slip.
- On screens, a voided item is struck through; a ticket left with nothing on it is Cancelled and no
  longer counts toward "all stations ready".
- On join, tickets are re-linked to the surviving order.

Hook points: `routes/orders.py` (create, add items, status), `routes/adjustments.py` (cancel order/line,
remove line, `apply_cancel`, `apply_line_cancel`), `routes/pos_tools.py` (`modify_line`, `move_table`,
`join_orders`).

## 5. Core change (generic)

Core approvals have no callback when a request ends without applying. Add an optional
`on_closed(payload, requester, outcome)` to `ApprovalAction`, called on Rejected / Withdrawn / Failed.
`fnb_outlets` uses it for **VOID REJECTED**. Core still imports no business module.

## 6. Print agent and printing

- `print_agent/` (new top-level folder): small Python program, run as a Windows service on a club PC on
  the LAN. Config: API URL and agent key only — printer IPs live in IMS.
- Loop: claim jobs (lease ~60 s, `FOR UPDATE SKIP LOCKED`) → send raw ESC/POS over TCP to `host:9100`
  → confirm printed, or report failure with the error. Heartbeat updates `last_seen_at`.
- Agent API (header `X-Agent-Key`, not a staff login): `GET /api/v1/kot/agent/jobs`,
  `POST /api/v1/kot/agent/jobs/{id}/printed`, `POST /api/v1/kot/agent/jobs/{id}/failed`.
- **Automatic retry with backoff; nothing is dropped.** An expired lease returns the job to the queue.
  Retries print "COPY" so a lost confirmation can't cause a dish to be cooked twice.
- **Alerts:** the till order card shows "KOT not printed — Rooftop Chinese" once a job has failed or waited
  ~30 s; the Manager dashboard gets an "Unprinted KOTs" attention list and an "agent offline" warning
  (no heartbeat for 60 s). New live topic `kot`.
- Render free tier: the agent's polling keeps the API awake (~744 of 750 free hours per month).

### KOT layout (80 mm, no prices)
Large **table number** at the top · venue · station · KOT number · NEW / ADDITIONAL (or COPY) ·
order code · pax · waiter · time · items: quantity, name, modifiers, line note · order preparation note.
Slips use the same header with the slip type in large text.

## 7. Tests

Backend (`tests/fnb_outlets`):
- override beats section mapping; unmapped items blocked;
- one ticket per station; add-on tickets carry only new lines; Draft fires on open;
- order status follows tickets; ingredients deducted once; Ready → Preparing on add-on;
- each slip kind fires on its trigger and to the right stations; void pending → approved / rejected;
- jobs claimed, re-queued on lease expiry, confirmed; wrong agent key rejected;
- join re-links tickets.

Agent: unit test for ESC/POS layout. Existing tests, `lint-imports`, frontend build must still pass.

## 8. Build order

1. Tables, default-station migration, setup API and screen.
2. Routing, tickets, `fire_kots`, till blocking.
3. Per-station status, Kitchen Queue tabs, till progress.
4. Slips and the core approval hook.
5. Print jobs, agent API, alerts, `print_agent/` Windows service.
