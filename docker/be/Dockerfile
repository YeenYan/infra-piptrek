FROM php:8.4-cli-alpine

# System dependencies
RUN apk add --no-cache \
    git \
    curl \
    zip \
    unzip \
    libpq-dev \
    oniguruma-dev \
    && docker-php-ext-install \
        pdo \
        pdo_pgsql \
        pcntl \
        mbstring

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copy composer files first for layer caching
COPY composer.json composer.lock ./

# Install PHP dependencies (warms the named volume on first run)
RUN composer install --no-scripts --no-autoloader --no-interaction

# Copy the rest of the application
COPY . .

RUN composer dump-autoload --optimize

# Embed entrypoint inline — no extra file needed in the repo
RUN printf '#!/bin/sh\nset -e\n\
\n\
if [ ! -f .env ]; then\n\
  if [ -f .env.example ]; then\n\
    cp .env.example .env\n\
    echo "[entrypoint] .env created from .env.example"\n\
  else\n\
    echo "[entrypoint] WARNING: no .env or .env.example found"\n\
  fi\n\
fi\n\
\n\
if ! grep -q "^APP_KEY=base64:" .env 2>/dev/null; then\n\
  php artisan key:generate --force\n\
  echo "[entrypoint] APP_KEY generated"\n\
fi\n\
\n\
if [ ! -f vendor/autoload.php ]; then\n\
  echo "[entrypoint] vendor missing — running composer install..."\n\
  composer install --no-scripts --no-interaction\n\
  composer dump-autoload --optimize\n\
fi\n\
\n\
echo "[entrypoint] running migrations..."\n\
php artisan migrate --force\n\
\n\
exec "$@"\n\
' > /usr/local/bin/docker-entrypoint.sh \
  && chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 8000

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]