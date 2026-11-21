# Pruebas del dominio — master plan
> **Status:** PENDING · 2026-11-20 · **historias de usuario de punta a punta: esperan P6–P10 y la revisión del catálogo (abajo) por el dueño**; S3/S4 (orden de pruebas existentes) siguen al final

Cada agente agrega pruebas sin criterio de dominio: `mjosefa-cms/tests/` tiene 34 pruebas en 20 archivos y la
mayoría prueba el framework, no la clínica (`TestMCPClosedByDefault`, `TestDefaultDeny_NoAuthorizeConfigured`,
`TestHealth`, `TestPWAConfig`, `TestProjectLayout`, `TestIntrospectionListsRPCRoutes`, `TestSplitIPs`…). Una prueba
que sería idéntica en cualquier otra app pertenece a la librería que dueña ese comportamiento (responsabilidad única).
Meta: abrir un repo y saber, en un documento, qué reglas de su negocio están protegidas y dónde.

## Preguntas abiertas

**Disparador (dueño, 2026-11-20):** probar a mano cada pantalla es caro y deja pasar fallos (RUT invisible, selector
que no abre, formulario vacío). Las piezas existen; falta probar **las historias de usuario** contra una base **real de
prueba** del motor que usa la app, sin mocks, sin repetir código de implementación en cada app, y con un documento
que diga qué historia falla y por qué. Hoy: `gotest` ya compila a wasm y corre pruebas en un navegador
(`wasmbrowsertest`), pero sin servidor; `appkit/backend.Build(app, b, s, dsn, schema)` ya arma el servidor completo
con cualquier DSN; `devbrowser` ya maneja Chrome (chromedp). Nada los une.

**P6 — Dónde vive cada historia.** Si cada app escribe sus historias, la agenda clínica se reescribe en cada app que
use `clinical_encounter`. A) **cada módulo trae sus historias** en `stories/`, y la app solo dice cuáles corre;
B) cada app escribe las suyas en `tests/`. Una historia necesita datos de otros módulos (una reserva confirmada para
la ficha): el módulo los pide con una interfaz propia, igual que hoy pide `AgendaReader` (`interfaces.go`), y la app
la entrega armada con las historias de los otros módulos. **Recomiendo A**: el código de la historia se escribe una vez,
junto al código que prueba. ¿Cuál evita el acoplamiento y es más mantenible? A: el módulo no importa a otro.
```go
// A — clinical_encounter/stories/agenda.go (lo escribe el módulo, una vez)
type Given interface { // lo que la historia necesita que ya exista, sin importar appointment_booking
	ConfirmedReservationToday(doctor, patient string) (reservationID string)
}
var DoctorSeesOnlyConfirmedAsChoosable = apptest.Story{Flow: FlowDayAgenda, Run: func(s *apptest.Session, g Given) {
	g.ConfirmedReservationToday("doctor-a", "ana")            // dato real, por la operación real
	s.SignInAs("doctor-a").Open(ui.ID)                        // navegador real sobre el wasm de la app
	s.Expect(selectsearch.ClsSsOption).Count(1).NotDisabled()
}}
// A — mjosefa-cms/tests/stories_test.go (lo único que escribe la app)
func TestStories(t *testing.T) { apptest.Run(t, config.App, services.Backend, screens.Browser, clinicalstories.All, bookingstories.All) }
// B — la app copia el cuerpo de cada historia en su tests/, y otra app lo vuelve a copiar
```
Respuesta:

**P7 — Cómo se ejecuta una historia.** A) **navegador real** (Chrome sin ventana) sobre el wasm compilado de la app y su
servidor real; los datos previos («Dado») se crean por las operaciones reales actuando como un usuario, nunca SQL a
mano; B) solo operaciones (sin navegador): rápido, pero no ve un RUT invisible ni un selector que no abre.
**Recomiendo A**: los fallos de hoy eran de pantalla; B queda para las reglas de negocio, que ya prueba cada módulo.
Respuesta:

**P8 — Qué base usa.** A) **una base nueva del mismo motor de la app por corrida** (Postgres para mjosefa-cms:
`mjosefa_cms_test_<hora>`), migrada con la migración de la app y borrada al final; la conexión de prueba sale del
keyring de desarrollo (skill **secrets**); B) una base de prueba fija que se limpia; C) memoria (`storage/mem`).
**Recomiendo A**: cada corrida empieza limpia, dos corridas no se pisan y prueba el motor real; C no es la base real.
Respuesta:

**P9 — Qué pieza las une.** Búsqueda en el ecosistema: `wasmbrowsertest` corre pruebas wasm en un navegador (sin
servidor), `devbrowser` maneja Chrome para el MCP, `appkit/backend.Build` arma el servidor. A) **librería nueva
`webtyp/apptest`**: arma el servidor con `backend.Build` sobre la base de P8, compila `web/browser` una vez, abre
Chrome con el motor de `devbrowser`, y da `Story`, `Session` (`SignInAs`, `Open`, `Click`, `Expect`) y `Run`;
`gotest` las corre con `gotest -stories` (lentas: no van en cada `gotest`); B) meterlo en `appkit` (le suma Chrome y
compilación a una librería isomórfica); C) extender `wasmbrowsertest` (la prueba corre dentro del navegador y no
puede abrir dos sesiones a la vez, que es lo que pide una historia «la recepción confirma → el doctor lo ve»).
**Recomiendo A**. ¿Cuál evita el acoplamiento y es más mantenible? A: una sola pieza depende de Chrome, y solo en pruebas.
Respuesta:

