# La pieza «aplicación»: webtyp entrega la app armada y la app solo la configura — master plan
> **Status:** IN PROGRESS · 2026-10-27 · cerradas F4 (`layout` v0.3.75), T1 (`view` v0.6.45), `jwt` v0.1.29, A (`appkit` v0.0.4) y A2 (`layout` v0.3.76); Fase M en curso con Antigravity: 12 cerrados y publicados, 4 en vuelo de `codejob` (`cash_manager`, `clinical_encounter`, `chat_room`, `appointment_booking`). **R espera P10–P13 del usuario**; P12 urgente: 3 de esos planes borran pantallas

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

**P12 — Pantallas que no son una lista (URGENTE: M en vuelo).** `view.Screen` solo acepta un `Presenter`
(lista + formulario), y 3 planes de M lo resuelven **borrando la pantalla real**: `appointment_booking` cambia
el asistente de reserva por una lista (`docs/PLAN.md:107-160`), `chat_room` la conversación por la lista de grupos
(`docs/PLAN.md:138`), `clinical_encounter` pierde el selector de paciente de la agenda de hoy (`ui/browser.go:209-240`).
Precedentes: Django admin (`ModelAdmin` + vistas propias), ActiveAdmin (`register` + `register_page`), react-admin
(`<Resource>` + `<CustomRoutes>`), Filament (Resources + Pages): dos tipos de pantalla, mismo menú y permiso.
A) `view.Screen` suma una hoja `Component func() dom.Component` (exactamente uno de `Presenter`/`Component`);
`layout/app` la envuelve; `chatview` pasa de `layout` a `components`. Costo: `view` depende de `dom`. Borra `Browser()` restantes.
B) Esos módulos conservan `Browser()` con `layout` hasta otra ola (dos formas de declarar pantallas: el ledger queda +1).
**Recomiendo A y pausar ya esos 3 planes** (los lleva Antigravity): sin A, M publica la pérdida. Respuesta:

**P10 — Cómo sabe «me» qué puede ver el admin.** Una pantalla que falta en `config.Modules()`
(`mjosefa-cms/config/config.go:96`) queda oculta incluso al admin: el navegador no entiende «*», el servidor lo
expande contra esa lista (`auth/user.go:399`). D7 iba a armarla desde las rutas, pero `authority` nace antes de
montarlas (`config/auth.go:255`) y nunca recibe el router: necesitaría un setter.
A) D7: `authMod.UseRoutes(r)` tras `appkit.MountServer` + `rbac.AllowedActions` por recurso (D3).
B) `rbac.SubjectGrants(proyecto, usuario) ([]model.Grant, error)`; «me» manda las filas tal cual («*:crud» incluido) y
`ProfileDTO.Allows` usa `model.Grant.Matches` (`model/rbac.go:208`), la misma regla del servidor.
Precedentes de B: CASL (manda `manage all` al cliente), Casbin `GetImplicitPermissionsForUser`, Spring `GrantedAuthority`.
Ledger B: conceptos −2 (`Resources`, `AllowedActions`), arranque sin orden, ⇒ borra lista, `rbacResolver` y su test.
**Recomiendo B.** Costo: un permiso mal escrito en una pantalla ya no la oculta al admin (sí al resto); lo atrapa Z. Respuesta:

**P11 — El Explorador de API con el menú nuevo.** No es un `view.Screen`; si la app lo agrega aparte de
`layout/app.BuildMenu`, `CanView` lo oculta a todos (solo conoce permisos de lo que construyó, `layout/app/menu.go:44-53`).
Y `"api_explorer"` está escrito 3 veces (`appkit/server.go:24`, `mjosefa-cms/config/config.go:26`, `apiexplorer/explorer.go:94`).
A) `BuildMenu` lo agrega siempre con `router.ResourceIntrospection` + `model.Read` (la constante pasa de `appkit` a
`router`, dueño de `/_routes`); la app no escribe nada (D10). Precedentes: Laravel Telescope (gate `viewTelescope`),
Django `admindocs`, Spring Boot Actuator: lo que trae el framework se registra solo, con su permiso.
B) La app lo agrega como hoja con permiso (con P12-A sería un `view.Screen{Component: …}`).
**Recomiendo A.** Respuesta:

