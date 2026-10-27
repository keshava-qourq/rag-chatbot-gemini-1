# Developer Guide

> Maintained by QuorQ AI SDLC. Text between `sdlc:auto` markers is regenerated; anything outside them is yours and is kept.

<!-- sdlc:auto:start status -->
_Up to date._
<!-- sdlc:auto:end status -->

<!-- sdlc:auto:start source -->
Generated from commit `4dde68e` · model `auto`
<!-- sdlc:auto:end source -->

<!-- sdlc:auto:start overview -->
RAG-chatbot-Gemini-1 is a generated two-app repository for a RAG chatbot with chat history and session IDs. The README describes it as using gemini-2.5-flash for answers and gemini-embedding-001 for embeddings, but no Gemini code is in the included files. `frontend/` is a Vite 6 + React 19 single-page app using react-router-dom 7 and Tailwind CSS 4. Per its README, its screens render seeded sample data. `src/lib/api.ts` provides an `apiFetch` helper for wiring screens to the real API later. `backend/` is a FastAPI 0.115 service with SQLAlchemy 2 and Pydantic 2. It currently exposes only `GET /health`. It defaults to a local SQLite database (`DATABASE_URL`, default `sqlite:///./app.db`) and creates tables from the models at startup. `docker-compose.yml` provides Postgres 16 and Redis 7 containers for local use.
<!-- sdlc:auto:end overview -->

<!-- sdlc:auto:start prerequisites -->
- Node.js and npm (no version is pinned; Vite 6 / React 19 tooling)
- Python 3.12 (the version used by backend/Dockerfile) with pip
- Docker and Docker Compose (optional: Postgres/Redis via docker-compose.yml, or building the backend image)
<!-- sdlc:auto:end prerequisites -->

<!-- sdlc:auto:start setup -->
1. cd backend && pip install -r requirements.txt
2. cd backend && uvicorn app.main:app --reload   # API on http://localhost:8000, health check at /health
3. cd frontend && npm install
4. cd frontend && npm run dev   # Vite dev server; the backend's default CORS allows http://localhost:5173
5. Optional: docker compose up -d   # starts Postgres 16 (user/password/db: app) on 5432 and Redis 7 on 6379; point DATABASE_URL at it to stop using SQLite
<!-- sdlc:auto:end setup -->

<!-- sdlc:auto:start run -->
| Name | Command | Description |
|---|---|---|
| backend dev | `cd backend && uvicorn app.main:app --reload` | Runs the FastAPI app with auto-reload on port 8000. |
| frontend dev | `cd frontend && npm run dev` | Starts the Vite dev server. |
| frontend build | `cd frontend && npm run build` | Bundles the SPA with vite build. This does not typecheck. |
| frontend preview | `cd frontend && npm run preview` | Serves the built bundle locally. |
| services | `docker compose up -d` | Starts the Postgres (db) and Redis (redis-queue) containers defined in docker-compose.yml. |
| backend image | `docker build -t rag-chatbot-backend backend && docker run -p 8000:8000 rag-chatbot-backend` | Builds and runs the backend container. The image tag is arbitrary. The container runs uvicorn on 0.0.0.0:8000 as a non-root user, with a /health HEALTHCHECK. |
| format frontend | `cd frontend && npm run format` | Rewrites frontend files with Prettier. |
| format backend | `cd backend && ruff format .` | Formats backend Python with Ruff. |
| lint fix | `cd frontend && npm run lint:fix` | Applies the ESLint fixes it can. |
<!-- sdlc:auto:end run -->

<!-- sdlc:auto:start structure -->
| Path | Purpose |
|---|---|
| `backend/` | FastAPI service, with its Dockerfile, requirements.txt, pytest.ini and ruff.toml. |
| `backend/app/` | Application package: main.py (app, CORS, /health), database.py (engine, SessionLocal, Base, get_db), models.py (SQLAlchemy models, none yet), schemas.py (Pydantic models incl. StubResponse), services.py (provides configured_services() used by /health), routers/ (generated routers, currently empty). |
| `backend/tests/` | pytest suite (test_health.py). |
| `frontend/` | Vite + React SPA, with package.json, vite.config.ts, tsconfig.json, eslint.config.js, .prettierrc and playwright.config.ts. |
| `frontend/src/` | App source: main.tsx entry point (BrowserRouter + App), index.css, test-setup.ts, and lib/ (api.ts API client, ui.tsx components, icons.tsx, navigate.ts). |
| `frontend/e2e/` | Playwright end-to-end specs (example.spec.ts). |
| `.quorq/` | Platform metadata (handoff.json). |
<!-- sdlc:auto:end structure -->

