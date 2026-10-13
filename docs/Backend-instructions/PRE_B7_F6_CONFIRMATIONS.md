# Backend agent: F6 data confirmations (add to the pre-B7 work)

Read this together with the pre-B7 instructions. The frontend agent is building the inventory lead's screens (F6)
now, from `prototype/index.html`. The rule in `docs/SERVICE_FRONTEND_PLAN.md` is: the contract carries every field
the prototype shows, and if a screen needs data the contract lacks, the contract is extended instead of dropping
or changing anything on screen. The B6 report does not let the frontend confirm the items below, because
section 6 says the request, count and waste endpoints "keep their shapes" without listing them.

**What to do:** for each item, check the real response/permission in the source, write the answer in your answer
file (exact field names, types, who may call it). If something is simply **missing and the fix is additive**
(a new optional field, or a permission the lead should have for its own location), add it, update
`docs/SERVICE_API_CONTRACT.md` §6 with a dated Changelog line, add a test in `tests/fnb_inventory/test_service_stock.py`
in both profiles, and say so. If the fix is **not** additive, or needs a product decision, do not do it: report it.
Everything stays confined to the lead's own kitchen location, as B6 already enforces.

## 1. Stock count: can the lead submit it? (the biggest one)
Prototype: the lead walks the stores, enters counted quantities, taps "Review & submit", and stock is set to what
was counted with the difference recorded as a count adjustment. In the existing frontend only
`manager`, `fnb_manager` and `finance_staff` may call `POST /inventory/fnb/counts/{id}/post`; counting
(`PUT .../counts`) is open to more roles.
- May `inventory_lead` call `POST /inventory/fnb/counts` (start), `PUT /inventory/fnb/counts/{id}/counts` (save)
  and `POST /inventory/fnb/counts/{id}/post` (submit) for its own location? If not, allow it for its own
  location (the same pattern as `fnb.receive`), refusing another location with the contract's 403 text style.
- A count list row and a count detail need, for the lead: `number`, `status`, `started_at`, `started_by_name`,
  `posted_at`, `posted_by_name`, `items` (lines), `counted`, `variance_value` (Rs, signed), and per line
  `ingredient_id`, `name`, `unit`, `storage_area`, `system_qty`, `counted_qty`, `variance`. The screen shows
  "Previous counts": date, items, who counted, variance in Rs.
- Is there at most one count in `Counting` per location? Starting a second one: what is returned?

## 2. Waste: cost per entry, and the note
Prototype Waste screen: a cost preview while typing, a log with a **Cost** column and a **By** column, and a
"By reason" bar chart in Rs.
- Does `GET /fnb/waste?days=7` (the shared `/kitchen/waste` handler) return, for the lead, per-entry `cost`
  (the monetary impact), `created_at`, `item_name`, `ingredient_id`, `quantity`, `unit`, `reason`, `note`,
  `logged_by_name`, `venue`? The existing kitchen screen hides costs for `kitchen_staff` and appears to join
  reason and note as `"Reason: note"` in one string. The frontend needs `reason` and `note` as **separate**
  fields and `cost` for `inventory_lead` (own location only).
- `GET /fnb/waste/options`: confirm the exact shape (`ingredients[]` with `id`, `name`, `unit`, `stock_qty`,
  `par_level`, `needs_restock`; `reasons[]`). For the cost preview it also needs the ingredient's average cost per
  unit (`avg_cost`) for the lead.
- The prototype's reasons are: Spoiled, Expired, Over-prep, Dropped or spilled, Returned by guest, Burnt or
  cooking error. Are these the values `reasons[]` returns? If the server has a different list, send the list as
  it is and say what it is; the frontend will use the server's list and the owner will decide about wording.
- `POST /fnb/waste` body: confirm `{ingredient_id, quantity, reason, note?, venue?}` and whether an inventory
  lead may omit `venue`. What error is returned for a quantity above what is on hand?

