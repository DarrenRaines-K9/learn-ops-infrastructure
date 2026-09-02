# Tech Stack (AI)

## 1. Run Questions

### 1a. Config Files

| Config File | Location | Config Value | What it's for | How it's used |
|---|---|---|---|---|
| `.env` | `learn-ops-api/.env` | `LEARN_OPS_HOST=database`, `LEARN_OPS_PORT=5432`, `LEARN_OPS_DB=learningplatform` | Django/Postgres connection, GitHub OAuth, Valkey cache, Slack tokens | Loaded by `docker-compose.yml` (`env_file`) into the `api` container; read in `LearningPlatform/settings.py` via `os.getenv(...)` for `SECRET_KEY`, `DATABASES`, `ALLOWED_HOSTS` |
| `.env` (secrets, not shown) | `learn-ops-api/.env` | `LEARN_OPS_DJANGO_SECRET_KEY=<random string>`, `LEARN_OPS_SECRET_KEY=<GitHub OAuth secret>`, `GITHUB_TOKEN=<GitHub PAT>`, `SLACK_TOKEN=<Slack bot token>` | Django cryptographic signing, GitHub OAuth login, GitHub API + Slack bot integrations | `SECRET_KEY` signs sessions/CSRF tokens; OAuth values used in the GitHub login flow; tokens used by API views that call GitHub/Slack |
| `.env.template` | `learn-ops-api/.env.template` | `LEARN_OPS_CLIENT_ID=replace_me`, `VALKEY_HOST=valkey`, `DEBUG=True` | Checked-in placeholder showing which env vars `learn-ops-api` expects | Devs copy this to `.env` and fill in real values; keeps real secrets out of git (`.gitignore` excludes `.env*` but keeps `!.env.template`) |
| `.env` | `learn-ops-client/.env` | `REACT_APP_API_URI=http://localhost:8000`, `REACT_APP_ENV="development"`, `GENERATE_SOURCEMAP=false` | Points the React app at the API and controls CRA build behavior | Injected at build/start time by Create React App (`react-scripts`); `REACT_APP_*` vars are inlined into the client bundle |
| `.env` | `learn-ops-infrastructure/.env` | `POSTGRES_DB=learningplatform`, `POSTGRES_USER=learnops`, `POSTGRES_PASSWORD=learnops123` | Bootstraps the shared Postgres container and `postgres_exporter` | Consumed via `env_file: ".env"` in `docker-compose.yml` for the `database` and `postgres_exporter` services |
| `.env` (secret) | `service-monarch/.env` | `GH_PAT=<GitHub PAT>`, `SLACK_WEBHOOK_URL=<Slack webhook>`, `SLACK_TOKEN=<Slack bot token>` | Auth for GitHub API polling and Slack notifications from the Monarch service | Loaded by `env_file: ".env"` in `service-monarch/docker-compose.yml`; read in `service/config/settings.py` via `os.getenv(...)` into the pydantic `Settings` model |
| `docker-compose.yml` | `learn-ops-infrastructure/docker-compose.yml` | `database` on port `5433:5432`, `api` on `8000:8000`/`5678:5678`, `client` on `3000:3000` | Orchestrates the full local stack: Postgres, Django API, React client, Prometheus, Grafana, postgres_exporter | `docker compose up` builds/starts each service on the shared `learningplatform` network, wiring `env_file`s and volume mounts from the sibling repos |
| `docker-compose.yml` | `learn-ops-infrastructure/valkey/docker-compose.yml` | `valkey` image on port `6379:6379`, `valkey-server --save 900 1`, `valkey-monitor` sidecar | Runs the Valkey (Redis-compatible) cache used by `learn-ops-api` | Started separately from the main stack; `learn-ops-api` connects to it via `VALKEY_HOST`/`VALKEY_PORT` env vars |
| `docker-compose.yml` | `service-monarch/docker-compose.yml` | `monarch` service, ports `8080:8080` (Prometheus metrics), `8081:8081` (log web UI) | Runs the standalone Monarch service in its own container | Builds from `service-monarch/Dockerfile`, joins the shared `learningplatform` network so it can reach other services |
| `prometheus.yml` | `learn-ops-infrastructure/prometheus.yml` | `scrape_interval: 15s`, job `django` scraping `api:8000` at `/metrics/metrics`, job `postgresql` scraping `postgres_exporter:9187` | Configures Prometheus metric scraping for observability | Mounted into the `prometheus` container in `docker-compose.yml` via `--config.file=/etc/prometheus/prometheus.yml` |
| `settings.py` | `learn-ops-api/LearningPlatform/settings.py` | `DEBUG=os.getenv("DEBUG","False")`, `ALLOWED_HOSTS=os.getenv("LEARN_OPS_ALLOWED_HOSTS")`, `CORS_ORIGIN_WHITELIST=(...)` | Central Django settings: security, CORS, DB, DRF, logging | Read at Django startup; pulls most values from `.env` via `os.getenv`, with `CORS_ORIGIN_WHITELIST` hardcoding `learningapi.nss.team`, `learning.nss.team` |
| `settings.py` (pydantic) | `service-monarch/service/config/settings.py` | `GITHUB_API_URL=https://api.github.com`, `GITHUB_RATE_LIMIT_PAUSE=5`, `PROMETHEUS_PORT=8080` | Typed configuration model for the Monarch service | `Settings(BaseModel)` reads `GH_PAT`/`SLACK_BOT_TOKEN` from `.env` (via pydantic `Config.env_file`) and hardcodes rate-limit/pause/port defaults |
| `pytest.ini` | `learn-ops-api/pytest.ini` | `DJANGO_SETTINGS_MODULE=LearningPlatform.test_settings`, `testpaths=LearningAPI/tests`, `addopts=--reuse-db --nomigrations` | Configures pytest/pytest-django for the API's test suite | Picked up automatically by `pytest`; points tests at a dedicated `test_settings` module and speeds up runs by reusing the test DB |
| `package.json` | `learn-ops-client/package.json` | `"engines": {"node": "22.13.0"}`, `"start": "react-scripts start"`, `"build:production": "env-cmd -f .env.production npm run build"` | Node/React app manifest: dependencies, scripts, target Node version | `npm install`/`npm start` read this; `env-cmd` scripts swap in environment-specific `.env` files for different build targets |
| `learn-ops-api.yaml` | `learn-ops-api/config/learn-ops-api.yaml` | `region: nyc`, `instance_size_slug: basic-xxs`, `github.branch: main` | DigitalOcean App Platform deployment spec for the API | Used by DigitalOcean's `doctl`/App Platform to provision the droplet, database, and auto-deploy on push to `main` |
| `nginx.api.conf` | `learn-ops-api/config/nginx.api.conf` | `server_name learningapi.nss.team`, `proxy_pass http://127.0.0.1:8000`, `listen 443 ssl` | Production nginx reverse proxy + TLS termination for the API | Deployed to the production host; routes HTTPS traffic on `learningapi.nss.team` to the Django app running on `127.0.0.1:8000` |
| `nginx.client.conf` | `learn-ops-api/config/nginx.client.conf` | `server_name learning.nss.team`, `root /home/.../learn-ops-client/build`, `try_files $uri $uri/ /index.html` | Production nginx config serving the built React app | Serves static files from the CI-built `build/` directory and falls back to `index.html` for client-side routing |
| `Dockerfile` | `learn-ops-api/Dockerfile` | `FROM python:3.11.11`, `EXPOSE 8000`, `CMD ["python3","manage.py","runserver","0.0.0.0:8000"]` | Builds the Django API container image | Used by `docker-compose.yml`'s `api.build` to create the image; installs deps via `pipenv` and runs the dev server |
| `Dockerfile` | `learn-ops-client/Dockerfile` | `FROM node:22.13.0`, `EXPOSE 3000`, `CMD ["npm","start"]` | Builds the React client container image | Used by `docker-compose.yml`'s `client.build`; runs `npm install` then the CRA dev server |
| `Dockerfile` | `service-monarch/Dockerfile` | `FROM python:3.11-slim`, `EXPOSE 8080`/`8081`, `CMD ["python","service/main.py"]` | Builds the Monarch service container image | Used by `service-monarch/docker-compose.yml`'s `monarch.build`; installs `requirements.txt` then runs the service entrypoint |

