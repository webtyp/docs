# La pieza «aplicación»: webtyp entrega la app armada y la app solo la configura — master plan
> **Status:** IN PROGRESS · 2026-10-27 · sin preguntas abiertas. Despachados (Jules): R0a `router`, R0c `view`, R1 `rbac`, R2 `auth`. Escritos en cola: R0b (`appkit`, `layout`), R3 (`mjosefa-cms`). Revisión: Antigravity. Se despacha junto con REACTIVE_RECORDS

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
| D16 (P11) | **El Explorador de API entra solo al menú**: `layout/app.BuildMenu` lo agrega siempre, con permiso `router.ResourceIntrospection` + `model.Read`. El valor sigue siendo `"api_explorer"` (las filas de permisos guardadas lo usan). La constante vive en `router`, dueño de `/_routes`, junto a `IntrospectionPath`; `appkit` borra `ResourceAPIExplorer`, `apiexplorer` la usa como `ModelName` y `mjosefa-cms` borra la suya | R0a, R0b, R3 |
| D17 (P13) | **Menú cerrado por defecto**: `platformd.Platform.CanView == nil` no muestra ninguna pantalla (hoy las muestra todas, `platformd.go:151,320`); los demos y tests pasan su función explícita | R0b | `webtyp/appkit`, `webtyp/layout` | D16, D17, D19 en `layout` | 📝 `appkit/docs/PLAN.md` (v0.0.5) despachar cuando salga router v0.4.9; `layout/docs/PLAN.md` (v0.3.77) cuando salgan router v0.4.9 y view v0.6.46 |
| D18 (P10) | **«me» manda los permisos tal como están guardados** («*:crud» incluido): `rbac.SubjectGrants(proyecto, usuario) ([]model.Grant, error)`; `auth.Permissions{Grants, ProjectID}` sin `Resources` ni `Resolver`; `ProfileDTO.Can(recurso, acción)` evalúa cada entrada con `model.Grant.Matches` (la misma regla que `model.AnyGrant` en el servidor). Se borran `auth.ActionResolver`, `rbacResolver`, `config.Modules()`, `tests/rbac_modules_test.go` y `rbac/docs/PLAN_ALLOWED_ACTIONS.md` | R1, R2, R3 |
| D13 (P8) | **El contrato de pantalla vive en `webtyp/view`** (`view.Screen`): el módulo declara sus pantallas como datos y `layout/app` las dibuja; ningún módulo importa `layout` | M | los 16 módulos con `ui/` | `Screens(...) []view.Screen`, sin `platformd` | 🟡 12 publicados. Los 4 que faltan van **en el mismo plan que F4 de REACTIVE_RECORDS**: `cash_manager` (plan reescrito: sin `Item()` ni lister a mano), `appointment_booking`, `chat_room`, `clinical_encounter` (planes nuevos: pantallas propias como `Component`, D20). Se despachan cuando salgan model v0.2.7, view v0.6.47 y (los 3 últimos) layout v0.3.78 |
| D19 (P12) | **Pantallas propias**: `view.Screen` suma `Component dom.Component`; `Validate` exige uno y solo uno de `Presenter`/`Component`; `layout/app.BuildMenu` lo pone en el menú con su permiso. `view` importa `dom` | R0c, R0b, M |
| D20 (P14) | **Pantallas propias armadas con piezas de `layout`**: `chatview` se queda en `layout` (usa `rightpanel`, y `layout` importa `components`: mudarlo crea un ciclo). Una pantalla `Component` puede armarse con `layout/chatview`, `layout/crudview` o `layout/rightpanel` (`chat_room`, `clinical_encounter`, `appointment_booking`); ese módulo importa esas piezas y publica cuando `layout` cambie (costo aceptado). Ningún módulo importa `layout/platformd` | M |
| D5 (P3) | **Pieza de sesión del navegador** en `webtyp/layout` (pedir «me», mostrar login o alta del primer admin, cerrar sesión, bloqueo por inactividad), armada con `layout/login`, `platformd` y `components/usermenu`; la app le da nombre, logo y formularios. Búsqueda: no existe en `components` (solo `usermenu`) ni en `layout` (solo la pantalla visual `login`) | B1 |
| D6 (P4) | **El armado del servidor sirve a `mjosefa-cms` y a `iam`**: los contratos los monta `appkit.MountServer` (rpc, MCP, `/_routes`, negar todo por defecto) y el transporte lo pone `webtyp/server` (leer el entorno, escuchar); las dos apps borran su `config/server.go` | B2 |
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
| R0b | `webtyp/layout`, `webtyp/appkit` | D16, D17 (y D19 en `layout`) | 🟢 cerradas: appkit v0.0.5, layout v0.3.77 |
| R0c | `webtyp/view` | D19: hoja `Component` | 🟢 cerrada v0.6.46 |
| R1 | `webtyp/rbac` | D18: `SubjectGrants` | 🟢 cerrada v0.0.22 |
| R2 | `webtyp/auth` | D18 (etapas 7–8 del plan de `cfaccess`) | 🟢 cerrada v0.0.80 |
| R3 | `mjosefa-cms` | menú desde `ui.Screens` + `BuildMenu`; `profile.Can`; `Permissions{Grants}`; borra `Modules()`, `rbacResolver`, `ResourceAPIExplorer`, explorador a mano, `rbac_modules_test.go`; «Catálogo» abre | 🟢 cerrada v0.0.66 |
| M | los 16 módulos con `ui/` | `Browser(...)` → `Screens(caller) []view.Screen`, sin `layout` en `go.mod` | 🟡 13 publicados (`device_manager` v0.2.72, `service_catalog` v0.6.12, `network_manager` v0.1.84, `staff_manager` v0.1.153, `pharmacy_manager` v0.1.36, `alert_manager` v0.1.16, `patient_directory` v0.1.62, `business_calendar` v0.5.16, `supply_manager` v0.2.34, `payment_manager` v0.4.15, `room_layout` v0.3.49, `agreement_manager` v0.3.7); los 3 restantes (`appointment_booking`, `chat_room`, `clinical_encounter`) despachados en vuelo tras layout v0.3.78 |
| B1 | `webtyp/layout`, `mjosefa-cms` | pieza de sesión (D5); la app borra `config/client.go` salvo su configuración, `loginscreen.go` y `html.go` | ⚪ diseño (gate de api-design) tras R |
| B2 | `mjosefa-cms`, `iam` | armado del servidor (D6): borran su `config/server.go` | ⚪ listo: `appkit` v0.0.4 publicado |
| B3 | `webtyp/access`, `staff_manager`, `mjosefa-cms` | primer administrador (D9) | ⚪ diseño, tras ACCESS_REUSE A3 |
| Z | `mjosefa-cms`, `iam` | aceptación: las dos apps usan la pieza; `config/` queda en su configuración; captura revisada de cada pantalla; test que recorre el menú como admin y como usuario sin permisos (todo enlace visible abre algo no vacío) | ⚪ al final |

