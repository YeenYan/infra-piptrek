# pipTrek Infrastructure

Infrastructure repository for the **pipTrek** application. This repo orchestrates Docker-based containerization for the backend and frontend services, which are managed as Git submodules.

---

## Project Overview

**infra-pipTrek** is the DevOps/infrastructure layer that:

- Contains all Dockerfiles and Docker Compose configuration
- References the backend and frontend applications as Git submodules
- Provides a single command to build and run the entire stack
- Keeps infrastructure concerns separated from application code

| Component                   | Technology                                          | Description             |
| --------------------------- | --------------------------------------------------- | ----------------------- |
| **Backend** (`pipTrek-be`)  | Laravel 12, PHP 8.2, GraphQL (Lighthouse), JWT Auth | REST/GraphQL API server |
| **Frontend** (`pipTrek-fe`) | Vue 3.5, TypeScript, Vite 8, Pinia, Vue Router      | Single Page Application |
| **Infrastructure**          | Docker, Docker Compose                              | Container orchestration |

---

## Repository Structure

```
infra-pipTrek/
├── docker/
│   ├── be/
│   │   └── Dockerfile          # Backend multi-stage Dockerfile (PHP 8.2 + Apache)
│   └── fe/
│       └── Dockerfile          # Frontend multi-stage Dockerfile (Node 22 + Nginx)
├── pipTrek-be/                 # Git submodule → Laravel backend (READ-ONLY)
├── pipTrek-fe/                 # Git submodule → Vue frontend (READ-ONLY)
├── docker-compose.yml          # Service orchestration
├── .gitmodules                 # Submodule configuration
└── README.md                   # This file
```

> **Important:** `pipTrek-be/` and `pipTrek-fe/` are Git submodules. Do **not** modify files inside them from this repository. Changes to application code must be made in their respective repositories.

---

## Prerequisites

Ensure the following tools are installed on your machine:

| Tool               | Minimum Version                  | Verify Command           |
| ------------------ | -------------------------------- | ------------------------ |
| **Git**            | 2.13+                            | `git --version`          |
| **Docker Engine**  | 20.10+                           | `docker --version`       |
| **Docker Compose** | V2 (bundled with Docker Desktop) | `docker compose version` |

> **Note:** Docker Desktop for Windows/macOS includes Docker Compose V2 by default. On Linux, you may need to install the `docker-compose-plugin` package separately.

---

## Getting Started

### 1. Clone the Repository with Submodules

Clone the infrastructure repository and initialize both submodules in a single command:

```bash
git clone --recurse-submodules https://github.com/YeenYan/infra-pipTrek.git
cd infra-pipTrek
```

The `--recurse-submodules` flag automatically clones `pipTrek-be` and `pipTrek-fe` into their respective directories.

### 2. Initialize Submodules (if cloned without --recurse-submodules)

If you already cloned the repo without the flag, initialize the submodules manually:

```bash
git submodule update --init --recursive
```

To verify submodules are properly initialized:

```bash
git submodule status
```

You should see commit hashes next to `pipTrek-be` and `pipTrek-fe` (no `-` prefix, which would indicate uninitialized).

### 3. Set Up Environment Variables

The backend requires certain environment variables. Create a `.env` file in the repository root (used by Docker Compose):

```bash
# Generate a Laravel application key (base64 encoded, 32 bytes)
APP_KEY=base64:YOUR_GENERATED_KEY_HERE

# Generate a JWT secret for authentication
JWT_SECRET=YOUR_JWT_SECRET_HERE
```

**To generate these values locally (if you have PHP/artisan access):**

```bash
# APP_KEY — run inside the pipTrek-be directory or any Laravel installation
php artisan key:generate --show

# JWT_SECRET — if jwt-auth is installed
php artisan jwt:secret --show
```

**Without local PHP**, you can generate a random base64 key:

```bash
# Linux/macOS
echo "base64:$(openssl rand -base64 32)"

# Or use any random 64-character hex string for JWT_SECRET
openssl rand -hex 32
```

