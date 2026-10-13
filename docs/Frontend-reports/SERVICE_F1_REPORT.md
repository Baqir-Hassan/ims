# Service roles — F1 foundation handoff

Date: 13 October 2026 (Asia/Karachi). **F1 complete; stopped here for owner review before F2.** Changes remain in the frontend working tree.

## What was built

The inherited foundation now compiles, renders and passes its checks: the scoped prototype stylesheet, 35-path icon set, formatting helpers, shared controls/overlays/tables/charts/stock components, core contract types, service sessions and device storage, device-aware API requests, and optional service layout/sign-in metadata in `RoleDefinition`.

This continuation completed or repaired:

- `app/core/service/style-probe.js`: gallery selectors plus the F2 entry, account, device set-up, PIN screen and PIN pad lists. It returns every matching element, including absent-state counts, with the required properties plus native `color-scheme`, `scrollbar-color` and `scrollbar-width`.
- `scripts/diff-service-styles.mjs`: strict comparisons, element counts and missing entries; checks viewport/theme, loaded fonts and nonempty coverage. No numeric tolerances or font aliases were used for the recorded checks.
- `scripts/check-service-foundation.mjs`: repeatable light/dark comparisons, screenshot pairs, width-axis checks, pseudo-element checks and React control interactions.
- `/app/_gallery`: kept the server route in `app/app/%5Fgallery/page.tsx`, moved its client implementation and reference samples to `app/core/service/gallery/`. Vite decoded the escaped client-module URL and failed hydration when the client code lived in the route folder. The production route now returns 404 and has a regression assertion.
- Gallery coverage: 27 samples, including all button variants in the inherited gallery, switches on/off, tabs, fields/errors, plain/warning/action toasts, sheet/drawer, sorted and empty tables, populated/empty charts, three stock levels, all four prototype movement kinds plus `showName`, all icons, and Archivo/Martian Mono width-axis examples.
- Font fidelity: next/font fallbacks now reproduce the prototype families exactly. `build/windows-fonts.ts` finishes vinext 0.0.50's font URL rewrite on Windows; cached filesystem URLs previously reached the browser, so it rendered fallback fonts. Both fonts now load from the existing vinext asset route in development and the build output in production.
- CSS isolation: `app/legacy.css` loads the existing four console sheets in their existing order in a lower cascade layer. Service defaults can then override specific legacy selectors. SVG presentation attributes survive; preflight borders, toast styles, native-control themes and inherited scrollbar colours are reset. The four old global scrollbar selectors exclude `.svc` and its descendants.
- Browser defaults: the frozen prototype runs in quirks mode because it has no doctype. Scoped default resets reproduce its table line height and two-line row spacing in the app's standards-mode document. All ported prototype rule bodies remain intact except the permitted font-variable substitutions.
- Formatting: restored the prototype's non-breaking spaces in `fmtRs` and `fmtQU`; preserved React text-node boundaries in the independent movement reference markup.
- API coverage: downloads and offline queue replay now also send `X-Device-Key` and announce service auth errors. Revocation forgets the key; expired tokens and PIN-change gates preserve queued work for the next sign-in. Requests preserve caller-supplied `Headers` objects and detect bearer tokens supplied through them.
- Six focused regression tests cover session persistence/expiry, device identity, auth events, downloads/replay, Windows font URLs and failed/incomplete style probes. The entry-route test also checks served font URLs and the production gallery 404.

Roles without a service layout still use the existing ManagementApp. New service role entries and `/app` sign-in belong to F2.

## Screenshot pairs and style probe

[Open all 54 screenshot pairs](service-f1/screenshots.html). Each pair shows **prototype on the left, app on the right**. The two source pages use 1366 × 860; the pair images crop each component from those viewports. There are also full viewport captures for each theme.

The prototype ran through the existing `ims-ui-prototype` server at :8765. Its demo data was reset and simulation turned off through the Prototype controls before each capture. Its independent reference markup was injected at the gallery's exact width and rendered by the prototype's own, unchanged stylesheet. The app used the separate `ims-frontend-svc` entry at :3025.

