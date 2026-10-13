# Service integration completion — 14 October 2026

Source integration fixes are implemented. Tests, real API execution, browser comparison and visual
sign-off were deliberately not run in this continuation, per the owner's instruction to skip tests.
Historical F8/F9 screenshots and failures remain historical evidence, not verification of these edits.

## Implemented

- Finance Open orders export adds only supplied SQL filters, removing untyped NULL binds that
  PostgreSQL cannot infer. Added optional department; the frontend exports only its displayed F&B scope.
- Store expiry consumes the aware Karachi expiry timestamp, distinguishes expired/tomorrow, and
  renders batch dates in Karachi. Store receipts and Manager purchasing use accepted_received_value.
- Invoice item reports compare uniquely linked accepted receipts without allocating ambiguous bills.
  Purchase supplier reports use selected-period delivered POs and quantities through period end,
  including rejected-only receipts. Price rows compare the latest two actual accepted receipt prices.
- Exceptions carry explicit title/area/priority; matching workbook columns are appended. Existing
  workbook columns are retained. Server report values now populate the existing frontend slots.
- Supplier-week Deliveries/Orders/Invoices controls and prototype comparison buckets are restored.
  Report sorts, signed price/difference formatting, comparison totals, supplier reliability pills,
  request status pills and transfer totals use the returned data.
- Manager Store summary now shows the location movement bridge rather than an ingredient list.
  Its read/export are inventory-owned and clearly use current average cost, not historical GL value.
- Sales transactions read table_code. Supplier cards use prototype reliability colours. Manager waste
  kitchen labels omit the redundant Kitchen suffix. Ingredient drawers show Not stocked here for
  absent cells and do not create those cells when saving other locations.
- Roster badges show actual clock-in/out and lateness; head-chef pass labels use the prototype wording.
  The existing guarded multi-shift state, same-ID edits, completed-attendance confirmation and
  server attendance/payroll restrictions remain. No simulated attendance was added.

## Boundaries and compatibility

Direct New PO remains blocked, untaxed-only received-value settlement remains, count variance remains
separate from recipe cost, and no implicit 8 percent overhead is inserted. No seed/migration, database
write, service restart, profile generation, commit, push or deployment was performed. Prototype and
historical evidence files were not changed. Existing unrelated checkout changes were preserved.
Definitions/permissions are synchronized in SERVICE_API_CONTRACT.md. Reference ownership is appended
in backend/docs/SERVICE_REFERENCE_MAP.md. No monolithic Rumanza import or new GL account is introduced.

## Validation scope

Static Python compilation, current generated frontend TypeScript checks, changed-file ESLint and import
boundaries are recorded at completion below. These checks are not either profile's runtime acceptance.
Unit/integration/browser tests, builds, downloads, physical printers/devices and hosted smoke are skipped.
A fresh visual capture is still required to establish which historical differences remain after these
source corrections; the old measured matrix cannot be declared clean without that work. Required
empty/loading/refusal states continue to preserve input and show actual server messages; their complete
browser acceptance matrix remains deferred with the other tests.

## Next acceptance when tests are resumed

Verify export SQL against isolated full/pilot APIs; partial/rejected/duplicate/ambiguous invoice receipts;
zero/partial/full/late/undated supplier deliveries within and after the selected period; price history
and zero prior price; Karachi expiry before/at/after midnight in another browser timezone; matching
stock-summary JSON/workbook and quantity reconciliation; actual roster/empty/refusal states; then
recapture prototype pairs. Existing F8/F9 source implementation is not a release sign-off.

## Static completion results

- Final direct TypeScript check passed against the current generated pilot registry.
- Changed-file ESLint passed with zero errors/warnings after naming the report cell renderer.
- Four changed backend Python sources compile in memory, without imports or database access.
- Import boundaries: three contracts kept, zero broken (195 files, 747 dependencies).
- Root/backend/frontend whitespace checks passed; Git's existing LF/CRLF notices are not failures.
- Rumanza index remained current. IMS index shows the new routes/helpers; a concurrent background
  indexer held its lock during one explicit sync attempt; the later retry completed and reported
  already up to date. No live lock was removed.

No automated tests, live queries, build, browser workflow/probe or deployment was run in this request.
The former export HTTP 500 is corrected in source; its runtime resolution awaits resumed acceptance.
