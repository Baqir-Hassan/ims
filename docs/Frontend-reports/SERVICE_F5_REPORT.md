# F5 — Cashier till hand-off

2026-10-13. Frontend work remains uncommitted. F6 has not started. Live B5 integration and physical receipt printing remain separate gates.

## Built

The outlet module now registers the lazy service-till slot for CashierApp. Bills are ordered by print time, with dining bills and live newly printed bill notifications. Settlement includes server totals, cash/card/bank transfer, split helpers, cash change, reference validation, partial payments, removal while unpaid, bill printing, discounts and approval waiting, department settlement, receipt and Next bill.

Payments show captured, signed, own-venue cash/card/bank amounts accepted during the current shift; reversed rows and other tenders are excluded. Register counting stays blind until counting starts, uses every prototype denomination plus coins, requires a variance note, and displays returned close history. Closing and opening the next shift remain server operations.

Writes use the existing API client with session/device identity, live till/orders/approvals refresh and authoritative results. Rejected requests retain entries. Offline payment retries retain client_ref, expose confirmed rejections for review/dismissal, and check authoritative acceptance before another collection. Final server settlement opens the receipt without another Close write. Receipt loading/retry and stale-response protection allow moving to the next bill safely. Print again uses browser printing.

Implementation is under app/modules/fnb_outlets/screens/till/: types, API, model, context, controller, frozen markup port, offline feedback and receipt print CSS. The outlet manifest registers the slot; the existing service style probe adds till selectors. Required visual deviations are recorded in the parent frontend plan. Backend, shared API contract and frozen prototype were not edited by the frontend.

## Contract and backend hand-off

[Integration notes](service-f5/INTEGRATION.md) record three owner-approved corrections:

- approval_manager_name on POS setup supplies the actual policy manager; absent/ambiguous names use the F&B Manager.
- checkout_discount_by_name on an Order supplies actual historical attribution; the current manager never substitutes for it.
- staff_meal_discount plus the corresponding gross service subtotal supplies the prototype's staff reduction. Missing fields preserve actual returned subtotal; the UI never assumes staff meals are free or derives financial reductions from line prices.

The backend agent's B5 report and source now implement the first two fields. The third remains an authorized additive proposal. Its exact prototype comparison uses a typed proposed response fixture and does not demonstrate current live backend support. Existing zero-value department settlement is distinct from the missing display breakdown.

Backend B3/B4/B5 notes were checked. B4 implements approved Served → Ready pickup Undo; the F4 Served → Preparing Recall proposal remains separate. B5 reports corrected payment identity/reversal, zero-settlement, attribution, shift timing, configured float and register guards. These are backend-agent source/test results, not a live frontend integration run. No server restart, migration, seed or printer configuration was performed here.

## Visual evidence

[Prototype/app pairs](service-f5/screenshots.html) cover 37 states at 1366×860, 768×1024 and 375×812, in light and dark themes. The prototype is reset with simulation off and a fixed clock. Typed test responses pass through the real API client. Historical shift rows absent from prototype open orders are test-only closed order projections, never product demo data.

Transient toast overlays are omitted from both screenshot columns; live notification and Open behavior have separate browser evidence. The probe compares screen geometry, actual text styles and wording without tolerances or font aliases. Register rows use the existing shared quirks-mode normalization: row geometry and actual b/small text are checked, while the non-text parent's deliberately normalized inline strut is outside the till probe.

[Required states](service-f5/required-states.html) contain 72 screenshots covering loading/error retry, retained rejection entries, generic manager fallback, receipt loading/retry after settlement, saved offline payments and staff-meal approval waiting. The owner approved their layouts and wording on 2026-10-13. Tender keyboard navigation and associated field labels preserve visual styling.

## Validation

Source/build tests use test transport fixtures; live migration/server integration, receipt device/physical printer checks and broader role acceptance remain required before production sign-off.

Browser verification completed: **222 pairs, zero measured style mismatches and zero wording differences**; **72 required-state screenshots across six viewport/theme variants** with behavior assertions passed. Regular light receipt and dark staff-meal receipt pairs were inspected directly.

[Focused behavior results](service-f5/evidence/behavior.json) record six passing checks: live printed-bill notification/Open plus tender keyboard; held-dish server rejection retaining reference; queued final payment replay opening receipt with unchanged identity and server shift timestamp; bearer/device identity; confirmed rejected payment review/dismissal before corrected collection; and delayed previous receipt unable to replace the next receipt.

Nine new unit tests cover returned money, staff breakdown and legacy fallback, captured/venue/shift/tender filtering including refunds, stable bill ordering/settlement time, original discount metadata and historical attribution, denominations/past-close scope, declared request payloads, direct/wrapped writes and normal conflicts/approvals.

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 57 passed |
| Pilot | Passed | Passed; same five image warnings | 57 passed |

Final full/pilot typecheck, lint, test and production-fixture scan logs are under service-f5/. Both dist/client/assets scans are clear of till fixture identities, prototype-source and __floorChange hooks. The full generated module/brand selection is restored. git diff --check passes. CodeGraph is synced and current. The final browser reruns use settled fonts and sequential dark captures; earlier font-loading flags/page reload timeouts are retained in intermediate logs, not counted as passing checks.

No commit, push, backend edit, shared contract edit or frozen prototype edit was made. The parent plan contains the required F5 accessibility and additional-state approvals. Stop at F5.
