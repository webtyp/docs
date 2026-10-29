# Revisión de los módulos restantes antes de entrar a una app — master plan
> **Status:** COMPLETED · 2026-10-29 · Los 14 módulos cerrados y publicados (A1–A14). Despacho y revisión: **Antigravity**

**De qué trata.** El MVP de `mjosefa-cms` ([MVP_DEPLOYMENT_ROADMAP](../../veltylabs/mjosefa-cms/docs/MVP_DEPLOYMENT_ROADMAP.md))
lleva solo `service_catalog`, `device_manager`, `staff_manager` y `room_layout`. Esta ola deja **los demás módulos de
`veltylabs/modules`** listos para entrar después, uno por uno: cada uno pasa la revisión de 6 puntos (skill
**app-modules**, «Before a module enters an app») y adopta las piezas publicadas el 2026-10-29 (decisiones D22, D26 y
D27 del roadmap: errores con `rpc.WriteError`, permisos por tabla con `model.ResourceOf`, guardar y borrar con
`view.MountSave`/`MountDelete`). **Ningún módulo entra a `mjosefa-cms` en esta ola:** se integran recién cuando el MVP
esté operativo, cada uno en su propio PR revisado.

**Cómo se trabaja (para Antigravity).**
- Al despachar un módulo: mover `docs/module_review/<módulo>.md` a `<módulo>/docs/PLAN.md` (eliminándolo de
  `docs/module_review/`) y ejecutar `codejob dispatch` (skill **codejob**). Un plan por repo; como máximo 15 en vuelo.
- Revisar cada PR con **plan-review** (`gotest` una vez, captura del demo, `langc sync .` limpio) y cerrar con
  `codejob close`; las cascadas largas en segundo plano real, nunca con `&`.
- Un defecto que resulte estar en una librería de `webtyp` sigue **bug-fix-flow** (prueba que falla, y se cierra la
  causa, no solo el caso).
- Marca de dueño en la celda Estado de cada fase en curso: `[Antigravity · AAAA-MM-DD HH:MM]` (regla de
  **master-plan**); una marca de otro agente con menos de 90 min no se toca.
- Ningún ejemplo, prueba ni documento de `webtyp` nombra un dominio del cliente (skill **documentation**).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| R1 | Cada módulo responde sus errores con `rpc.WriteError` y declara sus errores como `model.NotFound`/`Conflict`/`Invalid`, **manteniendo el texto y el código de hoy**; borra su `domainError`, `ValidationError`, `statusOf`/`writeError` | cada plan, etapa 1 |
| R2 | El permiso de una tabla es `model.ResourceOf(&XModel)`; lo que no es tabla, `model.FeatureResource` declarado una vez; ningún texto en `Requires` ni en `Permission` | etapa 2 |
| R3 | Las pantallas guardan y borran con `view.MountSave`/`MountDelete`; se borran los tipos que re-declaraban `{records}`/`{ids}` | etapa 3 (solo donde la pantalla guarda) |
| R4 | Claves foráneas declaradas (`Ref`) en toda columna `*_id` de una tabla propia | etapa 4 |
| R5 | Claves de traducción en inglés y traducciones **reales** en todos los idiomas del `lang.json` (nunca vacías ni de relleno) | etapa 5 |
| R6 | Demo que abre con `webtyp` y captura de cada pantalla en el PR | etapa 6 |
| R7 (P1, P2) | **El tenant y el usuario que actúa nunca vienen del navegador** (= D32 de MVP): el tenant entra por `Deps.TenantID` (configuración del servidor) y el usuario sale siempre de `ctx.UserID()` (`loopback.ActingAs` en pruebas, sin respaldo en el body); se borran `TenantId`, `ActorId`, `CreatorUserId`, `RecipientId` y similares de los args | fase B (un plan por módulo afectado tras cerrar A1–A14) |
| R8 (P4) | En `chat_room`, `ErrNotMember` es `model.NotFound` (404, no revela que una sala privada existe) y `ErrUnauthenticated` es `model.Invalid` (400, solo alcanzable como error de programación porque toda ruta lleva `Authenticated()`); no se agregan tipos a `model` | A5 (`chat_room`, etapa 1) |

