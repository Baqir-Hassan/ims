# F3 — Floor / waiter hand-off

2026-10-13. Frontend implementation is left in the working tree; no commits. F4 has not started. Live B3/B4 integration remains pending.

## Built

- The waiter role opens the owning outlet module’s `service-floor` slot through `/app`. Other roles keep their existing ManagementApp fallback.
- The prototype’s venue/service/clock header, Ready count, name/sign-out, Floor/Orders/Pickups navigation, plan/list views, table state/legend, Mine filter, and seating sheet are ported directly. Seating sends empty items with covers and Walk-in, Member, or Staff meal metadata.
- The order screen uses typed menu, availability, orders and tickets from the real API client. It preserves sections/chips, `/` search, required modifier groups, priced add-ons, allergy notes, seats and quantity. Drafts persist separately for each venue/order until a successful send; failures preserve them.
- Sent dishes group by station ticket/status. Held dishes persist on the server and Fire sends selected line IDs. Voids use the existing cancellation endpoint; a 202 retains the dish and shows pending manager approval.
- Print bill uses confirmation and locks the check. Reopen calls DELETE /bill and is disabled when a payment exists. Phones use the prototype’s full-screen Review & send check.
- Open-order cards show Mine/Everyone and station progress. Pickups sort mine first and oldest first, retain the prototype’s waiting timers, mark Served, and expose the proposed Undo reversal with normal server error handling. A live refresh announces a newly Ready own ticket.
- Data refresh uses the existing `useLive` hook and orders/kot/till/approvals topics. Fire joins the existing offline POST queue; floor/menu/availability join its GET cache. Repeated queued seating/Fire actions are guarded.

## Validation

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 39 passed |
| Pilot (`IMS_PROFILE=pilot`) | Passed | Passed; same five warnings | 39 passed |

Logs are retained under `service-f3/`: full/pilot typecheck, lint and test. The full generated module/brand selection was restored after the pilot checks. Floor fixture identifiers are absent from both profiles’ built client/server JavaScript. `git diff --check` is clean.

The [browser behavior evidence](service-f3/evidence/behavior-summary.json) records nine passing scenarios: seating validation/search shortcut, durable local drafts, held/Fire payloads, failed-send recovery, bill locking, pending void approvals, pickup sorting/serve/proposed Undo, own-ticket Ready notification, and bearer/device headers. Six new unit tests cover draft isolation/merging, payload boundaries, authoritative server totals plus drafts, modifier-group independence, request shapes, and approval/server errors.

## Screenshots and visual approval

[Prototype/app screenshot pairs](service-f3/screenshots.html) and [raw style-probe results](service-f3/evidence/summary.json) cover the floor, seating, order/menu, item modifiers, draft edits, sent station groups, bill confirmation/locking/reopen, void reasons, open orders, pickups and empty/availability states at 1366×860, 768×1024 and 375×812 in both themes. The phone full-screen check is included.

All 212 unique prototype/app pairs pass with zero style-probe mismatches and zero wording differences. Desktop and tablet have 35 states per theme; phones have 36 per theme, including the full-screen check. Raw per-state probe JSON and PNG pairs are retained. Desktop floor, dark modifier sheet, phone menu and phone full-screen check images were also inspected directly.

The prototype is reset, simulation is disabled after reset, and Appearance is System. Prototype-only controls are removed from captures. The app is checked with the legacy preference deliberately set to the opposite theme. Wording is checked as well as computed styles, without tolerances or font-name aliases.

The [additional required-state review](service-f3/required-states.html) contains 36 app screenshots: loading, API error/retry, persisted held dish, its Fire sheet, queued Fire feedback, and pending void approval. The owner approved the shown states on 2026-10-13: **Approve shown F3 states**. They use the prototype’s existing pieces; the frozen prototype has no matching rendered states. Offline Fire enqueue, duplicate guard and replay are checked at all six size/theme combinations.

This approval is recorded under Deviations in `D:\Projects\ims\docs\SERVICE_FRONTEND_PLAN.md`. That required note is the only F3 documentation edit outside `frontend/`; the prototype and shared contract are unchanged.

## Contract corrections and live integration

The owner authorized documenting and continuing with two proposed corrections. The [integration notes and CodeGraph reference map](service-f3/INTEGRATION.md) give the precise hand-off:

1. Order summary/detail needs `cost_centre_code: string | null` so another device can display the Staff meal department accepted at creation. The frontend accepts the proposed optional field; it invents no default department.
2. Pickup Undo needs Served → Ready on the existing ticket status endpoint. The current contract does not allow this reversal. The successful browser Undo scenario uses the proposed test response; real server support is still pending.

A read-only request to the running backend returned 404 for `/api/v1/cafe/floor?venue=Rooftop`. Its process was neither restarted nor modified. Real B3/B4 endpoint behavior, showcase data, server totals, permissions, concurrency and the proposed corrections still need live validation once the backend is available.

## Fixtures and reproduction

Typed fixtures are in `tests/service-floor-fixtures.ts`, derived from the frozen prototype’s seed data captured in `service-f3/evidence/prototype-source.json`. Browser scripts install responses behind the real API client. No product fixture/simulation is selected by the floor screen; missing endpoints show their actual error. The shared auth fixture provides the test session/device identity.

With the prototype on :8765 and the service preview on :3025 using pilot/Rykards selection, set `IMS_BROWSER_MODULE` to Playwright and `IMS_BROWSER_EXECUTABLE` to Edge/Chromium. Run `node scripts/check-service-floor.mjs`, `node scripts/check-service-floor-behavior.mjs`, and `node scripts/check-service-floor-states.mjs`. `IMS_F3_RECHECK_FAILED=1` preserves passing pairs and retries unfinished comparisons; `IMS_F3_WIDTH` and `IMS_F3_THEME` restrict a comparison run.