# SPEC 73 — Base Odoo 20 CE lead (docker + repos, sin migrar módulos)

> **Estado:** Implemented
> **Depende de:** — (infraestructura nueva paralela; convención de naming heredada del lead 19 y regla lead-only de AGENTS.md)
> **Fecha:** 2026-09-28
> **Objetivo:** Montar en `/home/odoo/lead` el mínimo indispensable para correr y ojear Odoo 20 CE en Docker — imagen propia a partir de la rama `20.0` de `odoo/odoo`, BD PostgreSQL con demo, compose, y las dos estructuras de repo vacías (`modulos_odoo_20`, `odoo20-skeleton`) commiteadas y pusheadas, sin migrar ningún módulo.

## Por qué existe esta spec

Los repos `modulos_odoo_20` y `odoo20-skeleton` ya existen en disco como clones vacíos de GitHub (`saymonset/modulos_odoo_20`, `saymonset/odoo20-skeleton`, rama `main` sin commits). El objetivo es replicar la arquitectura funcional del lead 19 (Dockerfile con clone de Odoo + entrypoint que crea/inicializa BD + compose db/web + bind mounts a `shared/{oca,extra}/<versión>`) pero **solo lo básico**: ni n8n, ni chatwoot, ni backups, ni módulos.

## Scope

**In:**

1. `modulos_odoo_20`: estructura mínima `shared/oca/20.0/` y `shared/extra/20.0/` (con `.gitkeep`), `README.md`, `AGENTS.md` mínimo (explica estructura y regla lead/prod), `.gitignore` (semilla del de 19, recortado).
2. `odoo20-skeleton/odoo20/`: esqueleto de despliegue con espejo de los nombres/archivos del 19 pero solo db+web:
   - `Dockerfile` — base `python:3.12-slim`, deps de sistema y wkhtmltopdf igual que 19, usuario `1001`, venv, `git clone --depth 1 -b 20.0 https://github.com/odoo/odoo.git /opt/odoo/odoo-core`, `pip install -r /opt/odoo/odoo-core/requirements.txt` + `debugpy pydevd-odoo` (no se copia el `requirements.txt` pinneado del 19: la rama 20.0 trae el suyo).
   - `entrypoint.sh` — copiado del 19 adaptado: `DB_NAME=dbodoo20`, inicialización `-i base` **con demo** (se quita `--without-demo=True`).
   - `docker-compose.yaml` — servicios `db-leads` (`pgvector/pgvector:pg15`, host `:5436`) y `web-leads` (`odoo-pers:20`, host `:38069`/`:38072`), red externa `odoo_network_20`, secrets por archivo, bind mounts a `/home/odoo/lead/modulos_odoo_20/shared/{extra,oca}/20.0`, volúmenes `./v20-leads/{pgdata,logs,odoo-web-data,config}`. (En 19 son dos archivos con `extends`; aquí se colapsa a uno solo: no hay segundo entorno que compartir.)
   - `generate_odoo_conf.sh` + `v20-leads/config/odoo.conf.dynamic` (plantilla con `__POSTGRES_PASSWORD__`, espejo del conf 19: `addons_path = /opt/odoo/odoo-core/addons,/opt/odoo/custom-addons/extra,/opt/odoo/custom-addons/oca`, `without_demo = all`, `workers = 2`).
   - `secrets/postgres_password.txt` (generada aleatoria, **gitignored**), `env-example` y `.env` (`COMPOSE_PROJECT_NAME=odoo20-lead`, sin secretos, commiteados como en 19).
   - Scripts numerados con memoria muscular del 19: `1_build_imagen.sh`, `3_stop-all.sh`, `4_start-all.sh`, `6_status_all_services.sh`, `7_logs_see_all_services.sh`.
   - `README.md` del skeleton: cómo build, up, url de ojeadura, credenciales.
3. Ejecución en host: `docker network create odoo_network_20`, build de `odoo-pers:20`, `up -d`, inicialización de `dbodoo20` con demo, verificación de ojeadura en `http://localhost:38069`.
4. Commit inicial + `push origin main` en ambos repos.

**Out of scope (para futuras specs):**

- Migración/port de módulos 19→20 (OCA vendored o extra).
- Servicios secundarios: n8n, chatwoot, postiz, pgadmin, redis, backups, crontab.
- anything en `/home/odoo/prod` (regla AGENTS.md: lead primero).
- CI/CD (`.github`, runner) para los repos nuevos.
- Config `.agents/`, `specs/`, `opencode.jsonc`, skills en `modulos_odoo_20`.
- Enterprise ni addons_path de enterprise.

## Modelo de datos

No introduce estructuras de datos Odoo nuevas. Introduce esta convención de naming (par 19 lead → 20 lead):

