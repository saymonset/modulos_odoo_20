# Docker Context — Rutas y Contenedores (Odoo 20 / lead20)

## Ubicación de trabajo

- `/home/odoo/lead20/modulos_odoo_20` — **único clone del repo de módulos** (no existe prod 20).
- `/home/odoo/lead20/odoo20-skeleton/odoo20/` — compose y scripts de la instancia lead 20.
- **Todavía no existe** `/home/odoo/prod/modulos_odoo_20` ni pipeline de despliegue 20.

## Contenedores (proyecto compose `odoo20-lead`)

| Propósito | Contenedor | Puertos (host) | BD |
|-----------|------------|----------------|-----|
| WEB | `odoo-20-web-leads` | 127.0.0.1:38069, :38072 (gevent) | `dbodoo20` en `odoo-db20-leads` |
| DB | `odoo-db20-leads` | 127.0.0.1:5436 → 5432 | imagen `pgvector/pgvector:pg16` |

Red: `odoo_network_20`. Imagen web: `odoo-pers:20`.

## Scripts (dentro de `odoo20/`)

`1_build_imagen.sh` · `3_stop-all.sh` · `4_start-all.sh` · `6_status_all_services.sh` ·
`7_logs_see_all_services.sh` · `generate_odoo_conf.sh`.

## Addons path (en `v20-leads/config/odoo.conf` del contenedor)

`/opt/odoo/odoo-core/addons,/opt/odoo/custom-addons/extra,/opt/odoo/custom-addons/oca`

Bind mounts (fuente → contenedor):
- `/home/odoo/lead20/modulos_odoo_20/shared/extra/20.0` → `/opt/odoo/custom-addons/extra`
- `/home/odoo/lead20/modulos_odoo_20/shared/oca/20.0` → `/opt/odoo/custom-addons/oca`

Otros volúmenes: `./v20-leads/odoo-web-data` (filestore), `./v20-leads/config`, `./v20-leads/logs`.

## Docker gotchas

1. Tras editar `.py`, borrar `__pycache__` y reiniciar: `docker restart odoo-20-web-leads`.
2. Upgrade CLI: `docker exec odoo-20-web-leads odoo -d dbodoo20 -u <modulo> --stop-after-init`
3. `Registry.new(db, update_module=True)` **no carga** `addons_path` custom. Para forzar recarga
   de vistas: subir `version` en `__manifest__.py`, reiniciar y Upgradar desde UI.
4. Snippets Python vía `docker exec ... python3 -c "..."` siempre en **una sola línea**.
5. Secret BD: `odoo20/secrets/postgres_password.txt` (user `odoo`). **No commitear** (ya está en `.gitignore`).
