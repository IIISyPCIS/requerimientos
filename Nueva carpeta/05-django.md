# 05 · Django

Stack: **Django + Gunicorn + Nginx (static/media) + PostgreSQL 16**.

## Estructura

```
proyecto/
├── config/                  # tu carpeta de settings (settings.py, wsgi.py)
├── manage.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env.example
└── docker/
    ├── entrypoint.sh
    └── nginx/default.conf   # ver 04-nginx.md (Plantilla A)
```

## `requirements.txt` (mínimo)

```txt
Django>=5.0,<6.0
gunicorn>=22.0
psycopg[binary]>=3.1
whitenoise>=6.6        # opcional si no usas nginx para static
```

> `psycopg[binary]` evita instalar compiladores y librerías del sistema.

## `Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# netcat para esperar a la BD en el entrypoint
RUN apt-get update \
    && apt-get install -y --no-install-recommends netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Dependencias (capa cacheable)
COPY requirements.txt .
RUN pip install -r requirements.txt

# Usuario no-root + carpetas de estáticos/media con permisos
RUN addgroup --system app && adduser --system --ingroup app app \
    && mkdir -p /app/staticfiles /app/media \
    && chown -R app:app /app

COPY --chown=app:app . .
COPY --chown=app:app docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER app
EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3", "--timeout", "60", "--access-logfile", "-"]
```

> Cambia `config.wsgi` por el módulo real de tu proyecto (`nombre_proyecto.wsgi`).

## `docker/entrypoint.sh`

```sh
#!/bin/sh
set -e

DB_HOST="${POSTGRES_HOST:-db}"
DB_PORT="${POSTGRES_PORT:-5432}"

echo "⏳ Esperando a PostgreSQL en ${DB_HOST}:${DB_PORT}..."
until nc -z "$DB_HOST" "$DB_PORT"; do
  sleep 1
done
echo "✅ PostgreSQL disponible"

if [ "${RUN_MIGRATIONS:-true}" = "true" ]; then
  echo "🔄 Aplicando migraciones..."
  python manage.py migrate --noinput
fi

if [ "${RUN_COLLECTSTATIC:-true}" = "true" ]; then
  echo "📦 Recolectando archivos estáticos..."
  python manage.py collectstatic --noinput
fi

# Superusuario opcional (solo si defines las 3 variables; no falla si ya existe)
if [ -n "$DJANGO_SUPERUSER_USERNAME" ] && [ -n "$DJANGO_SUPERUSER_PASSWORD" ] && [ -n "$DJANGO_SUPERUSER_EMAIL" ]; then
  python manage.py createsuperuser --noinput || true
fi

exec "$@"
```

## `docker-compose.yml`

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      DJANGO_SECRET_KEY: ${DJANGO_SECRET_KEY:?Falta DJANGO_SECRET_KEY}
      DJANGO_DEBUG: ${DJANGO_DEBUG:-False}
      DJANGO_ALLOWED_HOSTS: ${DJANGO_ALLOWED_HOSTS:?Falta DJANGO_ALLOWED_HOSTS}
      DJANGO_CSRF_TRUSTED_ORIGINS: ${DJANGO_CSRF_TRUSTED_ORIGINS:?Falta DJANGO_CSRF_TRUSTED_ORIGINS}
      POSTGRES_HOST: db
      POSTGRES_PORT: 5432
      POSTGRES_DB: ${POSTGRES_DB:?Falta POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER:?Falta POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD}
      DJANGO_SUPERUSER_USERNAME: ${DJANGO_SUPERUSER_USERNAME:-}
      DJANGO_SUPERUSER_EMAIL: ${DJANGO_SUPERUSER_EMAIL:-}
      DJANGO_SUPERUSER_PASSWORD: ${DJANGO_SUPERUSER_PASSWORD:-}
    volumes:
      - static_data:/app/staticfiles
      - media_data:/app/media
    expose:
      - "8000"
    depends_on:
      db:
        condition: service_healthy

  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - static_data:/app/staticfiles:ro
      - media_data:/app/media:ro
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

volumes:
  postgres_data:
  static_data:
  media_data:

networks:
  dokploy-network:
    external: true
```

**En Dokploy → Domains**: Service `nginx`, Port `80`, HTTPS activado.

## `settings.py` (fragmentos necesarios)

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.getenv("DJANGO_DEBUG", "False") == "True"

ALLOWED_HOSTS = [h.strip() for h in os.getenv("DJANGO_ALLOWED_HOSTS", "").split(",") if h.strip()]
CSRF_TRUSTED_ORIGINS = [o.strip() for o in os.getenv("DJANGO_CSRF_TRUSTED_ORIGINS", "").split(",") if o.strip()]

# Base de datos PostgreSQL
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ["POSTGRES_DB"],
        "USER": os.environ["POSTGRES_USER"],
        "PASSWORD": os.environ["POSTGRES_PASSWORD"],
        "HOST": os.getenv("POSTGRES_HOST", "db"),
        "PORT": os.getenv("POSTGRES_PORT", "5432"),
    }
}

# Detrás de Traefik/Nginx: confiar en el header de HTTPS
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
USE_X_FORWARDED_HOST = True
SESSION_COOKIE_SECURE = not DEBUG
CSRF_COOKIE_SECURE = not DEBUG

# Estáticos y media (coinciden con el nginx.conf y los volúmenes)
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

### Variante sin Nginx (WhiteNoise)

1. En `MIDDLEWARE`, justo después de `SecurityMiddleware`: `"whitenoise.middleware.WhiteNoiseMiddleware"`.
2. Elimina el servicio `nginx` del compose y en Dokploy asigna el dominio al servicio `app` puerto `8000` (añade `networks: [default, dokploy-network]` a `app`).
3. Para `media` de usuarios sigue haciendo falta nginx o un almacenamiento externo (S3).

## Comandos útiles

```bash
docker compose exec app python manage.py createsuperuser
docker compose exec app python manage.py shell
docker compose logs -f app
```

En Dokploy puedes abrir una terminal del contenedor desde la pestaña **Terminal** del servicio.
