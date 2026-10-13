# KOT routing — API contract (shared)

The backend and frontend agents both build against this file. Product rules are in
[KOT_ROUTING_PLAN.md](KOT_ROUTING_PLAN.md). Work split: [backend](KOT_BACKEND_PLAN.md) · [frontend](KOT_FRONTEND_PLAN.md).

**Change rule:** neither agent changes a shape here on its own. If something has to change, edit this file
in the same change and note it under "Changelog" at the bottom so the other side picks it up.

All paths are under `/api/v1`. Errors keep the existing form `{"detail": "<message>"}` with a plain-text
message the till can show as a toast.

## Common types

```ts
type Venue = "Indoor Cafe" | "Rooftop" | "Bar";
type StationMode = "screen" | "paper";            // "Paper + screen" | "Paper only"
type TicketStatus = "Placed" | "Preparing" | "Ready" | "Served" | "Cancelled";
type TicketKind = "new" | "additional";
type SlipKind = "void_pending" | "void" | "void_rejected" | "moved" | "joined" | "changed";
type PrintStatus = "queued" | "printed" | "failed" | "none";  // "none": station has no printer

interface Station { id: number; venue: Venue; name: string; mode: StationMode; printer_id: number | null;
  printer_name: string | null; sort_order: number; active: boolean }
interface Printer { id: number; name: string; host: string; port: number; paper_mm: 80 | 58; active: boolean;
  last_ok_at: string | null; last_error: string | null }
interface Agent { id: number; name: string; active: boolean; last_seen_at: string | null; online: boolean }
interface SectionMapping { venue: Venue; section: string; station_id: number | null }   // null = unmapped
interface ProductOverride { product_id: number; product_name: string; venue: Venue; section: string; station_id: number }

interface Slip { id: number; kind: SlipKind; text: string; created_at: string; print_status: PrintStatus }

interface TicketLine { order_line_id: number; name: string; quantity: number; modifiers: string[];
  note: string | null; cancelled: boolean }

interface Ticket {
  id: number; kot_no: string; kind: TicketKind; status: TicketStatus;
  order_id: number; order_code: string; venue: Venue; table_code: string | null; pax: number | null;
  member_name: string; waiter_name: string | null; ready_by: string; preparation: string | null; notes: string | null;
  station_id: number; station_name: string; station_mode: StationMode;
  created_at: string; accepted_at: string | null; ready_at: string | null;
  lines: TicketLine[]; slips: Slip[];             // slips: newest first
  print_status: PrintStatus; print_error: string | null;
  print_stuck: boolean;                         // KOT failed or unprinted for 30 s+, or a live paper-only ticket whose station has no printer
}

interface OrderKot {                             // the summary carried on an order
  ticket_id: number; kot_no: string; station_id: number; station_name: string; station_mode: StationMode;
  kind: TicketKind; status: TicketStatus; print_status: PrintStatus; print_error: string | null; print_stuck: boolean;
}
interface PrintAlert { station_name: string; what: string; since: string; error: string | null }
```

## Setup (write: `outlets.setup`; read: `outlets.setup`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/kot/setup` | — | `KotSetup` |
| POST | `/kot/stations` | `{venue, name, mode, printer_id, sort_order, active}` | `KotSetup` (201) |
| PUT | `/kot/stations/{id}` | same as POST | `KotSetup` |
| PUT | `/kot/section-map` | `{venue, section, station_id}` (`station_id` null = unmap) | `KotSetup` |
| PUT | `/kot/product-overrides/{product_id}` | `{station_id}` (null removes the override) | `KotSetup` |
| POST | `/kot/printers` | `{name, host, port, paper_mm, active}` | `KotSetup` (201) |
| PUT | `/kot/printers/{id}` | same as POST | `KotSetup` |
| POST | `/kot/printers/{id}/test` | — | `{queued: true}` (202) |
| POST | `/kot/agents` | `{name}` | `{agent: Agent, key: string}` (201) — `key` is only ever returned here |
| PUT | `/kot/agents/{id}` | `{active}` | `KotSetup` |

