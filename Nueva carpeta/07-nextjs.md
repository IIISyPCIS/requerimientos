# 07 · Next.js

Stack: **Next.js (modo `standalone`) + PostgreSQL 16**. No necesita Nginx: Traefik ya publica el dominio.

## Estructura

```
proyecto/
├── src/ o app/
├── package.json / package-lock.json
├── next.config.js
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env.example
└── docker/
    └── entrypoint.sh
```

## 1. Activar el modo `standalone`

`next.config.js` (o `.mjs` / `.ts`)

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};

module.exports = nextConfig;
```

Genera una carpeta mínima con solo lo necesario → imagen final de ~150 MB en vez de 1 GB.

## 2. `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1

# ───────── 1. Dependencias ─────────
FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ───────── 2. Build → target: builder ─────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Las variables NEXT_PUBLIC_* se "queman" en el build. Deben pasarse como ARG.
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
ENV NEXT_TELEMETRY_DISABLED=1

RUN npm run build

# ───────── 3. Runtime → target: runner ─────────
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    PORT=3000 \
    HOSTNAME=0.0.0.0

RUN addgroup --system --gid 1001 nodejs \
    && adduser --system --uid 1001 nextjs

COPY --from=builder --chown=nextjs:nodejs /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --chown=nextjs:nodejs docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER nextjs
EXPOSE 3000

ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "server.js"]
```

> Si no tienes carpeta `public/`, créala vacía (con un `.gitkeep`) o elimina la línea `COPY ... /public`.
> Con **pnpm** o **yarn** cambia `npm ci` por `pnpm install --frozen-lockfile` / `yarn install --frozen-lockfile`.

## 3. `docker/entrypoint.sh`

En Next.js el entrypoint solo espera a la BD (las migraciones se hacen en un servicio aparte, porque el modo `standalone` no incluye el CLI del ORM).

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

exec "$@"
```

## 4. `docker-compose.yml`

```yaml
services:
  # Tarea de un solo uso: aplica migraciones y termina.
  migrate:
    build:
      context: .
      target: builder            # imagen con node_modules y CLI del ORM
    command: npx prisma migrate deploy      # ← cambia según tu ORM (ver tabla abajo)
    restart: "no"
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
    depends_on:
      db:
        condition: service_healthy

  app:
    build:
      context: .
      target: runner
      args:
        NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-}
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      POSTGRES_HOST: db
      POSTGRES_PORT: 5432
      AUTH_SECRET: ${AUTH_SECRET:-}
      # añade aquí el resto de variables privadas de tu app
    expose:
      - "3000"
    networks:
      - default
      - dokploy-network
    depends_on:
      db:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3000/ >/dev/null 2>&1 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

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

networks:
  dokploy-network:
    external: true
```

**En Dokploy → Domains**: Service `app`, Port `3000`, HTTPS activado.

## Comando de migración según ORM

| ORM | `command` del servicio `migrate` |
|-----|----------------------------------|
| Prisma | `npx prisma migrate deploy` |
| Drizzle | `npx drizzle-kit migrate` |
| TypeORM | `npm run typeorm migration:run` |
| Sin ORM / sin migraciones | Elimina el servicio `migrate` y su `depends_on` |

## ⚠️ Puntos críticos de Next.js

| Problema | Solución |
|----------|----------|
| `NEXT_PUBLIC_*` llega vacía | Son de **build-time**: deben ir en `build.args` (y `ARG` en el Dockerfile). Cambiarlas exige **redeploy con rebuild**. |
| La app no responde desde fuera | Falta `HOSTNAME=0.0.0.0` (ya está en el Dockerfile). |
| Error al conectar a la BD durante el `build` | No consultes la BD en tiempo de build (`generateStaticParams`, `getStaticProps`), o usa `export const dynamic = "force-dynamic"`. |
| Prisma: `binaryTargets` | Añade `binaryTargets = ["native", "linux-musl-openssl-3.0.x"]` en `schema.prisma` para Alpine. |
| Imágenes de `next/image` lentas | Instala `sharp` (`npm i sharp`) — en Next 15 ya se incluye. |
