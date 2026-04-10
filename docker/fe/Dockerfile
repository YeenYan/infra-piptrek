# syntax=docker/dockerfile:1
###############################################################################
# Frontend Dockerfile — pipTrek-fe (Vue 3 + TypeScript + Vite)
###############################################################################
#
# This Dockerfile uses a multi-stage build to:
#   1. Install dependencies and compile the Vue SPA into static files
#   2. Serve the static files with Nginx (lightweight, production-grade)
#
# Build context: pipTrek-fe/ (the Vue application root)
# Dockerfile path: docker/fe/Dockerfile
#
# Standalone build command:
#   docker build -f docker/fe/Dockerfile -t username/piptrek-fe ./pipTrek-fe
#
# Docker Compose handles context/dockerfile mapping automatically.
###############################################################################


# =============================================================================
# Stage 1: Build Stage (Node.js — Install deps, type-check, and build)
# =============================================================================
# Purpose: Install npm dependencies and compile the Vue 3 SPA into static
#          HTML/CSS/JS files using Vite.
# Why Node 22 Alpine:
#   - package.json engines field requires "^20.19.0 || >=22.12.0"
#   - Node 22 is the current LTS line and satisfies the constraint
#   - Alpine variant minimizes the builder image size (~50MB vs ~350MB)
FROM node:22-alpine AS builder

WORKDIR /app

# Copy package manifests first to leverage Docker's layer caching.
# If package.json and package-lock.json haven't changed, Docker reuses
# the cached node_modules layer — skipping the expensive npm ci step.
COPY package.json package-lock.json ./

# Install exact dependency versions from package-lock.json.
# npm ci is preferred over npm install in CI/Docker builds because:
#   - It installs exact versions (fully reproducible)
#   - It's faster (skips version resolution)
#   - It removes any existing node_modules first (clean slate)
RUN npm ci

# Copy the rest of the application source code.
# This is done after npm ci so that source code changes don't
# invalidate the expensive node_modules cache layer.
COPY . .

# Build the Vue application for production.
# The "build" script in package.json runs:
#   run-p type-check "build-only {@}" --
# Which executes in parallel:
#   1. vue-tsc --build    (TypeScript type checking)
#   2. vite build         (production bundling, tree-shaking, minification)
# Output is written to dist/ by default (Vite's standard output directory).
RUN npm run build


# =============================================================================
# Stage 2: Production Image (Nginx — Static file server)
# =============================================================================
# Purpose: Serve the compiled static files with Nginx.
# Why nginx:alpine:
#   - Nginx is the industry standard for serving SPAs in production
#   - Alpine variant produces a ~25MB final image (extremely lightweight)
#   - The untagged 'alpine' variant tracks the latest stable Nginx on the
#     latest Alpine release, ensuring the most up-to-date security patches
#   - Nginx efficiently handles static files, gzip, caching headers, etc.
FROM nginx:alpine AS production

# OCI image metadata labels
LABEL maintainer="pipTrek Team"
LABEL description="pipTrek Frontend — Vue 3 SPA served via Nginx"
LABEL org.opencontainers.image.source="https://github.com/YeenYan/pipTrek-fe"

# Remove the default Nginx welcome page.
# We replace it with our own SPA-optimized configuration below.
RUN rm -rf /usr/share/nginx/html/*

# Write a custom Nginx configuration optimized for serving a Vue SPA.
# Uses Docker BuildKit's native COPY heredoc syntax (requires the
# # syntax=docker/dockerfile:1 directive at the top of this file).
# Key features of this configuration:
#   - try_files with fallback to /index.html for Vue Router's HTML5 history mode
#     (createWebHistory). Without this, direct navigation to /about would 404.
#   - Gzip compression reduces bandwidth for text-based assets (HTML, CSS, JS).
#   - Long cache TTL for hashed assets (Vite fingerprints filenames for busting).
#   - Short cache for index.html so users always get the latest app version.
#   - Security headers (X-Frame-Options, X-Content-Type-Options, etc.).
COPY <<'EOF' /etc/nginx/conf.d/default.conf
server {
    # Listen on port 80 inside the container.
    # The actual external port is mapped in docker-compose.yml or docker run -p.
    listen 80;
    server_name _;

    # Serve static files from the Vite build output directory.
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback: Vue Router uses HTML5 history mode (createWebHistory).
    # When a user navigates directly to /dashboard or refreshes, Nginx must
    # serve index.html so Vue Router can handle the route client-side.
    # Without this, any route other than / would return a 404.
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache hashed static assets aggressively.
    # Vite outputs JS/CSS with content hashes in filenames (e.g., index-abc123.js).
    # These filenames change when content changes, so it's safe to cache them
    # for a long time (1 year). This dramatically reduces bandwidth and load times.
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Do not cache index.html.
    # index.html references the current hashed asset filenames. If we cached it,
    # users would keep loading stale versions after a deployment. No-cache ensures
    # browsers always check for the latest index.html.
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # Security headers to harden the application.
    #   X-Frame-Options: Prevents the page from being embedded in iframes (clickjacking).
    #   X-Content-Type-Options: Prevents MIME-type sniffing attacks.
    #   X-XSS-Protection: Enables browser's built-in XSS filter.
    #   Referrer-Policy: Controls how much referrer info is sent with requests.
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Enable gzip compression for text-based content.
    # Reduces transfer sizes by 60-80% for HTML, CSS, JS, JSON, and SVG.
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/rss+xml
        image/svg+xml;
}
EOF

# Copy the compiled static files from the builder stage.
# Vite outputs the production build to dist/ which contains:
#   - index.html (entry point with hashed asset references)
#   - assets/     (JS, CSS, images with content-hashed filenames)
# Only this directory is copied — no node_modules, source code, or build tools.
COPY --from=builder /app/dist /usr/share/nginx/html

# Expose port 80 — Nginx's listening port.
# This documents the port for container users. Actual host mapping is
# configured via docker-compose.yml or the docker run -p flag.
EXPOSE 80

# Start Nginx in the foreground.
# "daemon off" keeps Nginx as PID 1 so Docker can monitor the process.
# If Nginx crashes, Docker detects it and can restart the container
# (depending on the restart policy configured in docker-compose.yml).
CMD ["nginx", "-g", "daemon off;"]
