# 🐳 Guía de Dockerización para desplegar en Dokploy

Esta guía explica cómo preparar **cualquier proyecto** (Django, Laravel, Next.js u otro) para desplegarlo en nuestro VPS con **Dokploy**, usando **PostgreSQL como base de datos obligatoria**.

> Objetivo: que todos los proyectos tengan la misma estructura, las mismas convenciones y que un despliegue sea simplemente `git push` + *Deploy*.

## 📚 Índice

| # | Documento | Contenido |
|---|-----------|-----------|
| 01 | [Conceptos y reglas de Dokploy](01-conceptos-y-reglas.md) | Cómo funciona Dokploy, reglas que **todo** proyecto debe cumplir |
| 02 | [Archivos comunes](02-archivos-comunes.md) | `.dockerignore`, `.gitignore`, `.env.example`, `entrypoint.sh`, healthchecks |
| 03 | [PostgreSQL](03-postgres.md) | Servicio de BD, volúmenes, backups, conexión |
| 04 | [Nginx](04-nginx.md) | Cuándo usarlo y configuraciones base |
| 05 | [Django](05-django.md) | Dockerfile, compose, entrypoint, settings |
| 06 | [Laravel](06-laravel.md) | Dockerfile (PHP-FPM + Nginx), compose, entrypoint |
| 07 | [Next.js](07-nextjs.md) | Dockerfile standalone, compose, migraciones |
| 08 | [Despliegue paso a paso en Dokploy](08-despliegue-dokploy.md) | Desde el repo hasta el dominio con HTTPS |
| 09 | [Troubleshooting y checklist](09-troubleshooting-checklist.md) | Errores comunes y lista de verificación final |

## 🗂️ Estructura recomendada del repositorio

```
mi-proyecto/
├── docker/
│   ├── entrypoint.sh          # Script de arranque (espera DB, migra, etc.)
│   └── nginx/
│       └── default.conf       # Solo si el proyecto usa Nginx
├── Dockerfile                 # Imagen de la aplicación
├── docker-compose.yml         # Servicios: app + db (+ nginx)
├── .dockerignore
├── .env.example               # Plantilla de variables (SÍ se sube)
├── .gitignore                 # Incluye .env (NO se sube el real)
└── ... código del proyecto
```

## ⚡ Resumen en 10 reglas

1. **PostgreSQL siempre** (`postgres:16-alpine`), con volumen con nombre.
2. **Nunca** subir `.env` al repositorio; solo `.env.example`.
3. Las variables reales se cargan en la pestaña **Environment** de Dokploy.
4. **No usar `ports:`** en el compose; usar `expose:` y asignar el dominio desde Dokploy.
5. **No usar `container_name`**.
6. La BD **no se expone** a internet (sin `5432:5432`).
7. Datos persistentes → **volúmenes con nombre**, nunca dentro del contenedor.
8. Contenedor de la app con **usuario no-root**.
9. El `entrypoint.sh` **espera a la BD**, ejecuta migraciones y luego lanza la app.
10. Cada servicio con **`restart: unless-stopped`** y **healthcheck** cuando sea posible.

## 🚀 Inicio rápido

1. Copia el `Dockerfile`, `docker-compose.yml`, `docker/entrypoint.sh` y `.dockerignore` del documento de tu tecnología.
2. Crea tu `.env.example` a partir de [02-archivos-comunes.md](02-archivos-comunes.md).
3. Sigue [08-despliegue-dokploy.md](08-despliegue-dokploy.md).
