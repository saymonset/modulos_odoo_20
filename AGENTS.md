# AGENTS.md — modulos_odoo_20

Repositorio de módulos para Odoo 20 CE. Monorepo espejo de `modulos_odoo` (18.0/19.0),
pero **sin contenido todavía**: la estructura base se montó en la SPEC 73 y la
migración/port de módulos se hará en specs futuras.

## Estructura

- `shared/oca/20.0/<modulo>` — módulos OCA vendored. No editar salvo migración puntual.
- `shared/extra/20.0/<modulo>` — módulos propios/terceros. Desarrollo activo aquí.
- Nunca mezclar OCA con extra; organizar por subcarpeta de versión.

## Regla crítica: lead vs prod

- Sesiones de código: correr opencode **solo** bajo `/home/odoo/lead/`.
- Si el working directory es `/home/odoo/prod/...`, detenerse y pedir al usuario
  mover la sesión a lead. Solo hotfix de emergencia confirmado.
- Todavía no existe un `prod/modulos_odoo_20` ni pipeline de despliegue 20
  (fuera del alcance de la SPEC 73).

## Docker (lead 20)

- Skeleton: `/home/odoo/lead/odoo20-skeleton/odoo20/` (compose db+web, imagen `odoo-pers:20`).
- Web: `:38069`, gevent: `:38072`, BD `dbodoo20` en host `:5436`, red `odoo_network_20`.
- Addons custom entran por bind mount a `/opt/odoo/custom-addons/{extra,oca}`.
