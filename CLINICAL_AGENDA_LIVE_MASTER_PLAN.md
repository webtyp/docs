# Agenda del doctor en vivo (ficha clínica) — master plan
> **Status:** IN PROGRESS · 2026-11-20 · A1–A3, A6 y la ronda de correcciones visuales (A7) en local con `replace`, verificados en vivo; espera que el dueño lo mire. Pendientes: A0 (PR #20), A4 (chat), A8 (`staffCaller` de booking)

**Para retomar:** skills `codejob` («gopush and local `replace`»), `events-queues-and-alerts` (regla de avisos), `app-modules`, `testing`. Probar en `~/Dev/Project/veltylabs/mjosefa-cms` (DEV_AUTOLOGIN abre la sesión de «Cesar», que es el profesional `1789255672828331302`).

**Qué pidió el dueño:** el doctor elige en un `selectsearch` una reserva de su día y ve las fichas de ese paciente; la lista cambia sola cuando entra o cambia una reserva. Sistema anterior y demo de referencia (`~/Dev/Archive/app-demo/modules/medicalhistory`):

![ficha](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-1-ficha.png) ![selector](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-2-selector.png) ![paciente](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-3-paciente.png)

**Qué funciona ahora (verificado en vivo por MCP, 2026-11-20 15:45):**
- Al abrir «Historial Clínico» se lee `my_agenda_result` (asíncrono, en el primer `OnAfterReload`); el selector: «Nombre, 84 años · RUT · 16:30 · Confirmada», la pendiente atenuada sin poder elegirse; sin buscador (`HideSearch`).
- Cabecera del selector en **una fila** (56–72 px, `OneLine`): nombre truncado con `fmt` (título completo en `title`), RUT y hora.
- Cambio de estado por la API (`PATCH`, PENDING→CONFIRMED) → el selector se actualiza **sin recargar** (`/notices`).
- Elegir una reserva recarga el presenter **estándar** filtrado por paciente (`view.WithFilter`): 53 filas «Mar 26 Abr 22 · onicotomia · marta navarrete» (año, fecha centrada, etiqueta neutra); al cambiar de paciente solo se ven las suyas (1).
- Elegir una ficha **llena el formulario** (motivo, diagnóstico, prescripción, doctor…).
- Tests: `go test` verde en `sse`, `view`, `appkit`, `components/selectsearch`, `staff_manager`, `client_directory`, `appointment_booking/tests`, `clinical_encounter` (incluye tests nuevos de cada pieza).

**Datos de prueba en la base local:** reservas `test-agenda-1` y `test-agenda-2` (nota «prueba agenda en vivo») de hoy para Cesar. Borrar cuando el dueño termine de mirar: `DELETE FROM reservation WHERE id LIKE 'test-agenda-%'`.

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1 · A) | Aviso por usuario sin datos: `sse.UserNotices` (un canal por usuario con sesión) montado por appkit en `appkit.NoticesPath` (`/notices`); `backend.Deps.Notices` (`Notifier`) y `backend.Deps.Subscriber`; `view.BrowserDeps.Notices`. appointment_booking avisa por `WatchStaffAgenda`, clinical_encounter traduce profesional → usuario (`UserByStaff`) y publica `AgendaChanged` | hecho en local |
| D2 | Selector con reservas activas (PENDING, CONFIRMED, COMPLETED, NO_SHOW, CONFLICTED) y su estado; solo CONFIRMED y COMPLETED se eligen (`Attendable`, `SsOption.Disabled`). `ActiveByStaff` reemplaza `ConfirmedByStaff` | hecho en local |
| D3 | Opción «Nombre, Edad años» / RUT / «hora - servicio · estado» (`AgendaEntry` suma `service_name`, `patient_age`, `reservation_status`) | hecho en local |
| D4 | Agenda al abrir la pantalla (primer `OnAfterReload`), fichas por el presenter estándar con `view.WithFilter` (sin paciente, la regla `Search` no trae nada), sin buscador, nada bloqueante; error de configuración visible en la pantalla | hecho en local |
| D5 | Fichas con `targetdate`: año en dos cifras, fecha centrada, etiqueta neutra (`Inset`, también en `targethour`), motivo truncado con `fmt`; `Item` de `medical_history` con el doctor como descripción | hecho en local |
| D6 | Correcciones del dueño (2026-11-20): cabecera del selector en una línea (`style.OneLine` nuevo en `widget/style`), nombre truncado con `fmt`, RUT visible, sin buscador (`crudview.Config.HideSearch` nuevo), ninguna implementación aparte del presenter estándar (`view.WithFilter` nuevo) | hecho en local |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A0 | clinical_encounter | `docs/PLAN.md` (part 2b, textos) con PR #20 abierto desde 2026-11-12, sin sesión viva: revisarlo en local (**plan-review**) y `codejob close`. Toca `errors.go`/`lang.json`, que este trabajo también tocó: conciliar | 🔴 |
| A1 | sse, view, appkit | **sse** (sin commit): `user_notices.go`, `tests/user_notices_test.go`. **view**: `browser.go` (`BrowserDeps.Notices`), `go.mod`. **appkit**: `app.go` (`NoticesPath`), `backend/backend.go` (`Notifier`, `Deps.Subscriber/Notices`), `backend/build.go`, `backend/notices.go` + `notices_wasm.go`, `browser/browser.go` (pantallas después de `ReadMe`), `browser/notices.go` + `notices_wasm.go`, `go.mod` (`replace` sse, view) | 🟢 local, espera revisión |
| A2 | appointment_booking, staff_manager, client_directory | **booking**: `booking.go` (`ActiveByStaff`), `module.go` (`Deps.Subscriber`), `agenda_watch.go`, `tests/agenda_watch_test.go`. **staff**: `module.go` (`UserByStaff`), `tests/test_readers_test.go`. **client_directory**: `readers.go` (`PatientBirthdate`), `tests/patient_test.go` | 🟢 local, espera revisión |
| A3 | clinical_encounter, components, mjosefa-cms | **clinical**: `interfaces.go`, `const.go`, `module.go`, `agenda.go`, `definition.go` + `definition_gen.go`, `rule_medical_history_search.go`, `ui/browser.go`, `lang.json`, `seed/demo.go`, `tests/agenda_test.go`, `go.mod` (`replace` components, view, form, model, dom, input, css, layout, color, palette, widget; `crud` subió a v0.0.19). **components**: `selectsearch/selectsearch.go`, `css.go`, `selectsearch_test.go` (`Disabled`). **mjosefa-cms**: `config/services/backend.go` (`Subscriber`, `Services`, `Notices`), `go.mod` (`replace` client_directory, sse) | 🟢 local, espera revisión |
| A7 | view, layout, widget, components, clinical_encounter, mjosefa-cms | **view**: `view.go` + `presenter.go` (`WithFilter`; el presenter recuerda lo ya listado para no mostrar filas de otro filtro como recién guardadas), `tests/filter_test.go`. **layout**: `crudview/crud.go` (`HideSearch`; ese archivo también trae cambios sin commit de Antigravity), `crudview/hide_search_test.go`. **widget**: `style/except.go`, `sheet.go`, `emit_primitives.go`, `emit_decls.go` (`OneLine`), `style/oneline_test.go`. **components**: `targetdate/{targetdate.go,css.go,row_test.go}`, `targethour/css.go`, `selectsearch/{selectsearch.go,css.go}`, `go.mod` (`replace` widget). **clinical**: `view.go` (`NewView(caller, opts...)`), `rule_medical_history_search.go` (sin paciente, nada), `ui/browser.go` (sin presenter propio), `lang.json`. **mjosefa-cms**: `go.mod` (`replace` widget) | 🟢 local, espera revisión |
| A8 | appointment_booking | `staffCaller` (`view.go`) reescribe `ReadArgs.Filter`: pasar a `view.WithFilter` + `WithNewRecord`. Su `ui/` no compila hoy por el trabajo de Antigravity (`calendarslider.New`): hacerlo cuando compile y verificar «Servicios por profesional» en vivo | 🔴 |
| A4 | chat_room | Volver a recibir su aviso por `/notices` (`Deps.Notices` + `ui.WithInbox` sobre `BrowserDeps.Notices`). chat_room no está en mjosefa-cms: no se puede verificar en vivo | 🔴 cuando el chat vuelva a la app |
| A9 | dom, layout, form | **dom**: `dom_backend.go` + `dom_frontend.go` (`Show` envuelve en `<slot>`, que es `display: contents`: el `<div>` vacío sumaba 8px de hueco bajo la cabecera del selector), `tests/show_test.go`, `docs/ARCHITECTURE.md`; `dom` además trae cambios sin commit de Antigravity (`dom_frontend.go` attrbool/state, `tests/uc_state_test.go`), ya sin agente activo. **layout**: `go.mod` (`replace` widget, `go mod tidy`). **form**: `go mod tidy` (faltaban entradas en go.sum). Verificado: selector 57,6px (56 + borde), arranque sin errores SSR | 🟢 local, espera revisión |
| A10 | widget, components, clinical_encounter | **widget**: `style/emit_values.go` (`glyphColor`: `Glyph(Subtle)` y `Glyph(Inactive)` pintan texto apagado; antes `Subtle` salía con el gris de fondo —RUT invisible en claro— e `Inactive` con `color: ;`), `emit_place.go`, `style/glyph_test.go`. **components**: `selectsearch/selectsearch.go` (`StartOpen`: abre sola la primera vez que tiene opciones y nada elegido), `selectsearch_test.go`. **clinical**: `ui/browser.go` (`StartOpen: true`). Verificado en vivo: abre al entrar, RUT `#6E6E73` | 🟢 local, espera revisión |
| A6 | devskills | `events-queues-and-alerts` (avisos), `widget-styling` (`OneLine`), `app-modules` (lista filtrada por selector), con líneas STATUS; sincronizado (`devskills -f`), sin publicar | 🟢 local |
| P | todos | Publicar en orden de dependencias con `gopush` (la cascada quita cada `replace`): dom → widget → sse → view → form → components → layout → appkit → staff_manager, client_directory, appointment_booking → clinical_encounter → mjosefa-cms; borrar las líneas STATUS de `events-queues-and-alerts`, `widget-styling` y `app-modules`, y este master | 🔴 tras revisión del dueño |

