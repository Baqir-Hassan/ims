# Backend agent: do this BEFORE starting B7

You are the backend agent for the "service roles" work (waiter, cashier, cook, inventory lead, F&B Store, F&B
Manager). You may not remember earlier sessions, so here is what you need.

**Read first, in this order:** `docs/SERVICE_API_CONTRACT.md` (§3 Pickups, §4 Kitchen), `docs/SERVICE_BACKEND_PLAN.md`
(B4), and your own `backend/docs/SERVICE_B4_REPORT.md`, which shows how you built the kitchen ticket lifecycle and
pickup Undo. You touch only `backend/` and the backend docs, as before. Do not edit `frontend/` or `prototype/`.
No commits, no server restart, no migration or seed of the pilot/public database (disposable `test_ims_*`
schemas only), exactly as in B3–B6.

Do the task in Part 1, then answer the questions in Part 2, and only then start B7.

---

## Part 1. Add the kitchen recall of a served ticket: `Served → Preparing`

### Why
The kitchen display's **Recall** panel lists recently bumped tickets, both **Ready** and **Served**, each with a
"Bring back" button. A cook uses it when a plate was sent out wrong or a table asks for it again. The frontend sends
`PATCH /kot/tickets/{id}/status` with `{"status": "Preparing"}` for either kind. The backend currently allows only
`Ready → Preparing` (and `Served → Ready` for the waiter's pickup Undo), so recalling a Served ticket is refused.
The owner decided the backend should support it, because the prototype is the agreed design.

### Behaviour to implement
1. **New transition `Served → Preparing`** on `PATCH /kot/tickets/{id}/status`. It returns the updated `Ticket`,
   like the other transitions.
2. **Who may do it:** the kitchen group `outlets.kitchen` (`cook`, `manager`, and the legacy roles already in that
   group). A `waiter` or `cashier` must not be able to (they have `Served → Ready` Undo only). The venue and
   station checks are the same as for `Ready → Preparing`:
   - a station cook may only change their own station's tickets, otherwise **403** `"That ticket belongs to <station>."`;
   - a pass cook (no station) may recall any station's ticket in the venue;
   - a ticket from another venue is refused the same way existing ticket writes refuse it.
3. **Effect:** the ticket becomes `Preparing` and behaves exactly like one recalled from `Ready`. Reuse the same
   code path and the same timestamp handling as `Ready → Preparing`, and also clear whatever "served" timestamp the
   `Served` state set. Do not clear or change the lines' `done` flags. Do not create a new KOT print job, slip or
   alert. A recalled ticket is the same ticket, not a new one.
4. **Order state:** recompute the order's state with the same helper the pickup Undo uses. An order that was
   `Served` is no longer Served once one of its tickets is `Preparing`, and it returns to Served only when every
   live ticket is Served again and no uncancelled held line remains (§3 Pickups).
5. **Stock:** do not consume ingredients again and do not restore any. Recall, like Undo, never changes stock.
6. **Refusals (409 unless stated), reusing existing wording where the same condition already has a message:**
   - ticket is not currently `Served` → the existing invalid-transition error;
   - ticket or order is cancelled → refuse (reuse the Undo wording for a cancelled order);
   - the order has **any** recorded payment, including a partial payment or a later reversal →
     `"Payments are already taken. Ask the cashier."` (same rule as Undo: a paid order has freed its table, so a
     kitchen ticket must not reopen on it).
7. **Concurrency:** take the same locks as Undo, and lock the order row *before* checking payment state, so a
   recall racing a cashier's payment cannot succeed after the payment commits (payment writes lock the order, then
   the venue shift).
8. **Live events and audit:** announce the same topics a ticket status change already announces (so the waiter's
   Pickups list drops the ticket and other kitchen screens refresh), and write the audit entry in the same style as
   the other status transitions.

### Docs to update (same change)
- `docs/SERVICE_API_CONTRACT.md` §4: add **`Served→Preparing`** to the transition list with the rules above. Add a
  line to §3 Pickups noting that a served ticket can also be recalled by the kitchen. Add a dated entry under
  **Changelog** ("approved by owner"). The change rule says neither side changes a shape alone, so the contract and
  code change together.
- `docs/SERVICE_BACKEND_PLAN.md` B4: add a bullet for the transition.
- `API_COVERAGE.md`, and an addendum to `backend/docs/SERVICE_B4_REPORT.md` (do not rewrite the report).

### Tests (full and pilot profiles, disposable schema)
Add them next to `tests/fnb_outlets/test_service_kitchen.py`:
1. Own-station cook recalls a Served ticket → 200, status `Preparing`, order no longer Served, `done` flags kept,
   **ingredient stock unchanged**, no new print job or slip.
2. A cook of another station → 403 with the exact station message; a pass cook succeeds; a cook of another venue
   is refused.
3. `waiter` and `cashier` → permission 403.
4. Refused after a partial payment, after full payment, and after a payment that was later reversed (exact
   message); refused for a cancelled ticket and a cancelled order.
5. A ticket that is not Served (Placed, Preparing, Ready) is refused for this transition, and the existing
   `Ready→Preparing` recall, `Ready→Served` and `Served→Ready` Undo still behave as before.
6. Round trip: Ready → Served → Preparing → Ready → Served, checking the order state after each step.
7. Live event topics announced; audit entry written.
8. If the test setup allows it, recall racing a payment on the same order.

Also rerun the existing B4 and KOT regressions, the import-boundary check and compilation, and record the
results in the addendum, with the same "split evidence" honesty as your earlier reports.

---

## Part 2. Answer these questions (no code change unless stated)

The frontend must match what the backend really returns, so it cannot guess. Answer from the source and put the
answers in a short section of the B4 addendum (or a new `SERVICE_PRE_B7_ANSWERS.md`). Quote field names exactly.

**A. Staff-meal totals.** The frontend once expected an extra `staff_meal_discount` field on `Order`. The owner has
decided the frontend will **not** use it and will show only the fields the contract returns. Confirm, for an order
with `guest_type: "Staff meal"`: what `subtotal`, `member_discount`, `tax_amount` and `total` contain, and what
`POST /cafe/orders/{id}/charge-department` charges and records. Do **not** add `staff_meal_discount`.

**B. Cost centre on orders.** The owner has decided the frontend will match the backend instead of the backend
adding `cost_centre_code` to order summaries. Confirm exactly:
1. Which responses carry `cost_centre_code`: `GET /cafe/orders/{id}`, the `POST` responses that return an `Order`,
   the receipt, `OrderSummary` lists. Is a cost-centre **id** (`cost_centre_id`) returned anywhere?
2. What body `POST /cafe/orders/{id}/charge-department` accepts today (`cost_centre_id`? `cost_centre_code`?), and
   how a cashier's device can learn the id for a code (a list endpoint, `/pos/setup`, the order itself).
3. What a cashier gets from `GET /cafe/orders?venue=<v>` **without** `open=true`, and with it: full `Order` objects
   (lines, payments, `subtotal`, `member_discount`, `total`, ...) or the `OrderSummary` projection? List the fields
   present. The till screen currently assumes full orders.

If an answer shows the cashier **cannot** charge a staff meal to its department with the contract as written (for
example no way to obtain the cost-centre id), do not invent anything: report it plainly and stop that item. Do not
change the contract for it without the owner.

---

## Report back
When done, give the owner: the files changed, the new test names and their results (honest about any split or
rerun evidence), the answers to Part 2, and the exact contract/plan lines you changed. Then wait for the owner's
go-ahead before starting B7.