| Elemento              | 19 lead                | 20 lead                  |
| --------------------- | ---------------------- | ------------------------ |
| Imagen                | `odoo-pers:19`         | `odoo-pers:20`           |
| Contenedor web        | `odoo-19-web-leads`    | `odoo-20-web-leads`      |
| Contenedor db         | `odoo-db19-leads`      | `odoo-db20-leads`        |
| Puertos web           | 28069 / 28072          | 38069 / 38072 (+10000)   |
| Puerto db (host)      | 5435                   | 5436                     |
| BD                    | `dbodoo19`             | `dbodoo20` (con demo)    |
| Red docker            | `odoo_network_19`      | `odoo_network_20`        |
| Volúmenes             | `./v19-leads/*`        | `./v20-leads/*`          |
| Addons bind           | `modulos_odoo/shared/{extra,oca}/19.0` | `modulos_odoo_20/shared/{extra,oca}/20.0` |
| Proyecto compose      | `odoo19-lead`          | `odoo20-lead`            |

Árbol final esperado:

```
/home/odoo/lead/modulos_odoo_20/          ├── README.md ├── AGENTS.md ├── .gitignore
│   └── shared/{oca,extra}/20.0/.gitkeep
/home/odoo/lead/odoo20-skeleton/odoo20/   ├── Dockerfile ├── entrypoint.sh ├── docker-compose.yaml
├── generate_odoo_conf.sh ├── env-example ├── .env ├── secrets/ ├── scripts 1_/3_/4_/6_/7_
└── v20-leads/{config,logs,pgdata,odoo-web-data}
```

## Implementation plan

1. Crear estructura de `modulos_odoo_20` (README, AGENTS.md mínimo, .gitignore, `shared/{oca,extra}/20.0/.gitkeep`) → commit inicial + push (`origin git@github.com:saymonset/modulos_odoo_20.git`, rama `main`).
2. Crear esqueleto en `odoo20-skeleton/odoo20/`: Dockerfile (rama `20.0`), entrypoint.sh (demo on), compose único db+web, conf dynamic + generate, env-example/.env, scripts de ciclo de vida, README → commit local (sin push todavía: falta verificar que construye).
3. Host: generar `secrets/postgres_password.txt` aleatorio, crear red `odoo_network_20`, `generate_odoo_conf.sh`, `docker build -t odoo-pers:20 .` (verifica que `20.0` instala clean con python 3.12).
4. `4_start-all.sh` → arrancan db+web; el entrypoint crea `dbodoo20` y la inicializa con `-i base` + demo (~3–5 min). Smoke: `docker logs odoo-20-web-leads` sin traceback, `curl -s http://localhost:38069/web/database/selector` devuelve HTML.
5. Ojeadura: login Manager (`admin/admin`, mismo criterio de conf que lead 19), verificar selector de BD y que una app con demo (Sales) muestra datos de ejemplo; confirmar que el stack 19 sigue intacto (`curl :28069`, `docker ps`).
6. Commit final del skeleton (incluidos ajustes que haya requerido el build) + push `origin main` en `odoo20-skeleton`.

## Acceptance criteria

- [x] `docker images` contiene `odoo-pers:20` construida sin errores desde la rama `20.0` de `odoo/odoo`.
- [x] `odoo-20-web-leads` y `odoo-db20-leads` en `Up`; db con healthcheck OK; proyecto `odoo20-lead`.
- [x] `http://localhost:38069` muestra el login de Odoo y el pie/`/web/database/selector` confirma versión 20.
- [x] `dbodoo20` inicializada **con demo**: al menos una app instalable/instalada muestra registros de ejemplo (p. ej. `SELECT count(*) FROM product_product` > baseline sin demo).
- [x] `addons_path` resuelve los mount points `/opt/odoo/custom-addons/{extra,oca}` (vacíos, sin error de registry).
- [x] Sin regresión en el stack 19: `:28069` y `:5435` siguen respondiendo; cero colisiones de puertos/nombres.
- [x] `git log` en ambos repos muestra el commit inicial en `main` y `git status` limpio tras el push (no están commiteados `secrets/*.txt` ni `v20-leads/{pgdata,logs,odoo-web-data}`).
- [x] `3_stop-all.sh` y `4_start-all.sh` levantan/apagan el stack 20 de forma idempotente.

## Decisiones

