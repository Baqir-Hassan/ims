# F&B release rehearsal: Render API + Cloudflare Worker

Phase 6 is in progress. The target chosen on 2026-10-10 is a Render free-tier Docker API for manual testing, with the frontend on Cloudflare Workers. The release is not deployed yet. At the owner's request, the published repository, hosting account/service identifiers, live URLs and app-build details remain unspecified; the configuration contains no account IDs or credentials.

## Local gates

Run from `backend`:

```powershell
.\.venv\Scripts\lint-imports.exe
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m tools.member_contracts
```

The release smoke uses a second, empty disposable PostgreSQL schema on the configured database. It disables demo seeding, delivery gateways and background jobs, then creates Manager/staff/member/menu/recipe/vendor data through the API. It checks purchase approvals, GRN/bill, kitchen transfers, member-account sale and tax receipt, confirmed attendance, approved/paid payroll, month-end billing, statements and collections. Inventory, AP, payroll payable and member AR must match their records. Billing tax issuance must not double-post; reversing billing must undo revenue/tax/AR. A separate returned-cheque regression checks that fee recovery reaches member AR once, bank charges remain an expense and a fee is not counted as receipt tender.

The frozen contract fixture covers 35 reference operations, with their parameters, request schemas and declared responses. Concrete profile/menu/account/receipt/device/pay-code responses are also exercised. Regenerate deliberately after reviewing reference changes:

```powershell
.\.venv\Scripts\python.exe -m tools.member_contracts --reference D:\Projects\RumanzaCMS\rumanza-backend
```

Run from `frontend`:

```powershell
npm ci
npm run typecheck
npm run lint
npm test
```

CI runs the full backend and frontend gates. The actual app team's F&B-only build still needs its own acceptance pass; schema comparisons are API evidence, not a substitute for device testing.

## Render setup

The owner must first create/publish the IMS Git repository and commit the reviewed files, as required by the deployment plan. This workspace currently has no Git repository, so it cannot be linked to Render yet. No commits or pushes have been made by Codex.

1. Create a **new, dedicated empty PostgreSQL database** for IMS rehearsal. Do not use the RumanzaCMS database or a shared production schema.
2. Create a Render Blueprint from the backend repository (rims-backend) using its `render.yaml`. It defines `ims-api-test` as a Docker web service on the free plan, uses `backend` as its Docker context, and probes `/ready`. Automatic deployment is off.
3. Supply `DATABASE_URL` privately in Render, plus `RUMANZA_ALLOWED_ORIGINS` as the exact Cloudflare frontend origin. Use no trailing slash; multiple allowed origins are comma-separated.
4. Render generates independent token/setup secrets. Keep them in Render's environment manager, never in source, logs or frontend configuration. Do not copy RumanzaCMS signing or gateway secrets. `IMS_ENVIRONMENT=staging` enforces the release configuration.
5. Start the manual deployment. The container respects Render's `PORT`; normal API lifespan startup creates the module schemas and base settings. `/ready` returns 503 if PostgreSQL is unavailable.
6. The Blueprint maps Render's automatic `RENDER_EXTERNAL_URL` to `RUMANZA_PUBLIC_URL`, so PDF/payment links use the API origin rather than localhost ([Render environment variables](https://render.com/docs/environment-variables)). Record the **actual** Render service URL. Visit `/health`, `/ready`, `/api/v1/system/modules` and `/api/v1/setup/status`. Docs/OpenAPI must be 404. Check Render logs for startup failure without copying credentials.
7. Bootstrap the first Manager through the admin UI with the private setup token. Remove the setup token after successful setup; restart is allowed without it. Repeat setup must fail. Rotating the signing secret invalidates existing staff/member tokens and pay codes.

The rehearsal keeps outgoing delivery disabled. Configure only sandbox/test gateway credentials when testing messages/tax integration. Outbox/statement generation still works while delivery is disconnected.

