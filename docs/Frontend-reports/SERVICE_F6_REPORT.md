# F6 — Venue stock (inventory lead) hand-off

2026-10-13. Committed on `akito/jolly-knuth-66fvdl` in rims-frontend. F7 has not started. Real B6 integration remains pending.

## Built

- **Shared desk shell** (`app/core/service/desk.tsx`): the prototype's `DeskShell` (side rail with the locked scope, grouped nav with count badges, the phone tab bar, main area). The service app now opens the slot `service-desk-<role>` first and `service-desk` second, because slot names are unique across modules and desk roles come from different modules; it also passes the "Signed in until …" text. F7 and F8 plug in the same way.
- **Inventory lead** (`fnb_inventory/screens/venue/`): `types.ts` (contract section 6 shapes), `api.ts`, `model.ts` (server data to the prototype's view), `controller.tsx` (live on the `stock` and `orders` topics, one write at a time, the server's message shown as sent) and `prototype-port.tsx` (InventoryApp, Overview, Stock, ItemDrawer, Composer, Requests, ReqDrawer, Stock count, Waste, translated one to one).
- The "Prototype only: play the next step" block in the request drawer is not shipped. Writes: request (`needed_by` as an ISO time with offset), withdraw, receive (quantities and reasons by line id), start / save / post / discard count, log waste. A count in progress keeps typed values on the device until it is submitted.
- Shared pieces touched: `Move` knows `dispatch` and `transit_loss`; `Level` copes with no par; one quirks-mode line-box rule for the count rows (the same technique as the kitchen and till).

## Validation

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 66 passed, 1 failed |
| Pilot | Passed | Passed; same warnings | 66 passed, 1 failed |

The one failure is the F3 floor unit test that reads `docs/service-f3/evidence/prototype-source.json`, which was never committed; it fails the same way on the untouched previous commit. Nine new unit tests cover the mapping rules (no invented cost, use or par; below-par order; request timeline and notes; waste reason/note and cost; count overlay; movement signs; pack rounding; needed-by wording). No fixtures reach the built client or server bundles.

## Prototype comparison (no screenshot pairs, by the owner's choice)

`node scripts/check-service-inventory.mjs` drives the prototype and the app through the same clicks with the same fixed clock at 1366×860 in both themes, and compares the computed style of every selector in the probe's new `inventory` list (about 150) plus all visible text: **41 states per theme, 82 of 82 identical**. States: overview and its delivery drawer; stock (filters, area, search, empty, selection, two ingredient drawers); requests (open / received / all, the four drawer statuses, a short receipt with reason); the composer (suggested, ingredient search, options, removed line, sent); the stock count (start, entries, expected, other area, review, submitted, restarted, discarded); waste (form, search, ready, over quantity, logged). The only allowed wording differences are listed in the script and under Deviations in the plan.

`node scripts/check-service-inventory-behavior.mjs` (6 scenarios, all passing): a failed load and retry; a refused receipt keeps the drawer, quantities and reason; movements that fail to load; count entries survive a reload and are cleared on submit, and only counted lines are saved; a double click sends one request; every request carries the bearer token and device key.

The comparison found and fixed four faults of mine: the most-depleted-first order of "Below par", the needed-by wording for past requests, the missing store note on a dispatched request, and a count whose area tab was chosen before the count existed. It also exposed values typed for one count leaking into the next.

## Needs the owner

1. Approve or change the Deviations recorded in the plan for F6 (extra states, the two movement kinds, the generic manager name, "No par set", the needed-by wording).
2. The prototype's overview says "used by orders **since 12:00**" and "before **tomorrow's 9:00** delivery". Both are kept word for word, but they are only true for a venue that opens at 12:00 and a delivery at 9:00. Say if they should follow the data.
3. Only managers could post a stock count before. F6 assumes the inventory lead may start, save, post and discard its own location's count; this and the other data the screens need are in `docs/Backend-instructions/PRE_B7_F6_CONFIRMATIONS.md`. Until the backend confirms, a missing field shows nothing invented and a missing endpoint shows its own error (Discard calls `DELETE /inventory/fnb/counts/{id}`, which is not in the contract).

## Reproducing

Prototype on :8765 (`python3 -m http.server 8765 --directory prototype`), app on :3025 with `IMS_PROFILE=pilot npm run dev -- --port 3025 --strictPort`. Set `IMS_BROWSER_MODULE` to a Playwright module and `IMS_BROWSER_EXECUTABLE` to Chromium. If the prototype's CDN scripts are blocked, set `IMS_PROTOTYPE_LIBS` to a folder with `react@18.3.1`, `react-dom@18.3.1` and `htm@3.1.1` installed from npm. `IMS_F6_THEME` and `IMS_F6_ONLY` narrow the comparison; `IMS_F6_SHOTS=<dir>` keeps screenshots only of states that differ.
