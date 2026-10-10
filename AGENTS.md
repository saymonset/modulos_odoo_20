# AGENTS.md — modulos_odoo_20

Repositorio de módulos para Odoo 20 CE. Monorepo espejo de `modulos_odoo` (18.0/19.0),
pero **sin contenido todavía**: la estructura base se montó en la SPEC 73 y la
migración/port de módulos se hará en specs futuras.

## Estructura

- `shared/oca/20.0/<modulo>` — módulos OCA vendored. No editar salvo migración puntual.
- `shared/extra/20.0/<modulo>` — módulos propios/terceros. Desarrollo activo aquí.
- Nunca mezclar OCA con extra; organizar por subcarpeta de versión.

## Regla crítica: lead vs prod

- Sesiones de código: correr opencode **solo** bajo `/home/odoo/lead20/`.
- Si el working directory es `/home/odoo/prod/...`, detenerse y pedir al usuario
  mover la sesión a lead. Solo hotfix de emergencia confirmado.
- Todavía no existe un `prod/modulos_odoo_20` ni pipeline de despliegue 20
  (fuera del alcance de la SPEC 73).
- `opencode.jsonc` apunta a `instructions.md` (clean code, respuestas cortas).

## Docker (lead 20)

- Skeleton: `/home/odoo/lead20/odoo20-skeleton/odoo20/` (compose db+web, imagen `odoo-pers:20`).
- Web: `:38069`, gevent: `:38072`, BD `dbodoo20` en host `:5436`, red `odoo_network_20`.
- Addons custom entran por bind mount a `/opt/odoo/custom-addons/{extra,oca}`.

## Contextos disponibles (carga bajo demanda)

| Contexto | Cuándo leer |
|----------|-------------|
| `.agents/context/testing.md` | Escribir o ejecutar tests |
| `.agents/context/cicd.md` | Modificar pipeline, deploy, runner |
| `.agents/context/docker.md` | Rutas, contenedores, bind mounts, gotchas |
| `.agents/context/odoo20.md` | Particularidades Odoo 20 (JSON-2/Bearer, quirks a verificar) |

## Skills

| Skill | Cuándo usar |
|-------|-------------|
| `odoo-guidelines` / `odoo-review` / `odoo-security` / `odoo-web-guidelines` / `odoo-git` | Skills **oficiales de odoo/odoo 20.0** (van juntas) |
| `odoo-19` | Referencia de conocimiento 19 para el port (no para código nuevo 20) |
| `odoo-development` | Desarrollo general Odoo (fuera de v20) |
| `spec` / `spec-impl` | Feature grande: `/spec` → approve → `/spec-impl` |
| `spec-verify` | Verificar un spec: `/spec-verify NN` (agente + comando en `opencode.jsonc`) |
| `context7-mcp` | Docs de librerías externas |
| `cavecrew` | Delegar a `cavecrew-investigator`/`builder`/`reviewer` para ahorrar contexto |
| `caveman` | Modo conciso: `/caveman` o "be less tokens"; sale con "normal mode" |
| `caveman-commit` | Mensaje de commit Conventional comprimido (`/commit`) |
| `caveman-review` | Revisión de diff comprimida (`/caveman-review`) |
| `caveman-compress` | Comprimir archivos de memoria / listas de todo |
| `caveman-help` | Referencia rápida de los modos caveman |
| `find-skills` | Descubrir o instalar skills nuevas |
