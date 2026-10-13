# F7 — F&B Store hand-off

2026-10-13. Committed on `akito/jolly-knuth-66fvdl` in rims-frontend. F8 has not started (it needs B7). Real integration remains pending.

## Built

- **F&B Store** (`fnb_inventory/screens/store/`), opened through the slot `service-desk-fnb_store_staff` on the shared `DeskShell`: `types.ts` (the shapes the existing store endpoints send), `api.ts`, `model.ts`, `controller.tsx` (live on the `stock` and `approvals` topics, one write at a time, the server's message shown as sent) and `prototype-port.tsx` (StoreApp, Overview, Dispatch, PickDrawer, Receiving, ReceiveDrawer, Purchase requests, PrComposer, Store stock, translated one to one).
- Writes: dispatch (the existing empty body when everything goes as approved; `dispatched_quantities` and a note only when a line is short), goods received (accepted **or** rejected lines, rejection needs a reason, no batch or expiry invented), purchase requests (one per vendor, in turn; a refusal stops the rest and says why).
- Also changed, found while building F7: the lead and the store share one kitchen-request mapping; the kitchen-request fields use the existing `approved_at` / `approved_by_name` (F6 had invented `decided_*`); legacy decimals are coerced to numbers; and composer lines (both roles) keep the order they were added in, which the first F6 check did not catch.

## Validation

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 73 passed, 1 failed |
| Pilot | Passed | Passed; same warnings | 73 passed, 1 failed |

The one failure is the F3 floor unit test that reads an evidence file that was never committed (it fails the same way on the untouched earlier commit). Seven new unit tests cover the store mapping. No fixtures reach the built bundles.

## Prototype comparison (no screenshot pairs, by the owner's choice)

`node scripts/check-service-store.mjs` drives the prototype and the app through the same clicks with the same fixed clock at 1366×860 in both themes and compares the computed style of every selector in the probe's `inventory` list plus all visible text: **26 states per theme, 52 of 52 identical**. States: overview and the delivery banner; the receive drawer (default, rejected with reason, short, price moved); dispatch and the pick list (default, one ticked, short, all ticked, after sending); receiving (list, a second order, after posting); purchase requests (list, composer, ingredient search, a line added, a line removed, after sending); store stock (default, below reorder, search, no match). The F6 check now has 84 of 84 identical (it gained the "ingredient added" state).

`node scripts/check-service-store-behavior.mjs` (8 scenarios, all passing): a failed load and retry; a refused dispatch keeps the pick list, ticks and quantities; a full send names no quantities and a double click sends once; a fully rejected line is posted with its reason and no invented batch or date; a refused receipt keeps what was entered; a purchase request that fails on the second vendor stops there and says why; no vendor means Send is disabled; the session and device headers.

The comparison found and fixed: the prototype's expiry wording rule (a batch's date is the moment it expires), the order of requests (kept as the server sends it), the vendor groups' order in the composer, and a 0.02px width difference caused by building "area · vendor" as one string instead of the prototype's separate text nodes.

## Needs the owner

1. Approve or change the Deviations recorded in the plan for F7 (extra states, data the server does not send, expiry wording).
2. **Expiry meaning.** The prototype reads a batch's date as the moment it expires, so a batch dated tomorrow says "Expires today". I copied that. Say if a date should mean "usable through that day" instead (the screen would then say "Expires today" only on the day itself).
3. **Dispatch with less than approved**, **vendor per ingredient** (the composer cannot send without it), **delivery time of day**, **perishable flag**, **batch on stock rows** and **read access to goods received** are not in the existing responses; they are asked of the backend in `docs/Backend-instructions/PRE_B7_F7_CONFIRMATIONS.md`. Until then: a missing field shows nothing invented, and Send on a purchase request stays disabled for an ingredient with no vendor.

## Reproducing

Prototype on :8765, app on :3025 (`IMS_PROFILE=pilot npm run dev -- --port 3025 --strictPort`), `IMS_BROWSER_MODULE`, `IMS_BROWSER_EXECUTABLE` and, if the prototype's CDN scripts are blocked, `IMS_PROTOTYPE_LIBS`, as in the F6 report. `IMS_F7_THEME` and `IMS_F7_ONLY` narrow the comparison; `IMS_F7_SHOTS=<dir>` keeps screenshots only of states that differ.
