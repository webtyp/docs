# Agenda del doctor en vivo (ficha clínica) — master plan
> **Status:** PENDING · 2026-11-20 · sin preguntas; sigue A0 (cerrar el PR #20) y el `## Design gate` de A1

**Para retomar:** skills `framework-map`, `app-modules`, `module-layout`, `events-queues-and-alerts`, `api-design`, `sign-in-and-permissions`, `ui-consistency`. Probar en `~/Dev/Project/veltylabs/mjosefa-cms` con DEV_AUTOLOGIN de un usuario que sea profesional (`staff_manager`).

**Qué pide el dueño (2026-11-20):** el doctor abre «Ficha Paciente», ve en un `selectsearch` las reservas de **su día** y elige al paciente que va a atender; ahí se cargan sus fichas. Si entra una reserva nueva o cambia el estado de una, lo ve sin recargar. Así era el sistema anterior (la demo de referencia ya lo hacía: `~/Dev/Archive/app-demo/modules/medicalhistory`, `medicalhistory.go:114-139`, `model.go:89-97`):

![ficha](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-1-ficha.png) ![selector](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-2-selector.png) ![paciente](../../veltylabs/modules/clinical_encounter/docs/img/agenda-old-3-paciente.png)

**Qué hay hoy (verificado):**
- `clinical_encounter/ui/browser.go:115` declara `var myAgenda` y nunca la lee: `StaffId` siempre vacío → el selector **no aparece nunca**. Y si se leyera, lo haría bloqueando dentro de `Screens()` con una llamada por paciente (`blockingCall`, N+1; `docs/BUGS_UI.md` CE-3).
- `MyAgenda` (`agenda.go`) solo trae reservas CONFIRMED (`appointment_booking/booking.go:296` `ConfirmedByStaff`) y no trae estado, edad ni nombre del servicio.
- La lista de fichas usa `targethour` (horas del día, `ui/browser.go:105`) desde que se creó la UI (`4dcf728`); la referencia usaba `targetdate` (día/fecha de cada consulta, panel derecho de la tercera captura). `MedicalHistory.Item()` ya pone `attention_at` en `Badge` (`definition.go:21`), que es lo que lee `targetdate`; `Description` lleva el estado en vez del doctor (`definition.go:20`).
- `selectsearch.SetOptions` ya cambia la lista en caliente (`components/selectsearch/selectsearch.go:172`).
- **El aviso al navegador se perdió con appkit:** `mjosefa-cms` tenía push por SSE (`ac494a3`: `/inbox` solo para usuarios con sesión, un canal por usuario, `sse.Publisher` en `events.Fanout`, `sse.Subscriber` en el navegador). La recomposición del MVP (`4577178`) lo quitó y appkit no lo trae: `view.BrowserDeps` (`view/browser.go:21`) no tiene suscriptor y nada monta el stream. `chat_room` sigue publicando `InboxTopic(userID)` (`message.go:155`) sin que nadie escuche.

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1 · A) | **Aviso por usuario, sin datos en el cable** (lo que existía en `mjosefa-cms` `ac494a3`): appkit monta **un solo** stream SSE para usuarios con sesión y cada uno recibe solo su canal; `view.BrowserDeps` trae el suscriptor y el id del usuario. `appointment_booking` avisa el cambio de agenda de un profesional por la interfaz que `clinical_encounter` ya consume (`interfaces.go`, como `ConfirmedByStaff`), sin que ningún módulo nombre a otro; `clinical_encounter` lo traduce al usuario del profesional y publica `AgendaTopic(userID)`; la pantalla vuelve a leer `my_agenda` y hace `SetOptions`. REACTIVE_RECORDS F5 agrega después su evento al mismo stream | `view`, `appkit`, `appointment_booking`, `clinical_encounter` |
| D2 | El selector muestra las reservas **activas** del día del profesional, cada una con su estado como etiqueta: PENDING, CONFIRMED, COMPLETED, NO_SHOW, CONFLICTED. CANCELLED, EXPIRED y RESCHEDULED no aparecen. Solo CONFIRMED y COMPLETED abren la ficha (dueño, 2026-11-20). `ConfirmedByStaff` se reemplaza por un lector que devuelve también el estado | `appointment_booking` (lector), `clinical_encounter` (`MyAgenda`, UI) |
| D3 | Cada opción del selector como en la referencia y el sistema anterior: «Nombre, Edad años», RUT debajo, «09:35 - Servicio» como etiqueta, más la etiqueta del estado (D2); `AgendaEntry` suma estado, edad y nombre del servicio | `clinical_encounter` |
| D5 | La lista de fichas del paciente vuelve a `targetdate` (dueño, 2026-11-20: «ya se había creado»): día de la semana / número / mes-año a la izquierda, motivo como título, **doctor** como etiqueta (`Item: model.ItemDescription` pasa de `status` a `doctor_name_snapshot`, `definition.go:20`); `targethour` sale de `ui/browser.go` | `clinical_encounter` |
| D4 | La agenda se carga **al abrir la pantalla**, nunca en `Screens()`, y las fichas del paciente al elegirlo, sin llamadas bloqueantes (cierra CE-1 y CE-3 de `docs/BUGS_UI.md`; CE-2: placeholder con `lang`) | `clinical_encounter/ui` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A0 | clinical_encounter | Cerrar el plan viejo: `docs/PLAN.md` (part 2b, textos) tiene el PR #20 abierto desde 2026-11-12 sin sesión viva → revisarlo en local (**plan-review**), terminar en su rama y `codejob close` | 🔴 |
| A1 | view, appkit (+ sse si falta algo) | D1: el stream y su suscriptor en `view.BrowserDeps`; `## Design gate` (**api-design**) revisado por el dueño antes de codificar | 🔴 |
| A2 | appointment_booking | D2 (lector con estado, reemplaza `ConfirmedByStaff`) y el aviso de cambio de agenda de D1 | 🔴 |
| A3 | clinical_encounter | D1–D5 | 🔴 tras A0–A2 |
| A4 | chat_room | Vuelve a recibir su aviso por el mismo stream (`ui.WithInbox`) | 🔴 tras A1 |
| A5 | mjosefa-cms | En vivo: dos sesiones; recepción crea y confirma una reserva → la lista del doctor cambia sola | 🔴 al final |
| A6 | devskills | `events-queues-and-alerts` / `app-modules`: cómo una pantalla recibe avisos | 🔴 tras A3 |

## Bloqueos
- Ninguno.

## Relación con otras olas
- REACTIVE_RECORDS F5: comparte el stream SSE de appkit (D1).
- mjosefa-cms UI_BUGS: `clinical_encounter/docs/BUGS_UI.md` CE-1–CE-3 se cierran en A3.
- LOG_NOISE: mismo chat de origen, sin dependencia.
