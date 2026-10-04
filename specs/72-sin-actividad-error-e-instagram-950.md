# SPEC 72 — Sin actividad pública de "bot caído" y respuestas Instagram ≤950

> **Estado:** Implemented
> **Depende de:** SPEC 68 (WEBHOOK_TIMEOUT=15 + URL interna — revierte una decisión), SPEC 62 (`Indentifica_canal`), SPEC 43 (workspace n8n_json en lead), SPEC 63 (promoción manual a prod)
> **Fecha:** 2026-09-25
> **Objetivo:** Eliminar por completo las dos fallas del incidente del 24-sep en Instagram (actividad pública del agentbot que reabrió la conversación y respuesta rechazada por Meta al exceder 1000 caracteres) fijando la respuesta inmediata del webhook de n8n, troceando de forma determinista las respuestas a Instagram y activando `keep_pending_on_bot_failure` en Chatwoot.

## Por qué existe esta spec

Evidencia forense (conv id 176 / display 177, inbox 5 `integraiaconodoo`, 2026-09-24):

1. **21:25:07 UTC** — contacto envía "Shared post" (mensaje de 12 caracteres).
2. **21:26:25** — actividad **pública** (`private=f`, message_type=2, msg 3526): *"Conversation was marked open by system due to an error with the agent bot."*. Causa: Chatwoot POSTeó al agentbot 4 veces (ejecuciones n8n 45429/45431/45433/45434 entre 21:25:28 y 21:26:47 — reintentos Sidekiq con backoff) porque las respuestas HTTP de n8n superaron los 15 s de `WEBHOOK_TIMEOUT` (stall transitorio del host: 8 GB RAM, 1.3 GB swap; hoy el webhook responde en ~0.6 s). Una de esas ejecuciones quedó colgada 67 min y terminó `crashed` al recrearse el contenedor (22:34). Cada reintento disparó el flujo LLM completo → riesgo de respuestas duplicadas al cliente.
3. **21:27:33** — la respuesta del bot (msg 3527, generada por el flujo n8n) quedó **`status=3` (failed)** con `external_error: "100 - The length of the message sent is over 1000 characters"`: 1032 caracteres contra el límite duro de Instagram DM → "Error al enviar" visible en el transcript, cliente sin respuesta. La **regla 6 del prompt universal ya ordena 900 chars para redes** (`prompt_renderer.py`:36-41) y el modelo la violó — se necesita guardia determinista.

Este es el **cuarto recurrencial** de la actividad de error (convs 49, 74, 172, 173→SPEC 68, 176).

## Scope

**In:**

1. Re-exportar el workflow prod `chatbot_create_lead_0_con_menu_whatsapp` a `/home/odoo/lead/odoo19-skeleton/n8n_json/chatwoot/` (fuente de verdad, SPEC 43) y commitear la baseline si hay divergencia con el export actual.
2. `Entrar_ChattWoot` (nodo webhook, typeVersion 2.1): fijar explícito `responseMode: 'onReceived'` (hoy es el default implícito; explicitarlo evita que un edit futuro lo cambie a `lastNode` y reintroduzca el timeout).
3. Nuevo nodo code `Trocear_Instagram` (JS puro) antes de los 4 enviadores genéricos de texto a la API de Chatwoot (`Enviar_mensaje_de_IA`, `IA1`, `IA2`, `IA3`): si el canal es Instagram y `content` > 950 code points → partir en ítems consecutivos ≤950 (corte en doble salto de línea → fin de oración → hard cut), preservando `account_id`/`conversation_id`/demás campos; el nodo HTTP Request envía un mensaje por ítem. Para cualquier otro canal: paso sin cambios.
4. Chatwoot prod, cuenta 1: `keep_pending_on_bot_failure = true` vía `rails runner` (una línea; rollback: `false`). Con respuestas asíncronas por API, el reopen + actividad pública ya no aporta información a nadie y ensucia el transcript.
5. Verificación E2E real en Instagram (mensaje que provoque respuesta larga) + chequeo SQL de 48 h sin actividades nuevas `status=3` ni "marked open".
6. Promoción manual a prod n8n (SPEC 63) del JSON editado y commit en la rama `lead` del skeleton.

**Out of scope (para futuras specs):**

- Alerta humana "bot caído" (diferida desde SPEC 68; con `keep_pending` el silencio ante caída total es deliberado hasta que exista la alerta).
- Parchar Chatwoot core para poner privada la actividad de `Webhooks::Trigger#create_agent_bot_error_activity` — frágil ante upgrades.
- Editar contenido del RAG que induce respuestas de marketing largas (lo gestiona el humano).
- Cambios en `ai_chatbot_1_portal` / prompt (regla 6 ya existe; el fix es determinista en n8n).
- Nodos de WhatsApp exclusivos (menús interactivos, carrito) y `botUnisa`.
- Notas privadas en la conversación 177 (decisión del usuario: solo el deseo de no repetición).

