# Testing Context — Odoo 19 (OCA Pattern)

## Convenciones de testing

- `@tagged("-at_install", "post_install")` en toda clase de test.
- `setUpClass` con env sin tracking: `mail_create_nolog=True`, `mail_create_nosubscribe=True`, `mail_notrack=True`, `no_reset_password=True`, `tracking_disable=True`.
- `Form` API para ejercitar onchanges/wizards.
- `unittest.mock.patch` para HTTP externo (`requests.get`/`requests.post`); `HttpCase` para controllers propios.
- `@mute_logger("odoo.models.unlink")` para suprimir logs esperados.
- `tests/__init__.py` con imports explícitos (no wildcard). `common.py` importado primero si existe.
- Fixtures binarias en `tests/fixtures/`.
- `Command` (`from odoo import Command`) para m2m/o2m.
- `invalidate_recordset()` tras writes antes de assertar campos computed.
- Skeleton base: ver `bcv_rate_update_venezuela/tests/common.py` o `ai_chatbot_1_portal/tests/common.py`.

## Cómo correr tests

```bash
# 1. Limpiar cache de bytecode
find shared/extra/19.0/<module> -name '__pycache__' -type d -exec rm -rf {} + 2>/dev/null; true

# 2. Upgrade + tests en una sola pasada (staging/pruebas)
docker exec odoo-19-web-leads python3 /opt/odoo/odoo-core/odoo-bin -d dbodoo19 \
    -u <module> --test-enable --stop-after-init --log-level=test

# Si el modulo tiene dependencias custom (ej. bcv_rate_update_venezuela),
# upgrade todo el chain: -u bcv_rate_update_venezuela,currency_rate_update_venezuela,currency_rate_update_base
```

Para PROD (solo verificar, no desarrollar):
```bash
docker exec odoo-19-web python3 /opt/odoo/odoo-core/odoo-bin -d dbodoo19 \
    -u <module> --test-enable --stop-after-init --log-level=test 2>&1 | tee /tmp/test_<module>.log
```

## Regla obligatoria

Todo módulo de `extra/19.0` modificado debe pasar sus tests (`--test-enable`) antes de hacer `git push`.