**P10 — Cómo se marca qué falla.** A) **`gotest -stories` escribe la columna «Estado» de `docs/TESTS.md`** de la app
(✅ o ❌ con la primera línea del fallo y la fecha) en cada corrida; B) lo marca a mano el agente. **Recomiendo A**:
marcado a mano se desactualiza; con A el documento siempre dice lo que pasó en la última corrida.
Respuesta:

## Catálogo de historias propuesto (mjosefa-cms) — revisar antes de escribir código

Por dominio (módulo) y flujo; «2 sesiones» = dos navegadores a la vez. Al cerrar P6–P10 cada fila pasa al
`docs/TESTS.md` de su módulo (D1). **Piloto: H13–H16** (la agenda clínica, lo que hoy se prueba a mano).

| # | Dominio · flujo | Historia: «como … quiero …, entonces …» |
|---|---|---|
| H1 | Acceso · entrar | Un profesional entra con su RUT desde un equipo registrado y ve el menú de su rol |
| H2 | Acceso · entrar | Desde un equipo no registrado no se puede entrar |
| H3 | Acceso · permisos | Un rol sin permiso no ve la pantalla, y la operación le responde «sin permiso» |
| H4 | Personal · alta | Se crea un profesional con especialidad y aparece para elegir en la reserva |
| H5 | Catálogo · alta | Se crea un servicio con duración y precio y aparece en la reserva |
| H6 | Calendario · horario | El horario del profesional limita las horas ofrecidas; un feriado no ofrece horas |
| H7 | Pacientes · alta | Se crea un paciente con RUT válido; un RUT inválido o repetido se rechaza con su mensaje |
| H8 | Pacientes · búsqueda | Se encuentra un paciente por nombre y por RUT |
| H9 | Reservas · mesón | La recepción reserva una hora libre con un paciente nuevo creado en el mismo paso |
| H10 | Reservas · mesón | Una hora ya tomada no se ofrece (2 sesiones reservando la misma hora: una gana) |
| H11 | Reservas · estados | Confirmar, cancelar y «no asistió» cambian el estado que se ve en la lista |
| H12 | Registros · en vivo | Lo que una sesión edita se ve en la otra sin recargar (2 sesiones) |
| H13 | Ficha · agenda del día | El doctor ve solo sus reservas de hoy con edad, RUT, hora y estado; la pendiente se ve y no se elige |
| H14 | Ficha · agenda del día | La agenda abre sola al entrar; al elegir una confirmada se listan las fichas de ese paciente, por fecha |
| H15 | Ficha · registro | Ficha nueva trae doctor y paciente puestos; al guardar aparece en la lista y al elegirla se llena el formulario |
| H16 | Ficha · agenda en vivo | La recepción confirma una reserva y la agenda del doctor la muestra sin recargar (2 sesiones) |
| H17 | Ficha · privacidad | Un doctor no ve las fichas de un paciente que no está en su agenda |

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | `docs/TESTS.md` por repo: una sección por área, diagrama si hay flujo, filas «regla → enlace a la prueba» | S3, S4 |
| D2 (P2) | Nombre `Test<Área>_<Regla>` | S3, S4 |
| D3 (P3) | Un archivo por área: `tests/<área>_test.go`; ayudantes en `tests/setup_test.go` | S3, S4 |
| D4 (P4) | Una app prueba solo su negocio + humo de arranque; lo del framework va a la librería dueña | S3, S4 |
| D5 (P5) | `gotest` falla si una prueba no está en `docs/TESTS.md` o una fila apunta a una prueba inexistente | S2 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S1 | `devskills` (**testing**, **plan-authoring**, **plan-review**) | reglas D1–D5 (sección «Domain tests») | 🟢 hecho 2026-11-05 |
| S2 | `devflow` (`gotest`) | control de D5 (sin `docs/TESTS.md` → solo aviso; con él → falla) | 🟢 hecho 2026-11-05 (`v0.4.165`) |
| S3 | `mjosefa-cms` | `docs/TESTS.md`; mover las pruebas del framework a su librería; renombrar | espera: cierre de DATA_IMPORT (repo con cambios sin commit y `replace` locales) |
| S4 | librerías y módulos | lo mismo, repo por repo, al tocarse | al final |
| E1 | dueño | revisar el catálogo (agregar, quitar, cambiar historias) y responder P6–P10 | 🔴 tuyo |
| E2 | `apptest` (nueva, `gonew`), `devflow` (`gotest -stories`) | gate de **api-design**; la pieza de P9 y el estado de P10 | tras E1 |
| E3 | `clinical_encounter`, `appointment_booking`, `mjosefa-cms` | piloto H13–H16 contra Postgres de prueba; cada fallo queda ❌ con su porqué en `docs/TESTS.md` y se arregla en la librería dueña (**bug-fix-flow**) | tras E2 |
| E4 | resto de módulos | H1–H12, H17, dominio por dominio | tras E3 |
| E5 | `devskills` (**testing**) | regla «historias de usuario»: dónde viven, cómo se corren, cómo se marcan | con E2 |

## Bloqueos

- S3: `mjosefa-cms` tiene trabajo sin commit (DATA_IMPORT) y dependencias con `replace` local; no se despacha hasta que quede limpio.

## Relación con otras olas

- CLINICAL_AGENDA_LIVE: sus arreglos se verificaron a mano por el MCP; el piloto E3 los vuelve historias automáticas.

- DATA_IMPORT (`mjosefa-cms`): sus pruebas nuevas (`pastdb`) ya siguen la regla cuando S1 cierre.