### 1b. How to Start It

The root [`Makefile`](Makefile) wraps `docker compose` (and two setup scripts) into named shortcuts. First-time setup is separate from the targets that actually start the running system:

**One-time / maintenance targets (not part of normal startup):**

| Target | Command it runs | What it does |
|---|---|---|
| `make setup` | `./scripts/setup.sh` | First-run wizard: checks/installs Docker, clones the sibling repos (`learn-ops-api`, `learn-ops-client`, `service-monarch`) into `~/workspace/lms`, and interactively writes their `.env` files (prompts for GitHub PAT, Slack tokens, DB creds, etc). Run once per machine before any `up*` target will work. |
| `make doctor` | `./scripts/setup.sh --doctor` | Same prerequisite checks as `setup`, but read-only — reports problems (missing Docker, bad versions, port conflicts) without cloning repos or writing `.env` files. Useful for diagnosing "why won't it start" without risking changes. |
| `make teardown` | `./scripts/teardown.sh` | Destructive: reverses `setup` — stops/deletes all containers, volumes and the `learningplatform` network, uninstalls Docker itself, deletes the cloned repos and their `.env` files, and prompts you to revoke the GitHub PAT. Not a "stop the system" command — use `make down` for that. |

**Targets that start the system, from smallest to largest scope:**

