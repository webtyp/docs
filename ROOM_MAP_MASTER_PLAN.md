# Mapa de ocupación (`room_layout`): la configuración única del recinto — master plan
> **Status:** PENDING · 2026-11-21 · esperan **P1 y P2** del dueño; D1 hecho en local (abre en «Operación»)

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

**P1 — De dónde sale «desde qué PC entra» un profesional.** Lo que se ve: el administrador asigna a la Dra. Rojas
al Box 2 lunes y miércoles en el mapa; el Box 2 tiene un PC; ella tiene que poder entrar desde ese PC sin hacer nada
más. Hoy no puede: entrar exige una fila en `staff_device` (`staff_manager/module.go:59-81`), que solo escriben el
setup del primer administrador y el importador `pastdb`. El dato ya existe en `room_layout`: `room_shift`
(ocupante + box + día), `room_space_equipment` (equipos del box) y `room_equipment.device_id` (el equipo de
`device_manager`, con su IP). A) **`staff_manager` sigue respondiendo «¿puede entrar desde esta IP?», pero lo
calcula preguntándole a `room_layout` qué equipos hay en los boxes donde ese profesional tiene turno** (interfaz
propia, sin importar el módulo); `staff_device` queda solo para quien no tiene box (el primer administrador).
B) `room_layout` escribe `staff_device` cada vez que se guarda un turno: dos copias del mismo dato que se
desalinean. **Recomiendo A**, con cualquier día que tenga turno (no solo hoy: un cambio de día o una hora extra no
deja a nadie afuera). ¿Cuál evita el acoplamiento y es más mantenible? A: un solo dato, `room_layout` no conoce el
login y `staff_manager` no conoce los boxes.
```go
// A — staff_manager/interfaces.go (lo único nuevo en staff_manager)
type WorkplaceDevices interface {
	DevicesOfOccupant(tenantID, occupantID string) (deviceIDs []string, err error) // equipos de los boxes con turno
}
// A — room_layout lo implementa leyendo room_shift → room_space_equipment → room_equipment.device_id
// B — room_layout/rule_room_shift_create.go: escribir/borrar filas staff_device en cada turno (y en update/delete)
```
Respuesta:

**P2 — Dónde vive el listado de profesionales.** Lo que se ve: hoy se crea al profesional en «Personal» y después
se va al mapa a asignarlo. A) **sigue en `staff_manager`** («Personal» queda para la ficha: RUT, especialidad), y
el panel «Profesionales» del mapa los lista y trae «+ Agregar profesional» (RUT, nombre, especialidad) que guarda
por la operación de `staff_manager`: el administrador nunca sale del mapa. B) trasladarlo entero a `room_layout`.
**Recomiendo A**: `room_layout` es genérico a propósito (habla de «ocupantes», sirve a un spa o un hotel:
`room_layout/AGENTS.md`), y `staff_manager` es el registro que ya leen la reserva, la ficha clínica y el login;
moverlo rompe a esos tres. ¿Cuál evita el acoplamiento y es más mantenible? A: cada módulo con una sola tarea.
```go
// A — room_layout/interfaces.go: el mapa agrega ocupantes por una interfaz (la app conecta staff_manager)
type OccupantWriter interface {
	AddOccupant(tenantID, document, name, category string) (occupantID string, err error)
}
// B — StaffMemberModel pasa a room_layout; booking, clinical y auth cambian de módulo
```
Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **El mapa abre siempre en «Operación»**; dibujar planta, boxes y equipos se entra desde «Configurar mapa» | `room_layout/ui` — hecho en local |
| D2 | **«Operación» muestra los dos pisos lado a lado si la pantalla lo permite** (escritorio), un piso con desliz horizontal en el teléfono, como la referencia; hoy dibuja uno a la vez (`ui/rootview.go` `shownFloors`) | `room_layout/ui` |
| D3 | **La asignación profesional → box → días es la única configuración**: agenda y equipos habilitados salen de ella (P1) | `room_layout`, `staff_manager` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| R1 | `room_layout` | D1: `NewMap` abre en `StageOperation` (constantes `StagePlan…StageOperation`), `tests/ui_start_test.go`, `docs/TESTS.md`; borrado `docs/PLAN.md` obsoleto (cumplido y publicado en v0.3.93) | 🟢 local, espera revisión |
| R2 | `room_layout` | D2 y el flujo de asignar de la referencia (tocar profesional → ver dónde calza → diálogo de días), comparado con `ui-reference` | tras P2 |
| R3 | `staff_manager`, `room_layout`, `mjosefa-cms` | P1: equipos habilitados desde la asignación | tras P1 |
| R4 | `room_layout`, `staff_manager` | historias `H-ROOM-<NN>` (asignar en el mapa) y la «Dado» de H-ACCESS-03 por el mapa | tras R2, R3 |

## Bloqueos

- 2026-11-21 · H-ACCESS-03 (`TEST_DOMAIN_MASTER_PLAN.md`) espera R3: el profesional no puede entrar hasta que un box asignado le habilite su PC.

## Relación con otras olas

- TEST_DOMAIN: cada fase de este master termina con su historia de usuario en verde, todo por pantalla.
