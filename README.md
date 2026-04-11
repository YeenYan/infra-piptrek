# pipTrek Infrastructure

Docker-based development environment for the pipTrek project.

## Project Overview

pipTrek is a full-stack application with a Laravel backend (GraphQL API) and a Vue 3 frontend (SPA). This repository orchestrates both services along with PostgreSQL using Docker Compose.

## Repository Structure

```
infra-pipTrek/
  docker/
    be/Dockerfile        # Backend Docker image
    fe/Dockerfile        # Frontend Docker image
  pipTrek-be/            # Laravel backend (Git submodule)
  pipTrek-fe/            # Vue 3 frontend (Git submodule)
  docker-compose.yml     # Service orchestration
  README.md
```

## Prerequisites

- [Docker](https://www.docker.com/) (with Docker Compose V2)
- [Git](https://git-scm.com/)

## Setup

```bash
# 1. Clone the repository with submodules
git clone --recurse-submodules <repo-url>

# 2. Navigate to the project
cd infra-pipTrek

# 3. Initialize submodules (if not already done)
git submodule update --init --recursive

# 4. Build and start all services
docker compose up --build
```

## Services

| Service    | URL                   | Description          |
| ---------- | --------------------- | -------------------- |
| Backend    | http://localhost:8000 | Laravel API          |
| Frontend   | http://localhost:5173 | Vue 3 SPA (Vite dev) |
| PostgreSQL | localhost:5432        | Database             |

## Useful Commands

```bash
# Start services in background
docker compose up -d

# Stop all services
docker compose down

# Rebuild images
docker compose build

# View logs
docker compose logs -f

# View logs for a specific service
docker compose logs -f be
```

## Notes

- **vendor** and **node_modules** are persisted via Docker named volumes
- Database migrations run automatically on backend startup
- `.env` is auto-generated from `.env.example` if missing
- `APP_KEY` is auto-generated if not set
- PostgreSQL data is persisted in the `postgres_data` volume

## Environment Variables

The backend uses these environment variables (set in docker-compose.yml):

| Variable      | Value    |
| ------------- | -------- |
| DB_CONNECTION | pgsql    |
| DB_HOST       | postgres |
| DB_PORT       | 5432     |
| DB_DATABASE   | piptrek  |
| DB_USERNAME   | postgres |
| DB_PASSWORD   | admin    |

## Troubleshooting

### PostgreSQL: `role "xxx" does not exist` / `password authentication failed`

This happens when the PostgreSQL volume was initialized with different credentials (`POSTGRES_USER` / `POSTGRES_PASSWORD`) than what's currently in `docker-compose.yml`. PostgreSQL only creates the user on **first initialization**—changing env vars afterwards has no effect on the existing volume.

**Fix:** Destroy the volume and let PostgreSQL reinitialize:

```bash
docker compose down -v
docker compose up --build -d
```

> **Warning:** `-v` removes **all** named volumes, including `postgres_data`. This deletes all database data.

---

### Backend: `.env` overrides `docker-compose.yml` environment

The backend `.env` file is bind-mounted into the container and takes precedence over the `environment` block in `docker-compose.yml`. If credentials differ between the two, the `.env` values win.

**Fix:** Ensure `DB_DATABASE`, `DB_USERNAME`, and `DB_PASSWORD` in `pipTrek-be/.env` match the `POSTGRES_*` values in `docker-compose.yml`.

---

### Frontend: `ERR_EMPTY_RESPONSE` / page won't load

Vite's dev server defaults to binding on `127.0.0.1` inside the container, which is not reachable from the host through Docker's port mapping.

**Fix:** Ensure `vite.config.ts` includes:

```ts
server: {
  host: '0.0.0.0',
  port: 5173,
}
```

---

### Migration: `cannot cast type bigint to uuid`

A migration is trying to cast an integer column (e.g. `role_id`) to UUID. This fails because the column contains integer values (1, 2, 3…) that aren't valid UUIDs.

**Fix:** Only cast columns that actually contain UUID-formatted data. Columns like `role_id` and `permission_id` that reference `bigIncrements` primary keys must stay as `bigint`.

---

### General: containers keep restarting

Check which container is failing:

```bash
docker compose logs -f
```

If the backend is crash-looping due to a migration or DB error, fix the root cause, then restart:

```bash
docker compose down
docker compose up --build -d
```