| Target | Command it runs | Scope | When to use it |
|---|---|---|---|
| `make up-api` | `docker compose up --build -d api` (then follows `api` logs) | Starts only the `api` service — but since `api` `depends_on: database` (with a healthcheck), Postgres is started too. Client, Prometheus, and Grafana stay down. | Backend-only work, e.g. testing API endpoints or running Django management commands without needing the React UI. |
| `make up-client-api` | `docker compose up --build -d api client` (then follows `api` + `client` logs) | Starts `api` (+ `database`, transitively) and `client`. Prometheus/Grafana/postgres_exporter stay down. | Full-stack feature work in the app itself, when you don't need the observability stack. |
| `make up` | `make pull` then `docker compose up --build -d`, then follows **all** logs | Starts every service defined in `docker-compose.yml`: `database`, `api`, `client`, `prometheus`, `grafana`, `postgres_exporter`. `pull` first refreshes any pre-built images (Postgres, Prometheus, Grafana). | The default "start everything" command — use this when you want the full stack including metrics/dashboards, or aren't sure which pieces you need. |
| `make restart` | `make pull`, then `docker compose down`, then `docker compose up --build -d`, then follows all logs | Same end state as `make up`, but forces a full stop/recreate first instead of reusing already-running containers. | After changing `docker-compose.yml`, `.env` files, or when containers are in a bad state and a plain `up` isn't picking up the change. |

**Supporting targets** (not for starting, but used alongside the above): `make down` stops the stack (keeps volumes), `make logs` tails logs for whatever's already running, `make ps` lists container status, `make reset` is `docker compose down -v --remove-orphans` (stops everything **and deletes volumes**, i.e. wipes the database) — use it when you want a clean-slate restart, not just a stop.

The Valkey cache (`learn-ops-infrastructure/valkey/docker-compose.yml`) is **not** wired into the Makefile at all — it's a separate compose file started manually with `docker compose -f valkey/docker-compose.yml up -d` if `learn-ops-api` needs the cache running.

### 1c. Where to Access It

| Service | Port | URL |
|---|---|---|
| React client (`learn-ops-client`) | `3000` (host) → `3000` (container) | http://localhost:3000 |
| Django API (`learn-ops-api`) | `8000` (host) → `8000` (container) | http://localhost:8000 |
| API remote debugger (debugpy, `learn-ops-api`) | `5678` (host) → `5678` (container) | N/A — attach a debugger client to `localhost:5678`, not a browser URL |
| Postgres (`database`) | `5433` (host) → `5432` (container) | N/A — connect with a DB client, e.g. `postgresql://learnops:learnops123@localhost:5433/learningplatform` |
| Valkey cache (`valkey`, separate compose file) | `6379` (host) → `6379` (container) | N/A — connect with a Redis-compatible client, e.g. `redis://localhost:6379` |
| Prometheus | `9090` (host) → `9090` (container) | http://localhost:9090 |
| Grafana | `3001` (host) → `3000` (container) | http://localhost:3001 |
| postgres_exporter | `9187` (host) → `9187` (container) | http://localhost:9187/metrics |
| Monarch service — Prometheus metrics (`service-monarch`) | `8080` (host) → `8080` (container) | http://localhost:8080/metrics |
| Monarch service — log web interface (`service-monarch`) | `8081` (host) → `8081` (container) | http://localhost:8081 |

Production (via nginx, from [`nginx.api.conf`](../learn-ops-api/config/nginx.api.conf) and [`nginx.client.conf`](../learn-ops-api/config/nginx.client.conf)): API at https://learningapi.nss.team (proxies to `127.0.0.1:8000`), client at https://learning.nss.team (serves the static `build/` output).

### 1d. Service Dependencies

Some of these are enforced at startup by `depends_on` in a `docker-compose.yml`; others are functional dependencies the code makes at request-time (compose will happily start the container, but a call will fail without the dependency reachable). Both are included since either kind can stop the system from actually working.

