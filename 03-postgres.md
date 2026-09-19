# 03 · PostgreSQL (obligatorio)

Todos los proyectos usan **PostgreSQL 16**. Hay dos formas de tenerlo en Dokploy.

## Opción A (estándar): PostgreSQL dentro del `docker-compose.yml`

La BD nace, vive y se despliega junto con el proyecto.

```yaml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:?Falta POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER:?Falta POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Falta POSTGRES_PASSWORD}
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
```

Los demás servicios se conectan usando **el nombre del servicio como host**:

```
host: db     puerto: 5432
```

Y esperan a que esté listo:

```yaml
depends_on:
  db:
    condition: service_healthy
```

### ⚠️ Puntos críticos

- `POSTGRES_USER/PASSWORD/DB` **solo se aplican la primera vez** (cuando el volumen está vacío). Si cambias la contraseña después en las variables, Postgres **no** la actualizará. Debes cambiarla con:
  ```bash
  docker exec -it <contenedor_db> psql -U <user> -c "ALTER USER <user> PASSWORD 'nueva';"
  ```
- Si necesitas empezar de cero (⚠️ borra los datos): elimina el volumen `postgres_data`.
- **Nunca** publiques el puerto 5432.

## Opción B: PostgreSQL creado desde Dokploy (*Databases*)

Dokploy → **Create Service → Database → PostgreSQL**. Ventajas: backups programados a S3 desde la UI y separación del ciclo de vida de la app.

Para que el compose la use hay que:

1. Unir el servicio de la app a la red `dokploy-network`.
2. Usar como host el **nombre interno** que muestra Dokploy en la BD (*Internal Host*), por ejemplo `miapp-db-abc123`.

```yaml
services:
  app:
    environment:
      POSTGRES_HOST: ${POSTGRES_HOST}   # internal host de Dokploy
    networks:
      - default
      - dokploy-network
```

## Conexión desde un cliente (DBeaver, pgAdmin, TablePlus)

No abras el puerto. Usa un **túnel SSH**:

```bash
ssh -L 5433:localhost:5432 usuario@IP_DEL_VPS
```

Como la BD no publica puerto, lo más práctico es entrar al contenedor:

```bash
docker exec -it $(docker ps -qf "name=db") psql -U $POSTGRES_USER -d $POSTGRES_DB
```

## Backups (Opción A)

### Backup manual

```bash
docker exec $(docker ps -qf "name=<proyecto>.*db") \
  pg_dump -U miapp_user -d miapp -Fc > backup_$(date +%F).dump
```

### Restaurar

```bash
cat backup_2025-01-01.dump | docker exec -i <contenedor_db> \
  pg_restore -U miapp_user -d miapp --clean --if-exists
```

### Backup automático con Dokploy

En **Schedules** del compose (o un cron del VPS) ejecuta el `pg_dump` y guarda el archivo en un volumen o súbelo a un almacenamiento externo (S3, Backblaze, etc.). **Un backup que solo existe en el mismo VPS no es un backup.**

## Variables estándar de conexión

Usamos siempre estas variables (cada framework las mapea a su configuración):

| Variable | Ejemplo |
|----------|---------|
| `POSTGRES_HOST` | `db` |
| `POSTGRES_PORT` | `5432` |
| `POSTGRES_DB` | `miapp` |
| `POSTGRES_USER` | `miapp_user` |
| `POSTGRES_PASSWORD` | `********` |

Si tu ORM pide una URL:

```
postgresql://POSTGRES_USER:POSTGRES_PASSWORD@POSTGRES_HOST:POSTGRES_PORT/POSTGRES_DB
```