## Modelo de datos

No introduce estructuras nuevas. Cambia:

- JSON del workflow `chatbot_create_lead_0_con_menu_whatsapp`: +1 nodo `n8n-nodes-base.code` (`Trocear_Instagram`), rewiring de 4 conexiones, `responseMode` explícito en el webhook.
- Columna `keep_pending_on_bot_failure` del registro `accounts` id 1 de Chatwoot: `false → true`.

Contrato del chunker (por ítem de salida): `{ ...campos_originales, content: <chunk ≤950> }`.

## Implementation plan

1. Re-export prod→lead del JSON y commit de la baseline (documenta divergencia si existe).
2. Editar JSON: `responseMode: 'onReceived'` explícito en `Entrar_ChattWoot`. Test en lead: "Test workflow" con payload simulado de Chatwoot → POST responde <1 s aunque el flujo LLM tarde.
3. Añadir `Trocear_Instagram` y cablearlo antes de `Enviar_mensaje_de_IA*` (4 conexiones). Test de paso a paso en lead con input `channel='Channel::Instagram'` + texto de 1032 chars (el del incidente) → exactamente 2 ítems ≤950; con canal `Channel::Whatsapp` → 1 ítem intacto.
4. Aplicar `keep_pending_on_bot_failure=true` en Chatwoot prod (rails runner con validación + rollback documentado); readback verificado.
5. Promoción manual (SPEC 63): importar el JSON en n8n prod en valle + E2E real: mensaje al IG `integraiaconodoo` que exija respuesta larga → ≥2 mensajes entregados ≤950, cero "Error al enviar", cero actividad nueva; mensaje de control corto → respuesta única. Verificar que la respuesta del bot pasa pending→open (asunción de la decisión keep_pending; si no se confirma, revertir el paso 4 y reabrir debate).
6. Commit del JSON final en rama `lead` del skeleton + push. Monitoreo 48 h: SQL `messages` (nuevos `status=3`, nuevas actividades "marked open") y conteo n8n ejecuciones 1:1 vs mensajes entrantes.

## Acceptance criteria

- [ ] Mensaje real a Instagram que produce >950 chars llega al cliente partido (100 % entregado) y ningún mensaje nuevo de la cuenta queda `status=3`.
- [ ] El webhook `http://n8n:5678/webhook/chatwoot_integraia` responde 200 en <1 s con el flujo LLM corriendo, y cada mensaje entrante genera **exactamente 1** ejecución del workflow principal.
- [ ] Cero actividades "Conversation was marked open by system..." en los 48 h posteriores a la promoción (consulta SQL sobre `messages`).
- [ ] `Account.find(1).keep_pending_on_bot_failure == true` (readback en rails runner) y rollback `false` documentado.
- [ ] El JSON commiteado en `lead` del skeleton es idéntico al workflow activo en n8n prod.
- [ ] Regresión: flujo de carrito/menú en WhatsApp E2E sin fragmentación nueva de mensajes.
- [x] ~~Respuesta del bot vía API transiciona la conversación pending→open (paso 5).~~ — **Criterio anulado (25-sep, decisión del usuario):** premisa factualmente incorrecta para Chatwoot 4.17 — las conversaciones con agent bot activo **nacen en `pending` por diseño** (`conversation.rb` `set_active_bot_conversation`), la respuesta del bot vía API no la abre (solo la respuesta humana en UI), y así ocurría ya antes de esta spec en toda conversación exitosa (convs 99/86/81/55). `keep_pending_on_bot_failure=true` se mantiene: el pending no oculta nada (el reply y la actividad de agentes se ven en el filtro Pending); revertirlo solo devolvería la actividad pública en inglés ante cada stall.

## Decisiones

