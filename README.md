# futbol-demo-deploy

Stack de despliegue de la **demo publica** del complejo de futbol 5 (para LinkedIn).
Levanta 3 servicios como un solo proyecto Compose (`futbol-demo`): frontend (nginx + React),
API (.NET en modo demo) y SQL Server, todo **aislado** del stack productivo.

## Que hay aca

| Archivo | Descripcion |
|---------|-------------|
| `docker-compose.yml` | Stack demo (db + api + frontend), parametrizado por `.env` |
| `.env.example` | Plantilla de variables (secretos + dominio) |
| `nginx/demo.conf` | Reverse proxy del host (un dominio, ruteo por path + TLS) |

## Requisitos

- Docker Engine + Compose.
- nginx en el host (para el reverse proxy) y `certbot` (para HTTPS).
- Un **dominio** con un registro DNS **A** apuntando a la IP del VPS.
  - Ej (DuckDNS gratis): `futbol-complejo-demo.duckdns.org` -> IP del VPS.

El frontend y la API se sirven bajo el **mismo dominio**, ruteando por path:
`/` -> frontend · `/api/` -> API. Asi hay un solo certificado TLS y no hay CORS cross-origin.

## Layout esperado

Este repo vive **al lado** de los dos repos de la aplicacion:

```
futbol-demo-deploy/
FutbolComplejoAPI/           <- git clone ... FutbolComplejoAPI
Frontend_FutbolComplejo/     <- git clone ... Frontend_FutbolComplejo
```

## Deploy (paso a paso)

```bash
# 1. Clonar los 3 repos (los de la app con los nombres exactos)
git clone https://github.com/SantiSSD/ComplejoFutbol5Api.git FutbolComplejoAPI
git clone https://github.com/SantiSSD/futbol-complejo-frontend.git Frontend_FutbolComplejo
git clone https://github.com/SantiSSD/futbol-demo-deploy.git

cd futbol-demo-deploy

# 2. Configurar secretos + dominio
cp .env.example .env
nano .env
#   DB_SA_PASSWORD=...        (min 8, complejidad)
#   JWT_KEY=...               (openssl rand -base64 48)
#   DEMO_WEB_ORIGIN=https://futbol-complejo-demo.duckdns.org
#   DEMO_API_URL=https://futbol-complejo-demo.duckdns.org/api/v1

# 3. Levantar el stack demo (construye imagenes)
docker compose --env-file .env up -d --build

# 4. Reverse proxy + HTTPS (en el host)
sudo cp nginx/demo.conf /etc/nginx/sites-available/futbol-demo.conf
sudo nano /etc/nginx/sites-available/futbol-demo.conf   # confirmar el dominio
sudo ln -s /etc/nginx/sites-available/futbol-demo.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d futbol-complejo-demo.duckdns.org
```

## Verificacion

```bash
docker compose --env-file .env ps          # db y api "healthy"; frontend Up
docker compose --env-file .env logs -f api-demo
curl -I https://futbol-complejo-demo.duckdns.org
curl -I https://futbol-complejo-demo.duckdns.org/api/v1/roles   # 401 (sin token) OK
```

## Credenciales demo

- Administrador: `admin` / `Admin123!`
- Recepcionista: `recepcionista` / `Recepcionista123!`

(Los datos son sinteticos y se **auto-resetean cada 6h**.)

## Comandos utiles

```bash
docker compose --env-file .env ps
docker compose --env-file .env logs -f api-demo
docker compose --env-file .env down          # baja (conserva datos)
docker compose --env-file .env down -v       # baja y borra datos de la demo
docker compose --env-file .env up -d --build # redeploy tras cambios
```

## Seguridad

- La demo corre **aislada**: base `FutbolComplejoDemo`, volumen `db-demo-data`, puertos atados a `127.0.0.1`.
- El backend tiene **triple guardia** para no tocar nunca la base productiva (flag + doble opt-in + verificacion del nombre de BD).
- La demo debe correr con **1 sola replica** (el gate/estado del auto-reset es in-memory).
- El `.env` real **no se versiona**.