**P13 — Un menú sin regla de permisos muestra todo.** `platformd.Platform.CanView == nil` muestra todas las pantallas
(`layout/platformd/platformd.go:151,320`); `iam` pasa «siempre sí» (`iam/modules/panel/panel.go:25`). A) `nil` = no mostrar
nada (cerrado por defecto, **api-harness**); los demos pasan su función. B) Dejarlo: `BuildMenu` nunca deja `nil`.
**Recomiendo A**, en el mismo plan de `layout` que P11/P12. Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | Cada pantalla del MVP se explica con `platformd.Describe` (regla de ui-consistency) | G1, I3 |
| D2 | Un dibujo por concepto, desde `webtyp/icons` (Font Awesome Free 6.7.2 solid); ningún módulo dibuja su propio ícono genérico | I1, I2, I3 |
| D3 | `rbac.Service` implementa `AllowedActions` (`auth.ActionResolver`); las apps borran `rbacResolver` (lo reemplaza `SubjectGrants` si P10=B) | R1 |
| D5 (P3) | **Pieza de sesión del navegador** en `webtyp/layout` (pedir «me», mostrar login o alta del primer admin, cerrar sesión, bloqueo por inactividad), armada con `layout/login`, `platformd` y `components/usermenu`; la app le da nombre, logo y formularios. Búsqueda: no existe en `components` (solo `usermenu` con el botón «Cerrar Sesión») ni en `layout` (solo la pantalla visual `login`) | B1 |
| D7 (P1) | **El servidor arma solo la lista de permisos** leyendo los `Requires` de las operaciones montadas (`router.Router.Routes()`); `auth.Permissions` deja de recibir `Resources` y las apps borran su lista a mano (`config.Modules()`). Cómo: P10 | R2 |
| D9 (P5) | **Primer administrador repartido por dueño**: `webtyp/access` crea el usuario y le da el rol de administrador (la misma pieza que usa `iam` por correo); `staff_manager` registra el equipo desde el que entró y su ficha. `access` no puede hacer la parte del equipo: `webtyp` no conoce los módulos de `veltylabs` (la dependencia va al revés). La app solo conecta las dos | B3 |
| D11 (P6) | **Repo nuevo** que une las partes (servidor, sesión, permisos, primer admin); la app escribe solo su configuración. Las partes siguen en sus dueños; el repo nuevo solo las une. Nombre: **`appkit`** (D12) | A |
| D12 (P7) | El repo nuevo se llama **`webtyp/appkit`**: «el kit para armar una app» (precedente: AppKit/UIKit de Apple); `appsetup` choca con el alta del primer admin («setup», 42 usos) y `appconfig` con la carpeta `config/` y con AWS AppConfig | A |
| D13 (P8) | **El contrato de pantalla vive en `webtyp/view`** (`view.Screen`, `view.Lookup`, `Screen.Validate`, publicado v0.6.44): el módulo declara sus pantallas como datos —hoja con `Presenter` y `Permission` (ayuda = `view.WithDescription`) o grupo con `Children` y `Help`, sin permiso propio— y `appkit` las dibuja; ningún módulo importa `layout`, así que publicar `layout` ya no arrastra a los módulos. Sin repo nuevo de contratos: `view` y `model` ya cumplen ese papel | A2, M |
| D14 (P9) | **`appkit` servidor opera sobre `router.Router` (`//go:build !browser`)**: borra `httpd`, `orm`, `env`, `dev.go` y `prod.go`. Expone `MountServer(r router.Router, cfg ServerConfig) error` con `MCPTools []mcp.ToolName`, rpc, `/_routes` protegido con `Requires(ResourceAPIExplorer, model.Read)` y `Register(r)`. El transporte concreto vive en `webtyp/server` o `cloudflare/edge` | A |
| D15 | **El contrato de chasis vive en la raíz `webtyp/layout` y el armado del menú en `webtyp/layout/app`** (sin acoplar a `platformd`): `UIModule`, `GroupModule`, `Described`, `Notifier` suben a `package layout` (con alias en `platformd`), `crudview` y `apiexplorer` dejan de importar `platformd`, y `layout/app.BuildMenu` devuelve `[]layout.UIModule` + `CanView` para que el consumidor elija su layout (`platformd` o uno propio) | A2 |
| D10 | **Lo que toda app webtyp trae sin escribirlo**: login (red local por RUT, correo, passkey), roles y permisos con alta del primer admin, menú que muestra solo lo que puedes usar, ayuda «?» en cada pantalla, íconos, auditoría, explorador de API, servidor que niega todo por defecto, instalable (PWA). Cada hueco contra esta lista es una fase | todas |
| D6 (P4) | **El armado del servidor sirve a `mjosefa-cms` y a `iam`**: lo entrega `webtyp/server` (leer el entorno, `BuildServer`, negar todo por defecto, montar rpc + MCP + `/_routes`); las dos apps borran su copia | B2 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| I1 | `webtyp/icons` | glifos `laptop`, `location`, `network`, `terminal` | 🟢 cerrada v0.0.33 |
| F1 | `webtyp/dom` | la señal es dueña de los hijos de `BindChildren` (igual en servidor y navegador; `Child(...)` al lado se rechaza) y las filas sin `Key` del primer dibujo se reemplazan al actualizar | 🟢 cerrada v0.13.29 |
| F2 | `webtyp/layout`, `form`, `dbsync`, `pwa` | dejaron de copiar a mano los hijos de la señal (`platformd` 5 sitios, opciones de `form`, bandeja de `dbsync`); `pwa` al `actionbutton.Primary` tipado; test de inicio con «?» dibujado una vez | 🟢 cerradas: layout v0.3.70, form v0.4.52, dbsync v0.3.10, pwa v0.2.13 |
| F3 | `webtyp/components` | `datatable`: las filas van en la señal | 🟢 cerrada v0.8.51 |
| G1 | `veltylabs/service_catalog` | `platformd.Describe` en el grupo, Servicios y Especialidades; verificado en `mjosefa-cms` con captura | 🟢 cerrada v0.6.4 |
| I2 | `webtyp/layout` (`apiexplorer`) | ícono `terminal` desde `webtyp/icons` | 🟢 cerrada v0.3.70 (con F2) |
| F4 | `webtyp/layout` (`platformd`) | el «?» del título se ve como texto suelto pequeño, sin forma de botón (captura 2026-10-24): darle estilo de botón con la receta de `widget/style` | 🟢 cerrada v0.3.75 |
| I3 | `veltylabs/device_manager` | Equipos `laptop`, Zonas `location`, Interfaces `network`; borra el path de la manzana de `ui/svg.go`; `platformd.Describe` en las 3 pantallas; demo pasa a `web/browser/main.go` | 🟢 cerrada v0.2.71 |
| R0 | `view`, `layout`, `router`, `components` | ya hechos: permiso obligatorio por pantalla (`view` v0.6.44), grupo sin permiso propio que abre su primer hijo visible (`layout` v0.3.76), `/_routes` con permiso siempre (`appkit` v0.0.4). Falta: hoja `Component` (P12), explorador solo (P11), `CanView` nil cerrado (P13) | ⚪ espera P11–P13 |
| R1 | `rbac` | `SubjectGrants` o `AllowedActions` (P10); `docs/PLAN_ALLOWED_ACTIONS.md` viejo se reescribe | ⚪ espera P10 |
| R2 | `auth` (etapa extra del `docs/PLAN.md` de `cfaccess`, M1c; ya desbloqueado: `jwt` v0.1.29) | «me» sin lista a mano (D7/P10) | ⚪ espera P10 y R1 |
| R3 | `mjosefa-cms` | `modules.Screens` + `layout/app.BuildMenu`; borra `rbacResolver`, `Modules()`, `ResourceAPIExplorer`, `tests/rbac_modules_test.go`; «Catálogo» abre. `iam` entra con las pantallas de `access` (ACCESS_REUSE), no aquí | ⚪ tras R0–R2 |
| B1 | `webtyp/layout`, `mjosefa-cms` | pieza de sesión (D5); la app borra `config/client.go` salvo su configuración, `loginscreen.go` y `html.go` | ⚪ diseño (gate de api-design) tras R |
| B2 | `appkit` (tramo 1), luego `mjosefa-cms` e `iam` borran su `config/server.go` | armado del servidor (D6) | ⚪ apps adoptan al publicar A |
| B3 | `webtyp/access`, `staff_manager`, `mjosefa-cms` | primer administrador (D9) | ⚪ diseño, tras ACCESS_REUSE A3 |
| A | `webtyp/appkit` (creado 2026-10-24 con `gonew`) | tramo 1: `MountServer` sobre el contrato `router.Router` (`//go:build !browser`, sin `httpd`, sin `orm`, sin `dev.go`/`prod.go`, con `MCPTools`, default `DenyAll`, y tests con `router/mock`) | 🟢 cerrada v0.0.4 |
| A2 | `webtyp/layout` (`layout` + `layout/app`) | promover contratos (`UIModule`, `GroupModule`, `Described`, `Notifier`) a la raíz `layout` (alias en `platformd`, desacoplar `crudview` y `apiexplorer`) y crear `layout/app.BuildMenu`: `[]view.Screen` → `[]layout.UIModule` + `CanView` sin acoplar a `platformd` | 🟢 cerrada v0.3.76 |
| M | los ~14 módulos con `ui/` | reemplazan `Browser(...)` por `Screens(caller) []view.Screen` y quitan `layout` de su `go.mod`, en el mismo cambio (una sola forma) | [Antigravity · 2026-10-27 12:36] 🟡 12 cerrados y publicados (`device_manager` v0.2.72, `service_catalog` v0.6.11, `network_manager` v0.1.84, `staff_manager` v0.1.153, `pharmacy_manager` v0.1.36, `alert_manager` v0.1.16, `patient_directory` v0.1.62, `business_calendar` v0.5.16, `supply_manager` v0.2.34, `payment_manager` v0.4.15, `room_layout` v0.3.49, `agreement_manager` v0.3.7); 4 en vuelo de `codejob` (`cash_manager`, `clinical_encounter`, `chat_room`, `appointment_booking`) |
| T1 | `webtyp/view` | `lang.json` con los 8 idiomas llenos | 🟢 cerrada v0.6.45 |
| Z | `mjosefa-cms`, `iam` | aceptación: las dos apps usan la pieza; `config/` queda en su configuración; captura revisada de cada pantalla; test que recorre el menú como admin y como usuario sin permisos (todo enlace visible abre algo no vacío) | ⚪ al final |

## Bloqueos

- 2026-10-27 · M: los planes en vuelo de `appointment_booking`, `chat_room` y `clinical_encounter` borran su pantalla no-lista (P12); no deben publicarse así. Los lleva Antigravity (marca 12:36): avisarle.
- 2026-10-24 · «Catálogo» sigue con el escenario en blanco en `mjosefa-cms` hasta que cierre R; no se agrega a mano a `config.Modules()` (sería otro parche).
- 2026-10-24 · Fallan en la cascada por causas ajenas a esta ola: `goflare` (assets del Worker) y `app` (recarga del navegador); no reciben las versiones nuevas hasta arreglarse. (`html` arreglado: v0.0.30.)

## Relación con otras olas

- MODULE_GUIDANCE (cerrada): D1 aplica su regla a las pantallas del MVP.
- ACCESS_REUSE: `access.EnsureRole` (alta por correo) y P5 tocan el mismo `config/auth.go`.
- MVP_DEPLOYMENT_ROADMAP (mjosefa-cms): M2 ejecutada en local; R y B vacían su `config/`.
- BUILD_DESTINATIONS: D4 rompe todos los módulos, igual que su D18/D19; conviene hacerlo en la misma pasada.
