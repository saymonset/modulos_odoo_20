# Plan — Ambiente IA lead20 (config desde lead/modulos_odoo)

## Estado (avanzado antes de bloquear plan mode)

- [x] Copiados `instructions.md`, `skills-lock.json`, `.agents/` (11 skills lead + contexts), `.opencode/plans/.gitkeep`
- [x] Skills oficiales Odoo 20 (odoo/odoo@20.0): `odoo-git`, `odoo-guidelines`, `odoo-review`, `odoo-security`, `odoo-web-guidelines` → `.agents/skills/`

## Pendiente (requiere salir de plan mode)

1. **Contextos** `.agents/context/`:
   - Borrar `odoo19.md`; crear `odoo20.md` con lo verificado hoy:
     - `/jsonrpc` REMOVIDO en 20 (ValueError). Usar `POST /json/2/<model>/<method>`, kwargs nombrados (422 si posicionales).
     - Auth: `Authorization: Bearer <api_key>`; Basic NO funciona. BD vía header `X-Odoo-Database: dbodoo20`.
     - odoo-mcp OK con `ODOO_TRANSPORT=json2` + `ODOO_JSON2_DATABASE_HEADER=1`; api key admin `25b3b...5da8` validada contra dbodoo20.
     - Arrastrar quirks de 19 (attrs, list, hasclass, t-set, COP) marcados "verificar al portear".
   - Reescribir `docker.md` para lead20: containers `odoo-20-web-leads` (127.0.0.1:38069/38072) + `odoo-db20-leads` (:5436, pgvector/pg16, db `dbodoo20`), red `odoo_network_20`, proyecto `odoo-20-lead`; compose `/home/odoo/lead20/odoo20-skeleton/odoo20/`; mounts `shared/{extra,oca}/20.0 → /opt/odoo/custom-addons/{extra,oca}`; addons_path `/opt/odoo/odoo-core/addons,/opt/odoo/custom-addons/extra,/opt/odoo/custom-addons/oca`; gotcha `__pycache__` + `docker restart odoo-20-web-leads`; no existe prod 20.
   - `cicd.md`: marcar pipeline 20 INEXISTENTE (SPEC 73 fuera de alcance); referencia el de 18/19 como modelo futuro.
2. **`opencode.jsonc`** (espejo lead + cambios 20):
   - `skills.paths`: `["~/.agents/skills/odoo-19", ".agents/skills"]`
   - agent `spec-verify` + commands `spec`/`spec-impl`/`spec-verify` idénticos.
   - MCP `odoo-server`: `ODOO_URL=http://127.0.0.1:38069`, `ODOO_DB=dbodoo20`, `ODOO_USERNAME=admin`, `ODOO_PASSWORD`/`ODOO_API_KEY=25b3b69ea4b6b8f98d8eebb40456a76dfe9a5da8`, `ODOO_TRANSPORT=json2`, `ODOO_JSON2_DATABASE_HEADER=1`, `ODOO_MCP_ENABLE_WRITES=1`.
   - `playwright` disabled como en lead. `instructions: ["instructions.md"]`.
3. **`AGENTS.md`** modulos_odoo_20: regla lead → "solo bajo `/home/odoo/lead20/`"; añadir tablas Contextos (testing/cicd/docker/odoo20) y Skills (incluidas las 5 oficiales 20 + odoo-19 referencia); quitar referencia n8n (no aplica a 20 todavía).
4. **Skeleton**: `docker-compose.yaml` líneas 44-45 `/home/odoo/lead/modulos_odoo_20/...` → `/home/odoo/lead20/modulos_odoo_20/...`; `docker compose up -d` (recrea web); verificar `docker exec odoo-20-web-leads ls /opt/odoo/custom-addons/oca` (web_responsive). Si existen dirs root-owned fantasmas en `/home/odoo/lead/`, reportar (no borrar sin confirmación).
5. **Verificación**: listar skills detectadas; prueba MCP read-only (version/db).
6. **Commits**: `chore: mirror lead agent config + odoo 20 official skills` (modulos_odoo_20) y `fix: bind mounts point to lead20 path` (skeleton). Sin push salvo indicación.
