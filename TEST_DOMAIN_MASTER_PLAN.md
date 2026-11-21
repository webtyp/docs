# Pruebas del dominio — master plan
> **Status:** IN PROGRESS · 2026-11-20 · **historias de usuario de punta a punta: P6–P10 cerradas (D6–D10); H-ACCESS-01 y 02 🟢; **H-ACCESS-03 🔴 espera P11** (cómo recibe acceso un profesional nuevo)**; S3/S4 (orden de pruebas existentes) siguen al final

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

**P11 — Cómo recibe acceso un profesional nuevo (bloquea H-ACCESS-03 y cualquier historia con un segundo usuario).**
Lo que se ve: el administrador crea a la Dra. Rojas en «Personal» con su RUT y su equipo, pero ella no puede entrar
nunca: el login responde «Access not authorized». La causa: entrar con RUT pide un usuario con identidad RUT
(`trustedip.RegisterRUT`), y hoy eso solo lo hacen el setup del primer administrador (`auth/trusted_ip/mode.go:41-50`)
y el importador de la base antigua (`mjosefa-cms/migrate/pastdb/staff.go:176-199`). Ninguna pantalla lo hace.
A) **al guardar un profesional con RUT, `staff_manager` le da acceso** por una interfaz propia que la app entrega
armada desde su modo de entrada (mismo patrón que `Registrar` para los equipos); el rol de permisos se sigue
asignando en «Permisos» (`access`). B) una pantalla nueva en `access`, «Dar acceso», pide RUT, nombre y rol, crea el
usuario, y `staff_manager` lo vincula por RUT. **Recomiendo A**: el administrador ya escribe el RUT y el equipo en el
formulario de personal, que es donde espera que la persona quede lista para entrar; con B escribe el mismo RUT dos
veces en dos pantallas. ¿Cuál evita el acoplamiento y es más mantenible? A: `staff_manager` no importa `auth`, solo
declara la interfaz; B suma a `access` un vínculo con el directorio de personal.
```go
// A — staff_manager/interfaces.go (el módulo declara lo que necesita, sin importar auth)
type SignInAccounts interface {
	GiveSignIn(rut, name string) (userID string, err error) // crea el usuario con identidad RUT, o devuelve el que ya existe
}
// A — mjosefa-cms/config/services/backend.go (la app lo arma desde su modo de entrada, como hoy Registrar)
staffmanager.Deps{..., Accounts: d.SignInAccounts}
// B — access/ui: pantalla «Dar acceso» con RUT + nombre + rol; staff_manager busca el usuario por RUT al guardar
```
Respuesta:

## Catálogo de historias (mjosefa-cms)

Id = `H-<DOMINIO>-<NN>` (D11): el número cuenta dentro de su dominio; agregar toma el siguiente de ese dominio,
quitar deja el hueco (nunca se renumera). Archivo `stories/h_<dominio>_<nn>_<flujo>.go` en el módulo dueño, así la
carpeta queda ordenada por dominio y número. «2 sesiones» = dos navegadores a la vez. Cada fila pasa al
`docs/TESTS.md` de su módulo (D1) cuando su historia se escribe. **Estado (D12):** 🔵 por revisar · 🟡 construyendo · 🟠 depurando · 🔴 no pasa · 🟢 lista e implementada. **Primero H-ACCESS-01** (dueño, 2026-11-20); las demás después.