## 3. Stock list and the ingredient drawer
Prototype Stock table: ingredient, storage area, on hand, par, level bar, **Avg cost**, value, last movement.
Drawer: on hand, par, **Cover (days)**, **Average use a day**, store pack size, value on hand, "to reach par", and
the movements list.
- Do the lead's scoped enquiry rows include `ingredient_id`, `name`, `unit`, `storage_area`, `pack_size`, `par`,
  `reorder_level`, `on_hand`, `value`, `avg_cost` (also when `on_hand` is 0) and `daily_use` (the same 14-day
  average the forecast uses, for **every** ingredient, not only the at-risk ones from `/fnb/forecast`)?
  If `avg_cost` or `daily_use` are missing, add them as optional numbers.
- `GET /inventory/fnb/enquiry/{ingredient_id}` for the lead: confirm `movements[]` rows carry
  `created_at`, `kind` (the full set, including `dispatch` and `transit_loss`), `change_qty`, `reference` (the
  request number, order code or reason: not only a free `note`), and `by_name`. The prototype shows the reference
  on every movement line.

## 4. Kitchen requests (list, drawer, create, receive)
Prototype Requests list: number, status pill, needed by, item count, **value**, last update. Drawer: a four-step
timeline (Requested, Approved, Dispatched, Received) with who and when, the F&B Manager's note, per-line
asked / approved (trimmed) / dispatched / received quantities with the short-receipt reason.
- Confirm the full set of `status` values and which are terminal (`Requested`, `Approved`, `Dispatched`,
  `Received`, `Received with variance`, `Rejected`, `Withdrawn`?), and whether `GET /fnb/kitchen-requests` for a
  lead needs the `kitchen` parameter or defaults to its own location.
- Request fields needed: `id`, `number`, `kitchen`, `status`, `needed_by` (ISO, echoed back), `remarks`,
  `requested_by_name`, `requested_at`, `decided_by_name`, `decided_at`, `decision_note`, `dispatched_by_name`,
  `dispatched_at`, `dispatch_note` (the storekeeper's note when sending, e.g. "Chicken boneless short by 0.5 kg at the store"; the
  drawer shows it next to the F&B Manager's note), `received_by_name`, `received_at`, and a request `value` (Rs at average cost), plus per line
  `id`, `ingredient_id`, `name`, `unit`, `requested_qty`, `approved_qty`, `dispatched_qty`, `received_qty`,
  `variance_reason`, `unit_cost`. Say which exist, and add the missing ones as optional fields.
- `POST /fnb/kitchen-requests` for a lead: confirm the body (`kitchen_location_id` may be omitted? `needed_by`,
  `remarks`, `lines[{ingredient_id, quantity}]`), and whether quantities are validated against `pack_size`.
- `POST /fnb/kitchen-requests/{id}/receive`: confirm `{received_quantities: {lineId: qty}, variance_reasons:
  {lineId: text}}`, that a short line without a reason is refused (and the exact message), and what is returned
  (the updated request?).
- `POST /fnb/kitchen-requests/{id}/withdraw`: only while `Requested`? Message otherwise?

## Report back
In the same answer file as the pre-B7 questions, add a section "F6 confirmations" with, per numbered item: what the
source does today, what you added (with test names), and anything you left for the owner to decide. Contract and
changelog edits go in the same change as the code.

---

## Additions found while building the F6 data layer

**5. Signs and a count discard.**
- `GET /fnb/overview`'s `recent_movements[].quantity` and `GET /inventory/fnb/enquiry/{id}`'s `change_qty`: confirm both
  are **signed from the location's point of view** (negative for orders, waste, transit loss and outgoing dispatch;
  positive for received and incoming), and say which sign `dispatch` and `count` carry at the *kitchen* location.
  The screen shows `+`/`−` from the sign and has no other source.
- The prototype's stock count has a **Discard** button that abandons a count in progress. Is there a way to cancel or
  delete a count in `Counting` status (for example `DELETE /inventory/fnb/counts/{id}` or a `Cancelled` status), and
  may the lead use it for its own location? Without it an abandoned count blocks starting the next one.