| Service | Depends On | Why |
|---|---|---|
| React client (`learn-ops-client`) | Django API | It has no data or logic of its own — every screen calls the API (via `REACT_APP_API_URI`) for auth, courses, cohorts, etc. |
| Django API (`learn-ops-api`) | Postgres (`database`) | The Django ORM is the system's source of truth (users, cohorts, courses); enforced by compose (`depends_on: database, condition: service_healthy`) so the API waits for a working DB before it boots. |
| Django API (`learn-ops-api`) | Valkey | Used as a request-time cache (e.g. `popular_queries`, team-matching results) to avoid recomputation — not required to boot, but those endpoints fail without it. |
| Django API (`learn-ops-api`) | GitHub API (external) | Backs the GitHub OAuth login flow and admin actions (checking org membership, managing classroom repos) using `GITHUB_TOKEN`. |
| Django API (`learn-ops-api`) | Slack (external) | Sends notifications (e.g. cohort/team events) via the Slack bot token. |
| Monarch (`service-monarch`) | GitHub API (external) | Its core job is polling and migrating GitHub repos — everything it does is a GitHub API call made with `GH_PAT`. |
| Monarch (`service-monarch`) | Valkey | Valkey *is* Monarch's datastore, not just a cache — migration state, pub/sub messaging, heartbeats, and its own log stream all live there; without it Monarch can't track progress or serve its log web UI. |
| Monarch (`service-monarch`) | Slack (external) | Posts migration status updates via the Slack webhook/bot token. |
| `postgres_exporter` | Postgres (`database`) | Queries Postgres directly to expose DB-level metrics for Prometheus; enforced by compose (`depends_on: - database`). |
| Prometheus | Django API (`api`) | `prometheus.yml` scrapes `api:8000/metrics/metrics` for application metrics; enforced by compose (`depends_on: - api`). |
| Prometheus | `postgres_exporter` | `prometheus.yml` also scrapes `postgres_exporter:9187` for DB metrics — a functional dependency the compose file doesn't declare, so Prometheus can start before the exporter is ready and just show a gap until it scrapes successfully. |
| Grafana | Prometheus | Grafana has no data of its own — its dashboards query Prometheus as the data source; enforced by compose (`depends_on: - prometheus`). |

### 1e. Main Entry Points

