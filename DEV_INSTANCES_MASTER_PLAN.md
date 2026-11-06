# Varios proyectos a la vez en `webtyp dev` — master plan
> **Status:** CLOSED · 2026-11-05 · todas las fases completadas y verificadas con mjosefa-cms y room_layout

**Qué duele hoy.** Con `mjosefa-cms` corriendo, abrir `webtyp dev` en `room_layout` para buscar un error de vista
**mata** el proyecto anterior: hay que detenerlo y volver a arrancar el otro. El agente vive lo mismo y peor: cuando
el demonio muere (por ejemplo, tras instalar un `webtyp` nuevo) sus herramientas MCP responden `ECONNREFUSED` y no
tiene cómo levantarlo. **Regla de la ola: lo que el dev hace en la TUI, el agente lo hace por MCP; si no, es un fallo.**

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

Ninguna (D1–D10 implementadas y publicadas).

## Fases
Orden de publicación: `tui` → `devflow` → `gorun` → `devtui` → `app` (cada uno usa al anterior).

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 0 | `devflow` | `Install` deja de matar al binario en uso (D10) | cerrada v0.4.166 |
| 0b | `devflow` | `gotest` sin `docs/TESTS.md` no avisa nada; si existe, se revisa como antes | cerrada v0.4.167 |
| 1 | `tui` | constante `tui.ProjectKey = "project"` | cerrada v0.1.4 |
| 2 | `gorun` | matar por ruta absoluta del ejecutable, no por nombre (D8); `server` ya lo usa | cerrada v0.0.29 |
| 3 | `devtui` | el cliente envía su proyecto en `/logs`, `webtyp/state`, `webtyp/action`; `Ctrl+C` ya no envía `stop` (D4, D9) | cerrada v0.4.70 |
| 4 | `app` | hasta 5 proyectos, herramientas, pestaña MCP, cliente, relanzarse al cambiar el binario (D1–D7, D9) | cerrada v0.7.62 (con `server` v0.2.111, `gorun` v0.0.29, `devflow` v0.4.167) |
| 5 | skill **webtyp-app** | varios proyectos, `list/stop_development`, `webtyp mcp &` como respaldo | cerrada v0.0.38 (sincronizada en todos los agentes) |
| 6 | `mjosefa-cms` + `room_layout` | correr los dos a la vez, detener uno desde la TUI y otro por MCP, reinstalar `webtyp` sin perderlos, capturas | cerrada (verificado en :8080 y :8081, capturas en `~/.gemini/antigravity/brain/`) |

## Bloqueos
Ninguno.

## Relación con otras olas
- Sale de DATA_IMPORT (M6: el demonio cayó tras instalar `app` v0.7.59). Cerrada, no bloquea nada.
- TUI_STAGES: liberado `app/docs/PLAN.md` para sus fases 3 y 4.