## Fases

| Fase | Repo | Qué falta | Estado |
|---|---|---|---|
| A1 | `agreement_manager` | **`Item()` en sus registros** (no compila con `view` ≥ v0.6.47), errores, 3 permisos const, traducciones (7 idiomas vacíos), demo nuevo | 🟢 cerrada v0.3.20 |
| A2 | `alert_manager` | errores (borra `wire.go`), 6 permisos literales, guardar/borrar de 4 pantallas, traducciones, demo nuevo | 🟢 cerrada v0.1.30 |
| A3 | `business_calendar` | errores (25 estados sueltos), 12 permisos literales, guardar/borrar de 3 pantallas, claves foráneas, traducciones | 🟢 cerrada v0.5.29 |
| A4 | `cash_manager` | errores (24 estados sueltos), permiso de módulo `cashmanager` → por tabla, claves foráneas, traducciones | 🟢 cerrada v0.2.19 |
| A5 | `chat_room` | errores (31 estados sueltos, R8), permiso const, guardar/borrar de grupos, traducciones | 🟢 cerrada v0.2.118 |
| A6 | `editorial` | errores (19 estados sueltos), 3 permisos const, guardar/borrar de publicaciones, traducciones, demo nuevo | 🟢 cerrada v0.0.45 |
| A7 | `network_manager` | errores (13 estados sueltos), 2 permisos const, guardar de ajustes, claves foráneas, traducciones | 🟢 cerrada v0.1.96 |
| A8 | `patient_directory` | errores (13 estados sueltos), 7 permisos literales/const, guardar de pacientes, claves foráneas, traducciones | 🟢 cerrada v0.1.76 |
| A9 | `payment_manager` | errores, 3 permisos const, **traducciones de relleno** en 7 idiomas (490 valores falsos) | 🟢 cerrada v0.4.28 |
| A10 | `pharmacy_manager` | errores (borra los `{records}` propios), 2 permisos literales, guardar/borrar de 2 pantallas, traducciones | 🟢 cerrada v0.1.52 |
| A11 | `supply_manager` | igual que `pharmacy_manager` | 🟢 cerrada v0.2.50 |
| A12 | `work_schedule` | errores, 1 permiso literal, claves foráneas, demo nuevo (traducciones completas) | 🟢 cerrada v0.1.43 |
| A13 | `appointment_booking` | errores, 30 permisos literales, `Close()` en `model_orm.go` (regenerar con `ormc`), **no tiene `lang.json`**, claves foráneas | 🟢 cerrada v0.3.52 |
| A14 | `clinical_encounter` | errores (13 estados sueltos), 7 permisos, guardar visitas, traducciones; incluye Etapa 0 para limpiar marcas de conflicto | 🟢 cerrada v0.2.31 |
| B | módulos con `args.TenantId` / `ActorId` | R7 (D32): el tenant desde `Deps.TenantID`, el usuario desde `ctx.UserID()` (`loopback.ActingAs` en pruebas); borrar esos campos de los args | ⚪ tras cerrar cada A1–A14 (un plan por repo) |

## Bloqueos

- Ninguno · Los 14 módulos (A1–A14) cerrados y publicados.

## Relación con otras olas

- MVP_DEPLOYMENT_ROADMAP (mjosefa-cms): dueño de las decisiones D22, D26, D27, D32 y de la revisión de 6 puntos; sus módulos (M2d, R1–R3, M2l) no están aquí. Los de esta ola entran a la app después del MVP.
- REACTIVE_RECORDS: `Item()` en los registros (A1).
- APP_COMPOSITION: pantallas `view.Screen`; su bloqueo «`view` importa `dom`» afecta a quien use la raíz de un módulo en un binario sin interfaz.
