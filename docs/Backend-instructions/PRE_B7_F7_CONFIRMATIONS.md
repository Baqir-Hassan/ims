# Backend agent: F7 data confirmations (F&B Store), add to the pre-B7 work

Read this together with `PRE_B7_RECALL_AND_ANSWERS.md` and `PRE_B7_F6_CONFIRMATIONS.md`. The frontend agent has built the
F&B Store's screens (F7) from `prototype/index.html`. The plan's rule is: the contract carries every field the prototype
shows, and if a screen needs data the contract lacks, the contract is extended instead of dropping or changing anything
on screen. Contract section 7 says the dispatch, purchase-request and goods-received endpoints "keep their shapes", without
listing them, so the frontend built to the shapes the existing F&B screens read and marked everything else optional.

**What to do:** for each item, check the real response or permission in the source and write the answer in your answer file
(exact field names, types, who may call it). If something is **missing and the fix is additive** (a new optional field, a
new optional request-body key, a permission the store should have), add it, update `docs/SERVICE_API_CONTRACT.md` §7 with a
dated Changelog line, add a test in both profiles, and say so. If the fix is not additive, or needs a product decision, do not
do it: report it. Keep every existing shape and message working.

## 1. Dispatch with trimmed quantities (the biggest one)
Prototype pick list: the storekeeper ticks each line, may **send less than approved** (capped at what the store holds), and the
shortfall is visible to the venue and the F&B Manager. Today `POST /fnb/kitchen-requests/{id}/dispatch` is called with `{}`.
The frontend now sends `{}` when everything goes as approved, and otherwise
`{"dispatched_quantities": {"<request line id>": <qty>, ...}, "note": "Fresh milk short by 11 L"}`.
- Does dispatch accept `dispatched_quantities` (all lines, by line id)? If not, add it as optional: omitted means the approved
  quantity, as now. Refuse a quantity above the store's stock, or above the approved quantity, with a plain message.
