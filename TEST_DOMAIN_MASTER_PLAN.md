# Pruebas del dominio — master plan
> **Status:** IN PROGRESS · 2026-11-24 · **20/20 historias 🟢 en verde contra Postgres real + Chrome** (catálogo completo de `mjosefa-cms` implementado y pasando en local; listo para revisión y publicación del dueño)

Cada agente agrega pruebas sin criterio de dominio: `mjosefa-cms/tests/` tiene 34 pruebas en 20 archivos y la
mayoría prueba el framework, no la clínica (`TestMCPClosedByDefault`, `TestDefaultDeny_NoAuthorizeConfigured`,
`TestHealth`, `TestPWAConfig`, `TestProjectLayout`, `TestIntrospectionListsRPCRoutes`, `TestSplitIPs`…). Una prueba
que sería idéntica en cualquier otra app pertenece a la librería que dueña ese comportamiento (responsabilidad única).
Meta: abrir un repo y saber, en un documento, qué reglas de su negocio están protegidas y dónde.

**Disparador (dueño, 2026-11-20):** probar a mano cada pantalla es caro y deja pasar fallos (RUT invisible, selector
que no abre, formulario vacío). Las piezas existen; falta probar **las historias de usuario** contra una base **real de
prueba** del motor que usa la app, sin mocks, sin repetir código de implementación en cada app, y con un documento
que diga qué historia falla y por qué. Hoy: `gotest` ya compila a wasm y corre pruebas en un navegador
(`wasmbrowsertest`), pero sin servidor; `appkit/backend.Build(app, b, s, dsn, schema)` ya arma el servidor completo
con cualquier DSN; `devbrowser` ya maneja Chrome (chromedp). Nada los une.

## Preguntas abiertas

Ninguna. (P13 resuelta en local 2026-11-23: `view.Saver.Save` llena el registro enviado con la respuesta del servidor y `layout/crudview` pasa a editar el registro creado, serializa guardados en vuelo y limpia `ServerOnly` al crear una ficha nueva.)

## Catálogo de historias (mjosefa-cms)

Id = `H-<DOMINIO>-<NN>` (D11): el número cuenta dentro de su dominio; agregar toma el siguiente de ese dominio,
quitar deja el hueco (nunca se renumera). Archivo `stories/h_<dominio>_<nn>_<flujo>.go` en el módulo dueño, así la
carpeta queda ordenada por dominio y número. «2 sesiones» = dos navegadores a la vez. Cada fila pasa al
`docs/TESTS.md` de su módulo (D1) cuando su historia se escribe. **Estado (D12):** 🔵 por revisar · 🟡 construyendo · 🟠 depurando · 🔴 no pasa · 🟢 lista e implementada. **Primero H-ACCESS-01** (dueño, 2026-11-20); las demás después.

