# 06 · Laravel

Stack: **Laravel + PHP-FPM 8.3 + Nginx + PostgreSQL 16** (+ cola y scheduler opcionales).

## Estructura

```
proyecto/
├── app/ ... (código Laravel)
├── composer.json / composer.lock
├── package.json                # si compilas assets con Vite
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env.example
└── docker/
    ├── entrypoint.sh
    └── nginx/default.conf      # ver 04-nginx.md (Plantilla B)
```

## `Dockerfile` (multi-stage con 2 targets: `app` y `web`)

```dockerfile
# syntax=docker/dockerfile:1

# ───────── 1. Dependencias PHP ─────────
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts --no-autoloader --prefer-dist
COPY . .
RUN composer dump-autoload --optimize --no-dev --no-scripts

# ───────── 2. Assets (Vite) ─────────
# Si NO usas Vite/npm, elimina este stage y la línea COPY --from=assets de más abajo
FROM node:20-alpine AS assets
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
COPY --from=vendor /app/vendor ./vendor
RUN npm run build

# ───────── 3. App (PHP-FPM)  → target: app ─────────
FROM php:8.3-fpm-alpine AS app

RUN apk add --no-cache \
        postgresql-client postgresql-dev libzip-dev icu-dev oniguruma-dev \
        libpng-dev libjpeg-turbo-dev freetype-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j"$(nproc)" pdo_pgsql pgsql zip intl bcmath gd pcntl opcache

# Ajustes de producción para PHP
RUN { \
      echo "opcache.enable=1"; \
      echo "opcache.validate_timestamps=0"; \
      echo "opcache.memory_consumption=128"; \
      echo "upload_max_filesize=50M"; \
      echo "post_max_size=50M"; \
      echo "memory_limit=256M"; \
    } > /usr/local/etc/php/conf.d/app.ini

WORKDIR /var/www/html

COPY --chown=www-data:www-data . .
COPY --from=vendor --chown=www-data:www-data /app/vendor ./vendor
COPY --from=assets --chown=www-data:www-data /app/public/build ./public/build
COPY --chown=www-data:www-data docker/entrypoint.sh /entrypoint.sh

RUN chmod +x /entrypoint.sh \
    && mkdir -p storage/app/public storage/framework/cache storage/framework/sessions storage/framework/views storage/logs bootstrap/cache \
    && chown -R www-data:www-data storage bootstrap/cache \
    && ln -sfn /var/www/html/storage/app/public public/storage

USER www-data
EXPOSE 9000
ENTRYPOINT ["/entrypoint.sh"]
CMD ["php-fpm"]

# ───────── 4. Web (Nginx con los archivos públicos) → target: web ─────────
FROM nginx:1.27-alpine AS web
COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=app /var/www/html/public /var/www/html/public
```

> Si tu proyecto **no usa Vite**, quita el stage `assets` y la línea `COPY --from=assets ...`.

## `docker/entrypoint.sh`

```sh
#!/bin/sh
set -e

DB_HOST="${POSTGRES_HOST:-db}"
DB_PORT="${POSTGRES_PORT:-5432}"

echo "⏳ Esperando a PostgreSQL en ${DB_HOST}:${DB_PORT}..."
until pg_isready -h "$DB_HOST" -p "$DB_PORT" -q; do
  sleep 1
done
echo "✅ PostgreSQL disponible"

# Carpetas que un volumen vacío podría haber ocultado
mkdir -p storage/app/public storage/framework/cache storage/framework/sessions storage/framework/views storage/logs

php artisan package:discover --ansi

if [ "${RUN_MIGRATIONS:-true}" = "true" ]; then
  echo "🔄 Ejecutando migraciones..."
  php artisan migrate --force
fi

if [ "${APP_ENV}" = "production" ]; then
  php artisan config:cache
  php artisan route:cache
  php artisan view:cache
  php artisan event:cache
fi

exec "$@"
```

## `docker-compose.yml`

```yaml
x-laravel-env: &laravel-env
  APP_NAME: ${APP_NAME:-Laravel}
  APP_ENV: ${APP_ENV:-production}
  APP_KEY: ${APP_KEY:?Falta APP_KEY}
  APP_DEBUG: ${APP_DEBUG:-false}
  APP_URL: ${APP_URL:?Falta APP_URL}
  LOG_CHANNEL: stderr
  DB_CONNECTION: pgsql
  DB_HOST: db
  DB_PORT: 5432
  DB_DATABASE: ${POSTGRES_DB:?Falta POSTGRES_DB}
  DB_USERNAME: ${POSTGRES_USER:?Falta POSTGRES_USER}
  DB_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD}
  POSTGRES_HOST: db
  POSTGRES_PORT: 5432
  SESSION_DRIVER: ${SESSION_DRIVER:-database}
  CACHE_STORE: ${CACHE_STORE:-database}
  QUEUE_CONNECTION: ${QUEUE_CONNECTION:-database}

services:
  app:
    build:
      context: .
      target: app
    restart: unless-stopped
    environment:
      <<: *laravel-env
    volumes:
      - laravel_storage:/var/www/html/storage
    expose:
      - "9000"
    depends_on:
      db:
        condition: service_healthy

  web:
    build:
      context: .
      target: web
    restart: unless-stopped
    volumes:
      - laravel_storage:/var/www/html/storage:ro
    expose:
      - "80"
    networks:
      - default
      - dokploy-network
    depends_on:
      - app

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── Opcional: worker de colas ──
  queue:
    build:
      context: .
      target: app
    restart: unless-stopped
    command: php artisan queue:work --sleep=3 --tries=3 --max-time=3600
    environment:
      <<: *laravel-env
      RUN_MIGRATIONS: "false"
    volumes:
      - laravel_storage:/var/www/html/storage
    depends_on:
      - app

  # ── Opcional: scheduler (cron de Laravel) ──
  scheduler:
    build:
      context: .
      target: app
    restart: unless-stopped
    command: sh -c "while true; do php artisan schedule:run --verbose --no-interaction; sleep 60; done"
    environment:
      <<: *laravel-env
      RUN_MIGRATIONS: "false"
    volumes:
      - laravel_storage:/var/www/html/storage
    depends_on:
      - app

volumes:
  postgres_data:
  laravel_storage:

networks:
  dokploy-network:
    external: true
```

**En Dokploy → Domains**: Service `web`, Port `80`, HTTPS activado.

> Si no necesitas colas ni scheduler, borra los servicios `queue` y `scheduler`.

## Detalles importantes

- **`APP_KEY`** es obligatoria. Genérala localmente con `php artisan key:generate --show` y pégala en Dokploy.
- **Proxy de confianza**: en `bootstrap/app.php` (Laravel 11+):
  ```php
  ->withMiddleware(function (Middleware $middleware) {
      $middleware->trustProxies(at: '*');
  })
  ```
  En Laravel 10, en `app/Http/Middleware/TrustProxies.php` pon `protected $proxies = '*';`.
  Sin esto, los enlaces y assets se generan con `http://` y aparece *mixed content*.
- **`storage` es un volumen**: los archivos subidos (`storage/app/public`) persisten entre despliegues y nginx los sirve vía el enlace `public/storage`.
- **Logs** a `stderr` para verlos en Dokploy → *Logs*.

## Comandos útiles

```bash
docker compose exec app php artisan migrate:status
docker compose exec app php artisan tinker
docker compose exec app php artisan db:seed --force
docker compose logs -f app web
```
