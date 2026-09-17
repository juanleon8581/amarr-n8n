# 1.1.0 — 2026-09-16

## Added

- Workflow `Taiga → Discord`: nuevo proyecto `ESTRELLAS WEBCAM` mapeado a su propio canal de Discord.
- Workflow `Taiga → Discord`: enrutamiento por atributo personalizado `DEPARTAMENTO` (épicas, US, tareas, issues) — si el objeto trae ese atributo, prima sobre el canal por proyecto. Cubre los 9 departamentos definidos en `ESTRELLAS WEBCAM` (Contabilidad, Soporte, Asesores, RRHH, Fotografía, Gerencia, Comercial, Marketing, Desarrollo).
- Workflow `Taiga → Discord`: nodo **Verify Taiga Signature** — valida el header `x-taiga-webhook-signature` (HMAC-SHA1 sobre el raw body) antes de procesar el evento; rechaza peticiones sin firma válida. Requiere `NODE_FUNCTION_ALLOW_BUILTIN=crypto` (agregado a `docker-compose.yml`) y "Raw Body" activado en el nodo Webhook.
- Workflow `Taiga → Discord`: embeds enriquecidos — avatar del actor (`embed.author`), logo del proyecto (`embed.thumbnail`), y formato correcto del diff para custom attributes y campos con valores objeto (antes salían como "modificado" genérico o `[object Object]`).

## Fixed

- Workflow `Taiga → Discord`: el diff de `custom_attributes` (ej. cambios en DEPARTAMENTO) se perdía — venía en formato distinto al resto del diff y caía al caso genérico.
