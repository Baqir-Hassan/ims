# KOT routing — backend plan

Owner: **backend agent.** Touches only `backend/`, the new top-level `print_agent/`, and backend docs
(`API_COVERAGE.md`, `ARCHITECTURE.md`). Does not edit `frontend/`.
Build against [KOT_API_CONTRACT.md](KOT_API_CONTRACT.md); product rules in [KOT_ROUTING_PLAN.md](KOT_ROUTING_PLAN.md).

Ship the phases in order. **B1–B4 make the contract usable by the frontend**, so land those first.

## B1. Schema and day-one stations — `modules/fnb_outlets/schema.py`
- Add the tables in KOT_ROUTING_PLAN §1 (`kitchen_stations`, `kitchen_station_sections`,
  `kitchen_station_products`, `kot_printers`, `kot_print_agents`, `kot_tickets`, `kot_ticket_lines`,
  `kot_slips`, `kot_print_jobs`) and `order_lines.station_id`, using the existing idempotent
  `CREATE ... IF NOT EXISTS` / `ADD COLUMN IF NOT EXISTS` style.
- In `ensure()`: for each venue **with no stations yet**, create the default station ("Cafe Kitchen",
  "Rooftop Kitchen", "Bar"; mode `screen`; no printer) and map every distinct section on that venue's menu
  (`Cafe Menu` / `Rooftop Menu` / `Bar Menu`) to it. Running it again must change nothing.
- Register doc numbering `kot` → `KOT-` in the manifest.

## B2. Setup and routing endpoints — new `routes/kot_setup.py`, `services/kot_routing.py`
- All setup endpoints in the contract. Writes need `outlets.setup`, return the full `KotSetup`, and audit
  (`kot.station_saved`, `kot.section_mapped`, `kot.override_saved`, `kot.printer_saved`, `kot.agent_created`, ...).
- Agent keys: `secrets.token_urlsafe(32)`, store only a hash, return the key once.
- `resolve_station(database, venue, product)`: override → (venue, section) → `None`. One function used by
  routing, order create and add-items.
- `GET /kot/routing` (`outlets.serve`).
- Announce live topic `kot` on every change; add `"kot"` to the manifest's `live_topics`.

## B3. Routing orders into tickets — `routes/orders.py`, new `services/kot.py`
- `_price_item`: resolve the station; raise the contract's 409 when `None`; return `station_id`.
  `insert_line` stores it on `order_lines.station_id`.
- `fire_kots(database, order_id, kind, actor_id)`: lines that are live and not yet on a ticket → one ticket
  per station (KOT number from core numbering) → queue a print job per ticket (B6). Call from:
  - `create_cafe_order` unless `save_as == "Draft"` → `new`;
  - `update_order_status` when leaving Draft → `new`;
  - `add_order_items` → `additional`.
- Offline replays (`client_ref` already seen) must not fire again.
- `expanded_order` and `list_orders`: add `kot`, `print_alerts` and each item's `station_id` / `station_name`.

## B4. Per-station status — `services/kot.py`, new `routes/kot_tickets.py`, `services/stock.py`
- `GET /kot/tickets`, `PATCH /kot/tickets/{id}/status` per the contract.
- `sync_order_status(database, order_id, actor_id)` after every ticket change:
  - any live ticket Preparing/Ready → order **Preparing** (call `consume_order_ingredients` as today);
  - all live tickets Ready → order **Ready** (send the existing `queue_notice`);
  - an add-on ticket on a Ready order → back to **Preparing**. Do this through a narrow exception, not by
    loosening `check_transition` for everyone.
- `update_order_status` for orders with tickets: refuse Preparing/Ready and refuse Served until all
  tickets are Ready (contract messages). Orders without tickets behave exactly as today.
- Line cancelled or removed → its `kot_ticket_lines` row shows `cancelled`; a ticket with no live lines
  becomes `Cancelled` and drops out of the "all ready" check. Order cancelled → its tickets `Cancelled`.
- Order Served → its tickets `Served`.

## B5. Slips — `services/kot.py`, `routes/adjustments.py`, `routes/pos_tools.py`, core approvals
Each slip is a `kot_slips` row plus a print job, sent only to stations holding affected items, skipping
tickets already Served. Comps print nothing.

| Slip | Hook |
|---|---|
| `void` | `remove_order_line`, `cancel_order_line_now`, `force_cancel_order`, `update_order_status` → Cancelled |
| `void_pending` | `cancel_order` and `force_cancel_order_line` when `approvals.gate(...)` returns a pending request |
| `void` (approved) | `apply_cancel`, `apply_line_cancel` — the normal void path |
| `void_rejected` | the new core hook below, on Rejected / Withdrawn / Failed |
| `moved` | `move_table` (stations with unfinished tickets) |
| `joined` | `join_orders` — also re-point the merged order's `kot_tickets.order_id` to the surviving order |
| `changed` | `modify_line`, only if the line is already on a ticket |

