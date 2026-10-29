# La pieza «aplicación»: webtyp entrega la app armada y la app solo la configura — master plan
> **Status:** IN PROGRESS · 2026-10-28 · sin preguntas abiertas. R0–R3 cerradas (menú por permiso, explorador solo, «me» con permisos guardados). Faltan: 3 módulos de M en vuelo, B1–B3 (diseño) y la aceptación Z. Revisión: Antigravity

**Disparador (mjosefa-cms, 2026-10-24):** «Catálogo» aparece en el menú pero al tocarlo no se ve
nada; Equipos, Zonas e Interfaces muestran el mismo dibujo (una manzana); el Explorador de API no
tiene ícono; ninguna pantalla explica para qué sirve. Y `mjosefa-cms/config/` tiene 1.407 líneas,
de las que unas 1.100 tendría que repetir cualquier otra app con login (`iam` ya repite parte:
`iam/config/server.go`, `iam/config/auth.go`). **Regla del dueño:** webtyp entrega la pieza lista;
la app solo la configura.

**Por qué pasa (2026-10-24):** `webtyp` tiene 135 librerías, pero ninguna que diga «esto es una
app» (login, permisos, menú y servidor ya armados juntos, como traen Django, Rails o Laravel). Cada
app los pega a mano, y lo que cruza varias piezas (por ejemplo «el menú muestra solo lo que puedes
usar») no lo tiene ninguna. Esta ola construye esa pieza; R, B1, B2 y B3 son sus partes, y termina
cuando `mjosefa-cms` e `iam` la usan (regla de **master-plan**: una ola cierra en la app de referencia).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D13 (P8) | **El contrato de pantalla vive en `webtyp/view`** (`view.Screen`): el módulo declara sus pantallas como datos (`Screens(...) []view.Screen`) y `layout/app.BuildMenu` las dibuja con su permiso; ningún módulo importa `layout/platformd` | M | los 16 módulos con `ui/` | `Screens(...) []view.Screen`, sin `platformd` | 🟡 13 publicados; en vuelo `appointment_booking`, `chat_room`, `clinical_encounter` (pantallas propias como `Component`, D20). Reciben además la corrección de ítems de REACTIVE_RECORDS (G4) |
| D20 (P14) | **Pantallas propias armadas con piezas de `layout`**: una pantalla `Component` (D19, ya publicada en view v0.6.46) puede armarse con `layout/chatview`, `layout/crudview` o `layout/rightpanel` (`chat_room`, `clinical_encounter`, `appointment_booking`); ese módulo importa esas piezas y publica cuando `layout` cambie (costo aceptado). `chatview` no se muda: usa `rightpanel` y `layout` importa `components` (sería un ciclo) | M |
| D5 (P3) | **Pieza de sesión del navegador** en `webtyp/layout` (pedir «me», mostrar login o alta del primer admin, cerrar sesión, bloqueo por inactividad), armada con `layout/login`, `platformd` y `components/usermenu`; la app le da nombre, logo y formularios. Búsqueda: no existe en `components` (solo `usermenu`) ni en `layout` (solo la pantalla visual `login`) | B1 |
| D6 (P4) | **El armado del servidor sirve a `mjosefa-cms` y a `iam`**: los contratos los monta `appkit.MountServer` (rpc, MCP, `/_routes`, negar todo por defecto) y el transporte lo pone `webtyp/server` (leer el entorno, escuchar); las dos apps borran su `config/server.go`. La base la elige cada app (invariante 6 de **project-layout**) | B2 |
| D9 (P5) | **Primer administrador repartido por dueño**: `webtyp/access` crea el usuario y le da el rol de administrador (la misma pieza que usa `iam` por correo); `staff_manager` registra el equipo desde el que entró y su ficha. `access` no puede hacer la parte del equipo: `webtyp` no conoce los módulos de `veltylabs`. La app solo conecta las dos | B3 |
| D10 | **Lo que toda app webtyp trae sin escribirlo**: login (red local por RUT, correo, passkey), roles y permisos con alta del primer admin, menú que muestra solo lo que puedes usar, ayuda «?» en cada pantalla, íconos, auditoría, explorador de API, servidor que niega todo por defecto, instalable (PWA). Cada hueco contra esta lista es una fase | todas |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| I1 | `webtyp/icons` | glifos `laptop`, `location`, `network`, `terminal` | 🟢 cerrada v0.0.33 |
| F1 | `webtyp/dom` | la señal es dueña de los hijos de `BindChildren` | 🟢 cerrada v0.13.29 |
| F2 | `webtyp/layout`, `form`, `dbsync`, `pwa` | dejaron de copiar a mano los hijos de la señal | 🟢 cerradas: layout v0.3.70, form v0.4.52, dbsync v0.3.10, pwa v0.2.13 |
| F3 | `webtyp/components` | `datatable`: las filas van en la señal | 🟢 cerrada v0.8.51 |
| G1 | `veltylabs/service_catalog` | ayuda en el grupo, Servicios y Especialidades | 🟢 cerrada v0.6.4 |
| I2 | `webtyp/layout` (`apiexplorer`) | ícono `terminal` | 🟢 cerrada v0.3.70 |
| F4 | `webtyp/layout` (`platformd`) | el «?» del título con forma de botón | 🟢 cerrada v0.3.75 |
| I3 | `veltylabs/device_manager` | íconos propios de Equipos, Zonas e Interfaces y su ayuda | 🟢 cerrada v0.2.71 |
| A | `webtyp/appkit` | `MountServer` sobre `router.Router` | 🟢 cerrada v0.0.4 |
| A2 | `webtyp/layout` | contratos de chasis en la raíz `layout` + `layout/app.BuildMenu` | 🟢 cerrada v0.3.76 |
| T1 | `webtyp/view` | `lang.json` con los 8 idiomas | 🟢 cerrada v0.6.45 |
| R0a | `webtyp/router` | `ResourceIntrospection` | 🟢 cerrada v0.4.9 |
| R0b | `webtyp/layout`, `webtyp/appkit` | explorador solo en el menú (`router.ResourceIntrospection`), menú cerrado por defecto, hoja `Component` en el menú | 🟢 cerradas: appkit v0.0.5, layout v0.3.77 |
| R0c | `webtyp/view` | hoja `Component` en `view.Screen` | 🟢 cerrada v0.6.46 |
| R1 | `webtyp/rbac` | `SubjectGrants`: los permisos tal como están guardados | 🟢 cerrada v0.0.22 |
| R2 | `webtyp/auth` | «me» manda los permisos guardados (`*:crud`); `ProfileDTO.Can` | 🟢 cerrada v0.0.80 |
| R3 | `mjosefa-cms` | menú desde `ui.Screens` + `BuildMenu`; `profile.Can`; `Permissions{Grants}`; borra `Modules()`, `rbacResolver`, `ResourceAPIExplorer`, explorador a mano, `rbac_modules_test.go`; «Catálogo» abre | 🟢 cerrada v0.0.66 |
| M | los 16 módulos con `ui/` | `Browser(...)` → `Screens(caller) []view.Screen`, sin `layout` en `go.mod` | 🟡 13 publicados (`device_manager` v0.2.72, `service_catalog` v0.6.12, `network_manager` v0.1.84, `staff_manager` v0.1.153, `pharmacy_manager` v0.1.36, `alert_manager` v0.1.16, `patient_directory` v0.1.62, `business_calendar` v0.5.16, `supply_manager` v0.2.34, `payment_manager` v0.4.15, `room_layout` v0.3.49, `agreement_manager` v0.3.7); los 3 restantes (`appointment_booking`, `chat_room`, `clinical_encounter`) despachados en vuelo tras layout v0.3.78 |
| B1 | `webtyp/layout`, `mjosefa-cms` | pieza de sesión (D5); la app borra `config/client.go` salvo su configuración, `loginscreen.go` y `html.go` | ⚪ diseño (gate de api-design) tras R |
| B2 | `mjosefa-cms`, `iam` | armado del servidor (D6): borran su `config/server.go` | ⚪ listo: `appkit` v0.0.4 publicado |
| B3 | `webtyp/access`, `staff_manager`, `mjosefa-cms` | primer administrador (D9) | ⚪ diseño, tras ACCESS_REUSE A3 |
| Z | `mjosefa-cms`, `iam` | aceptación: las dos apps usan la pieza; «Catálogo» abre (R3 cerrada: verificar con captura); `config/` queda en su configuración; captura revisada de cada pantalla; test que recorre el menú como admin y como usuario sin permisos (todo enlace visible abre algo no vacío) | ⚪ al final |

