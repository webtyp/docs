# Mapa de ocupación (`room_layout`): la configuración única del recinto — master plan
> **Status:** IN PROGRESS · 2026-11-21 · sin preguntas; D1 hecho en local; ahora R3 (D5) y D6, luego R2; D1 hecho en local (abre en «Operación»)

**Regla del dueño (2026-11-21):** `room_layout` es la pieza central: ahí se elige al profesional y se le asigna el
box donde trabaja y qué días. Esa asignación es la única configuración y todo sale de ella: su agenda y desde qué
PC puede entrar (el box tiene o no un PC). La experiencia debe ser fluida: abrir en operación, mostrar los dos pisos
si la pantalla lo permite y asignar profesionales ubicándolos en las plantas. Referencia de diseño:
`veltylabs/modules/room_layout/docs/ui-reference/operacion-pantalla-principal-html`.

Hoy · Propuesta (la referencia):
![hoy](../../veltylabs/modules/room_layout/docs/img/operacion-1-hoy.png)
![propuesta](../../veltylabs/modules/room_layout/docs/img/operacion-2-propuesta.png)

Para retomar: skills **app-modules**, **module-layout**, **api-design**, **api-harness**, **testing** (historias
de usuario: todo por pantalla); probar en `veltylabs/mjosefa-cms` (MCP de webtyp) y con
`go test -tags stories -run TestStories ./tests/` (ver `TEST_DOMAIN_MASTER_PLAN.md`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **El mapa abre siempre en «Operación»**; dibujar planta, boxes y equipos se entra desde «Configurar mapa» | `room_layout/ui` — hecho en local |
| D2 | **«Operación» muestra los dos pisos lado a lado si la pantalla lo permite** (escritorio), un piso con desliz horizontal en el teléfono, como la referencia; hoy dibuja uno a la vez (`ui/rootview.go` `shownFloors`) | `room_layout/ui` |
| D3 | **La asignación profesional → box → días es la única configuración**: agenda y equipos habilitados salen de ella (P1) | `room_layout`, `staff_manager` |
| D5 (P1) | **El PC habilitado sale del turno** (A simplificada): `staff_manager` declara `DeviceAssignments{UsesDevice(tenantID, occupantID, deviceID) (bool, error)}` y `IsTrustedIP` acepta su fila `staff_device` (solo el primer administrador) o `UsesDevice == true`; `room_layout` lo responde (turno activo del ocupante → equipos del box → `device_id`); la app conecta con un adaptador que resuelve `room_layout` al preguntar (`FindBuiltModule`): los módulos siguen sin importarse y ninguno necesita al otro al construirse | `staff_manager`, `room_layout`, `mjosefa-cms` |
| D6 (P3) | **La pantalla se llama «Ocupación»** (clave `Occupancy`): nombra la tarea, no el dibujo | `room_layout/ui`, `lang.json` |
| D4 (P2) | **El personal sigue en `staff_manager`**; el panel «Profesionales» del mapa trae «+ Agregar profesional» (RUT, nombre, especialidad) que guarda por la operación de `staff_manager` (interfaz propia de `room_layout`, la app la conecta): el administrador no sale del mapa | `room_layout/ui` + `interfaces.go`, `mjosefa-cms` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| R1 | `room_layout` | D1: `NewMap` abre en `StageOperation` (constantes `StagePlan…StageOperation`), `tests/ui_start_test.go`, `docs/TESTS.md`; borrado `docs/PLAN.md` obsoleto (cumplido y publicado en v0.3.93) | 🟢 local, espera revisión |
| R2 | `room_layout` | D2, D4 y el flujo de asignar de la referencia (tocar profesional → ver dónde calza → diálogo de días), comparado con `ui-reference` | tras R3 |
| R3 | `staff_manager`, `room_layout`, `mjosefa-cms` | D5 y D6 | 🟡 [Claude · 2026-11-21] en curso |
| R4 | `room_layout`, `staff_manager` | historias `H-ROOM-<NN>` (asignar en el mapa) y la «Dado» de H-ACCESS-03 por el mapa | tras R2, R3 |

## Bloqueos

- 2026-11-21 · H-ACCESS-03 (`TEST_DOMAIN_MASTER_PLAN.md`) espera R3: el profesional no puede entrar hasta que un box asignado le habilite su PC.

## Relación con otras olas

- TEST_DOMAIN: cada fase de este master termina con su historia de usuario en verde, todo por pantalla.
