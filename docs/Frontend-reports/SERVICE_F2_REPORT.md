# F2 — Sign-in hand-off

2026-10-13. Changes are left in the working tree; no commits. F3 has not started.

The frontend implementation is built and tested. The owner approved the additional required auth-state layouts and wording on 2026-10-13, resolving their visual review. Live backend integration and the showcase seed remain pending.

## Built

- `/app` provides the prototype's Entry, AccountForm, shared-device roster and PinPad. `/staff` and `/admin` redirect there; the signed-in account selects its role.
- Manager-only setup posts the venue/name, revokes the setup session and stores the returned device key. A failed logout can retry without creating the device again. Resetting the device requires confirmation.
- The PIN pad supports keypad and keyboard input, deletion, wrong-PIN messages and shake, device locking, and automatic roster recovery after the Manager unlocks it. The roster uses the existing `useLive` hook and the `devices` topic.
- First-PIN change keeps the restricted token out of the workspace and saved session, checks that the two entries agree, shows server validation messages and accepts only the fresh token returned by `/auth/my-pin`.
- Own-computer sessions persist; PIN and walk-up sessions stay in tab storage. Shared sessions end after five minutes without pointer/key input, except cooks. The server's `expires_at` controls session end, including after the tab becomes visible again.
- API and live-stream auth failures return to the appropriate sign-in state; device revocation also deletes the key. The existing live stream now sends `X-Device-Key` too.
- The new roles are registered in their owning modules. `roles.consolidated` is explicitly selected in the pilot: the Manager inherits the old management workspaces, hidden role definitions disappear, and Manager/Store receive their service metadata. The full build keeps the old role definitions and labels.
- Until the module shells are ported in F3–F8, authenticated roles use their currently permitted ManagementApp workspaces. Password sessions show “Signed in until …”. Roles without a service layout retain ManagementApp.
- The old backend's password response remains supported using its actual signed-token expiry and existing role/scope. The running backend can retain that older behavior until its B2 code is loaded.
- `/app` renders the prototype's `viewport-fit=cover`; a small Vite compatibility transform fills vinext 0.0.50's omitted viewport field, with a rendered-HTML regression check. The entry clock/date are populated after hydration so saved sessions and browser/server clock differences do not cause a hydration failure.

## Validation

| Profile | Type check | Lint | Build and tests |
|---|---|---|---|
| Full | Passed | Passed; five existing image warnings | 33 passed |
| Pilot (`IMS_PROFILE=pilot`) | Passed | Passed; same five warnings | 33 passed |

The browser behavior checks cover wrong-PIN retry, locking and automatic unlock, first-PIN restriction and fresh-token replacement, reset confirmation, keyboard roster navigation, own/walk-up persistence, idle activity and cook exemption, expiry, device revocation, and live-stream device headers/auth failure. See [behavior evidence](service-f2/evidence/behavior-summary.json).

## Screenshots and style probe

[Prototype/app pairs](service-f2/screenshots.html) cover the start screen, own-computer form and remember checkbox, empty-password error, setup login and rejected Store account, venue choices, roster and all its filters, PIN pad with empty/two-digit states, and walk-up form.

Both pages use 1366×860, 768×1024 and 375×812 in light and dark. The prototype is reset, simulation is disabled after reset, and Appearance is System. Demo chips, prototype controls and the demo PIN hint are omitted from the comparison as required by the plan. The app is checked with the legacy theme deliberately set to the opposite preference.

All 96 comparable screenshot pairs have zero style-probe mismatches, with no tolerances or font-name aliases. Raw probe results and the coverage totals are in [summary.json](service-f2/evidence/summary.json). The comparison also checks visible wording on the supplemental states, and waits for fonts, hydration and transitions before measuring. Source images were inspected, including the desktop roster and phone PIN-change sheet.

The prototype has no doctype. Its default form margin is reproduced inside the service root, and interpolated numeric text nodes are preserved to match its exact glyph measurements. These fixes reproduce the reference rather than change its design.

## Accessibility deviation

The roster's tablist now has named tabs, `aria-selected`, one Tab stop and ArrowLeft/ArrowRight/Home/End navigation. This has no visual or wording difference and is recorded under Deviations in `D:\Projects\ims\docs\SERVICE_FRONTEND_PLAN.md`, which also records the owner-approved additional auth states. These plan notes are the only F2 edits outside `frontend/`.

## Approved additional auth states

[Required real-auth state review](service-f2/auth-states.html) contains 30 app-only screenshots: wrong PIN, device locked, first-PIN change, its server error, and device reset confirmation, each at all required sizes/themes.

The frozen `PinPad` accepts any four digits and never renders an error or lock. It also has no first-PIN-change or reset-confirmation sheet. F2 and the contract require these behaviors. The implementation uses the prototype's existing Sheet, auth-form, error and shake pieces; the PIN-error wording comes from the server. The owner reviewed and approved the shown sheet headings, field labels, confirmation wording and required-state compositions on 2026-10-13.

**Owner decision recorded:** Approve shown states, in response to the review of `service-f2/auth-states.html`. The 30 screenshots document those approved additions; the 96 directly comparable pairs remain the exact-prototype evidence. No prototype or contract files were edited. The approval is also recorded under Deviations in the frontend plan.

## Fixtures and backend follow-up

Typed fixtures live in `tests/service-auth-fixtures.ts` and are installed through the browser test transport behind the API client. They derive the Rooftop staff and till from the prototype's STAFF/seedDevices and mock the B2 login shape, logout, device creation/roster, PIN and PIN change. No simulation or fixture authentication is active in the app; a missing server endpoint produces its API error. Fixture keys/tokens are absent from the built client/server JavaScript.

B2 auth/session/device source appeared during this frontend phase. A final read-only request to the running backend on :8000 still returned 404 for `/api/v1/auth/device`, so its live B2 integration is not available yet. The 12-hour/14-day B2 lifetime, server lock counter, session revocation and real seeded data still need backend integration validation. No new contract fields or contract edits are requested.

## Reproducing the browser evidence

Run the existing prototype on :8765 and the service frontend preview on :3025 with the pilot module/brand selection. Set `IMS_BROWSER_MODULE` to a Playwright module path and `IMS_BROWSER_EXECUTABLE` to Edge/Chromium, then run `node scripts/check-service-sign-in.mjs` and `node scripts/check-service-auth-behavior.mjs` from `frontend/`. `IMS_F2_RECHECK_FAILED=1` retries unfinished comparisons; `IMS_AUTH_RECHECK` selects behavior scenarios by regular expression. Both tools use test-only responses and retain evidence under `docs/service-f2/`.