- Does it accept `note`, and does the request then return it as `dispatch_note` with `dispatched_by_name` / `dispatched_at`
  (the request drawer for the kitchen shows it beside the F&B Manager's note)? If the server should generate the shortfall note
  itself, say so and the frontend will stop sending it.
- Response: the updated request (`KitchenRequest`)?

## 2. Listing kitchen requests for the store
The store lists every kitchen's requests: approved ("ready to pick"), requested ("waiting for the F&B Manager"), dispatched,
and received in the last two days. The frontend calls `GET /fnb/kitchen-requests` with **no** `kitchen` or `status` and
expects all venues' requests, with these fields: `id`, `number`, `kitchen` (name), `status`, `needed_by` (ISO),
`requested_by_name`, `requested_at`, `approved_by_name`, `approved_at`, `dispatched_by_name`, `dispatched_at`, `received_at`,
and lines `id`, `ingredient_id`, `name`, `unit`, `requested_qty`, `approved_qty`, `dispatched_qty`, `received_qty`.
- Is that call valid for `fnb_store_staff` as written? If a parameter is required, say which.
- Confirm the status strings (`Requested`, `Approved`, `Dispatched`, `Received`, `Received with variance`, `Rejected`,
  `Withdrawn`) and the order the list comes back in (the screen shows it as sent).

## 3. Store stock rows (`GET /inventory/fnb/enquiry`, no location filter)
Prototype Store stock and the pick list show, per ingredient: area, **vendor**, on hand, reorder level, par, level, average cost,
value, and the **batch and its use-by date**. The purchase-request composer groups lines **by vendor**.
Today a row has `id`, `name`, `unit`, `avg_cost`, `by_location[<location id>] = {quantity, par_level, reorder_level}`.
The frontend reads these optional extras and shows nothing when they are missing:
- `storage_area`, `pack_size` (B6 added both to the ingredient: also on this list for the store?).
- `vendor_id`, `vendor_name`: the ingredient's usual supplier. **Without this the composer cannot group or send a purchase
  request** (the screen then says "Vendor not set" and disables Send). Where does an ingredient's supplier live today? Add it
  to the row if it is a single supplier; if an ingredient can have several, say how the frontend should pick one.
- `next_batch: {batch_no, expiry_date}`: the batch used first (FIFO) at the F&B Store, or `null`.

## 4. Approved purchase orders (`GET /fnb/purchase-orders?status=Approved`)
Prototype "Deliveries due": each order has a **delivery time** ("Rawal delivers PO-1046 at 10:30", "Overdue", and a banner when
a delivery is within four hours). Today the order has `expected_delivery_date` only.
- Add `expected_delivery_at` (ISO with offset, optional) if the time is known or can be captured when the order is raised.
  Without it the screen shows the day only and shows the "delivers at …" banner only once the day has passed.
- Lines: `id`, `ingredient_id`, `item_name`, `unit`, `quantity`, `received_quantity`, `estimated_unit_cost` ✓? And a
  `perishable` boolean (the receive drawer asks for a **use-by date only for perishables**; today the frontend asks for it
  unless `perishable` is `false`).
- `request_number`, `estimated_total` ✓?

## 5. Receiving (`POST /fnb/goods-received`) and the list of receipts
Body the frontend sends: `{po_id, notes: null, lines: [{po_line_id, quantity_received, quantity_rejected, reject_reason,
unit_cost, batch_no: null, expiry_date}]}`. A line is sent when it has accepted **or** rejected stock (a fully rejected line has
`quantity_received: 0`). `batch_no` is `null` and the frontend invents neither a batch number nor an expiry.
- Is a fully rejected line accepted (B6 says fully rejected receipts are handled), and what does a `null` `batch_no` mean: does
  the server make one up? The prototype shows a batch on the store stock row.
- Return `grn_code` in the response (the toast reads "GRN-0882 posted. Store stock is updated and the invoice goes to the F&B
  Manager."). Today the response has `bill`; is `grn_code` there?
- `GET /finance/goods-received` feeds the "Goods received" table: `grn_code`, `po_code`, `vendor_name`, `created_at`,
  `total_received_value`, and `lines[] {item_name, quantity_ordered, quantity_received, quantity_rejected, ...}` (the table lists
  what was rejected and what came short). **May `fnb_store_staff` call it?** If not, allow read access for the store.
- What does `expiry_date` mean: the last day the batch may be used, or the first day it may not? (The prototype reads the date as the
  moment it expires, the start of that day, so a batch dated tomorrow reads "Expires today". Say which the server means.)

## 6. Purchase requests
- `GET /fnb/purchase-requests` rows: `number`, `vendor_name`, `status`, `estimated_total`, `converted_po_code`, `created_at`,
  `reason`, `lines[] {item_name, quantity, estimated_unit_cost}`. Which status strings exist (`Pending`, `Converted`,
  `Rejected`, `Awaiting second approval`, ...)? And the F&B Manager's rejection reason: is it returned (the frontend reads
  `decision_note`)? The prototype shows it under "Rejected".
- `POST /fnb/purchase-requests`: the frontend sends one request per vendor with
  `{vendor_id, needed_by_date: null, reason, lines: [{ingredient_id, quantity, estimated_unit_cost}]}` and `reason` defaults to
  "Store restock". Confirm, and that the response is the created request (`number`).
- `GET /fnb/vendors` returns `{id, name}`?

## 7. Batches expiring (`GET /fnb/batches/expiring?location_id=<store>&days=2`)
Confirm `location_id` takes the F&B Store's id, and the rows `{id, ingredient_id, name, unit, batch_no, expiry_date, remaining,
location}` (`batch_no` may be null for unbatched stock?).

## Report back
Add an "F7 confirmations" section to the same answer file: per numbered item, what the source does today, what you added (with
test names), and anything left for the owner to decide. Contract and changelog edits go in the same change as the code.
