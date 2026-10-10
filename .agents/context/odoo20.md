# Odoo 20 Specific — Cambios verificados en este entorno

## Transporte API (verificado en lead20, dbodoo20)

- `/jsonrpc` legacy fue **REMOVIDO**: devuelve `ValueError` (dict update). Usar JSON-2.
- Endpoint: `POST /json/2/<model>/<method>` con body `{"params": {...}}` — kwargs **nombrados**
  (posiciónales → 422 `missing a required argument`).
- Auth: `Authorization: Bearer <api_key>`; Basic auth **ya no funciona** ("use an API Key with a
  Bearer Authorization header").
- BD vía header `X-Odoo-Database: <dbname>`.
- `odoo-mcp` funciona con `ODOO_TRANSPORT=json2` + `ODOO_JSON2_DATABASE_HEADER=1` (igual que 19).

## Vistas XML (arrastrado de 19 — verificar al portear cada módulo)

- `attrs` no existe: atributos directos `invisible="..."` / `required="..."`.
- List view: `<list>` en vez de `<tree>`.

## QWeb (arrastrado de 19 — verificar al portear)

- `hasclass('...')` en vez de `contains(@class, ...)`.
- `t-set` no se comparte entre bloques `<xpath>` — inline la búsqueda en cada uso.

## Monedas (arrastrado de 19 — verificar al portear)

- COP no tiene xml_id garantizado: `env.ref('base.COP', raise_if_not_found=False) or env['res.currency'].sudo().search([('name','=','COP')], limit=1)`.
- Campos monetarios con `$` ambiguo (USD/COP): añadir etiqueta explícita `COP`.

## Overrides de métodos core

Mantener la firma exacta del padre, sin `*args/**kwargs`. En 20, contrastar la firma con el
código real del contenedor (`/opt/odoo/odoo-core/...`) o Context7 antes de asumir la de 19.