- **Sí:** esqueleto mínimo (solo odoo+db) — el usuario pidió "solo lo básico"; los ~30 archivos del 19 (backup, crontab, postiz, chatwoot, n8n) no aplican.
- **No:** duplicar el par de archivos compose con `extends` del 19 — un solo `docker-compose.yaml` basta sin segundo entorno.
- **Sí:** puertos 38069/38072 + db 5436 — verificado libre con `ss -tlnp`; patrón +10000 sobre lead 19.
- **Sí:** BD con demo — el objetivo declarado es "poderlo ojear"; el `--without-demo` del lead 19 existe para tests deterministas (SPEC 05), que aquí no aplican.
- **Sí:** `pip install -r /opt/odoo/odoo-core/requirements.txt` de la propia rama 20.0 en vez de copiar el `requirements.txt` pinneado del 19 — evita drift de versiones.
- **Sí:** base `python:3.12-slim` y receta wkhtmltopdf idénticas al 19 — mínimo cambio; verificación de `MIN_PY_VERSION` de 20.0 ocurre en el build (paso 3) y su bump es trivial en el Dockerfile.
- **No:** migrar módulos, tocar prod, ni montar n8n/chatwoot — cada uno, si llega, en su spec.
- **Sí:** `admin_passwd = admin` y bind 0.0.0.0 como el lead 19 — consistencia; el riesgo queda anotado abajo.
- **Sí:** definición rápida — el usuario respondió "sí" a las 5 recomendaciones del bloque de preguntas sin apertura de debate.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| Odoo 20 exige python > 3.12 o deps incompatibles y el build revienta | El requirements viene del propio core 20.0; si el build falla, bump a `python:3.13-slim` (una línea) — se resuelve en el paso 3 antes de commitear |
| wkhtmltopdf del recipe 19 descontinuado/inútil en 20 (cambio de renderer) | No bloquea la ojeadura (solo PDFs); se mantiene la receta y cualquier ajuste va en spec futura |
| `without_demo = all` en el conf anula el `-i base` con demo del entrypoint | El entrypoint pasa `--without-demo=False` explícito en la inicialización; verificado en criterion de demo del paso 4 |
| Consumir RAM del host (8 GB) con un segundo stack Odoo encendido | `workers = 2` igual que 19; si hay presión, `3_stop-all.sh` apaga el 20 (SPEC 69/70 ya cubren priorización del host) |
| Credenciales débiles expuestas (admin/admin, 0.0.0.0) | Mismo criterio aceptado para lead 19; BD 20 sin datos reales y sin integraciones; host no público |

## What is **not** in this spec

- Migración de módulos 19→20 (OCA ni extra).
- n8n / chatwoot / postiz / pgadmin / redis / backups / crontab.
- Nada en `/home/odoo/prod`.
- CI/CD de los repos nuevos.
- `.agents/`, `specs/`, `opencode.jsonc` en `modulos_odoo_20`.

Cada uno de esos, si aterriza, va en su propia spec.

## Registro de implementación (28-sep, rama `spec-73-odoo20-lead-base`)

- **Paso 1:** `modulos_odoo_20` con `shared/{oca,extra}/20.0/.gitkeep` + README + AGENTS + .gitignore → commit raíz `f98265f` y push `origin/main` ✓.
- **Paso 2:** esqueleto `odoo20-skeleton/odoo20/` (15 archivos, commit `529970d`) — compose único db+web como decidió la spec.
- **Desviación 1 (obligada, materialización del riesgo "Odoo 20 exige deps"):** `pgvector/pgvector:pg15` es incompatible con Odoo 20 — su `res_groups._recompute_user_share` usa `ANY_VALUE()` (PostgreSQL ≥16) y la init de `dbodoo20` crasheaba (`UndefinedFunction`). Fix: imagen db `pgvector/pgvector:pg16` (commits `9049030`+`cf4afdd`). `python:3.12-slim` sí sirvió (`MIN_PY_VERSION=(3,12)`).
- **Desviación 2:** desde 19.0 `without_demo` es booleano: `all` en el conf → warning "invalid boolean"; se cambió la plantilla a `without_demo = True` (equivalente funcional) y el entrypoint/-i de apps pasan `--without-demo=False` explícito para la demo.
- **Adaptación de permisos:** sin `sudo` en la sesión de ejecución → no se puede `chown 1001` el config desde el host; se mantiene `user: "1001:1001"` (igual que 19) con `chmod a+rwX` sobre `v20-leads/{config,logs,odoo-web-data}` (documentado en README). `generate_odoo_conf.sh` ya no usa `sudo chown`.
- **Pasos 3–5:** red `odoo_network_20`, imagen `odoo-pers:20` (2.25 GB, `odoo-bin --version` = `Odoo Server 20.0`), `up -d`, init de `dbodoo20` OK. Verificado: login `:38069` HTTP 200, selector 200, cero CRITICAL/Tracebacks tras el cambio a pg16, cero warnings `invalid addons directory` en el ciclo actual. Para el criterio de demo se instaló `sale_management` con demo vía CLI (`-i sale_management --without-demo=False --no-http --stop-after-init`): `product_product`=46, `res_partner`=44. Regresión 19 intacta (`:28069` 200, `:5435` OK). Stop/start idempotentes probados; proyecto compose `odoo20-lead`.
- **Paso 6:** push `origin/main` del skeleton (`cf4afdd`); `secrets/*.txt` y `v20-leads/{pgdata,logs,odoo-web-data}` y el `odoo.conf` renderizado quedan gitignored; `git status` limpio en ambos repos.
