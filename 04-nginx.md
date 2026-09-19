# 04 · Nginx

## ¿Cuándo usarlo?

Dokploy ya trae **Traefik** (dominio + HTTPS). Nginx **dentro del proyecto** solo se añade cuando aporta algo:

| Tecnología | ¿Nginx en el compose? | Motivo |
|------------|:---------------------:|--------|
| **Django** | Recomendado | Sirve `/static/` y `/media/` sin pasar por gunicorn. (Alternativa simple: WhiteNoise, sin nginx) |
| **Laravel** | **Obligatorio** | PHP-FPM no habla HTTP; necesita un servidor web delante (FastCGI). |
| **Next.js** | No | Node ya sirve HTTP y Traefik hace el resto. |

## Reglas

- El servicio nginx es el que recibe el dominio en Dokploy (puerto `80`).
- **No** configures SSL en nginx: lo hace Traefik.
- Reenvía el header `X-Forwarded-Proto` que manda Traefik, para que la app sepa que el usuario entró por HTTPS.
- Sube el tamaño máximo de subida (`client_max_body_size`).

## Plantilla A: Nginx como proxy inverso (Django, Node, etc.)

`docker/nginx/default.conf`

```nginx
upstream app_server {
    server app:8000;          # nombre del servicio : puerto interno
}

server {
    listen 80;
    server_name _;

    client_max_body_size 50M;

    # Archivos estáticos y media (volúmenes compartidos con la app)
    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        access_log off;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /app/media/;
        expires 7d;
        access_log off;
    }

    location / {
        proxy_pass http://app_server;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $http_x_real_ip;
        proxy_set_header X-Forwarded-For   $http_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;

        proxy_read_timeout 60s;
        proxy_redirect off;
    }
}
```

> Se usan `$http_x_forwarded_*` (los que manda Traefik) en vez de `$scheme` / `$remote_addr`, que dentro de la red Docker siempre serían HTTP y la IP de Traefik.

## Plantilla B: Nginx + PHP-FPM (Laravel)

`docker/nginx/default.conf`

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;

    client_max_body_size 50M;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;                  # servicio php-fpm
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param HTTPS $http_x_forwarded_proto if_not_empty;
        fastcgi_read_timeout 60s;
    }

    location ~* \.(?:css|js|jpg|jpeg|gif|png|svg|ico|webp|woff2?)$ {
        expires 30d;
        access_log off;
        add_header Cache-Control "public";
        try_files $uri =404;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## Cómo montarlo en el compose

**Opción 1 (simple): bind mount del archivo de configuración**

```yaml
nginx:
  image: nginx:1.27-alpine
  restart: unless-stopped
  volumes:
    - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
  expose:
    - "80"
  networks:
    - default
    - dokploy-network
  depends_on:
    - app
```

**Opción 2 (más robusta): construir la imagen con la config incluida** — ver el target `web` en [06-laravel.md](06-laravel.md).

## Verificar la configuración

```bash
docker compose exec nginx nginx -t      # valida sintaxis
docker compose logs -f nginx            # logs
```