**Core hook (generic, no module imports):** add optional
`on_closed: Callable[[dict, dict, str], None] | None = None` to `ApprovalAction` in `core/registry.py`, and
call it from `core/services/approvals.py` where a request becomes Rejected, Withdrawn or Failed (including
scheduled failures). Register `on_closed` for `pos_void` and `pos_line_void` in the outlets manifest.
Update ARCHITECTURE.md (E6).

## B6. Print jobs, agent API, alerts — `services/kot_print.py`, new `routes/kot_agent.py`
- A job's `payload_json` is **structured**, not pre-rendered; the agent renders ESC/POS:
  `{type: "kot"|"slip"|"test", title, copy: bool, venue, station, table_code, kot_no, kind, order_code, pax,
  waiter, time, lines: [{quantity, name, modifiers[], note, cancelled}], preparation, notes, slip_text,
  paper_mm, printer: {host, port}}`.
- A station with no printer: no job is created (`print_status: "none"`). A paper-only station is the exception to "nothing to report": its live tickets count as unprinted (`print_stuck`, `print_alerts`, `/kot/alerts`) because the slip is the only way the kitchen sees them.
- Agent auth: dependency reading `X-Agent-Key`, comparing it against stored hashes, refusing inactive agents,
  and updating `last_seen_at`. Not a staff token.
- `GET /kot/agent/jobs?limit=10` claims queued jobs (or jobs whose lease has expired) with
  `FOR UPDATE SKIP LOCKED`, sets `lease_until = now + 60s`, and increments `attempts`.
- `POST /kot/agent/jobs/{id}/printed` → printed; update the printer's `last_ok_at`; announce `kot`.
- `POST /kot/agent/jobs/{id}/failed {stage: "connect"|"send", error}` → back to queued; update the
  printer's `last_error`.
- **COPY flag:** set `copy=true` on the next claim only if a previous attempt might have printed
  (lease expired after a claim, or a `send`-stage failure). A `connect` failure prints nothing, so no COPY.
- `GET /kot/alerts`: jobs unprinted for 30 s+ or with an error, and agents with no heartbeat for 60 s.
  Register core attention list `kot_unprinted` with the same rows.
- `POST /kot/printers/{id}/test` queues a `test` job.

## B7. Print agent — `print_agent/` (new top-level folder)
- Python 3.12, cross-platform code, deployed as a Windows service. Dependencies: `requests` only; ESC/POS
  is written as raw bytes, without a printer library.
- `config.toml`: `api_url`, `agent_key`, `poll_seconds` (default 2). Commit only `config.example.toml`.
- Own `print_agent/.gitignore` (the root one no longer covers Python): `config.toml`, `__pycache__/`,
  `*.py[cod]`, `.venv/`, `.pytest_cache/`, `logs/`.
- Loop: claim → for each job, open a TCP connection to `printer.host:port` (5 s timeout) → send → report.
  Exponential backoff per printer (2 s → 60 s) while it fails; other printers keep printing.
- Layout per KOT_ROUTING_PLAN §6, 48 columns at 80 mm / 32 at 58 mm: large table number, then bold slip
  title, items, notes, and a cut.
- `README.md`: install Python, configure, install as a service with NSSM (`nssm install ims-print-agent ...`),
  read the logs, use the test print button.
- Tests: rendering (80/58 mm, COPY, ADDITIONAL, each slip kind) and a fake TCP printer for
  connect/send failures.

## B8. Tests — `backend/tests/fnb_outlets/test_kot.py` (use `tests/fnb_helpers.py`)
- Default stations created once, all sections mapped; a second `ensure` changes nothing.
- An override beats the section mapping; an unmapped item gets 409 on create and on add-items.
- One ticket per station; add-on carries only new lines; Draft fires on open; offline replay fires nothing.
- Order status follows tickets; ingredients deducted once; Ready → Preparing on add-on; Served refused until all Ready.
- Each slip fires on its trigger and to the right stations; void pending → approved / rejected / withdrawn.
- Jobs are claimed, re-queued after the lease, confirmed; the COPY rule holds; a wrong or inactive key gets 401.
- Join re-points tickets; `GET /kot/alerts` reports stale jobs and offline agents.
- Keep `lint-imports`, the full `pytest` suite and `tests/release` green; add the new routes to `API_COVERAGE.md`.
