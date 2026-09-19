# 02 · Archivos comunes a todos los proyectos

## 1. `.dockerignore`

Evita enviar basura al build (más rápido, imágenes más pequeñas y sin filtrar secretos).

```gitignore
# Control de versiones
.git
.gitignore
.github

# Secretos y entorno
.env
.env.*
!.env.example

# Docker
Dockerfile*
docker-compose*.yml
.dockerignore

# Sistema / IDE
.DS_Store
.idea
.vscode
*.log

# Node / Next.js
node_modules
.next
npm-debug.log*

# Python / Django
__pycache__
*.pyc
*.pyo
.venv
venv
db.sqlite3
staticfiles
media

# PHP / Laravel
vendor
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
bootstrap/cache/*
public/hot

# Tests y docs
tests
docs
*.md
```

> ⚠️ Ajusta según tu proyecto. Ejemplo: en Laravel **no** ignores `vendor` si lo instalas fuera del build (aquí se instala dentro del Dockerfile, por eso se ignora).

## 2. `.gitignore` (mínimo relacionado con Docker)

```gitignore
.env
.env.local
.env.production
*.log
```

## 3. `.env.example`

Este archivo **sí se sube** al repo. Sirve de plantilla; los valores reales van en Dokploy → *Environment*.

```env
# ── General ──────────────────────────────
APP_ENV=production
APP_DOMAIN=miapp.midominio.com

# ── PostgreSQL (obligatorio en todos los proyectos) ──
POSTGRES_DB=miapp
POSTGRES_USER=miapp_user
POSTGRES_PASSWORD=CAMBIAR_por_una_clave_larga_y_aleatoria
POSTGRES_HOST=db
POSTGRES_PORT=5432

# ── Específico del framework (deja solo lo que uses) ──
# Django
DJANGO_SECRET_KEY=CAMBIAR
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=miapp.midominio.com
DJANGO_CSRF_TRUSTED_ORIGINS=https://miapp.midominio.com

# Laravel
APP_KEY=base64:CAMBIAR
APP_DEBUG=false
APP_URL=https://miapp.midominio.com

# Next.js
NEXT_PUBLIC_API_URL=https://miapp.midominio.com/api
AUTH_SECRET=CAMBIAR
```

Generar claves seguras:

```bash
openssl rand -base64 32          # contraseñas / secretos
openssl rand -hex 32             # alternativa hexadecimal
php artisan key:generate --show  # APP_KEY de Laravel
python -c "import secrets; print(secrets.token_urlsafe(50))"  # Django SECRET_KEY
```

> 💡 **Contraseña de Postgres**: evita caracteres como `@ : / # ?` si construyes una `DATABASE_URL`, porque rompen la URL.

## 4. `docker/entrypoint.sh` (patrón general)

Se ejecuta **cada vez** que arranca el contenedor. Responsabilidades:

1. Esperar a que PostgreSQL acepte conexiones.
2. Ejecutar migraciones / preparar el framework.
3. Lanzar el proceso principal con `exec "$@"` (para que reciba las señales de Docker).

Plantilla base:

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

# --- Tareas específicas del framework aquí ---

exec "$@"
```

Recuerda darle permisos de ejecución **antes de hacer commit**:

```bash
chmod +x docker/entrypoint.sh
git update-index --chmod=+x docker/entrypoint.sh
```

> ⚠️ **Fin de línea**: el archivo debe guardarse con **LF**, no CRLF (Windows). Si ves `exec ./entrypoint.sh: no such file or directory`, es casi seguro el CRLF. Añade un `.gitattributes`:
> ```
> *.sh text eol=lf
> ```

## 5. Healthchecks

Un healthcheck permite que `depends_on` espere a que un servicio esté realmente listo.

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 10s
```

> El doble `$$` evita que Compose interpole la variable en el host; se resuelve dentro del contenedor.

## 6. Buenas prácticas para todos los Dockerfile

- Usa **multi-stage builds** para dejar solo lo necesario en la imagen final.
- Copia primero los archivos de dependencias (`package.json`, `requirements.txt`, `composer.json`) y luego el código: aprovecha la **caché de capas**.
- Corre la app con **usuario no-root**.
- Fija versiones de imágenes base (`python:3.12-slim`, `node:20-alpine`, `php:8.3-fpm-alpine`).
- No copies secretos dentro de la imagen.
