###############################################################################
# Backend Dockerfile — pipTrek-be (Laravel 12 + PHP 8.4)
###############################################################################
#
# This Dockerfile uses a multi-stage build strategy to produce a lean,
# production-ready image for the Laravel backend application.
#
# Stages:
#   1. composer-builder  — Installs PHP (Composer) dependencies
#   2. node-builder      — Compiles Vite frontend assets (Tailwind CSS + JS)
#   3. production        — Final image with PHP 8.4 + Apache serving the app
#
# Build context: pipTrek-be/ (the Laravel application root)
# Dockerfile path: docker/be/Dockerfile
#
# Standalone build command:
#   docker build -f docker/be/Dockerfile -t username/piptrek-be ./pipTrek-be
#
# Docker Compose handles context/dockerfile mapping automatically.
###############################################################################


# =============================================================================
# Stage 1: Composer Dependency Installation
# =============================================================================
# Purpose: Install production PHP dependencies in an isolated stage.
# Why php:8.4-cli-bookworm instead of composer:2:
#   - The full composer:2 image carries many OS-level CVEs.
#   - Using Debian Bookworm PHP CLI + copying just the composer binary reduces
#     the vulnerability surface while keeping full PHP compatibility.
#   - Bookworm (Debian 12) receives aggressive security patching.
#   - Only the vendor/ directory is carried forward to the final image.
FROM php:8.4-cli-bookworm AS composer-builder

# Copy the Composer binary from the official Composer image.
# This is a multi-stage COPY — only the binary is extracted, the composer:2
# image's OS-level vulnerabilities do not affect this stage's base layer.
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Install system packages needed by Composer to download and extract archives.
RUN apt-get update && apt-get install -y --no-install-recommends unzip \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only Composer manifests first to leverage Docker's layer caching.
# If these files haven't changed, Docker reuses the cached vendor/ layer,
# dramatically speeding up rebuilds when only application code changes.
COPY composer.json composer.lock ./

# Install PHP dependencies for production.
#   --no-dev               : Excludes dev packages (phpunit, faker, mockery, etc.)
#   --no-interaction       : Non-interactive mode (required for CI/Docker builds)
#   --no-scripts           : Skips post-install scripts (they need full app context)
#   --prefer-dist          : Downloads pre-built archives (faster than git clones)
#   --optimize-autoloader  : Generates a classmap for faster class resolution
#   --ignore-platform-reqs : Skips extension checks — safe here because the
#                            production stage installs all required extensions.
#                            This avoids needing to install PHP extensions in
#                            this throwaway build stage.
RUN composer install \
    --no-dev \
    --no-interaction \
    --no-scripts \
    --prefer-dist \
    --optimize-autoloader \
    --ignore-platform-reqs


# =============================================================================
# Stage 2: Frontend Asset Compilation (Vite + Tailwind CSS)
# =============================================================================
# Purpose: Build the backend's CSS/JS assets using Vite with laravel-vite-plugin.
# The backend's package.json defines Vite to compile:
#   - resources/css/app.css  (Tailwind CSS v4)
#   - resources/js/app.js
# Output goes to public/build/ with cache-busted filenames.
# Why Node 22: The project ecosystem targets modern Node (>=20.19 or >=22.12).
FROM node:22-alpine AS node-builder

WORKDIR /app

# Copy package manifests first for layer caching (same strategy as Composer).
COPY package.json package-lock.json ./

# Install Node dependencies using npm ci for reproducible builds.
# npm ci installs exact versions from package-lock.json and is faster
# than npm install in CI environments because it skips version resolution.
RUN npm ci

# Copy Vite config and source asset files required for the build.
# Only the files Vite needs are copied — not the full app — to minimize
# cache invalidation (changes to PHP files won't trigger a JS rebuild).
COPY vite.config.js ./
COPY resources/ ./resources/

# Run the Vite production build defined in package.json as "build": "vite build".
# This compiles, tree-shakes, and minifies all CSS/JS assets into public/build/.
RUN npm run build


# =============================================================================
# Stage 3: Production Image (PHP 8.4 + Apache on Debian Bookworm)
# =============================================================================
# Purpose: Assemble the final production image.
# Why php:8.4-apache-bookworm:
#   - Bundles Apache + mod_php in a single container (simpler orchestration)
#   - Well-suited for Laravel (public/index.php is the single entry point)
#   - Apache's mod_rewrite handles Laravel's clean URL routing
#   - PHP 8.4 satisfies the ^8.2 requirement from composer.json
#   - Bookworm (Debian 12) is the latest stable Debian release with
#     the most up-to-date security patches, reducing known CVEs
FROM php:8.4-apache-bookworm AS production

# OCI image metadata labels
LABEL maintainer="pipTrek Team"
LABEL description="pipTrek Backend — Laravel 12 GraphQL API (Lighthouse + JWT)"
LABEL org.opencontainers.image.source="https://github.com/YeenYan/pipTrek-be"