| Service | Startup File | Routes / URL Config File |
|---|---|---|
| Django API (`learn-ops-api`) | `learn-ops-api/manage.py` (dev server, run by the Dockerfile's `CMD`), `learn-ops-api/LearningPlatform/wsgi.py` (WSGI entry point for production/gunicorn) | `learn-ops-api/LearningPlatform/urls.py` (root URLconf — registers the DRF router and includes app-level URLs), `learn-ops-api/LogViewer/urls.py` (included at `logs/` for in-app log inspection) |
| React client (`learn-ops-client`) | `learn-ops-client/src/index.js` | `learn-ops-client/src/components/ApplicationViews.js` |
| Monarch (`service-monarch`) | `service-monarch/service/main.py` | `service-monarch/service/custom_logging/web_interface.py` (Flask routes for the log web interface: `/`, `/health`, `/api/logs`, `/api/log-levels`, `/api/services`) |
| Postgres (`database`) | Not found — off-the-shelf `postgres:16` image, no application code in this repo | Not found — no HTTP routes; accessed via the Postgres wire protocol, not URL routing |
| Valkey (`valkey`) | Not found — off-the-shelf `valkey/valkey:latest` image, no application code in this repo | Not found — no HTTP routes; accessed via the Redis protocol, not URL routing |
| Prometheus | Not found — off-the-shelf `prom/prometheus:latest` image, configured only via `learn-ops-infrastructure/prometheus.yml` (not a startup/entry file) | Not found — no app-defined routes; its own built-in web UI/API, not something this repo defines |
| Grafana | Not found — off-the-shelf `grafana/grafana:latest` image, no application code in this repo | Not found — no app-defined routes; its own built-in web UI, not something this repo defines |
| `postgres_exporter` | Not found — off-the-shelf `quay.io/prometheuscommunity/postgres-exporter` image, no application code in this repo | Not found — exposes a fixed `/metrics` endpoint built into the image, not defined in this repo |

## 2. Services

| Service Name | Tech Stack (including version) | Purpose |
|---|---|---|
| Django API (`learn-ops-api`) | Python 3.11.11; Django 5.2.17; Django REST Framework 3.18.0; django-allauth 0.54.0 + dj-rest-auth 4.0.1 (GitHub OAuth); psycopg2-binary 2.9.12 (Postgres driver); valkey 6.1.1 (cache client); gunicorn 26.1.0 (WSGI server); django-cors-headers 4.9.0; django-prometheus 2.5.0; structlog 23.1.0 + django-structlog 5.0.0 (logging) — versions from `learn-ops-api/Pipfile.lock` | Backend REST API for the learning platform: manages courses, cohorts, students, assessments, and GitHub-based auth/repo operations; the system of record backed by Postgres. |
| React client (`learn-ops-client`) | Node.js 22.13.0; React 16.13.1; React Router DOM 5.2.0; React Scripts (Create React App) 5.0.1; Chart.js 4.4.1 + react-chartjs-2 5.2.0; Radix UI themes/components 1.x — versions from `learn-ops-client/package.json` | Single-page frontend for staff and students to browse courses/cohorts, manage assessments, and view team/project data by calling the Django API. |
| Monarch (`service-monarch`) | Python 3.11 (`python:3.11-slim` base image, unspecified patch version); Flask 3.0.3 (log web interface); pydantic 2.10.4 (typed settings/models); valkey 6.0.2 (state store/pub-sub client); prometheus-client 0.21.1; structlog 24.4.0; tenacity 9.0.0 (retry logic); requests 2.32.3 — versions from `service-monarch/requirements.txt` | Standalone service that polls/migrates GitHub repositories using a GitHub PAT, tracks migration state and heartbeats in Valkey, exposes Prometheus metrics, and posts status updates to Slack. |
| Postgres (`database`) | PostgreSQL 16 (`postgres:16` image) | Primary relational datastore for the Django API — stores users, cohorts, courses, and all other application data. |
| Valkey cache (`valkey`) | Valkey (`valkey/valkey:latest` — unspecified pinned version) | Redis-compatible in-memory store; used by `learn-ops-api` as a request cache and by Monarch as its primary state/pub-sub/log store. |
| Prometheus | Prometheus (`prom/prometheus:latest` — unspecified pinned version) | Scrapes and stores time-series metrics from the Django API and `postgres_exporter` for observability. |
| Grafana | Grafana (`grafana/grafana:latest` — unspecified pinned version) | Dashboarding UI that visualizes metrics queried from Prometheus. |
| `postgres_exporter` | postgres_exporter (`quay.io/prometheuscommunity/postgres-exporter:latest` — unspecified pinned version) | Exposes Postgres internal metrics in Prometheus format so Prometheus can scrape database health/performance data. |

## 3. System Overview

This is a cohort-based learning management system built for a coding bootcamp (the `NssUser` model name and GitHub-Classroom-style workflow point to Nashville Software School), architected as a Django REST API backed by Postgres, a separate React single-page frontend, and a standalone GitHub-automation microservice (`service-monarch`) tied together over a shared Valkey cache/pub-sub channel. Its domain model reflects the day-to-day mechanics of running a bootcamp: a curriculum hierarchy of courses, books, and projects; cohorts of students who move through that curriculum on a schedule (including breaks); self- and instructor-driven assessments scored against weighted learning objectives; capstone projects with proposal/status tracking; and team/group-project formation with an auto-provisioned GitHub repo per team. It solves the problem of coordinating a curriculum, a rotating set of student cohorts, and repo-based project work — including the tedious mechanics of creating repos, adding student collaborators, and migrating issue-ticket templates into each new repo — without instructors and staff having to do that GitHub bookkeeping by hand.

From a user's perspective, the application lets students track their own progress through a personal dashboard: viewing a cohort event calendar, setting and reviewing personal learning goals, taking self-assessments, submitting client- and server-side project proposals, checking assessment requirements, and updating their linked Slack identity. Instructors get a much broader set of tools: creating and managing cohorts (including inviting/adding students), authoring the curriculum itself (courses, books, and projects), building weekly project teams from a cohort's students, creating and editing assessments, tracking foundations-exercise completion, and logging one-on-one feedback for a student. Underlying all of this, GitHub OAuth login also drives repo-centric features — cohort GitHub Classroom project links, per-team repositories, and (via the Monarch service) automatic migration of issue tickets from a template repo into each team's new repo once it's created.

Three roles use the system, distinguished by Django auth groups and an `is_staff` flag rather than a single role field, and each sees a different app entirely once logged in. Students are the default role with no elevated group membership, and are restricted to the self-service dashboard, calendar, goals, assessments, and proposal views described above. Staff members (flagged `is_staff` and placed in a `Staff` group) get a narrower, coaching-oriented view limited to foundations-exercise tracking, alongside admin-only API access to curriculum configuration endpoints. Instructors (members of the `Instructors` group, either bootstrapped via an `INSTRUCTOR_USERNAME` environment variable or granted by an existing instructor) get the full authoring experience — cohort management, curriculum editing, team-building, assessment creation, and student feedback — and are the only role permitted to create, update, or delete cohorts and similar protected resources at the API layer. Instructors can also flip a "mimic" toggle in the client to preview the app as a student would, for testing and support purposes, without leaving their own account.