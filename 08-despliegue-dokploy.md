# 08 · Despliegue paso a paso en Dokploy

## Requisitos previos

- VPS con Dokploy instalado y accesible (`http://IP_DEL_VPS:3000`).
- Un dominio o subdominio con un registro **DNS tipo A** apuntando a la IP del VPS.
- Repositorio en GitHub/GitLab/Gitea con los archivos de esta guía.

## Checklist local antes de subir

```bash
# 1. Probar que el compose es válido
docker compose config

# 2. Probar el build y el arranque en local
cp .env.example .env      # edita valores de prueba
docker compose up --build

# 3. Verificar que responde y que la BD se conecta
```

Si funciona en local, funcionará en Dokploy.

## Paso 1 · Crear el proyecto

1. Dokploy → **Projects** → **Create Project** (ej.: `Mi Cliente`).
2. Dentro del proyecto → **Create Service** → **Compose**.
3. Nombre: `miapp-prod`. Tipo: **Docker Compose**.

## Paso 2 · Conectar el repositorio

En la pestaña **General** del compose:

| Campo | Valor |
|-------|-------|
| Provider | GitHub / GitLab / Git |
| Repository / Branch | tu repo y `main` |
| Compose Path | `./docker-compose.yml` |
| Trigger Type | *On Push* (auto-deploy) |

> Repositorio privado: conecta la cuenta de GitHub desde **Settings → Git** o usa una **SSH key** de deploy.

## Paso 3 · Variables de entorno

Pestaña **Environment**: pega el contenido de tu `.env.example` con **valores reales**:

```env
POSTGRES_DB=miapp
POSTGRES_USER=miapp_user
POSTGRES_PASSWORD=<clave_larga_aleatoria>
...
```

> Toda variable con `:?` en el compose (obligatoria) hará fallar el deploy si falta. Es intencional.

## Paso 4 · Dominio

Pestaña **Domains** → **Add Domain**:

| Campo | Django | Laravel | Next.js |
|-------|--------|---------|---------|
| Service Name | `nginx` | `web` | `app` |
| Port | `80` | `80` | `3000` |
| Host | `miapp.midominio.com` | igual | igual |
| HTTPS | ✅ | ✅ | ✅ |
| Certificate | Let's Encrypt | Let's Encrypt | Let's Encrypt |

## Paso 5 · Desplegar

1. Botón **Deploy**.
2. Pestaña **Deployments** → revisa el log del build.
3. Pestaña **Logs** → selecciona el servicio (`app`, `db`, `nginx`) y comprueba que arrancan bien.

Debes ver líneas como:

```
✅ PostgreSQL disponible
🔄 Aplicando migraciones...
```

## Paso 6 · Verificación

- [ ] `https://miapp.midominio.com` carga con candado 🔒.
- [ ] Login / registro funciona (escribe en la BD).
- [ ] Los estáticos cargan (CSS/JS/imágenes).
- [ ] Al hacer un `git push` se redepliega solo.
- [ ] Los datos siguen ahí tras un redeploy (el volumen persiste).

## Actualizaciones

```
git push origin main  →  webhook  →  Dokploy build + redeploy automático
```

- Cambios en variables de entorno → **Deploy** manual (o *Rebuild* si son `NEXT_PUBLIC_*`).
- **Rollback**: pestaña *Deployments* → redeploy de una versión anterior.

## Buenas prácticas de operación

1. Activa **backups** de la BD y súbelos a almacenamiento externo (ver [03-postgres.md](03-postgres.md)).
2. Usa una rama `main` (producción) y otra `develop` (staging) con servicios Compose separados.
3. No borres volúmenes desde Docker/Dokploy sin backup.
4. Revisa **Monitoring** de Dokploy (CPU / RAM / disco) periódicamente.
5. Limpia imágenes viejas: Dokploy → **Settings → Server → Clean Docker** (o `docker system prune -af` con cuidado).