# Install system libraries required by PHP extensions.
# Each library is annotated with the extension it supports:
#   libpng-dev, libjpeg-dev, libfreetype6-dev → GD (image processing)
#   libzip-dev                                → zip (archive handling)
#   libicu-dev                                → intl (internationalization)
#   libsqlite3-dev                            → pdo_sqlite (default DB driver)
#   libpq-dev                                 → pdo_pgsql (PostgreSQL driver)
#   unzip                                     → Composer package extraction
# --no-install-recommends avoids pulling in unnecessary packages.
# rm -rf /var/lib/apt/lists/* cleans the apt cache to reduce image size.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libicu-dev \
    libsqlite3-dev \
    libpq-dev \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Configure and install PHP extensions required by Laravel and project deps.
#   pdo_mysql  : MySQL driver (production database option per config/database.php)
#   pdo_sqlite : SQLite driver (default DB_CONNECTION in .env.example)
#   pdo_pgsql  : PostgreSQL driver (available in config/database.php)
#   gd         : Image processing with FreeType + JPEG support
#   zip        : ZIP archive handling (Composer, file uploads)
#   intl       : Internationalization (locale-aware formatting)
#   bcmath     : Arbitrary-precision math (used by some Laravel/package features)
#   opcache    : Bytecode cache — critical for PHP production performance
# -j$(nproc) parallelizes compilation using all available CPU cores.
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
    pdo_mysql \
    pdo_sqlite \
    pdo_pgsql \
    gd \
    zip \
    intl \
    bcmath \
    opcache

# Enable Apache mod_rewrite.
# Laravel's public/.htaccess uses rewrite rules to direct all requests
# to public/index.php (the front controller). Without mod_rewrite,
# any route other than / would return a 404 from Apache.
RUN a2enmod rewrite

# Set the Apache document root to Laravel's public/ directory.
# This is critical for security — only public/ is web-accessible,
# preventing direct access to .env, composer.json, app/, config/, etc.
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public

# Reconfigure Apache's VirtualHost and global config to use the new document root.
# Also enable AllowOverride All so Laravel's public/.htaccess rewrite rules work.
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/sites-available/*.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf \
    && sed -ri -e 's/AllowOverride None/AllowOverride All/g' \
        /etc/apache2/apache2.conf

# Write a production-optimized PHP configuration.
#   opcache.*              : Bytecode caching eliminates re-parsing PHP files.
#                            validate_timestamps=0 means opcache never checks if
#                            files changed (safe in immutable containers).
#   memory_limit=256M      : Sufficient for typical Laravel + GraphQL workloads.
#   upload_max_filesize=64M: Allows larger file uploads if needed.
#   post_max_size=64M      : Must be >= upload_max_filesize.
#   expose_php=Off         : Hides "X-Powered-By: PHP" header (security hardening).
RUN { \
    echo 'opcache.enable=1'; \
    echo 'opcache.memory_consumption=128'; \
    echo 'opcache.interned_strings_buffer=8'; \
    echo 'opcache.max_accelerated_files=10000'; \
    echo 'opcache.validate_timestamps=0'; \
    echo 'opcache.revalidate_freq=0'; \
    echo 'memory_limit=256M'; \
    echo 'upload_max_filesize=64M'; \
    echo 'post_max_size=64M'; \
    echo 'expose_php=Off'; \
    } > /usr/local/etc/php/conf.d/production.ini

# Set the working directory to the Laravel project root.
WORKDIR /var/www/html

# Copy the full application source from the build context (pipTrek-be/).
# This includes all PHP files, config, routes, views, migrations, etc.
COPY . .

# Overlay production Composer dependencies from Stage 1.
# This replaces any vendor/ that came with the COPY . . above,
# ensuring only production (--no-dev) packages are included.
COPY --from=composer-builder /app/vendor ./vendor

# Overlay compiled Vite assets from Stage 2.
# The public/build/ directory contains hashed, minified CSS/JS files
# produced by `npm run build` via laravel-vite-plugin.
COPY --from=node-builder /app/public/build ./public/build

# Create Laravel's required writable directories and set ownership.
# Laravel writes to these at runtime:
#   storage/framework/cache/data  — File-based cache
#   storage/framework/sessions    — File-based sessions
#   storage/framework/views       — Compiled Blade templates
#   storage/logs                  — Application log files
#   bootstrap/cache               — Cached config, routes, services
#   database                      — SQLite database file (if using SQLite)
# www-data is Apache's runtime user — it must own these directories.
# 775 permissions allow the user and group to read/write/execute.
RUN mkdir -p \
    storage/framework/cache/data \
    storage/framework/sessions \
    storage/framework/views \
    storage/logs \
    bootstrap/cache \
    database \
    && chown -R www-data:www-data storage bootstrap/cache database \
    && chmod -R 775 storage bootstrap/cache database

# Remove files that should not be in the production image.
# These are development/CI artifacts that are unnecessary and could
# leak information if the container is compromised.
RUN rm -rf \
    tests \
    .env.example \
    phpunit.xml \
    .git \
    node_modules

# Copy the entrypoint script from the Docker build context.
# Note: The entrypoint script lives in docker/be/ in the infra repo,
# but we need it accessible in the container. It's passed via build args
# or we embed it directly below instead.

# Expose port 80 — Apache's default HTTP port.
# This is documentation for container users; actual port mapping is
# configured in docker-compose.yml or the docker run -p flag.
EXPOSE 80

# Use Apache's foreground mode as the container command.
# apache2-foreground keeps Apache running in the foreground so Docker
# can monitor the process. If Apache crashes, Docker detects it and
# can restart the container (depending on restart policy).
CMD ["apache2-foreground"]
