# Sonrisas de Cristal — infraestructura

Un solo `docker-compose.yml` que levanta todo: **PostgreSQL**, **API (FastAPI)**, **pgAdmin**, **postgres_exporter**, **Prometheus** y **Grafana**. El túnel de Cloudflare se agregará después.

```
sonrisas_cristal_infra/   <- este repo (solo compose + configuración)
sonrisas_cristal_api/     <- código de la API   (build: ../sonrisas_cristal_api)
sonrisas_cristal_db/      <- esquema e init SQL (build: ../sonrisas_cristal_db)
```

Los tres repos deben ser carpetas hermanas.

## Servicios

| Servicio | Puerto en tu PC | Para qué |
|---|---|---|
| `api` | 8000 | API REST (se quitará cuando entre Cloudflare) |
| `grafana` | 3000 | Dashboards (usuario y contraseña del `.env`) |
| `pgadmin` | 5050 | Administrar la BD |
| `prometheus` | 127.0.0.1:9090 | Solo local; Grafana lo lee por la red interna |
| `postgres`, `postgres_exporter` | — | Solo dentro de la red Docker |

## Primer arranque

1. Detén los compose viejos de los otros repos (sin `-v`): `docker compose down` en `sonrisas_cristal_api` y en `sonrisas_cristal_db`. Usan los mismos puertos.
2. `copy .env.example .env` y llena los valores (`JWT_SECRET`, `METRICS_TOKEN`, contraseñas, `DATABASE_URL` con la contraseña codificada).
3. `docker compose up -d --build`
4. (Opcional, datos de demo) `docker compose exec api python -m scripts.seed`
5. Grafana: <http://localhost:3000> → carpeta **Sonrisas de Cristal** → *API* y *PostgreSQL*.
6. Prometheus: <http://localhost:9090/targets> debe mostrar `api`, `postgres` y `prometheus` en **UP**.

Este compose usa volúmenes nuevos (`sonrisas_postgres_data`, ...): la BD empieza **vacía**. Los datos del compose viejo siguen en su volumen y no se tocan.

## Cómo se conecta el monitoreo

- **API → Prometheus:** la API expone `GET /metrics` con `Authorization: Bearer <METRICS_TOKEN>`. El token sale del mismo `.env` y llega a Prometheus como secreto de Docker (`/run/secrets/metrics_token`).
- **PostgreSQL → Prometheus:** `postgres_exporter` entra como el usuario `monitor` (rol `pg_monitor`, sin acceso a los datos). Ese usuario lo crea `init/03-monitor-user.sh` del repo de la BD.
- **Prometheus → Grafana:** el datasource y los dos dashboards se cargan solos desde `grafana/provisioning` y `grafana/dashboards`.

## Si la BD ya existía

Los scripts de `init/` solo corren con un volumen vacío. Si tu volumen ya tiene datos y no existe el usuario `monitor`, créalo una vez:

```
docker compose exec postgres psql -U postgres -d sonrisas_cristal -c "CREATE ROLE monitor LOGIN PASSWORD '<MONITOR_DB_PASSWORD>'; GRANT pg_monitor TO monitor;"
```

## Pendiente

- Túnel de Cloudflare (quitar el puerto 8000 y poner `TRUSTED_PROXIES` con la subred de `sonrisas_sonrisas_net`).
- Publicar las imágenes en GHCR y cambiar `build:` por `image:`.
- Alertas (Alertmanager) si hacen falta.