| Check | Light | Dark |
|---|---:|---:|
| Selector groups | 119 | 119 |
| Elements measured | 372 | 372 |
| App vs independent markup under service.css | 0 mismatches | 0 mismatches |
| App vs actual prototype stylesheet | 0 mismatches | 0 mismatches |
| Scrollbar/switch pseudo-elements | 0 mismatches | 0 mismatches |
| React controls | Pass | Pass |
| Browser exceptions | 0 | 0 |

The saved legacy theme was deliberately **opposite** the system theme in both runs. Service screens, native checkboxes and scrollbars still matched the prototype. Archivo loads with `62% 125%` width range; Martian Mono with `75% 112.5%`. Rendered table-code and kitchen-header glyphs narrow at their prototype stretch values; their measurements match the reference.

Evidence: [summary](service-f1/evidence/summary.json), [light diff](service-f1/evidence/light-diff.json), [dark diff](service-f1/evidence/dark-diff.json), [light font/pseudo details](service-f1/evidence/light-fonts-and-pseudos.json), [dark font/pseudo details](service-f1/evidence/dark-fonts-and-pseudos.json). Raw probe outputs accompany them.

Full viewport captures: [light prototype](service-f1/evidence/light-prototype-viewport.png), [light app](service-f1/evidence/light-app-viewport.png), [dark prototype](service-f1/evidence/dark-prototype-viewport.png), [dark app](service-f1/evidence/dark-app-viewport.png).

The phone/tablet sign-in, floor and till captures belong to their respective later phases; F1 contains the shared-component gallery.

A [source audit](service-f1/evidence/source-audit.json) checked 677 prototype CSS rule bodies with no missing bodies and confirmed that all 35 icon paths match. A [legacy style comparison](service-f1/evidence/legacy-check.json) compared `/`, `/staff` and `/admin` against the original unlayered console styles in both themes: 0 differences across 34, 79 and 25 elements respectively.

## Full and pilot validation

| Profile | Typecheck | Lint | npm test (includes production build) |
|---|---|---|---|
| Full | Pass | Pass; 5 existing image warnings | 27 passed, 0 failed |
| Pilot (`IMS_PROFILE=pilot`) | Pass | Pass; same 5 warnings | 27 passed, 0 failed |

The brand assertion reads the generated brand, so both Rumanza/full and Rykards/pilot pass. The gallery production 404 and served font URL assertions pass in both builds. Generated module/brand files were restored to the full profile after validation.

Logs are under the ignored `.wrangler/f1-{full,pilot}-{typecheck,lint,test}.log` files in this checkout.

## Deviations

No intentional visual or wording deviations. The prototype and shared API contract were left unchanged. The default-reset and Windows font fixes reproduce the agreed design; they do not introduce another design.

## Contract questions

None blocking F1. The existing later-phase question remains for F6: the contract includes movement kinds `dispatch` and `transit_loss`, while the prototype's movement icon/label map has only `sales`, `received`, `waste` and `count`. Resolve that when F6 needs those states.

## Fixtures

Only the dev-only gallery's typed component samples/reference markup remain. F1 adds no runtime API fixtures. F2's unavailable endpoints will need typed fixtures behind its API client, as the owner requested, with removal tracked as real endpoints become available.

## Separate parent-folder edits and preview details

The parent `D:\Projects\ims` is outside the frontend Git repository. Two explicitly requested parent files changed:

- `ARCHITECTURE.md`: rewritten Frontend styling section describing both systems, the cascade isolation, the faithful default resets and the prototype reference. Other architecture sections were preserved.
- `.claude/launch.json`: added only the `ims-frontend-svc` entry on :3025.

The requested `preview_start`/browser preview tools were unavailable in this session; the app browser and Node tools also failed during Windows sandbox setup. Validation used the configured entry from PowerShell and a separate headless session of the installed Edge browser through the bundled Playwright runtime. No Bash server launch was used. The existing prototype server was reused.

To repeat the browser checks, start those two preview entries, set `IMS_BROWSER_MODULE` to an available Playwright module and `IMS_BROWSER_EXECUTABLE` to the browser executable if required, then run `node scripts/check-service-foundation.mjs` from `frontend/`. Compare saved outputs directly with `node scripts/diff-service-styles.mjs reference.json app.json`.
