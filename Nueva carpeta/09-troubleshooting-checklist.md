# 09 · Troubleshooting y checklist final

## 🔧 Errores frecuentes

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| `502 Bad Gateway` / `404 page not found` (de Traefik) | Puerto o servicio incorrecto en *Domains* | Verifica que *Service Name* y *Port* coincidan con el compose (`expose`). Revisa que el servicio esté en `dokploy-network`. |
| `exec /entrypoint.sh: no such file or directory` | Fin de línea CRLF (Windows) | Convierte a LF y añade `.gitattributes` con `*.sh text eol=lf`. |
| `exec /entrypoint.sh: permission denied` | Sin permiso de ejecución | `chmod +x docker/entrypoint.sh` y `git update-index --chmod=+x`. |
| `password authentication failed for user` | Cambiaste `POSTGRES_PASSWORD` con el volumen ya creado | Cambia la clave con `ALTER USER`, o borra el volumen (⚠️ pierde datos). |
| `could not translate host name "db"` | La app no comparte red con `db`, o el servicio se llama distinto | El host debe ser el **nombre del servicio** del compose. |
| `port is already allocated` | Usaste `ports:` | Cambia a `expose:` y configura el dominio en Dokploy. |
| Página sin CSS/JS (Django) | No hay `collectstatic` o el volumen no se comparte con nginx | Revisa `static_data` en `app` y `nginx`, y el `alias` del nginx.conf. |
| Django: `CSRF verification failed` | Falta `CSRF_TRUSTED_ORIGINS` con `https://` | `DJANGO_CSRF_TRUSTED_ORIGINS=https://tu-dominio.com` |
| Django: `Bad Request (400)` | `ALLOWED_HOSTS` sin tu dominio | Añádelo a `DJANGO_ALLOWED_HOSTS`. |
| Laravel: assets con `http://` (mixed content) | Proxies no confiables | `trustProxies(at: '*')` + `APP_URL=https://...` |
| Laravel: `No application encryption key` | Falta `APP_KEY` | Genera con `php artisan key:generate --show`. |
| Laravel: `Permission denied` en `storage/logs` | Volumen creado con permisos root | Entra al contenedor como root y `chown -R www-data:www-data storage`. |
| Laravel: `403` / `File not found` en imágenes subidas | Falta el enlace `public/storage` o el volumen no está en `web` | Revisa el `ln -s` del Dockerfile y que `web` monte `laravel_storage`. |
| Next.js: variable `NEXT_PUBLIC_*` vacía | No se pasó como build arg | Añádela en `build.args` y haz *Rebuild*. |
| Next.js: app arranca pero no responde | Falta `HOSTNAME=0.0.0.0` | Ya incluido en el Dockerfile; no lo quites. |
| Build muy lento / se queda sin memoria | VPS con poca RAM | Añade swap (`fallocate -l 2G /swapfile`) o construye la imagen en CI y usa `image:`. |
| Certificado SSL no se genera | DNS no apunta al VPS o puertos 80/443 cerrados | Verifica con `dig tu-dominio.com` y el firewall. |

## 🔍 Comandos de diagnóstico

```bash
# Ver contenedores del proyecto
docker ps --format "table {{.Names}}\t{{.Status}}"

# Logs en vivo de un servicio
docker logs -f <nombre_contenedor>

# Entrar a un contenedor
docker exec -it <nombre_contenedor> sh

# Revisar el compose ya interpolado con las variables
docker compose config

# Probar conexión a la BD desde la app
docker exec -it <contenedor_app> sh -c 'nc -zv db 5432'

# Espacio en disco usado por Docker
docker system df
```

## ✅ Checklist final antes de entregar un proyecto

**Repositorio**
- [ ] `Dockerfile` con multi-stage y usuario no-root
- [ ] `docker-compose.yml` sin `ports:` ni `container_name`
- [ ] `.dockerignore` presente
- [ ] `.env.example` completo y **sin** secretos reales
- [ ] `.env` en `.gitignore`
- [ ] `docker/entrypoint.sh` con permisos de ejecución y fin de línea LF
- [ ] `docker/nginx/default.conf` (si aplica)

**Base de datos**
- [ ] PostgreSQL 16 con volumen con nombre
- [ ] Healthcheck (`pg_isready`) y `depends_on: service_healthy`
- [ ] Puerto 5432 **no** publicado
- [ ] Plan de backups definido

**Aplicación**
- [ ] `DEBUG` desactivado en producción
- [ ] Secretos (`SECRET_KEY`, `APP_KEY`, `AUTH_SECRET`) generados aleatoriamente
- [ ] Confía en el proxy (`X-Forwarded-Proto`)
- [ ] Archivos subidos en volumen persistente
- [ ] Logs a `stdout`/`stderr`

**Dokploy**
- [ ] Variables cargadas en *Environment*
- [ ] Dominio → servicio y puerto correctos, HTTPS activo
- [ ] Auto-deploy por webhook configurado
- [ ] Probado un redeploy sin pérdida de datos
