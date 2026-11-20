# Agenda del doctor en vivo (ficha clínica) — master plan
> **Status:** PENDING · 2026-11-20 · esperando P1 del dueño (qué canal lleva el aviso al navegador)

**Para retomar:** skills `framework-map`, `app-modules`, `module-layout`, `events-queues-and-alerts`, `api-design`, `sign-in-and-permissions`, `ui-consistency`. Probar en `~/Dev/Project/veltylabs/mjosefa-cms` con DEV_AUTOLOGIN de un usuario que sea profesional (`staff_manager`).

**Qué pide el dueño (2026-11-20):** el doctor abre «Ficha Paciente», ve en un `selectsearch` las reservas de **su día** y elige al paciente que va a atender; ahí se cargan sus fichas. Si entra una reserva nueva o cambia el estado de una, lo ve sin recargar. Así era el sistema anterior:

![ficha](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-1-ficha.png) ![selector](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-2-selector.png) ![paciente](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-3-paciente.png)

**Qué hay hoy (verificado):**
- `clinical_encounter/ui/browser.go:115` declara `var myAgenda` y nunca la lee: `StaffId` siempre vacío → el selector **no aparece nunca**. Y si se leyera, lo haría bloqueando dentro de `Screens()` con una llamada por paciente (`blockingCall`, N+1; `docs/BUGS_UI.md` CE-3).
- `MyAgenda` (`agenda.go`) solo trae reservas CONFIRMED (`appointment_booking/booking.go:296` `ConfirmedByStaff`) y no trae estado, edad ni nombre del servicio.
- `selectsearch.SetOptions` ya cambia la lista en caliente (`components/selectsearch/selectsearch.go:172`).
- **El aviso al navegador se perdió con appkit:** `mjosefa-cms` tenía push por SSE (`ac494a3`: `/inbox` solo para usuarios con sesión, un canal por usuario, `sse.Publisher` en `events.Fanout`, `sse.Subscriber` en el navegador). La recomposición del MVP (`4577178`) lo quitó y appkit no lo trae: `view.BrowserDeps` (`view/browser.go:21`) no tiene suscriptor y nada monta el stream. `chat_room` sigue publicando `InboxTopic(userID)` (`message.go:155`) sin que nadie escuche.

## Preguntas abiertas
**P1 — ¿Por qué canal se entera la pantalla del doctor?** Lo que el doctor ve romperse hoy: la lista no cambia hasta recargar la página. Hay dos piezas del ecosistema que ya hacen la mitad:
A) **Aviso por usuario (`/inbox`, lo que existía):** appkit monta un solo stream SSE para usuarios con sesión; cada usuario recibe solo su canal. Cuando cambia una reserva, `clinical_encounter` (servidor) avisa al usuario del profesional, sin datos en el cable; la pantalla vuelve a leer `my_agenda`. Sirve igual para el chat (`chat_room` ya publica así).
B) **Registros reactivos (REACTIVE_RECORDS D7/F5, diseñado y sin conectar):** cada escritura queda en `changelog`, el SSE avisa «hay cambios hasta la versión N», el navegador baja las filas nuevas a `view.Records` y la pantalla escucha las reservas. Genérico, pero baja filas de `appointment_booking` a un módulo que no puede importarlo, y con 171.170 reservas importadas hay que acotar por fila antes de usarlo.
**¿Cuál evita el acoplamiento y es más mantenible?** A: ningún módulo nombra a otro (el aviso viaja por la interfaz de `interfaces.go` que ya usa `MyAgenda`) y no viajan datos. **Recomiendo A**, con el stream pensado como **el único** que monta appkit: F5 de REACTIVE_RECORDS después agrega su evento «versión N» al mismo stream, sin un segundo canal.
```go
// hoy (clinical_encounter/ui/browser.go): nada avisa; la lista queda fija
picker.Options = opts

// A — el navegador escucha el canal del usuario con sesión (d.Notices lo arma appkit)
d.Notices.Subscribe(clinicalencounter.AgendaTopic(d.UserID), func(events.Event) {
	reloadAgenda() // vuelve a llamar my_agenda y hace picker.SetOptions(...)
})
// A — servidor: appointment_booking avisa por la interfaz que clinical_encounter ya consume
m.agenda.OnStaffAgendaChange(func(staffID string) {
	userID := m.doctors.UserByStaff(staffID)
	m.notices.Publish(events.Event{Topic: AgendaTopic(userID)})
})

// B — el navegador escucha las filas de reservas que llegan a view.Records
d.Records.OnChange(reservationModelName, func() { reloadAgenda() }) // nombre de otro módulo
```
Respuesta:

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | La pantalla se refresca sola por push SSE (dueño, 2026-11-20); el canal concreto lo fija P1 | según P1 |
| D2 | El selector muestra las reservas **activas** del día del profesional, cada una con su estado como etiqueta: PENDING, CONFIRMED, COMPLETED, NO_SHOW, CONFLICTED. CANCELLED, EXPIRED y RESCHEDULED no aparecen. Solo CONFIRMED y COMPLETED abren la ficha (dueño, 2026-11-20). `ConfirmedByStaff` se reemplaza por un lector que devuelve también el estado | `appointment_booking` (lector), `clinical_encounter` (`MyAgenda`, UI) |
| D3 | Cada fila como en el sistema anterior: «Nombre Edad Años», debajo «09:35 - Servicio», RUT; `AgendaEntry` suma estado, edad y nombre del servicio. Obvio, sin pregunta | `clinical_encounter` |
| D4 | La agenda se carga **al abrir la pantalla**, nunca en `Screens()`, y las fichas del paciente al elegirlo, sin llamadas bloqueantes (cierra CE-1 y CE-3 de `docs/BUGS_UI.md`; CE-2: placeholder con `lang`) | `clinical_encounter/ui` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A0 | clinical_encounter | Cerrar el plan viejo: `docs/PLAN.md` (part 2b, textos) tiene el PR #20 abierto desde 2026-11-12 sin sesión viva → revisarlo en local (**plan-review**), terminar en su rama y `codejob close` | 🔴 |
| A1 | view, appkit (+ sse si falta algo) | El stream de P1 y su suscriptor en `view.BrowserDeps`; requiere `## Design gate` (**api-design**) | 🔴 espera P1 |
| A2 | appointment_booking | D2 (lector con estado) y el aviso de cambio de agenda | 🔴 espera P1 |
| A3 | clinical_encounter | D2–D4 y la suscripción de A1 | 🔴 tras A0–A2 |
| A4 | chat_room | Vuelve a recibir su aviso por el mismo stream (`ui.WithInbox`) | 🔴 tras A1 |
| A5 | mjosefa-cms | En vivo: dos sesiones; recepción crea y confirma una reserva → la lista del doctor cambia sola | 🔴 al final |
| A6 | devskills | `events-queues-and-alerts` / `app-modules`: cómo una pantalla recibe avisos | 🔴 tras A3 |

## Bloqueos
- Ninguno.

## Relación con otras olas
- REACTIVE_RECORDS F5: comparte el stream SSE de appkit (P1).
- mjosefa-cms UI_BUGS: `clinical_encounter/docs/BUGS_UI.md` CE-1–CE-3 se cierran en A3.
- LOG_NOISE: mismo chat de origen, sin dependencia.
