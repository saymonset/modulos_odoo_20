# SPEC 74 — nginx lead20.integraia.lat para Odoo 20 CE

> **Estado:** Implemented
> **Depende de:** SPEC 73 (stack 20 corriendo en `:38069/:38072`, `proxy_mode=True`, master `admin`)
> **Fecha:** 2026-09-29
> **Objetivo:** Exponer el Odoo 20 lead detrás del nginx del host con HTTPS (`https://lead20.integraia.lat`) replicando el patrón del bloque `lead.integraia.lat` del 19 en un archivo vhost aislado, sin tocar los bloques existentes y cerrando el acceso directo por puerto al final.

## Por qué existe esta spec

SPEC 73 dejó el 20 accesible solo por puerto directo (`0.0.0.0:38069`) "porque es lo básico". Para uso cómodo y seguro hay que tratarlo igual que el 19 lead: subdominio con certificado del host, sin nginx dentro de Docker y con `proxy_mode=True` (ya puesto). El patrón exacto a copiar está en `/etc/nginx/sites-enabled/jumpjibe.com.conf`, bloque `lead.integraia.lat` → upstreams `odoo_leads`/`odoo_leads_longpolling`.

## Scope

**In:**

1. Archivo nuevo `/etc/nginx/sites-available/odoo20.conf` + symlink en `sites-enabled`: upstreams `odoo20_leads` (`127.0.0.1:38069`) y `odoo20_leads_longpolling` (`127.0.0.1:38072`); server blocks 80→301 https y 443 ssl http2 espejo del bloque lead 19 (snippets `ssl.conf`/`letsencrypt.conf`, headers proxy, caché `/web/static/`, `/longpolling/` con upgrade websocket, timeouts 720–900 s, `client_max_body_size 50M`, logs `odoo20_leads.{access,error}.log`). `jumpjibe.com.conf` NO se toca.
2. Certificado certbot `lead20.integraia.lat` (http-01 vía nginx, `sudo`).
3. Fuente versionada en el skeleton: `odoo20-skeleton/odoo20/nginx-lead20.conf` + sección "nginx" en su README → commit + push `origin main`.
4. Cierre del acceso directo: puertos del `docker-compose.yaml` a `"127.0.0.1:38069"` y `"127.0.0.1:38072"` (patrón prod 19 con 18069), `--force-recreate web-leads`, verificación solo por dominio, commit.
5. Verificación E2E: login 200 por `https://lead20.integraia.lat`, longpolling funcional, `lead.integraia.lat` (19) intacto, `nginx -t` OK.

**Out of scope (para futuras specs):**

- Cambios en los server blocks existentes (`lead`, `integraia.lat`, n8n, chatwoot, postiz, pgadmin, temporal).
- Cloudflare proxy (orange) / WAF sobre el subdominio: DNS plano, registro A.
- Cierre de los `0.0.0.0` del 19 lead (fuera de alcance; si se quiere, spec propia).
- Nada en prod; monitorización; CI.

## Modelo de datos

Sin estructuras Odoo nuevas. Artefactos:

| Artefacto | Detalle |
| --- | --- |
| `/etc/nginx/sites-available/odoo20.conf` | vhost nuevo (symlink en `sites-enabled`) |
| `/etc/letsencrypt/live/lead20.integraia.lat/` | cert emitido por certbot |
| `odoo20/nginx-lead20.conf` | copia versionada en `odoo20-skeleton` |
| `docker-compose.yaml` | `ports` web → `127.0.0.1:38069/38072` |

## Implementation plan

1. **Gate DNS (usuario):** el usuario crea el registro A `lead20.integraia.lat → 147.93.179.254` (TTL 300) en el panel de la zona; el plan solo verifica `dig +short lead20.integraia.lat @1.1.1.1` = la IP. Sin resolución, no se continúa (certbot fallaría y gasta rate-limit; se usa `--dry-run` antes del real).
2. Escribir `odoo20-skeleton/odoo20/nginx-lead20.conf` (espejo del bloque lead 19 con nombres `odoo20*`) → commit local en el skeleton.
3. Aplicar en host con `sudo` (contraseña solo por stdin, nunca escrita en archivos): copiar a `sites-available/odoo20.conf`, symlink, `certbot --nginx -d lead20.integraia.lat`, dejar el bloque 443 tal cual el archivo propio con las rutas de cert del emisor → `nginx -t` → `systemctl reload nginx`. Rollback: borrar symlink + reload.
4. E2E: `curl -sI https://lead20.integraia.lat/web/login` = 200 con cert válido; `http://` = 301; `/longpolling/` responde vía dominio; `https://lead.integraia.lat` (19) sigue 200; grep `server_name` antes/después idéntico salvo el nuevo.
5. `docker-compose.yaml`: bind `127.0.0.1` de 38069/38072 → `up -d --force-recreate web-leads` → verificar por dominio y que `ss -tlnp` ya no muestra `0.0.0.0:38069`.
6. README del skeleton (sección nginx: cómo re-aplicar en otro host) + commit final + push `origin main`.