Render free services sleep after 15 idle minutes and may take about a minute to wake. Their background jobs run only while awake; scheduled approvals, billing delivery and tax sync are not continuously available on this rehearsal tier. Free Render PostgreSQL expires after 30 days, so choose it only for disposable testing and export needed data before expiry. These are [Render's documented free-tier limits](https://render.com/docs/free). Docker/secret/health settings follow the [Blueprint reference](https://render.com/docs/blueprint-spec).

## Cloudflare Worker

The Worker configuration is `frontend/wrangler.jsonc`, with name `ims-fnb-test`, Node compatibility, static assets and Images bindings. Change the name only if the chosen Cloudflare account already uses it. Confirm the intended account before deployment; this config does not contain an account token.

`NEXT_PUBLIC_API_URL` is a **build-time** value; changing a Worker runtime variable after building does not update browser code. From `frontend`, use the actual Render URL:

```powershell
$env:NEXT_PUBLIC_API_URL = "https://YOUR-ACTUAL-RENDER-SERVICE.onrender.com/api/v1"
$env:IMS_MODULES = "members,finance,catalog,fnb_inventory,fnb_outlets,payroll_shifts"
npx wrangler login
npx wrangler whoami
npm run deploy
```

The predeploy script rejects missing/local/insecure API URLs and incomplete module selections. The Cloudflare Vite plugin builds the deployable Worker at `dist/server/index.js` and redirects Wrangler to the generated configuration through `.wrangler/deploy/config.json`. Worker artifact validation and the rendered-route tests run during the local gate. A local `npx wrangler deploy --dry-run` also validated packaging and the ASSETS/IMAGES bindings without publishing.

After deployment, set Render's allowed origin to the actual Worker/custom-domain origin and manually redeploy the API if needed. Keep the frontend and backend module lists identical.

## Live read-only check

From `backend`, substitute both actual URLs:

```powershell
.\.venv\Scripts\python.exe -m tools.release_check --api-url https://YOUR-API.onrender.com/api/v1 --frontend-origin https://YOUR-WORKER.workers.dev --expect-setup initialized
```

This checks PostgreSQL readiness, all seven modules, Manager setup status, hidden docs, CORS and HTML responses for the Worker landing/staff/admin routes. It only sends GET/OPTIONS and never seeds or mutates records. For the first check before bootstrap, use `--expect-setup empty`.

## Manual acceptance and app-team hand-off

On the deployed rehearsal service, repeat the smoke workflow using the six roles. Check POS/kitchen screens, approval restrictions, the till receipt, outlet P&L with labour, month-end statement PDF and trial balance. Verify desktop/mobile layout, offline order/payment replay and live kitchen updates. Reconnect after Render wakes; ensure a failed network request does not duplicate a paid sale.

The app team must point the F&B-only build at the IMS API and exercise register/login, profile/photo/password, notifications/devices, billing preferences/documents, menu/hours/tables/reservations, on-course ordering, member account/statement/pay-code and receipts. Features must report account, notifications and F&B; Golf/Academy flows stay hidden. Verify that one member cannot access another member's orders/notifications/account and that staff tokens do not work on member-only paths.

Record the build identifier and pass/fail evidence when this stage is resumed. The owner requested that the published IMS repository, Render service/account/database, Cloudflare account/actual origin and app-build details remain blank. Live deployment and mobile acceptance remain pending.

## Recovery and promotion

Use only disposable manual-test records until acceptance is complete. Preserve a database export and the previous API image/Worker version before a later production release. Render can redeploy a previous commit and Cloudflare can restore a previous Worker version; database DDL/data do not roll back with application code. Restore a separately verified database backup if data rollback is required. Disable tax/message sending while recovering.

Production promotion uses separate signing/setup/gateway secrets and a separate database, exact allowed origins and hidden docs. Provision an always-on service before relying on automated billing, notifications or scheduled approvals.

## Verification recorded on 2026-10-10

- Baseline backend suite: all 125 existing tests passed, including all payroll cases.
- Final 143-test run: 127 passed and 17 errors in 1:08:29; PostgreSQL closed connections during fixture startup and cleanup. Retrying only the cached errors passed all 17 checks in 16:12. These are passing results across runs; no single uninterrupted final full-suite pass is recorded.
- After a fresh read-only PostgreSQL probe succeeded, the targeted retry passed all three checks: fresh-install workflow/reconciliation, receipt API suspense/PDC/cancellation/dishonour/gateway workflow, and returned-cheque fee journal/tender regression (3 passed). The final suite now collects 144 tests, all with passing results across the full run and targeted retries.
- Release configuration/contract/live-check unit subset: 17 passed.
- Import boundaries: all three contracts kept.
- Frontend: typecheck/build/Worker artifact checks and all 11 tests passed; lint has zero errors and five existing image warnings.
- Cloudflare: Wrangler dry-run validated package and ASSETS/IMAGES bindings without deployment.
- Local built frontend: landing, staff and admin routes returned HTTP 200 HTML.
- Browser automation could not initialize because its Windows sandbox helper failed; visual QA is unverified. Docker CLI is unavailable, so an actual container build/Render startup remains a hosted gate.
- The reference member contract matches all 35 captured operations. The app team's actual F&B-only build was not available for testing.

## Rumanza F&B pilot

Deploy the pilot with the profile in `profiles/pilot.env` (the same modules and features in `backend/` and `frontend/`; the frontend copy also names the brand):

- API: the `ims-api-pilot` service in `render.yaml` (its own PostgreSQL database; `IMS_MODULES` / `IMS_FEATURES` from the profile). Two free Render services share one account's free hours, so keep only one running all the time.
- Worker: `IMS_PROFILE=pilot` picks the profile's modules and features at build time. In Cloudflare Workers Builds (Git-connected), set the build variables `IMS_PROFILE=pilot` and `NEXT_PUBLIC_API_URL=https://<pilot-api>/api/v1`, then retry the build; a Workers Build without `IMS_PROFILE` fails rather than shipping every module. From a PC: `IMS_PROFILE=pilot NEXT_PUBLIC_API_URL=https://<pilot-api>/api/v1 npm run deploy` (the predeploy check refuses a build that doesn't match the profile). The Worker name stays the one in `wrangler.jsonc`; to deploy the pilot as the separate `ims-fnb-pilot` Worker instead, also set `CLOUDFLARE_ENV=pilot` for the build (the Cloudflare Vite plugin picks the environment at build time, not with `wrangler deploy --env`).
- Branding: `IMS_BRAND=rykards` in `frontend/profiles/pilot.env` makes the Worker present as Rykards, with the product named "Inventory Management System" (sidebar, entry screens, tab title, export file names); the full profile stays Rumanza ("Club Management System"). The brands are in `frontend/app/core/brands.json`. The Rykards emblem is `frontend/public/brand/rykards-mark.png` (also the tab icon, `rykards-favicon.png`); a brand can instead set a wide `logo` image, or neither and get a monogram tile. Purchase orders, Z-reports, statements and receipts keep the club's name: they are the club's own documents.
- First start: bootstrap the Manager as for the full release, then in POS setup set the Member guest type's discount, the checkout discount limit and the F&B tax rate, and map every menu section to a kitchen station (unmapped items can't be ordered).
- Demo data instead (for showing the pilot, never on a database with real data): from `backend/`, with `DATABASE_URL`, `IMS_ENVIRONMENT=staging` and the profile's `IMS_MODULES` / `IMS_FEATURES` set, run `python -m scripts.seed --password <12+ characters>` instead of bootstrapping (it creates the Manager). It adds the demo staff (manager@, kitchen.cafe@, kitchen.rooftop@, kitchen.bar@, fnbmanager@, fnbstore@, finance@rumanza.pk, all with that password; each kitchen login works in its own venue), vendors, ingredients and opening stock, menus with recipes, kitchen requests and waste, a week of rosters and attendance, kitchen stations per venue (Pakistani, Chinese, Bakery and Coffee Bar paper-only, …), a 10% member discount, a week of paid sales and today's open orders at every KOT stage. `--no-showcase` leaves out the stations, discount and till activity. The seed is one transaction; on a slow or distant database it can take ten minutes or more, and a dropped connection rolls it back, so run it again.
- Checks: `pytest tests/release/test_pilot_profile.py` (pilot on its own and the upgrade to the full product), and a browser check that no member, payroll, ledger or tax screens appear for any role.
- Adding modules later: add them (or features) to the profile, rebuild the Worker and restart the API on the same database; then run `POST /cafe/member-refs/link` if members was added. See `docs/PILOT_PLAN.md` §6.