## Bloqueos

- 2026-10-29 · `view` importa `dom` desde R0c (`view.Screen.Component dom.Component`, `view/screen.go:42`): todo binario que importe la raíz de un módulo con presentadores (p. ej. `access`) arrastra `dom`; rompe `iam` (`tests/layering_test.go:66`, su `edge` no puede llevar UI) al subir a `access` v0.0.28. Falta decidir dónde vive `Screen` o cómo tipar `Component` sin `dom`.
- 2026-10-28 · Revisión y cierres: Antigravity (temporizador de 15 min). Todo lo necesario está en este master, en REACTIVE_RECORDS y en el `docs/PLAN*.md` de cada repo.
- 2026-10-27 · El menú cerrado por defecto deja sin pantallas los demos que crean `platformd.Platform` sin `CanView`: `access/web/browser/main.go` (su plan en vuelo debe pasar `Menu.CanView`).
- 2026-10-24 · Fallan en la cascada por causas ajenas a esta ola: `goflare` (assets del Worker) y `app` (recarga del navegador).

## Relación con otras olas

- ACCESS_REUSE: `access.EnsureRole` (alta por correo) y D9 tocan el mismo `config/auth.go`; el menú de `iam` llega con las pantallas de `access`.
- MVP_DEPLOYMENT_ROADMAP (mjosefa-cms): R2 viaja en el plan de `cfaccess` (su M1c); R y B vacían `config/`.
- REACTIVE_RECORDS: los 3 módulos de M en vuelo reciben su corrección G4; su F5 vive en `appkit`.
- BUILD_DESTINATIONS: M rompe todos los módulos, igual que su D18/D19; conviene hacerlo en la misma pasada.