| Id | Módulo · flujo | Historia | Estado |
|---|---|---|---|
| H-ACCESS-01 | access · entrar | Un profesional entra con su RUT desde un equipo registrado y ve el menú de su rol | 🟢 lista (`auth/stories`, 6 s) |
| H-ACCESS-02 | access · entrar | Desde un equipo no registrado no se puede entrar | 🟢 lista (el navegador entra desde loopback; el setup se hace desde la IP de red del equipo) |
| H-ACCESS-03 | access · permisos | Un rol sin permiso no ve la pantalla, y la operación le responde «sin permiso» | 🔴 no pasa: ningún flujo da acceso a un segundo profesional (P11) |
| H-STAFF-01 | staff_manager · alta | Se crea un profesional con especialidad y aparece para elegir en la reserva | 🔵 por revisar |
| H-CATALOG-01 | service_catalog · alta | Se crea un servicio con duración y precio y aparece en la reserva | 🔵 por revisar |
| H-CALENDAR-01 | business_calendar · horario | El horario del profesional limita las horas ofrecidas; un feriado no ofrece horas | 🔵 por revisar |
| H-PATIENT-01 | client_directory · alta | Paciente con RUT válido se crea; RUT inválido o repetido se rechaza con su mensaje | 🔵 por revisar |
| H-PATIENT-02 | client_directory · búsqueda | Se encuentra un paciente por nombre y por RUT | 🔵 por revisar |
| H-BOOKING-01 | appointment_booking · mesón | La recepción reserva una hora libre con un paciente nuevo creado en el mismo paso | 🔵 por revisar |
| H-BOOKING-02 | appointment_booking · mesón | Una hora ya tomada no se ofrece (2 sesiones reservando la misma hora: una gana) | 🔵 por revisar |
| H-BOOKING-03 | appointment_booking · estados | Confirmar, cancelar y «no asistió» cambian el estado que se ve en la lista | 🔵 por revisar |
| H-RECORDS-01 | view (cualquier módulo) · en vivo | Lo que una sesión edita se ve en la otra sin recargar (2 sesiones) | 🔵 por revisar |
| H-ENCOUNTER-01 | clinical_encounter · agenda del día | El doctor ve solo sus reservas de hoy con edad, RUT, hora y estado; la pendiente se ve y no se elige | 🔵 por revisar |
| H-ENCOUNTER-02 | clinical_encounter · agenda del día | La agenda abre sola al entrar; al elegir una confirmada se listan las fichas de ese paciente, por fecha | 🔵 por revisar |
| H-ENCOUNTER-03 | clinical_encounter · registro | Ficha nueva trae doctor y paciente puestos; al guardar aparece en la lista y al elegirla se llena el formulario | 🔵 por revisar |
| H-ENCOUNTER-04 | clinical_encounter · agenda en vivo | La recepción confirma una reserva y la agenda del doctor la muestra sin recargar (2 sesiones) | 🔵 por revisar |
| H-ENCOUNTER-05 | clinical_encounter · privacidad | Un doctor no ve las fichas de un paciente que no está en su agenda | 🔵 por revisar |

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

## Trabajo local sin publicar (H-ACCESS-01, 02)

El dueño publicó la cascada el 2026-11-20 23:17 (server v0.2.142, staff_manager, layout, mjosefa-cms v0.0.133…): los arreglos de H-ACCESS-01 ya están publicados. Queda local:

- **apptest** (solo `docs/PLAN.md` publicado): `apptest.go` (`Story[G]`, `Stories[G]`, `UsingGiven`, `Run`), `app.go` (compila una vez `web/server` `-tags dev`, la migración y el sitio; por historia: base nueva, migración, servidor sin `DEV_AUTOLOGIN`, Chrome sin ventana; `ThisDevice`/`AnotherDevice` (IP de red del equipo; sin ella la historia se salta), `PostJSON` → `Reply{Status, Body, Location}`, `Logf`, `Fatalf`), `session.go` (`NewSession`, `OpenHome`, `FillField`, `Submit`, `Find`, `FindField`, `ExpectVisible/Gone/Count/AtLeast/SomeText`, `Click`; al fallar: PNG en `/tmp/apptest-failures/`, URL y texto). `go.mod`: `replace` postgres.
- **postgres**: `empty_database.go` (`NewEmptyDatabase`), `backup.go` lo usa, `tests/empty_database_test.go`, README.
- **auth/stories** (módulo aparte para que `auth` no cargue Chrome): `given.go` (`Given{MenuEntry, SignInMessage}`), `h_access_01_sign_in.go`, `h_access_02_unregistered_device.go`, `go.mod` (`replace` apptest, postgres).
- **layout**: `login/login.go` exporta `ClsMessage` (dónde el login escribe la respuesta).
- **mjosefa-cms**: `tests/stories_test.go` ya publicado; `go.mod` con `replace` apptest, auth/stories, postgres, layout.
- Correr: `cd mjosefa-cms && go test -tags stories -run TestStories -v ./tests/` (2 historias, ~7 s).
- Publicar (tras revisión): postgres → layout → apptest → auth/stories → mjosefa-cms.
- Hallazgo (este dominio): las pruebas de los módulos usan `storage/mem`, que no revisa llaves foráneas; el fallo del primer administrador solo apareció contra Postgres.

## Bloqueos

- S3: `mjosefa-cms` tiene trabajo sin commit (DATA_IMPORT) y dependencias con `replace` local; no se despacha hasta que quede limpio.

## Relación con otras olas

- CLINICAL_AGENDA_LIVE: sus arreglos se verificaron a mano por el MCP; el piloto E3 los vuelve historias automáticas.

- DATA_IMPORT (`mjosefa-cms`): sus pruebas nuevas (`pastdb`) ya siguen la regla cuando S1 cierre.