**Trabajo de otro agente en los mismos repos (no commitear ni revertir por error):** UI_BUGS de Antigravity (`veltylabs/mjosefa-cms/docs/UI_BUGS_MASTER_PLAN.md`): `appkit/browser/browser.go` (`GroupScreens`), `view/screen.go`, casi todo `components` salvo `selectsearch`, `appointment_booking/{definition*,lang.json,svg.go,ui/*}`, `staff_manager/{docs,rule_staff_member_delete.go,tests/test_staff_delete_test.go}`, `clinical_encounter/{tests/model_test.go,ui/svg.go}`, en mjosefa-cms todo menos `config/services/backend.go` y las dos líneas `replace`. `appointment_booking/ui` no compila hoy por ese trabajo (`calendarslider.New`).

## Bloqueos
- Ninguno.

## Relación con otras olas
- REACTIVE_RECORDS F5: comparte el stream `/notices` (D1).
- mjosefa-cms UI_BUGS: CE-1–CE-3 de `clinical_encounter/docs/BUGS_UI.md` quedan resueltos por A3.
- Hallazgos de otros dominios, por mover a su master: (1) un `crud.ReadArgs.Filter` codifica también los campos de resultado del modelo y un GET falla con un array (`MyAgendaResult.entries`; parche local `OmitEmpty`) → ARCHITECTURE; (2) `webtyp dev` no recompila el cliente cuando cambia la carpeta `ui/` de un módulo con `replace` → LOG_NOISE/devflow; (3) `browser_screenshot` del MCP vence en toda página (también en `/favicon.svg`).