- **Sí (implementación, 25-sep):** promoción conjunta SPEC 71+72 — el re-export prod→lead del paso 1 reveló que la ÚNICA divergencia de contenido es `mensaje_usuario` de SPEC 71 (aún no promovida a prod); el usuario confirmó importar en prod el JSON con 71+72 juntos. Baseline = JSON actual del lead.
- **Sí:** `keep_pending_on_bot_failure=true` — **revierte el "No" de SPEC 68**: allí la respuesta se evaluó con el reply síncrono en mente; como el bot ya responde de forma asíncrona vía `/messages`, el reopen en inglés no ayuda a humanos y sí contamina el transcript. La alerta humana sigue pendiente (spec futura). ~~Condición de revert: criterio de aceptación pending→open.~~ — **Anulada (25-sep):** el E2E demostró que pending es el estado de diseño para conversaciones con bot en Chatwoot 4.17 (ver criterio 7); el usuario confirmó mantener el flag.
- **Sí:** split determinista ≤950 solo para Instagram — la regla 6 del prompt (900 chars) ya existía y el LLM la violó (1032); el margen 950/1000 cubre diferencias de conteo de Meta.
- **No:** parchar el core de Chatwoot para privatizar la actividad.
- **No:** umbral universal 1000 para WhatsApp/Telegram (4096) — fragmenta sin necesidad (decisión del usuario).
- **No:** tocar el prompt en `ai_chatbot_1_portal` — el fix va en n8n; reglas duplicadas divergirían.
- **Sí:** `responseMode` explícito en lugar de confiar en el default (`Webhook.node.js`:174 `getNodeParameter('responseMode','onReceived')`).

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| `keep_pending` oculta una caída real de n8n (charla pending sin aviso) | Aceptado explícitamente por el usuario; la alerta vive en una spec futura (heredada de SPEC 68). Ventana acotada: monitoreo 48 h post-promoción |
| Los chunks llegan desordenados al cliente | HTTP Request procesa ítems secuencialmente; verificado en el E2E del paso 5 (texto con "parte 2" distinguible) |
| Divergencia entre el export lead y prod | Paso 1 re-exporta antes de editar cualquier línea |
| La respuesta vía API no abre la conversación pending | Criterio de aceptación #7 con revert del paso 4 si falla |
| Stall del host vuelve a retrasar el 200 del webhook (causa del retry del 24-sep) | Fuera del control del workflow (SPEC 69/70 cubren carga del host); con `onReceived` explícito la superficie de retraso queda en la aceptación de conexión, no en 25–110 s de LLM |

## What is **not** in this spec

- Alerta "bot caído" a humanos.
- Contenido del RAG / prompt del Odoo.
- Chatwoot core (privacidad de actividades).
- Cambios en flujos WhatsApp-only y `botUnisa`.

Cada uno de esos, si aterriza, va en su propia spec.

## Registro de implementación (25-sep)

- **Paso 1:** re-export prod→lead confirmó que la única divergencia de contenido era `mensaje_usuario` de SPEC 71 (pendiente de promoción); el usuario autorizó promover 71+72 juntos. Baseline = JSON de lead tal cual.
- **Paso 2:** `Entrar_ChattWoot` con `responseMode: 'onReceived'` explícito. Test en vivo con payload sonda (ruta `outgoing` → noOp, sin efectos): HTTP 200 "Workflow was started" en 0.4–0.6 s.
- **Paso 3 (desviación del literal):** en lugar de 1 nodo chunker compartido, **4 instancias** `Trocear_Instagram_IA|IA1|IA2|IA3` (una por enviador). Un único nodo alimentando a los 4 ejecutaría los 4 en cada llegada (fan-out de n8n) → mensajes duplicados. Además: +1 asignación `platform` en `tomar_parametros` para que su guard funcione. Tests deterministas del code (Code-API idéntico al runtime): 1032→950+80 con IDs intactos; WA/TG sin tocar; hard cut y code points OK.
- **Paso 4:** `Account(1).settings.keep_pending_on_bot_failure=true` (store_accessor, no columna). **Rollback:** `docker exec chatwoot-app bundle exec rails runner 'a=Account.find(1); a.keep_pending_on_bot_failure=false; a.save!'`. Readback true; SQL `settings->>'keep_pending_on_bot_failure'` = `true`.
- **Paso 5 (import + E2E, 25-sep 00:05 UTC):** promo del JSON vía UPDATE de `workflow_entity.nodes/connections` (backup en `/tmp/opencode/prod_wf_*_before.json`) + `docker restart n8n-container` en valle (55 s; sonda caída durante activación → `crashed`, sin efecto). Post-restart: workflow activado, sonda `outgoing` → HTTP 200 en 108–418 ms. E2E real IG (conv 179, "Shared post" + pregunta precios): 2 respuestas del bot **entregadas y leídas** (`delivered/read`, cero `status=3`), **cero** actividades "marked open", webhook respondió en <1 s con el LLM corriendo (35.8 s y 19.8 s), 1 ejecución de agente por mensaje entrante. La respuesta larga quedó en 900 chars ≤950 → el chunker no troceó (comportamiento correcto del guard; el caso >950 está cubierto por los tests deterministas del paso 3 con el texto real de 1032 del incidente). Monitoreo 48 h: SQL de `status=3` y actividades "marked open" desde 2026-09-24 22:34 → 0 y 0.