## Bloqueos

- 2026-10-27 · **Revisión: la coordina Antigravity** (temporizador de 15 min, regla de MASTER_PLANS.md). Este chat se cerró; todo lo necesario está en este master, en REACTIVE_RECORDS y en el `docs/PLAN*.md` de cada repo.
- 2026-10-27 · D17 (menú cerrado por defecto) deja sin pantallas los demos que crean `platformd.Platform` sin `CanView`: `access/web/browser/main.go` (sin plan todavía) y los de los 3 módulos de M (sus planes ya lo corrigen).
- 2026-10-24 · «Catálogo» sigue con el escenario en blanco en `mjosefa-cms` hasta que cierre R3; no se agrega a mano a `config.Modules()`.
- 2026-10-24 · Fallan en la cascada por causas ajenas a esta ola: `goflare` (assets del Worker) y `app` (recarga del navegador).

## Relación con otras olas

- ACCESS_REUSE: `access.EnsureRole` (alta por correo) y D9 tocan el mismo `config/auth.go`; el menú de `iam` llega con las pantallas de `access`.
- MVP_DEPLOYMENT_ROADMAP (mjosefa-cms): R2 viaja en el plan de `cfaccess` (su M1c); R y B vacían `config/`.
- REACTIVE_RECORDS: se despacha en la misma pasada; su F4 va en el plan de M de cada módulo y su F5 en `appkit`.
- BUILD_DESTINATIONS: M rompe todos los módulos, igual que su D18/D19; conviene hacerlo en la misma pasada.