```ts
interface KotSetup {
  venues: Venue[];
  stations: Station[];
  sections: Record<Venue, string[]>;      // every section on that venue's menu
  section_map: SectionMapping[];          // one row per (venue, section) in `sections`
  product_overrides: ProductOverride[];
  printers: Printer[];
  agents: Agent[];
}
```

Validation (409/422 with message): station name unique per venue; a section or override may only point at
a station of the same venue; the product must be on that venue's menu; deactivating a station that still has
sections/overrides mapped is refused ("Move its sections first").

## Routing for the till (`outlets.serve`) — cached for offline use

| Method | Path | Returns |
|---|---|---|
| GET | `/kot/routing?venue=Rooftop` | `{venue, stations: {id, name, mode}[], items: Record<string, number \| null>}` |

`items` maps every product id on that venue's menu to its resolved station (override, else section map), or
`null` when it has none. Only active stations are listed.

## Orders (existing endpoints, extended)

- `POST /cafe/orders` and `POST /cafe/orders/{id}/items` refuse any item without a station:
  **409** `"No kitchen station for <item> (section <section> at <venue>). Ask the Manager to map it."`
- `GET /cafe/orders`, `GET /cafe/orders/{id}` and every endpoint returning an order now include
  - `kot: OrderKot[]` (empty for Draft orders and for orders placed before this feature),
  - `print_alerts: PrintAlert[]` (KOTs/slips failed or unprinted for 30 s+, plus live paper-only tickets that never got a print job because the station has no printer; `ticket_id` is null for a slip),
  - and on each item: `station_id: number | null`, `station_name: string | null`.
- `PATCH /cafe/orders/{id}/status`, for an order that has tickets:
  - `Preparing` / `Ready` → **409** `"Update the station tickets instead"` (order status follows tickets);
  - `Served` → **409** `"Still preparing: <station>, <station>"` unless every live ticket is Ready.
  Orders without tickets keep today's behaviour.
- Cancel, remove line, cancel line, move table, join and modify line keep their request/response shapes;
  their slips are printed server-side and appear in `Ticket.slips`.

## Tickets (`outlets.serve`)

| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/kot/tickets?venue=Rooftop[&station_id=3][&include_done=false]` | — | `Ticket[]` |
| PATCH | `/kot/tickets/{id}/status` | `{status: "Preparing" \| "Ready", reason?: string}` | `Ticket` |
| POST | `/kot/tickets/{id}/reprint` | — | `Ticket` |

- Default list: tickets in Placed/Preparing/Ready for orders that are not Served/Cancelled, oldest first.
- Steps go Placed → Preparing → Ready only; anything else is **409**.
- The backend does not restrict which screen ticks a ticket; the frontend decides where buttons appear.
- **Accepting by hand:** for a paper-only station whose KOT has not printed (`print_stuck`), `Preparing`
  needs a `reason` (3+ characters) or it is **409** `"The KOT has not printed. Say why you are accepting this ticket by hand."`.
  Screen stations are unaffected. The audit entry `kot.ticket_status_changed` carries `print_override: true` and the reason.
- **Reprint** (Placed/Preparing tickets only; otherwise **409**): a job that never printed is refreshed (current
  lines, current station printer) and retried at once; a job that did print goes out again marked COPY. **409** if
  the station has no printer, its printer is switched off, every item is cancelled, or an agent is sending the job
  right now. Audited as `kot.ticket_reprinted`. The ticket's own status is untouched.

## Alerts (`outlets.serve`)

| Method | Path | Returns |
|---|---|---|
| GET | `/kot/alerts` | `{unprinted: {ticket_id, order_code, table_code, venue, station_name, what, since, error}[], agents_offline: Agent[]}` |

The Manager dashboard also gets a core attention list, `kot_unprinted`, from the backend; the frontend
needs no work for that one.

## Live events

New live topic **`kot`**, announced on any ticket, slip, print-job or setup change. Order status changes
keep announcing **`orders`**.

## Changelog
- (empty)
