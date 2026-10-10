# CI/CD Context — Odoo 20 (lead20)

## Estado actual

**No existe pipeline CI/CD para Odoo 20.** Tampoco clon prod (`/home/odoo/prod/modulos_odoo_20`).
Todo despliegue es manual en la instancia lead (`odoo-20-web-leads`).

## Referencia: el pipeline de 18/19 (`modulos_odoo`)

Modelo a replicar cuando exista prod 20 (SPEC futura):

1. `changes` — detecta módulos `extra/<versión>` tocados + deps en orden topológico
2. `lint` — compileall, claves de manifest, anti-patrón `attrs=` en XML
3. `test` — update + `--test-enable` en staging
4. `deploy` — merge `--ff-only` en clon prod, `-u` sin tests, restart + health check

Ver `/home/odoo/lead/modulos_odoo/.agents/context/cicd.md` (runner self-hosted, auth SSH por
ssh-agent socket, fallback manual). No aplicar ciegamente: puertos/contenedores 20 difieren
(38069/5436, `odoo-20-web-leads`).

## Fallback manual en lead20

```bash
cd /home/odoo/lead20/odoo20-skeleton/odoo20
docker exec odoo-20-web-leads odoo -d dbodoo20 -u <cadena> --stop-after-init
docker restart odoo-20-web-leads
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:38069/web/login
```