### 4. Build the Docker Images

Build both backend and frontend images:

```bash
docker compose build
```

This will:

1. **Backend:** Install Composer dependencies → Build Vite assets → Create PHP 8.2 + Apache image
2. **Frontend:** Install npm dependencies → Type-check and build Vue SPA → Create Nginx Alpine image

### 5. Start the Services

```bash
docker compose up -d
```

| Service              | URL                                                            | Description           |
| -------------------- | -------------------------------------------------------------- | --------------------- |
| **Backend**          | [http://localhost:8000](http://localhost:8000)                 | Laravel API           |
| **GraphQL Endpoint** | [http://localhost:8000/graphql](http://localhost:8000/graphql) | Lighthouse GraphQL    |
| **Health Check**     | [http://localhost:8000/up](http://localhost:8000/up)           | Backend health status |
| **Frontend**         | [http://localhost:3000](http://localhost:3000)                 | Vue SPA               |

---

## Running Services

### Start Services

```bash
# Start all services in detached mode (background)
docker compose up -d

# Start with real-time log output (foreground)
docker compose up
```

### Stop Services

```bash
# Stop and remove containers (preserves volumes/data)
docker compose down

# Stop and remove containers AND volumes (destroys data)
docker compose down -v
```

### Restart Services

```bash
# Restart all services
docker compose restart

# Restart a specific service
docker compose restart backend
docker compose restart frontend
```

### Rebuild Services

```bash
# Rebuild images and restart (e.g., after submodule updates)
docker compose up -d --build

# Force rebuild without cache
docker compose build --no-cache
docker compose up -d
```

### View Logs

```bash
# Follow logs from all services
docker compose logs -f

# Follow logs from a specific service
docker compose logs -f backend
docker compose logs -f frontend
```

### Execute Commands Inside Containers

```bash
# Open a shell in the backend container
docker compose exec backend bash

# Run artisan commands
docker compose exec backend php artisan migrate
docker compose exec backend php artisan tinker

# Run artisan with fresh migration and seed
docker compose exec backend php artisan migrate:fresh --seed
```

---

## Docker Hub Workflow

### Build and Tag Images

Build images with Docker Hub-compatible tags:

```bash
# Backend image
docker build -f docker/be/Dockerfile -t username/piptrek-be:latest ./pipTrek-be

# Frontend image
docker build -f docker/fe/Dockerfile -t username/piptrek-fe:latest ./pipTrek-fe
```

Replace `username` with your Docker Hub username or organization name.

### Tag with Version Numbers

```bash
# Tag with a specific version
docker tag username/piptrek-be:latest username/piptrek-be:1.0.0
docker tag username/piptrek-fe:latest username/piptrek-fe:1.0.0
```

### Push to Docker Hub

```bash
# Log in to Docker Hub
docker login

# Push backend image
docker push username/piptrek-be:latest
docker push username/piptrek-be:1.0.0

# Push frontend image
docker push username/piptrek-fe:latest
docker push username/piptrek-fe:1.0.0
```

### Pull and Run from Docker Hub

On a deployment server (no source code needed):

```bash
docker pull username/piptrek-be:latest
docker pull username/piptrek-fe:latest

# Run backend
docker run -d \
  --name piptrek-backend \
  -p 8000:80 \
  -e APP_KEY=base64:YOUR_KEY \
  -e JWT_SECRET=YOUR_SECRET \
  -e DB_CONNECTION=sqlite \
  -e DB_DATABASE=/var/www/html/database/database.sqlite \
  username/piptrek-be:latest

# Run frontend
docker run -d \
  --name piptrek-frontend \
  -p 3000:80 \
  username/piptrek-fe:latest
```

---

## Troubleshooting

### Build Errors

#### "COPY failed: file not found" or empty build context

**Cause:** Submodules are not initialized. The `pipTrek-be/` or `pipTrek-fe/` directories are empty.

**Fix:**

```bash
git submodule update --init --recursive
```

#### "npm ci" fails with lockfile errors

**Cause:** `package-lock.json` is out of sync with `package.json` in a submodule.

**Fix:** Update the submodule to the latest commit where lockfiles are in sync:

```bash
cd pipTrek-fe  # or pipTrek-be
git pull origin main
cd ..
git add pipTrek-fe
git commit -m "Update pipTrek-fe submodule"
```

#### Docker build runs out of memory

**Cause:** The TypeScript type-check (`vue-tsc --build`) or Vite build can be memory-intensive.

**Fix:** Increase Docker's memory allocation in Docker Desktop settings (recommended: 4GB+).

---

### Missing Environment Variables

#### "No application encryption key has been specified"

**Cause:** `APP_KEY` is not set or is empty.

**Fix:** Generate and set the key in your `.env` file or docker-compose environment:

```bash
# Generate a key
echo "base64:$(openssl rand -base64 32)"

# Add to .env file in repo root
APP_KEY=base64:generated_value_here
```

#### "JWT secret is not defined"

**Cause:** `JWT_SECRET` is not configured.

**Fix:**

```bash
# Generate a secret
openssl rand -hex 32

# Add to .env file in repo root
JWT_SECRET=generated_value_here
```

---

### Submodule Issues

#### Submodule directories are empty after clone

**Cause:** Repository was cloned without `--recurse-submodules`.

**Fix:**

```bash
git submodule update --init --recursive
```

#### Submodule is in "detached HEAD" state

This is normal behavior for submodules. The parent repo pins each submodule to a specific commit. To update a submodule to the latest upstream commit:

```bash
cd pipTrek-be  # or pipTrek-fe
git checkout main
git pull
cd ..
git add pipTrek-be
git commit -m "Update pipTrek-be submodule to latest"
```

#### "fatal: not a git repository" inside submodule

**Cause:** The `.git` directory may be missing or corrupted.

**Fix:**

```bash
# Remove and re-initialize
rm -rf pipTrek-be  # or pipTrek-fe
git submodule update --init --recursive
```

---

### Container Startup Issues

#### Backend container exits immediately

**Check logs:**

```bash
docker compose logs backend
```

**Common causes:**

- Missing `APP_KEY`: See [Missing Environment Variables](#missing-environment-variables)
- Port 8000 already in use: Change the host port in `docker-compose.yml` (e.g., `8001:80`)
- PHP extension errors: Rebuild the image with `docker compose build --no-cache backend`

#### Frontend shows "502 Bad Gateway" or blank page

**Check logs:**

```bash
docker compose logs frontend
```

**Common causes:**

- Build failed silently: Rebuild with `docker compose build --no-cache frontend`
- Nginx config error: Check `docker compose exec frontend nginx -t`

#### Containers can't communicate with each other

**Cause:** Network configuration issue.

**Verify the network exists:**

```bash
docker network ls | grep piptrek
```

**Test connectivity from frontend to backend:**

```bash
docker compose exec frontend wget -qO- http://backend:80/up
```

#### "port is already allocated" error

**Cause:** Another process is using port 8000 or 3000 on the host.

**Fix:** Either stop the conflicting process or change the host port in `docker-compose.yml`:

```yaml
ports:
 - "8001:80" # Change 8000 to 8001 (or any free port)
```

---

## Architecture Decisions

| Decision                             | Rationale                                                                                                |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| **Multi-stage builds**               | Keeps final images small by discarding build tools (Composer, Node, npm)                                 |
| **php:8.2-apache** for backend       | Single-process container simplifies orchestration; Apache + mod_rewrite handles Laravel routing natively |
| **nginx:stable-alpine** for frontend | ~25MB image; Nginx efficiently serves static SPAs with gzip and caching                                  |
| **SQLite as default DB**             | Matches the project's `.env.example` default; no additional database service required for basic setup    |
| **Named volumes**                    | Persists database and storage data across container restarts                                             |
| **Bridge network**                   | Enables DNS-based service discovery (containers reference each other by service name)                    |

---

## License

Refer to individual submodule repositories for license information.
