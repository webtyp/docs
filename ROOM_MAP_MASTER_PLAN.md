# Mapa de ocupación (`room_layout`): la configuración única del recinto — master plan
> **Status:** IN PROGRESS · 2026-11-21 · sin preguntas; R1, R2a, R2b, R3 y D4 hechos en local; sigue **R2d** (D10–D13, interfaz del dueño) y luego el diálogo de asignar (D8, D9); D1 hecho en local (abre en «Operación»)

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
| D7 (P4) | **Un solo modelo para lo que hay en un espacio: `RoomArtifact`** (qué, dónde, estado) con ids blandos al módulo dueño de cada cosa (`device_id` → `device_manager` para los que tienen red). Un tipo nuevo es un valor de `kind`, sin tablas; una columna nueva solo cuando aparece un **módulo dueño** nuevo (pocos), nunca por tipo. `next_maintenance` e `items` se mudan a su módulo dueño (mantención, stock) cuando exista. `Equipment`, `RoomEquipment` y la pantalla «Equipamiento» (vacíos) se borraron | hecho en local |
| D8 (P5) | **La especialidad del turno se elige en el diálogo de asignar** (lista del catálogo) y al guardar el espacio queda habilitado para ella si no lo estaba (en la misma transacción) | `room_layout` (`shift_tx.go`, `ui`) |
| D9 (P6) | **«Mañana» y «tarde» salen del horario de la clínica**: mañana = apertura → 13:00, tarde = 14:00 → cierre (`HoursReader`); sin horario, 09–13 y 14–18 | `room_layout` |
| D10 | **Una sola forma de ver los pisos en todas las etapas** (dibujar, espacios, equipos, operación): todos los pisos ajustados al ancho, dos a la vista si caben y uno si no, sin recortar la planta ni desplazarla por dentro; una celda pintada muestra sus líneas horizontales y verticales (capturas del dueño `docs/img/dueno-2026-11-21-1{3,4,5}.png`) | `room_layout/ui` |
| D11 | **Cabeceras de una o dos filas**: en «Configurar mapa» los pasos numerados llevan su ayuda en una línea y desaparece «Siguiente: …»; la cuadrícula del edificio va debajo del mapa con botones chicos (se usa poco); el nombre del piso se entiende como campo (etiqueta y ayuda). En «Operación»: Planificar/Ahora, Semana/Mes, Ocupación/Funcionamiento y «Configurar mapa» en una o dos filas, sin el pie «← Equipamiento» (captura `dueno-2026-11-21-12.png`) | `room_layout/ui` |
| D12 | **Las alertas se ven sin desplazarse** (arriba del panel lateral, no al final) | `room_layout/ui` |
| D13 | **Profesionales: `selectsearch` que filtra por nombre y por área** (especialidad); el buscador actual no filtra | `room_layout/ui`, `staff_manager` (área del ocupante por interfaz) |
| D4 (P2) | **El personal sigue en `staff_manager`**; el panel «Profesionales» del mapa trae «+ Agregar profesional» (RUT, nombre, especialidad) que guarda por la operación de `staff_manager` (interfaz propia de `room_layout`, la app la conecta): el administrador no sale del mapa | `room_layout/ui` + `interfaces.go`, `mjosefa-cms` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| R1 | `room_layout` | D1: `NewMap` abre en `StageOperation` (constantes `StagePlan…StageOperation`), `tests/ui_start_test.go`, `docs/TESTS.md`; borrado `docs/PLAN.md` obsoleto (cumplido y publicado en v0.3.93) | 🟢 local, espera revisión |
| R2a | `room_layout`, `device_manager`, `view`, `mjosefa-cms` | Hecho en local y verificado en vivo (PC `pc11` vinculado a «pc info1» en la base de desarrollo). **device_manager**: `queries.go` `DeviceOptions`, `tests/device_options_test.go`. **room_layout**: `interfaces.go` `DeviceReader`, `module.go` `Deps.Devices`, `definition.go` `DeviceOptionModel` + `rule_device_option_read.go`, `artifact.go` `KindHasNetwork`, `ui/stage3_artefactos.go` «Equipo de red», `ui/types.go` `DeviceID`/`DeviceData`, `ui/data.go` (carga equipos; **`save` crea con `Saver` y edita con `Updater`**: antes toda edición de algo existente en el mapa —piso, espacio, equipo— se rechazaba), `ui/rootview.go` (**la etapa activa se redibuja en cada `Refresh`**: antes tocar un equipo no mostraba su panel), `lang.json`, `tests/device_option_test.go`, `tests/artifact_save_test.go`. **view**: `view.go` (la documentación de `Saver` ya no promete «reemplazar»: solo crea). **mjosefa-cms**: `backend.go` (`Devices`, `devicesOf`), `go.mod` (`replace` device_manager). `definitionc` instalado desde el código (v0.1.49 generaba la firma vieja de `Search`) | 🟢 local |
| R2b | `room_layout`, `widget`, `mjosefa-cms` | Hecho en local: lee los turnos semanales al abrir (`ui/data.go` `fillShifts`), quitar un turno lo borra en el servidor (`deleteShift`), bloques `MorningStartMin…AfternoonEndMin` (`block.go`) y ocupación por cruce de minutos (`ui/stage4_operacion.go`), fuera el botón falso «Asignar lunes mañana» (`tests/map_data_test.go` `TestMap_LoadsWeeklyShifts`). **D2 hecho**: «Operación» muestra todos los pisos en fila, dos a la vista en escritorio y uno en el teléfono (`widget/style` **`PagesInView(n)`** nuevo para `ScrollRow`, con test; `ui/css.go` `PartFloorStrip`; planta ajustada al ancho `planGridFit`; captura `docs/img/operacion-3-dos-pisos.png`). **Arreglado el congelamiento al crear/editar espacios** que introduje en R2a: `Refresh` vuelve a ser fino y solo los paneles laterales de «Equipamiento» y «Operación» se rearman (`rebuildAside`). Falta el diálogo de asignar: espera P5 y P6 | 🟢 local |
| R2c | `room_layout`, `staff_manager`, `mjosefa-cms` | **D4 hecho en local y verificado en vivo** (captura `docs/img/operacion-4-agregar-profesional.png`, sin enviar datos): `room_layout/ui/browser.go` (`OccupantForm`, `Steps{NewOccupant}`, `Screens(d, steps)`), `ui/stage4_operacion.go` («+ Agregar profesional» abre el alta en un diálogo y relee al guardar), `ui/rootview.go` (`newOccupant`), `seed/demo.go`, `tests/ui_view_test.go`, `lang.json`; `staff_manager/ui/professional_form.go` (nuevo: `ProfessionalForm`, misma operación y reglas que «Personal», incluida la cuenta de entrada D13) + `ui/css.go` (etiqueta arriba, un campo por fila) + `tests/ui_css_test.go`, `lang.json`, `go.mod` (html, css, widget); `mjosefa-cms/config/screens/browser.go` (`roomScreens`). Falta frente a la referencia: panel de profesionales con avatar y filtro, barra superior sin el pie «← Equipamiento», nombres que no se corten | 🟢 local |
| R2d | `room_layout`, `staff_manager` | D10, D11, D12, D13 | 🟡 [Claude · 2026-11-21] en curso |
| R2f | `lang` (`langc`), `room_layout` | **Ayuda sin traducir arreglada en local**: `langc` en modo librería solo conocía los tipos del propio módulo, así que no sabía que `view.Screen.Help` es `lang.Text` y además leía solo literales (no constantes) en la regla 7; la sincronización borraba la ayuda traducida de cualquier pantalla. `lang/langc/langc.go` (`scanFor`: aprende los tipos de las dependencias), `langc/scan.go` (regla 7 con `textArg`), `tests/langc_test.go` (2 casos, fallan sin el arreglo); `langc` instalado desde el código; `room_layout/lang.json` (ayuda de «Ocupación», «Confirm», «Delete» en 8 idiomas); skill **langc** regla 7 | 🟢 local |
| R2e | `room_layout` | D8 y D9: el diálogo de asignar (profesional → especialidad → días × mañana/tarde → guardar) | tras R2d |
| R3 | `staff_manager`, `room_layout`, `mjosefa-cms` | Hecho en local. **room_layout**: `artifact.go` (nuevo: funciones de artefacto, tipos `Artifact*` y estados `ArtifactStatus*` del dominio, `UsesDevice`), `definition.go` (`RoomArtifact.device_id`; fuera `Equipment`, `RoomEquipment`, sus temas; filtro del tablero `artifact_kind`), `definition_gen.go` (regenerado; `crud` subido a v0.0.19), `day.go` (tablero y libres leen artefactos), `checks.go`, `order.go`, `rows.go`, `view.go`, `migrate/migrate.go`, `seed/seed.go` (siembra artefactos), `ui/browser.go` (D6 y sin «Equipamiento»), `ui/svg.go`, `ui/types.go`, `lang.json`; borrados `equipment.go`, 5 `rule_room_*equipment*`, `tests/equipment_test.go`, `tests/equipment_device_test.go`; `tests/uses_device_test.go` nuevo; ajustados `rules_test`, `screen_save_test`, `view_test`, `widgets_test`, `seed_test`, `orm_coverage_test`, `ui_view_test`, `migrate_test`, `docs/TESTS.md`. **staff_manager**: `DeviceAssignments`, `Deps.Assignments`, `IsTrustedIP`, `tests/trust_assignment_test.go`. **mjosefa-cms**: `config/services/backend.go` (`roomAssignments`: busca `room_layout` al preguntar), `errors.go` (`errRoomsType`), `go.mod` (`replace` room_layout). Tablas `room_equipment` y `room_space_equipment` quedan en la base (vacías) hasta borrarlas con `migrate -drop` | 🟢 local, espera revisión |
| R4 | `room_layout`, `staff_manager` | historias `H-ROOM-<NN>` (asignar en el mapa) y la «Dado» de H-ACCESS-03 por el mapa | tras R2, R3 |

## Bloqueos

- 2026-11-21 · H-ACCESS-03 (`TEST_DOMAIN_MASTER_PLAN.md`) espera R3: el profesional no puede entrar hasta que un box asignado le habilite su PC.

## Relación con otras olas

- TEST_DOMAIN: cada fase de este master termina con su historia de usuario en verde, todo por pantalla.