| Id | Módulo · flujo | Historia | Estado |
|---|---|---|---|
| H-ACCESS-01 | access · entrar | Un profesional entra con su RUT desde un equipo registrado y ve el menú de su rol | 🟢 lista (`auth/stories`, 6 s) |
| H-ACCESS-02 | access · entrar | Desde un equipo no registrado no se puede entrar | 🟢 lista (el navegador entra desde loopback; el setup se hace desde la IP de red del equipo) |
| H-ACCESS-03 | access · permisos | Un rol sin permiso no ve la pantalla, y la operación le responde «sin permiso» | 🟢 lista (`auth/stories`, 4 s; 2026-11-23) |
| H-STAFF-01 | staff_manager · alta | Se crea un profesional con especialidad y aparece para elegir en la reserva | 🟢 lista (`staff_manager/stories`, 3 s; 2026-11-23) |
| H-CATALOG-01 | service_catalog · alta | Se crea un servicio con duración y precio y aparece en la reserva | 🟢 lista (`service_catalog/stories`, 3 s; 2026-11-23) |
| H-CALENDAR-01 | business_calendar · horario | El horario del profesional limita las horas ofrecidas; un feriado no ofrece horas | 🟢 lista (`business_calendar/stories`, 4 s; 2026-11-23) |
| H-PATIENT-01 | client_directory · alta | Paciente con RUT válido se crea; RUT inválido o repetido se rechaza con su mensaje | 🟢 lista (`client_directory/stories`, 5 s; 2026-11-23) |
| H-PATIENT-02 | client_directory · búsqueda | Se encuentra un paciente por nombre y por RUT | 🟢 lista (`client_directory/stories`, 5 s; 2026-11-23) |
| H-BOOKING-01 | appointment_booking · mesón | La recepción reserva una hora libre con un paciente nuevo creado en el mismo paso | 🟢 lista (`appointment_booking/stories`, 4 s; 2026-11-23) |
| H-BOOKING-02 | appointment_booking · mesón | Una hora ya tomada no se ofrece (2 sesiones reservando la misma hora: una gana) | 🟢 lista (`appointment_booking/stories`, 5 s; 2026-11-23) |
| H-BOOKING-03 | appointment_booking · estados | Confirmar, cancelar y «no asistió» cambian el estado que se ve en la lista | 🟢 lista (`appointment_booking/stories`, 5 s; 2026-11-24) |
| H-RECORDS-01 | view (cualquier módulo) · en vivo | Lo que una sesión edita se ve en la otra sin recargar (2 sesiones) | 🟢 lista (`service_catalog/stories`, 3 s; 2026-11-24) |
| H-ENCOUNTER-01 | clinical_encounter · agenda del día | El doctor ve solo sus reservas de hoy con edad, RUT, hora y estado; la pendiente se ve y no se elige | 🟢 lista (`clinical_encounter/stories`, 6 s; 2026-11-23) |
| H-ENCOUNTER-02 | clinical_encounter · agenda del día | La agenda abre sola al entrar; al elegir una confirmada se listan las fichas de ese paciente, por fecha | 🟢 lista (`clinical_encounter/stories`, 7 s; 2026-11-23) |
| H-ENCOUNTER-03 | clinical_encounter · registro | Ficha nueva trae doctor y paciente puestos; al guardar aparece en la lista y al elegirla se llena el formulario | 🟢 lista (`clinical_encounter/stories`, 6 s; 2026-11-23) |
| H-ENCOUNTER-04 | clinical_encounter · agenda en vivo | La recepción confirma una reserva y la agenda del doctor la muestra sin recargar (2 sesiones) | 🟢 lista (`clinical_encounter/stories`, 6 s; 2026-11-23) |
| H-ENCOUNTER-05 | clinical_encounter · privacidad | Un doctor no ve las fichas de un paciente que no está en su agenda | 🟢 lista (`clinical_encounter/stories`, 5 s; 2026-11-23) |
| H-ROOM-01 | room_layout · alertas | Dos turnos que se pisan en el mismo box (por ejemplo, llegados de la importación de Pa100T) se ven en rojo en «Alertas» y en el box; al mover uno a mano, la alerta desaparece (dueño, 2026-11-23) | 🟢 lista (`room_layout/stories`, 4 s; 2026-11-23) |
| H-ROOM-02 | room_layout · autogestión | Un profesional entra con su sesión y en «Ocupación» ve solo los boxes donde tiene turno; puede acomodar su propio turno (día, horario, box libre) sin pasar por el administrador (dueño, 2026-11-23) | 🟢 lista (`room_layout/stories`, 5 s; 2026-11-23) |
| H-ROOM-03 | room_layout · autogestión | Un profesional no puede cambiar ni borrar el turno de otro: la pantalla no ofrece la acción y la operación responde «sin permiso» (dueño, 2026-11-23) | 🟢 lista (`room_layout/stories`, 5 s; 2026-11-23) |

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | `docs/TESTS.md` por repo: una sección por área, diagrama si hay flujo, filas «regla → enlace a la prueba» | S3, S4 |
| D2 (P2) | Nombre `Test<Área>_<Regla>` | S3, S4 |
| D3 (P3) | Un archivo por área: `tests/<área>_test.go`; ayudantes en `tests/setup_test.go` | S3, S4 |
| D4 (P4) | Una app prueba solo su negocio + humo de arranque; lo del framework va a la librería dueña | S3, S4 |
| D5 (P5) | `gotest` falla si una prueba no está en `docs/TESTS.md` o una fila apunta a una prueba inexistente | S2 |
| D6 (P6) | **Cada módulo trae sus historias** en `stories/`; lo que necesita de otros módulos lo pide con una interfaz propia (`Given`), que la app entrega armada con las historias de esos módulos; la app solo lista cuáles corre | E2, E3 |
| D7 (P7) | **Navegador real** (Chrome sin ventana) sobre el wasm de la app y su servidor real; los datos previos se crean con las operaciones reales actuando como un usuario: sin mocks, sin SQL a mano | E2 |
| D8 (P8) | **Base nueva del motor de la app por corrida** (Postgres en mjosefa-cms), migrada con la migración de la app y borrada al final; la conexión de prueba sale del keyring de desarrollo (**secrets**) | E2 |
| D9 (P9) | **Librería nueva `webtyp/apptest`** (`Story`, `Session`, `Run`) sobre `appkit/backend.Build` y el Chrome de `devbrowser`; se corre con `gotest -stories`, no en cada `gotest` | E2 |
| D10 (P10) | **`gotest -stories` escribe la columna «Estado»** de `docs/TESTS.md` (🟢, o 🔴 con la primera línea del fallo y la fecha; colores en D12) | E2 |
| D11 | **Id de historia `H-<DOMINIO>-<NN>`**, número por dominio, sin renumerar; archivo `stories/h_<dominio>_<nn>_<flujo>.go`; la subprueba lleva el id (`TestStories/H-ENCOUNTER-01`) y la fila de `docs/TESTS.md` también (dueño, 2026-11-20) | E2, E5 |
| D12 | **Colores de estado de una historia**: 🔵 por revisar, 🟡 construyendo, 🟠 depurando, 🔴 no pasa, 🟢 lista e implementada; `gotest -stories` escribe 🟢/🔴, el agente marca 🟡/🟠 mientras trabaja (dueño, 2026-11-20) | E2, E5 |
| D13 (P11) | **Al guardar un profesional con RUT, `staff_manager` le da su cuenta de entrada** por su interfaz `SignInAccounts` (`GiveSignIn(rut, name)`); `appkit` la entrega en `Deps.SignInAccounts`, conectada tarde al modo que sepa dar cuentas (`auth.GivesSignIn`, que `trusted_ip` implementa); sin modo que pueda, guardar responde `ErrNoAccountGiver`. Además corrige que el profesional heredaba el `UserId` del administrador que lo creaba | hecho en local |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S1 | `devskills` (**testing**, **plan-authoring**, **plan-review**) | reglas D1–D5 (sección «Domain tests») | 🟢 hecho 2026-11-05 |
| S2 | `devflow` (`gotest`) | control de D5 (sin `docs/TESTS.md` → solo aviso; con él → falla) | 🟢 hecho 2026-11-05 (`v0.4.165`) |
| S3 | `mjosefa-cms` | `docs/TESTS.md`; mover las pruebas del framework a su librería; renombrar | espera: cierre de DATA_IMPORT (repo con cambios sin commit y `replace` locales) |
| S4 | librerías y módulos | lo mismo, repo por repo, al tocarse | al final |
| E2 | `apptest`, `postgres`, `auth/stories`, `server`, `staff_manager`, `layout`, `mjosefa-cms` | Hecho en local para H-ACCESS-01 (etapas 1, 3–6 y la historia). Correr: `cd mjosefa-cms && go test -tags stories -run TestStories -v ./tests/`. Falta: etapa 2 (`fetch` cookie jar) y `SignedInAs`/`DEV_AUTOLOGIN` (las piden las historias con datos previos), etapa 7 (`gotest -stories` escribe el Estado; hoy se marca a mano aquí) | 🟢 local, espera revisión |
| E3 | `clinical_encounter`, `appointment_booking`, `mjosefa-cms` | piloto H-ENCOUNTER-01…04 contra Postgres de prueba; cada fallo queda ❌ con su porqué en `docs/TESTS.md` y se arregla en la librería dueña (**bug-fix-flow**) | tras E2 |
| E4 | resto de módulos | el resto del catálogo, dominio por dominio | tras E3 |
| E5 | `devskills` (**testing**) | regla «historias de usuario»: dónde viven, cómo se corren, cómo se marcan | con E2 |

