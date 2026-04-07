# Med Tracker

Med Tracker is an open source web application for logging **as-needed (pro re nata / PRN)** medications. You define medications and how dosing is measured (dose per unit), then record when you took a dose and how much. The goal is a simple, self-hosted tool that works well on both phones and desktops.

The project is designed to run **locally via Docker** for the API and database, with a **React** front end for day-to-day use.

---

## Table of contents

- [Med Tracker](#med-tracker)
  - [Table of contents](#table-of-contents)
  - [Features](#features)
  - [Architecture](#architecture)
  - [Tech stack](#tech-stack)
  - [Repository layout](#repository-layout)
  - [Prerequisites](#prerequisites)
  - [Getting started](#getting-started)
    - [1. Clone the repository](#1-clone-the-repository)
    - [2. Start the database and API](#2-start-the-database-and-api)
    - [3. Run the front end (development)](#3-run-the-front-end-development)
    - [4. Production build of the front end (optional)](#4-production-build-of-the-front-end-optional)
  - [Configuration](#configuration)
    - [Environment variables](#environment-variables)
  - [API and documentation](#api-and-documentation)
  - [Front-end development](#front-end-development)
  - [Database](#database)
  - [Troubleshooting](#troubleshooting)
  - [Roadmap](#roadmap)
  - [License](#license)
  - [Maintainer](#maintainer)

---

## Features

**Implemented today**

- Health check endpoint for monitoring and smoke tests.
- Docker Compose stack: PostgreSQL plus the FastAPI service with live reload during development.

**Planned**

- Create and manage **medications** and **dose per unit** (for example, mg per tablet).
- **Log** each dose with a timestamp and quantity taken.
- **Calendar-style view** of all logged doses (future iteration).

---

## Architecture

Med Tracker follows a classic **three-tier** layout:

1. **Browser** — React (Vite) single-page app, responsive layout for mobile and desktop.
2. **API** — FastAPI application exposing JSON over HTTP.
3. **Data** — PostgreSQL for durable storage; SQLAlchemy models and Alembic migrations define the schema.

In **local development**, the front end usually runs on the Vite dev server (port **5173**) and talks to the API on port **8000**. A **dev proxy** maps browser requests to `/api/*` onto the API so you can use relative URLs and avoid CORS issues for those routes. The API also allows CORS from the Vite origin for direct calls if you prefer.

In **production** (not fully wired in this repository yet), you would typically build the static front end, serve it behind a reverse proxy or CDN, and run the API and database as separate services with secrets supplied by the environment.

---

## Tech stack

| Layer | Technology | Role |
|--------|------------|------|
| Runtime / containers | Docker, Docker Compose | API + database orchestration |
| API | [FastAPI](https://fastapi.tiangolo.com/) | HTTP API, validation, OpenAPI |
| Server | [Uvicorn](https://www.uvicorn.org/) | ASGI server for FastAPI |
| ORM & migrations | [SQLAlchemy](https://www.sqlalchemy.org/), [Alembic](https://alembic.sqlalchemy.org/) | Database access and schema migrations |
| Database | [PostgreSQL](https://www.postgresql.org/) 16 | Primary data store |
| Driver | [psycopg](https://www.psycopg.org/) 3 | PostgreSQL driver for Python |
| Config | [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) | Typed settings from environment variables |
| Front end | [React](https://react.dev/), [TypeScript](https://www.typescriptlang.org/) | UI |
| Build tool | [Vite](https://vite.dev/) | Dev server and production bundles |

---

## Repository layout

```
med-tracker/
├── backend/                 # FastAPI application
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── __init__.py
│       └── main.py          # Application entry and routes (grows over time)
├── frontend/                # Vite + React + TypeScript
│   ├── package.json
│   ├── vite.config.ts       # Dev proxy: /api → FastAPI
│   └── src/
├── docker-compose.yml       # PostgreSQL + API
├── .env.example             # Example environment variables
├── LICENSE
└── README.md                # Short project blurb (see also this file)
```

---

## Prerequisites

- **Docker** and **Docker Compose** (Compose V2 plugin or `docker compose`), for the database and API.
- **Node.js** (LTS recommended) and **npm**, for the front-end dev server and builds.

Optional but useful:

- `curl` or a browser for hitting `/health` and OpenAPI docs.
- A PostgreSQL client if you want to inspect the database directly.

---

## Getting started

### 1. Clone the repository

```bash
git clone <repository-url>
cd med-tracker
```

### 2. Start the database and API

From the repository root:

```bash
docker compose up --build
```

This starts:

- **PostgreSQL** on host port **5432** (default credentials are defined in `docker-compose.yml`; treat them as development-only).
- **FastAPI** on **http://localhost:8000** with `--reload` and the `backend` folder mounted into the container so code changes apply immediately.

Wait until the database health check passes and the API is listening. Verify the API:

```bash
curl -s http://localhost:8000/health
```

You should see JSON similar to `{"status":"ok"}`.

### 3. Run the front end (development)

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL Vite prints (typically **http://localhost:5173**). The dev server proxies requests under **`/api`** to **http://localhost:8000** (see `frontend/vite.config.ts`), so from the browser you can use paths like `/api/health` without configuring a separate API base URL for local development.

### 4. Production build of the front end (optional)

```bash
cd frontend
npm run build
```

The static output is written to `frontend/dist`. Serving that folder and wiring it to the same host as the API is a deployment concern you can add when you are ready.

---

## Configuration

### Environment variables

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | SQLAlchemy database URL for the API. |

Copy `.env.example` to `.env` and adjust if you run Alembic or other tools **on the host** against PostgreSQL bound to `localhost`. The Compose file sets `DATABASE_URL` inside the `api` service for container-to-container access to the `db` service.

Example for tools on the host talking to the exposed Postgres port:

```bash
DATABASE_URL=postgresql+psycopg://medtracker:medtracker@localhost:5432/medtracker
```

**Security note:** The default Compose credentials are for local development only. For any shared or internet-exposed deployment, use strong secrets, restrict network access, and avoid committing real passwords.

---

## API and documentation

- **Interactive docs (Swagger UI):** [http://localhost:8000/docs](http://localhost:8000/docs) when the API is running.
- **Alternative OpenAPI (ReDoc):** [http://localhost:8000/redoc](http://localhost:8000/redoc).

The current codebase exposes at least:

- `GET /health` — Liveness-style check returning `{"status": "ok"}`.

As features are added, new routes will appear here and in the OpenAPI schema automatically.

---

## Front-end development

- **Dev server:** `npm run dev` (Vite).
- **Lint:** `npm run lint`.
- **Preview production build:** `npm run build` then `npm run preview`.

**CORS:** The FastAPI app allows origins `http://localhost:5173` and `http://127.0.0.1:5173` so you can call the API directly from the Vite origin during development.

**Proxy:** Paths beginning with `/api` are proxied to the backend with the `/api` prefix removed. For example, a request from the browser to `http://localhost:5173/api/health` is forwarded to `http://localhost:8000/health`.

---

## Database

- **Engine:** PostgreSQL 16 (Alpine image in Compose).
- **Persistence:** A named Docker volume (`postgres_data`) stores database files so data survives container restarts unless you remove the volume.
- **Migrations:** Alembic is listed in `backend/requirements.txt` for schema evolution; migration layout can be added as models are introduced.

**Resetting local data** (destructive):

```bash
docker compose down -v
```

The `-v` flag removes the named volume and clears PostgreSQL data.

---

## Troubleshooting

| Issue | What to check |
|--------|----------------|
| API will not start | Ensure port **8000** is free, or change the published port in `docker-compose.yml`. |
| Database “unhealthy” | Wait a few seconds; if it persists, run `docker compose logs db`. |
| Front end cannot reach API | Confirm the API is up (`curl http://localhost:8000/health`). If using `/api/...` URLs, confirm the Vite dev server is running and `vite.config.ts` proxy matches your API port. |
| CORS errors when calling `http://localhost:8000` directly from the app | Either use the `/api` proxy on port 5173 or add your front-end origin to `CORSMiddleware` in `backend/app/main.py`. |
| Port **5432** already in use | Stop the conflicting Postgres instance or change the host port mapping for the `db` service. |

---

## Roadmap

- [ ] Data models and Alembic migrations for medications, units, and dose logs.
- [ ] REST (or similar) endpoints for CRUD and listing.
- [ ] Front-end screens to create medications and log doses.
- [ ] Calendar or timeline view of logged medications.

---

## License

This project is licensed under the **GNU General Public License v3.0**. See the [LICENSE](./LICENSE) file in the repository.

---

## Maintainer

**Andrew Cincotta** — Financial Analyst; project author.

For questions or contributions, use the repository’s issue tracker and pull requests according to project norms.
