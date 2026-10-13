# F4 — Kitchen display hand-off

2026-10-13. Frontend implementation is left in the working tree; no commits. F5 has not started. Real B4 integration remains pending.

## Built

- Cooks open the outlet module service-kitchen slot through /app. Station cooks see their assigned station; the pass sees every station in its venue and can select a station tab. The legacy KitchenQueue remains available for ManagementApp roles.
- The frozen KitchenApp, Ticket, AllDay and PassView markup is ported directly: always dark; Open/Late/Oldest/Avg ticket/clock header; oldest-first station rail; fresh/watch/late ages; new/additional/cooking stamps; allergy/options/seats; void/pending/rejected/moved slips; All day quantities excluding crossed and voided lines; Recall; and empty states.
- Start, Ready, Undo, line crossing, Recall and Send out call the real ticket endpoints. Writes are server-authoritative, guarded against simultaneous actions, and reload after success or rejection. The pass sends only Ready tickets and retains successful station progress when another station fails; retry sends only the remaining Ready ticket.
- Paper-only Start requires a reason when print_stuck. The reason remains entered after a rejection. Live refresh uses the existing useLive hook and kot/orders/approvals topics, without a separate polling loop.
- New Placed tickets can play the original two-tone chime after a user gesture. Initial/backlog tickets and station-view changes are silent; muting persists on the device. A screen wake lock is requested, reacquired after visibility returns, and released on unmount. Refusal does not interrupt cooking.

Module code is in app/modules/fnb_outlets/screens/kitchen/{types.ts,api.ts,model.ts,context.ts,controller.tsx,prototype-port.jsx,display-effects.ts}; the outlet manifest registers its lazy slot. Core changes add the kitchen style-probe selectors, preserve prototype quirks-mode inline line boxes in service.css, and give the shared kitchen toast overlay a dark theme. The backend hand-off review also adds the already approved optional StaffUpdate.email to app/core/service/types.ts.

## Validation

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 48 passed |
| Pilot (IMS_PROFILE=pilot) | Passed | Passed; same five image warnings | 48 passed |

Logs are under service-f4/: full/pilot typecheck, lint and test, plus visual-retry.log, behavior.log and states.log. Both production fixture scans are clear, and the full generated module/brand selection is restored. git diff --check is clean. CodeGraph was synced after the frontend changes.

[Browser behavior evidence](service-f4/evidence/behavior-summary.json) records nine passing scenarios: station scope/always-dark layout, silent initial/pre-gesture tickets, persisted crossed lines/void guard, gesture chime/mute persistence, failed Ready/success/Undo, keyboard station tabs/silent view changes, Ready/proposed Served Recall and server refusal, partial Send out recovery, and sign-out cleanup plus bearer/device identity.

Nine new unit tests cover defensive scope and oldest ordering, age boundaries and completed-ticket averages, All day totals, active pass orders retaining served station rows, stable Recall ordering, gesture/audio cleanup, wake-lock reacquisition/refusal, delayed wake-lock cleanup, and real request shapes/server conflicts. The additional-state script also verifies loading/retry and paper reason validation, rejection preservation, audited payload and subsequent acceptance. Audio/wake APIs are controlled doubles in the browser checks; physical sound/device behavior is a separate integration check.

## Screenshots and visual approval

[Prototype/app screenshot pairs](service-f4/screenshots.html) and [raw style-probe results](service-f4/evidence/summary.json) cover 31 desktop states at 1366×860, always dark: station/pass defaults and closed sides, crossing, placed/started/bumped/recalled tickets, new/additional/void/slips, empty rails/All day/Recall, all five pass station tabs, Ready/Served Recall, paper Start/Ready, all-stations Ready and Send out, mute, and persistence of dark mode under a light system preference.

All 31 pairs have zero style-probe mismatches and zero wording differences. Station and pass default pairs were inspected directly. The prototype is reset with simulation off; test-only typed transport fixtures use its seed data. Probe comparisons use no tolerances or font aliases. The app has the legacy preference deliberately set to light; the kitchen stays dark.

The [required-state review](service-f4/required-states.html) contains five screenshots: loading, server error/retry, empty/entered reason for accepting an unprinted paper ticket, and server rejection. The owner approved these on 2026-10-13: **Approve shown F4 states**. They reuse the prototype boot/empty, Sheet, field, buttons and toast, which have no matching rendered states in the frozen prototype.

Accessibility: station tabs have a name, tab semantics, aria-selected, one Tab stop and ArrowLeft/ArrowRight/Home/End navigation. The reason textarea has an associated label. Visual styling and wording are preserved. The tab accessibility deviation and approved additional states are recorded in D:/Projects/ims/docs/SERVICE_FRONTEND_PLAN.md. That required note is this phase's only edit outside frontend; backend and prototype were not edited.

## Contract questions and backend hand-off

[Integration notes and CodeGraph source/target map](service-f4/INTEGRATION.md) document the owner-authorized Served → Preparing Recall proposal. The prototype offers Ready and Served tickets; the current shared service contract permits Ready → Preparing only. The frontend shows the actual server rejection. Successful Served Recall remains a proposed fixture scenario until the backend owner agrees and implements lifecycle/venue/station checks. No shared contract edit was made by the frontend.

The backend B1–B3 notes were reviewed from backend/docs/SERVICE_B3_FRONTEND_HANDOFF.md and SERVICE_B3_REPORT.md. B3 is implemented in source, but that does not establish that the existing server has loaded it or that its database has been migrated. B4 still owns station enforcement, line-done writes, pickup/recall and cooked voids. No server restart, migration or seed was performed by this review.

The approved StaffUpdate.email correction is applied; F8 must require email and a starting password for PIN-to-password role changes. F3 cost-centre summaries and Served → Ready pickup Undo remain pending. The hand-off records the approved gross subtotal/payment-status semantics and held-dish/payment/reopen guards for F5. Current floor totals already consume gross subtotal and trust returned total.

Real B4 permissions, concurrent writes, held-line order-state guards, history scope, live events, showcase data and physical device/printer checks remain integration work. No live-backend write success is claimed from fixtures.

## Fixtures and reproduction

Test fixtures are in tests/service-kitchen-fixtures.ts, derived from the frozen prototype snapshot in service-f4/evidence/prototype-source.json. scripts/service-kitchen-browser.mjs installs typed responses behind the real client and labels the proposed served reversal. Product screens import no fixtures, simulation or substitute successful server responses; missing endpoints show their actual error.

With the prototype on :8765 and the service preview on :3025 using pilot/Rykards, set IMS_BROWSER_MODULE to Playwright and IMS_BROWSER_EXECUTABLE to Edge/Chromium. Run node scripts/check-service-kitchen.mjs, node scripts/check-service-kitchen-states.mjs and node scripts/check-service-kitchen-behavior.mjs. IMS_F4_RECHECK_FAILED=1 retains passing comparisons and reruns unfinished/failed pairs. Unit tests run with node --test tests/service-kitchen.test.mjs or npm test.