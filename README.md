# IMS

Modular club management system: a shared **core** plus pluggable **modules**. The F&B first deployment
(F&B outlets and inventory, catalog, finance, members, F&B payroll and shifts) is ported from RumanzaCMS;
see [FNB_DEPLOYMENT_PLAN.md](FNB_DEPLOYMENT_PLAN.md) for the plan and the module rules.

```
backend/    FastAPI API (app/core + app/modules/<name>), tests, import-linter contracts
frontend/   React 19 / Vinext app for Cloudflare Workers (app/core + app/modules/<name>)
```

## Modules

A backend module is a package under `backend/app/modules/` whose `manifest.py` defines
`MODULE = Module(...)`: its routers, schema, seed, roles and what it registers with other modules
(journal sources, revenue sources, GRN line handlers, background tasks, approval policies, ...).
Its frontend half lives in `frontend/app/modules/<same name>/manifest.ts`.

`IMS_MODULES` (comma separated) picks the modules to run and build; leave it empty for all of them.
`core` is always on. A module whose dependency is not enabled stops the API at startup, and
`lint-imports` fails when a module imports one it does not list in `depends_on`.

Implemented modules: core, members, finance, catalog, F&B inventory, F&B outlets and F&B payroll/shifts.

## Backend

```bash
cd backend
python -m venv .venv
.venv/Scripts/python -m pip install -r requirements.txt   # .venv/bin/python on Linux/macOS
.venv/Scripts/python -m uvicorn app.main:app --reload
```

Settings go in `backend/.env` (see `.env.example`): at least `DATABASE_URL`, `RUMANZA_TOKEN_SECRET` and
`RUMANZA_SETUP_TOKEN`. On first start the API creates each enabled module's tables; an empty database
then asks for the first Manager (`POST /api/v1/setup/bootstrap` with the setup token). For a development
database with demo accounts (`manager@rumanza.pk` / `demo123`, plus each module's staff):

```bash
.venv/Scripts/python -m scripts.seed
```

Checks (database tests run in a throwaway schema on `DATABASE_URL` and are skipped without one):

```bash
.venv/Scripts/lint-imports
.venv/Scripts/python -m pytest
```

## Frontend

```bash
cd frontend
npm install
npm run dev
```

`npm run dev`, `build`, `lint` and `typecheck` first regenerate `app/modules.generated.ts` from
`IMS_MODULES` in the shell environment. `npm test` builds and runs `tests/*.test.mjs`.

## Docker

`docker compose up --build` runs the API on :8000 and the frontend on :3000; `IMS_MODULES` from the
shell is passed to both.

## Release rehearsal

Phase 6 prepares a Render free-tier Docker API and Cloudflare Worker frontend.
Use [the release runbook](docs/FNB_RELEASE.md) for private environment configuration,
first-Manager setup, local/live checks and the outstanding mobile-build acceptance.
The Render Blueprint is `backend/render.yaml`; the separate test Worker is configured in
`frontend/wrangler.jsonc`. Source rules and data flows are documented in
[ARCHITECTURE.md](ARCHITECTURE.md) and [the F&B DFD](docs/FNB_DFD.md).

`IMS_ENVIRONMENT=staging` (or `production`) enforces release secrets, HTTPS CORS,
no demo seeding and hidden API docs. `/ready` verifies PostgreSQL connectivity.
Deployment must rebuild the frontend with the actual Render `/api/v1` URL.