## Trabajo local sin publicar (H-ACCESS-01…03, D13)

El dueño publicó la cascada el 2026-11-20 23:17 (server v0.2.142, staff_manager, layout, mjosefa-cms v0.0.133…): los arreglos de H-ACCESS-01 ya están publicados. Queda local:

- **apptest** (solo `docs/PLAN.md` publicado): `apptest.go` (`Story[G]`, `Stories[G]`, `UsingGiven`, `Run`), `app.go` (compila una vez `web/server` `-tags dev`, la migración y el sitio; por historia: base nueva, migración, servidor sin `DEV_AUTOLOGIN`, Chrome sin ventana; `NewSession`/`NewSessionOnAnotherDevice` (un navegador que entra por la IP de red del equipo; sin ella la historia se salta), `Logf`, `Fatalf`; sin puertas traseras: no hay llamadas HTTP para preparar datos), `session.go` (`NewSession`, `OpenHome`, `FillField`, `Submit`, `Find`, `FindField`, `ExpectVisible/Gone/Count/AtLeast/SomeText`, `Click`; al fallar: PNG en `/tmp/apptest-failures/`, URL y texto). `go.mod`: `replace` postgres.
- **postgres**: `empty_database.go` (`NewEmptyDatabase`), `backup.go` lo usa, `tests/empty_database_test.go`, README.
- **apptest** además: `OpenScreen` (busca solo dentro de la pantalla con `widget.Current`: el chasis deja montadas todas), `FindButton`, `ExpectState`, `ExpectPageText`, `Count`, `StatusOf` (GET con la sesión del navegador); `Click` hace clic sobre el elemento (no en coordenadas: la pantalla entra deslizándose); `FillField` sale del campo (el guardado automático se dispara al confirmar).
- **auth/stories** (módulo aparte para que `auth` no cargue Chrome): `given.go` (`Given{MenuEntry, SignInMessage, AddPersonWithoutRole, AdminOnlyRead}`), `h_access_01_sign_in.go`, `h_access_02_unregistered_device.go`, `h_access_03_no_role.go`, `go.mod` (`replace` apptest, postgres).
- **auth** (D13): `user.go` (`GivesSignIn`), `trusted_ip/rut.go` (`Authenticator.GiveSignIn`), `tests/give_sign_in_test.go`.
- **appkit** (D13): `backend/backend.go` (`SignInAccounts`, `Deps.SignInAccounts`), `backend/accounts.go` (`lateAccounts`, `ErrNoAccountGiver`), `backend/build.go`, `backend/signin.go`, `tests/sign_in_accounts_test.go`, `docs/TESTS.md`; `go.mod`: `replace` auth.
- **staff_manager** (D13): `interfaces.go` (`SignInAccounts`), `module.go` (`Deps.Accounts`), `rule_staff_member_create.go` (`giveSignIn`; ya no copia el `UserId` del creador), `tests/test_staff_sign_in_test.go`, `docs/TESTS.md`; módulo nuevo `stories/` (`AddProfessional`: agrega un profesional por la pantalla «Personal»).
- **devflow**: `gotest.go` (`listWasmTestPackages`) + `gotest_destinations.go`: el paso «browser» salta un submódulo sin pruebas de navegador (un `stories/` es todo `!wasm`); `tests/gotest_destinations_test.go`. El mismo repo trae cambios sin commit de otra tarea (`go_mod.go`, `go_tidy.go`, `docs/GONEW.md`, `tests/go_tidy_test.go`, `tests/gonew_test.go`): no tocarlos.
- **layout**: `login/login.go` exporta `ClsMessage` (dónde el login escribe la respuesta).
- **mjosefa-cms**: `tests/stories_test.go` (Given de acceso completo), `config/services/backend.go` (`Accounts: d.SignInAccounts`); `go.mod` con `replace` apptest, auth, auth/stories, appkit, staff_manager, staff_manager/stories, postgres, layout.
- Correr: `cd mjosefa-cms && go test -tags stories -run TestStories -v ./tests/` (3 historias, ~20 s; H-ACCESS-03 roja por P12).
- Publicar (tras revisión): devflow, postgres, layout, auth → appkit → staff_manager → apptest → auth/stories, staff_manager/stories → mjosefa-cms.
- Hallazgo (este dominio): las pruebas de los módulos usan `storage/mem`, que no revisa llaves foráneas; el fallo del primer administrador solo apareció contra Postgres.

## Bloqueos

- 2026-11-21 · H-ACCESS-03 espera `ROOM_MAP_MASTER_PLAN.md` (P1, R3): desde qué PC entra un profesional sale de su asignación a un box en `room_layout`, no de un campo en «Personal» (dueño).

- S3: `mjosefa-cms` tiene trabajo sin commit (DATA_IMPORT) y dependencias con `replace` local; no se despacha hasta que quede limpio.

## Relación con otras olas

- CLINICAL_AGENDA_LIVE: sus arreglos se verificaron a mano por el MCP; el piloto E3 los vuelve historias automáticas.

- DATA_IMPORT (`mjosefa-cms`): sus pruebas nuevas (`pastdb`) ya siguen la regla cuando S1 cierre.
