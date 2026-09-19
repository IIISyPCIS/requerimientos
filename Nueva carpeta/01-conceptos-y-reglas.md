# 01 · Conceptos y reglas de Dokploy

## ¿Qué es Dokploy?

Dokploy es un PaaS *self-hosted* que corre sobre Docker. Nos da:

- Despliegue desde **GitHub / GitLab / Gitea / Bitbucket** con webhooks (auto-deploy).
- **Traefik** integrado como reverse proxy → dominios y certificados **HTTPS (Let's Encrypt)** automáticos.
- Gestión de variables de entorno, logs, monitoreo y backups.

## Tipos de aplicación que usaremos

| Tipo | Cuándo usarlo |
|------|---------------|
| **Docker Compose** ✅ (recomendado) | Proyecto con app + BD (+ nginx, workers, etc.) en un solo despliegue. **Es el estándar en esta guía.** |
| **Application** (Dockerfile) | App suelta que usa una BD creada aparte desde *Databases* en Dokploy. |

## Flujo de una petición

```
Usuario ──HTTPS──▶ Traefik (Dokploy) ──HTTP──▶ [nginx | app] ──▶ PostgreSQL
                    (SSL, dominio)              (red interna)     (red interna)
```

- **Traefik** termina el SSL. Dentro de nuestros contenedores todo viaja en HTTP.
- Por eso las apps deben confiar en el header `X-Forwarded-Proto` (ver cada framework).

## Reglas obligatorias para el `docker-compose.yml`

### ✅ Hacer

- Usar `expose:` para indicar el puerto interno del servicio.
- Declarar la red externa de Dokploy en el servicio que recibe tráfico:
  ```yaml
  networks:
    - default
    - dokploy-network
  ```
  ```yaml
  networks:
    dokploy-network:
      external: true
  ```
- Usar **volúmenes con nombre** para datos persistentes (BD, media, storage).
- Leer la configuración con `${VARIABLE}` (Dokploy genera el `.env` desde su pestaña *Environment*).
- Agregar `restart: unless-stopped`.

### ❌ No hacer

| Mal | Por qué |
|-----|---------|
| `ports: - "80:80"` | Choca con Traefik y otros proyectos del VPS. |
| `container_name: mi-app` | Rompe redeploys y despliegues aislados. |
| `ports: - "5432:5432"` | Expone la BD a internet. |
| Guardar datos en carpetas del repo (`./data`) | El repo se limpia en cada deploy y se pierden. |
| Subir `.env` con secretos | Riesgo de seguridad. |
| Usar `latest` como tag | Despliegues no reproducibles. Fija versión (`postgres:16-alpine`). |

## Cómo llegan las variables a la app

```
Dokploy → pestaña Environment → genera archivo .env junto al compose
        → docker compose lee ${VAR} → se pasan a `environment:` del servicio
```

Cada variable que la app necesite **debe estar declarada en `environment:`** del servicio (o usar `env_file: .env`).

## Dominios

En la pestaña **Domains** de Dokploy se configura:

- **Service Name**: el servicio del compose que recibe el tráfico (`nginx`, `web` o `app`).
- **Port**: el puerto interno (el que pusiste en `expose`).
- **HTTPS**: activado, certificado `Let's Encrypt`.
- **Host**: `miapp.midominio.com` (el DNS tipo A debe apuntar a la IP del VPS).
