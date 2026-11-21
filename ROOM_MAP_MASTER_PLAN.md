# Mapa de ocupación (`room_layout`): la configuración única del recinto — master plan
> **Status:** PENDING · 2026-11-21 · P2 cerrada (D4); esperan **P1** (confirmar A simplificada) y **P3** (nombre); D1 hecho en local (abre en «Operación»)

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

**P1 — De dónde sale «desde qué PC entra» un profesional.** Respuesta del dueño: **A**, «¿es lo más simple? ¿se
puede simplificar?». Análisis, contando piezas nuevas:

| Opción | Piezas nuevas | Problema |
|---|---|---|
| A tal como estaba (`DevicesOfOccupant` → lista) | 1 interfaz en `staff_manager`, 1 lectura en `room_layout`, 1 cableado | **dependencia circular**: `room_layout` ya pide a `staff_manager` sus ocupantes al construirse (`mjosefa-cms/config/services/backend.go:82-98`) |
| B (copiar a `staff_device` en cada turno) | escribir/borrar en crear, editar y borrar turno | dos copias que se desalinean |
| C (`room_layout` pasa a ser el registro del login) | `IsTrustedIP` y `RegisterFirstAdmin` en `room_layout`, más «usuario → ocupante» desde `staff_manager` | mete el login en un módulo genérico y mueve el alta del primer administrador |
| **A simplificada** | 1 interfaz de **una pregunta sí/no** en `staff_manager`, 1 lectura en `room_layout`, 1 adaptador de ~5 líneas en la app | ninguno |

**Recomiendo A simplificada, que ya es lo mínimo**: un solo dato (el turno), una pregunta en vez de una lista
(«¿este ocupante usa este equipo?» es una consulta, no cargar y recorrer todos sus equipos en cada login), y el
círculo se corta en la app, que es la raíz de composición: `staff_manager` solo hace esa pregunta al entrar alguien,
nunca al construirse, así que la app le entrega un adaptador que busca `room_layout` ya armado en ese momento
(`backend.FindBuiltModule`, que ya existe). Menos que esto solo es B o C, y los dos agregan piezas o duplican datos.
```go
// staff_manager/interfaces.go
type DeviceAssignments interface {
	UsesDevice(tenantID, occupantID, deviceID string) (bool, error) // ¿tiene turno en un box que tiene este equipo?
}
// staff_manager.IsTrustedIP: su fila staff_device (solo el primer administrador) O UsesDevice(...) == true
// room_layout: UsesDevice = room_shift activo del ocupante → room_space_equipment → room_equipment.device_id == deviceID
// mjosefa-cms/config/services/backend.go: adaptador que llama a FindBuiltModule(d, roomlayout.Definition.Name) al preguntar
```
Respuesta: A — ¿confirmas la versión simplificada?

**P3 — Nombre de la pantalla (hoy «Mapa»).** «Mapa» nombra el dibujo (el medio), no la tarea; la regla de nombres
(**api-design**, prueba del novato) pide la intención. Lo que el administrador hace ahí cada día es decidir quién
ocupa qué box y cuándo, y ver cuánto se ocupa (la referencia ya llama «Ocupación» a la capa principal y al
indicador). A) **«Ocupación»**: dice la tarea en una palabra, sirve igual a un spa o un hotel (el módulo es genérico)
y no choca con «Boxes», «Pisos» ni «Equipamiento», que ya están en el menú. B) «Asignación de boxes»: precisa pero
larga y habla de «box», palabra que el módulo evita. C) «Plano»: igual que «Mapa», nombra el dibujo. **Recomiendo A.**
```go
// room_layout/ui: la etiqueta del menú (clave en inglés, traducida en lang.json)
const Label lang.Text = "Occupancy" // es: «Ocupación» — antes "Map" («Mapa»)
```
Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **El mapa abre siempre en «Operación»**; dibujar planta, boxes y equipos se entra desde «Configurar mapa» | `room_layout/ui` — hecho en local |
| D2 | **«Operación» muestra los dos pisos lado a lado si la pantalla lo permite** (escritorio), un piso con desliz horizontal en el teléfono, como la referencia; hoy dibuja uno a la vez (`ui/rootview.go` `shownFloors`) | `room_layout/ui` |
| D3 | **La asignación profesional → box → días es la única configuración**: agenda y equipos habilitados salen de ella (P1) | `room_layout`, `staff_manager` |
| D4 (P2) | **El personal sigue en `staff_manager`**; el panel «Profesionales» del mapa trae «+ Agregar profesional» (RUT, nombre, especialidad) que guarda por la operación de `staff_manager` (interfaz propia de `room_layout`, la app la conecta): el administrador no sale del mapa | `room_layout/ui` + `interfaces.go`, `mjosefa-cms` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| R1 | `room_layout` | D1: `NewMap` abre en `StageOperation` (constantes `StagePlan…StageOperation`), `tests/ui_start_test.go`, `docs/TESTS.md`; borrado `docs/PLAN.md` obsoleto (cumplido y publicado en v0.3.93) | 🟢 local, espera revisión |
| R2 | `room_layout` | D2, D4 y el flujo de asignar de la referencia (tocar profesional → ver dónde calza → diálogo de días), comparado con `ui-reference` | tras P3 (nombre) |
| R3 | `staff_manager`, `room_layout`, `mjosefa-cms` | P1: equipos habilitados desde la asignación | tras P1 |
| R4 | `room_layout`, `staff_manager` | historias `H-ROOM-<NN>` (asignar en el mapa) y la «Dado» de H-ACCESS-03 por el mapa | tras R2, R3 |

## Bloqueos

- 2026-11-21 · H-ACCESS-03 (`TEST_DOMAIN_MASTER_PLAN.md`) espera R3: el profesional no puede entrar hasta que un box asignado le habilite su PC.

## Relación con otras olas

- TEST_DOMAIN: cada fase de este master termina con su historia de usuario en verde, todo por pantalla.