<!-- sdlc:auto:start conventions -->
- Frontend imports use the `@/` alias for `src/` (configured in vite.config.ts and tsconfig.json).
- Component tests (Vitest + Testing Library, jsdom, globals) live beside components as `src/**/*.{test,spec}.{ts,tsx}`. Playwright specs live in `e2e/` only.
- TypeScript runs in `strict` mode but with `noImplicitAny` and `strictNullChecks` disabled.
- Frontend is checked with ESLint and formatted with Prettier. Backend is linted and formatted with Ruff.
- Frontend API calls are meant to go through `apiFetch` in src/lib/api.ts. It prefixes VITE_API_BASE_URL (default http://localhost:8000) and adds a Bearer token when VITE_API_SECRET_KEY is set.
- Backend is generated with one router per architecture component. Generated routes are stubs returning `StubResponse` (status `not_implemented`) until implemented.
- Database access uses the request-scoped `get_db` dependency. All models inherit `Base` from app/database.py.
- There are no migrations: `Base.metadata.create_all` runs at startup. The code comments recommend replacing this with Alembic before storing real data.
- Configuration comes from env vars: DATABASE_URL and ALLOWED_ORIGINS (comma-separated CORS origins) for the backend; VITE_API_BASE_URL, VITE_API_SECRET_KEY and E2E_BASE_URL for the frontend. A root .env.example exists, but its contents were not reviewed.
<!-- sdlc:auto:end conventions -->

<!-- sdlc:auto:start testing -->
| Name | Command | Description |
|---|---|---|
| backend tests | `cd backend && pytest` | Runs the pytest suite in tests/. pytest.ini sets pythonpath so `app` is importable. |
| backend lint | `cd backend && ruff check .` | Lints the backend with Ruff. |
| frontend unit | `cd frontend && npm test` | Runs Vitest component tests under src/. Passes when no tests exist. |
| frontend lint | `cd frontend && npm run lint` | Runs ESLint over the app. The platform uses this as a gate. |
| format check | `cd frontend && npm run format:check` | Verifies Prettier formatting without writing changes. |
| typecheck | `cd frontend && npm run typecheck` | Runs tsc --noEmit. This is not part of the build. |
| e2e install | `cd frontend && npm run test:e2e:install` | Downloads the Chromium browser for Playwright. |
| e2e | `cd frontend && npm run test:e2e` | Runs the Playwright specs in e2e/. Without E2E_BASE_URL, it installs and starts the backend on 127.0.0.1:8000, builds and previews the frontend on 127.0.0.1:4173, and tests against those. With E2E_BASE_URL set, it tests that URL instead. |
<!-- sdlc:auto:end testing -->

<!-- sdlc:auto:start troubleshooting -->
| Problem | Solution |
|---|---|
| `npm run dev` or `npm run build` fails to resolve `@/App`. | main.tsx imports `@/App`, and the frontend README refers to src/App.tsx, src/screens/ and src/lib/brand.ts, but none of these appear in the file tree. Restore or create src/App.tsx (and the screens it routes to) before running the frontend. |
| `npm run typecheck` reports errors even though the app builds. | This is expected: `vite build` does not typecheck, and the README notes generated screens may fail tsc (e.g. an untyped `useState({})`). Add types to fix it. |
| Browser blocks API calls with CORS errors. | The backend only allows http://localhost:5173 and http://127.0.0.1:5173 by default. Set ALLOWED_ORIGINS (comma-separated) to the frontend's actual origin. |
| Frontend calls the wrong API host. | Set VITE_API_BASE_URL at build or dev time. Otherwise it defaults to http://localhost:8000. |
| `pytest` cannot import `app`. | Run pytest from the backend/ directory so pytest.ini (pythonpath = .) applies. |
| Playwright fails because no browser is installed. | Run `cd frontend && npm run test:e2e:install` first. |
| Data unexpectedly lives in backend/app.db, or the Postgres container sits unused. | The backend uses SQLite unless DATABASE_URL is set. Point DATABASE_URL at the docker-compose Postgres (user/password/db `app`, port 5432; psycopg is installed). |
| docker compose fails because ports 5432 or 6379 are in use. | Stop the local Postgres/Redis using those ports, or change the host port mappings in docker-compose.yml. |
<!-- sdlc:auto:end troubleshooting -->
