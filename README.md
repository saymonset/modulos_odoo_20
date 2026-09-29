# modulos_odoo_20

Repositorio de módulos (OCA vendored + propios) para **Odoo 20 CE**.

Nació vacío y así está hoy: solo la estructura base para montar el stack
docker de lead (`odoo20-skeleton/odoo20/`, SPEC 73). **No se migró ningún
módulo desde 19**; cada port irá en su propia spec.

## Estructura

- `shared/oca/20.0/<modulo>` — módulos OCA vendored. No editar salvo migración puntual.
- `shared/extra/20.0/<modulo>` — módulos propios/terceros. Aquí irá el desarrollo activo.

Nunca mezclar OCA con extra; organizar por subcarpeta de versión.

## Stack docker (lead)

| Elemento | Valor |
|----------|-------|
| Web | `http://localhost:38069` (gevent `:38072`) |
| Contenedores | `odoo-20-web-leads` / `odoo-db20-leads` |
| BD | `dbodoo20` (PostgreSQL en `:5436`), inicializada **con demo** |
| Bind mounts | `shared/extra/20.0` → `/opt/odoo/custom-addons/extra`, `shared/oca/20.0` → `/opt/odoo/custom-addons/oca` |

Ciclo de vida con los scripts de `~/lead/odoo20-skeleton/odoo20/`
(`4_start-all.sh`, `3_stop-all.sh`, `6_status_all_services.sh`, `7_logs_see_all_services.sh`).