## Acceptance criteria

- [x] `https://lead20.integraia.lat/web/login` HTTP 200 con candado válido (certbot, sin mixed-content).
- [x] `http://lead20.integraia.lat` responde 301 a https.
- [x] Longpolling funcional vía dominio (chatter abre sin errores de conexión en consola).
- [x] `nginx -t` OK; los `server_name` preexistentes inalterados; `https://lead.integraia.lat` (19) sigue OK.
- [x] `odoo20/nginx-lead20.conf` y el compose con `127.0.0.1` commiteados y pusheados en `odoo20-skeleton` `main`; README documenta el vhost; `git status` limpio.
- [x] `ss -tlnp` muestra `127.0.0.1:38069/38072` (acceso directo por IP:puerto caído).
- [x] Login `dbodoo20` (demo) funciona por el dominio.

## Decisiones

- **Sí:** vhost aislado en `sites-available/odoo20.conf` + symlink — no inflar `jumpjibe.com.conf` (petición explícita del usuario; rollback es borrar un archivo).
- **Sí:** espejo literal del bloque `lead.integraia.lat` (snippets/timeouts/longpolling) — consistencia operativa con el 19.
- **No:** contenedor nginx dedicado — el 19 usa nginx del host.
- **No:** tokens/API de DNS — el registro A lo crea el usuario manualmente (una línea, 30 s); el plan lo trata como gate con `dig`.
- **Sí:** cerrar `0.0.0.0:38069/38072` → `127.0.0.1` al terminar (patrón prod 19) — un Odoo con master `admin` no debe escuchar en la calle; reversible en 2 líneas si se necesita acceso directo puntual.
- **Sí:** `--dry-run` de certbot antes del emit real — protege el rate-limit de ACME si el DNS no propagó.
- **Seguridad:** la contraseña sudo se usa vía stdin en la sesión de ejecución, no se escribe en ningún archivo; se recomienda rotarla (quedó en el chat).

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| DNS sin propagar al ejecutar → certbot falla | Gate del paso 1 con `dig @1.1.1.1` + `--dry-run` previo |
| Reload de nginx afecta el 19 u otros servicios | `nginx -t` antes de reload; conf separado; rollback = borrar symlink + reload |
| El cierre a `127.0.0.1` rompe algún uso actual por IP:38069 | Es decisión aceptada del usuario; revert de 2 líneas en compose documentado en README |
| Certbot renueva con config de su bloque autogenerado y diverge del archivo versionado | El 443 final queda escrito a mano (solo rutas de cert); la renovación es in-place por symlink del mismo archivo |

## What is **not** in this spec

- Proxy/WAF Cloudflare sobre `lead20`.
- Cambios a los dominios/bloques existentes del host (incluido cerrar el 0.0.0.0 del 19 lead).
- Prod, monitorización, CI.

Cada uno de esos, si aterriza, va en su propia spec.

## Registro de implementación (29-sep, rama `spec-74-nginx-lead20-odoo20`)

- **Paso 1:** A record creado por el usuario; gate `dig @1.1.1.1 lead20.integraia.lat` = 147.93.179.254 ✓.
- **Paso 2:** `odoo20-skeleton/odoo20/nginx-lead20.conf` (commit `d5888ad`), espejo literal del bloque lead 19.
- **Paso 3 (adaptación):** `--dry-run` no aplica al modo `--nginx` de este certbot (solo a `certonly`) → se emitió con `certbot certonly --nginx -d lead20.integraia.lat` partiendo de un vhost provisional 80-only en `/tmp`; luego el archivo versionado (con rutas estándar `/etc/letsencrypt/live/lead20.integraia.lat/*`) reemplazó al provisional y `nginx -t` + reload. `diff` conf host == versionado ✓.
- **Desviación (justificada por AC de chatter):** añadido `location /websocket` → upstream longpolling/gevent (commit en skeleton). Evidencia: `:38072/websocket` responde 101 (handshake correcto); el espejo 19 solo enrutaba `/longpolling/` (route legacy que en 19 ya devolvía 500 y en 20, 404). Verificado por dominio: 400 sobre HTTP/2 de curl → 101 forzando `--http1.1` (los navegadores usan HTTP/1.1 para WS; el error RuntimeError "bind the websocket" del log era del boot del gevent aún levantando).
- **Paso 4:** login https 200, http 301, cert Let's Encrypt CN=lead20 válido, ALPN h2 para tráfico normal, ws 101; `https://lead.integraia.lat` (19) 200 antes y después.
- **Paso 5:** compose `127.0.0.1:38069/38072` + recreate; `ss -tlnp` confirma solo-localhost; re-E2E por dominio OK.
- **Paso 6:** README (sección nginx con re-aplicación en otro host, prerequisitos DNS/snippets, rollback y revert del bind; fijados `without_demo = True` y chmod de dirs; push `97491be` a `origin main`); `git status` limpio en skeleton y `modulos_odoo_20`. Contraseña sudo usada solo por stdin, no escrita en archivos del repo.
