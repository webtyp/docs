# La pieza «aplicación»: webtyp entrega la app armada y la app solo la configura — master plan
> **Status:** IN PROGRESS · 2026-11-02 · B1 y B2 publicados (`appkit` v0.1.2, `server` v0.2.106, `app` v0.7.49, `mjosefa-cms` v0.0.107); quedan B3 (primer administrador en `access`, tras ACCESS_REUSE A3) y Z (aceptación con `iam`)

**Disparador (mjosefa-cms, 2026-10-24):** menú roto e íconos repetidos, y `config/` con 1.407 líneas
que toda app con login repite (`iam` también). **Regla del dueño:** webtyp entrega la pieza lista; la
app solo la configura; un módulo se configura con un constructor y listo.

**Hoy (2026-11-02):** `mjosefa-cms` tiene 1.554 líneas de Go fuera de `ui`/tests — `config/client.go` 355,
`auth.go` 292, `server.go` 219, `modules/server.go` 131, `loginscreen.go` 85, `html.go` 76,
`migrate.go` 47 + `cmd/migrate/main.go` 41 —; `iam/config` 2.027. `appkit` solo trae `MountServer`
(`appkit/server.go`). `web/browser/main.go` adivina un método (`p.(interface{ Init(Ctx) })`) y llama
`Init(nil)`. `modules/tenant.go` guarda constantes de instalación que son configuración. Cada módulo
tiene su propio `New(db, Deps{IDs, Publisher, TenantID, …})` y la app escribe a mano los adaptadores
entre módulos (`specialtiesAsCategories`, `staffAsOccupants` en `modules/server.go`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D21 (P15) | **Una app es una declaración**: `config/app.go` con `appkit.App{Name, Logo, Tenant, Timezone, Login, Modules}`; `web/browser/main.go` = `appkit.Browser(config.App)`, `web/server/main.go` (generado) = `appkit.Server(config.App)`. Se borran `client.go`, `server.go`, `auth.go`, `loginscreen.go`, `html.go`, `modules/`, `tenant.go` y el `Init(nil)` adivinado | B1, B2 |
| D22 (P16, P19) | **Un módulo es un valor**: cada módulo exporta `var Module = appkit.Module{Name, Migrate, Build}` en su subpaquete **`server/`** (P19: en la raíz sería un ciclo con `migrate/`; `ui/` = navegador, `server/` = servidor); `appkit.Deps` trae lo común (base, ids, eventos, tenant, zona horaria). Las pantallas siguen en `ui.Screens` (B1) | B2e + los 18 módulos |
| D23 (P17) | **Cables entre módulos por interfaz**: el que necesita declara la interfaz (`roomlayout.CategorySource`); `appkit` entrega el único módulo construido que la cumple y falla al arrancar si falta o hay dos; se borran los adaptadores de la app | B2 |
| D25 (P20) | **Un valor por binario**: `config/app.go` (`appkit.App`, `!browser`) y `config/browser.go` (`appkit.Browser{Brand, Screens, DefaultScreen, IdleSeconds}`, `browser`); lo compartido es `config/brand.go` (`appkit.Brand{Name, LogoMark}`); `web/browser/main.go` = `appkit.RunBrowser(config.Browser)`; `html.go` = `appkit.LoginPages`/`LoadingHTML` | B1 |
| D24 (P18) | **Una sola forma de migrar, para toda app (servidor o Worker)**: `migrate/main.go` en la raíz, generado por webtyp (la app no lo escribe): `appkit.Migrate(config.App)` recorre `App.Modules` y corre cada `Module.Migrate` (los `migrate/` de los módulos no cambian). Lo ejecuta siempre ese binario: `webtyp dev` al arrancar (reemplaza la sincronización por `Ormc.SetSyncer`, para no dejar dos caminos), el supervisor de SELFHOST tras el `pg_dump` (ajusta su D14: `appkit.Server` no migra) y el `pre-deploy` del Worker. Se borran `config/migrate.go` y `cmd/migrate/`; el paso legado `device.ip` pasa a `device_manager/migrate` | B2, `app`, SELFHOST, `device_manager` |
| D-TUI (de TUI_SIGNAL, cerrada 2026-11-02) | La pantalla de primera configuración que `mjosefa-cms` arma en `config/client.go` (subtítulo «First setup: create the administrator», hoy la única clave de su `config/lang.json`; la TUI lo avisa como línea `LANG`) pasa a la librería que entrega el login | `layout/login` o `access` |
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
| M | los 16 módulos con `ui/` | `Browser(...)` → `Screens(caller) []view.Screen` | 🟢 cerrada (verificado 2026-11-02: ningún módulo tiene `Browser()`; los 3 últimos con pantallas propias, D20) |
| B1 | `webtyp/appkit` + `mjosefa-cms` | `appkit.Brand`, `LoginForm`, `LoginScreen`, `LoginPages`, `LoadingHTML` (`browser.go`) y `appkit.Browser` + `RunBrowser` (`run_browser.go`, wasm), textos en `lang.json` (es); `mjosefa-cms` borra `client.go`, `loginscreen.go` y el `Init(nil)`; Go de la app 1.554 → 641 líneas | 🟢 appkit v0.1.2, mjosefa-cms v0.0.107 |
| B2a | `webtyp/appkit` | `App`, `Module`, `Deps`, `Find`, `BuildServer`, `Server`, `Steps`, `RunMigrations`, `Migrate` (D6, D22–D24) | 🟢 cerrada v0.1.0 (PR #6 de Jules, revisado) |
| B2b | `webtyp/server` | generador: `config/` con `var App` → `web/server/main.go` = `appkit.Server(config.App)` y `migrate/main.go` = `appkit.Migrate(config.App)` (`server.HasApp`) | 🟢 server v0.2.106 |
| B2c | `webtyp/app` + `server` | `webtyp dev` corre `go run ./migrate` (línea `MIGRATE`) con `appkit.App`; mensaje «config declares App (appkit)»; `WATCH WATCH` duplicado corregido | 🟢 app v0.7.49; falta ver la línea `MIGRATE` con una base local |
| B2d | `device_manager` + `mjosefa-cms` | paso legado `device.ip` en `device_manager/migrate.LegacyDeviceIP`; `mjosefa-cms` borra `config/legacy_device_ip.go` | 🟢 `device_manager` v0.2.95; `mjosefa-cms` local sin publicar |
| B2e | `service_catalog`, `staff_manager`, `room_layout`, `device_manager` | `server/server.go` con `var Module`; `CategoryOptions`, `OccupantOptions`, `FindByIP(ip)` en el módulo dueño; `appkit` v0.1.1 suma `Auth.Operations` | 🟢 device_manager v0.2.96, service_catalog v0.6.40, staff_manager v0.1.181, room_layout v0.3.76, appkit v0.1.1. Los otros 14 módulos, cuando una app los use |
| B2f | `mjosefa-cms` | `config/app.go`; `auth.go` → `buildAuth`; borra `server.go`, `migrate.go`, `legacy_device_ip.go`, `cmd/`, `modules/server.go`, `tenant.go`, `prod.go`, `dev.go`; `migrate/main.go`; pruebas sobre `buildTestApp` | 🟢 mjosefa-cms v0.0.107 |
| B3 | `webtyp/access`, `staff_manager`, `mjosefa-cms` | primer administrador (D9) | ⚪ diseño, tras ACCESS_REUSE A3 |
| Z | `mjosefa-cms`, `iam` | aceptación: las dos apps usan la pieza; «Catálogo» abre (R3 cerrada: verificar con captura); `config/` queda en su configuración; captura revisada de cada pantalla; test que recorre el menú como admin y como usuario sin permisos (todo enlace visible abre algo no vacío) | ⚪ al final |

## Bloqueos

- 2026-10-29 · `view` importa `dom` desde R0c (`view.Screen.Component dom.Component`, `view/screen.go:42`): todo binario que importe la raíz de un módulo con presentadores (p. ej. `access`) arrastra `dom`; rompe `iam` (`tests/layering_test.go:66`, su `edge` no puede llevar UI) al subir a `access` v0.0.28. Falta decidir dónde vive `Screen` o cómo tipar `Component` sin `dom`.
- 2026-10-28 · Revisión y cierres: Antigravity (temporizador de 15 min). Todo lo necesario está en este master, en REACTIVE_RECORDS y en el `docs/PLAN*.md` de cada repo.
- 2026-10-27 · El menú cerrado por defecto deja sin pantallas los demos que crean `platformd.Platform` sin `CanView`: `access/web/browser/main.go` (su plan en vuelo debe pasar `Menu.CanView`).
- 2026-10-24 · Fallan en la cascada por causas ajenas a esta ola: `goflare` (assets del Worker) y `app` (recarga del navegador).

## Relación con otras olas

- ACCESS_REUSE: `access.EnsureRole` (alta por correo) y D9 tocan el mismo `config/auth.go`; el menú de `iam` llega con las pantallas de `access`.
- SELFHOST_DEPLOY: D24 cambia quién migra en su D14 (el binario `migrate`, no el servidor al arrancar).
- MVP_DEPLOYMENT_ROADMAP (mjosefa-cms): R2 viaja en el plan de `cfaccess` (su M1c); R y B vacían `config/`.
- REACTIVE_RECORDS: los 3 módulos de M en vuelo reciben su corrección G4; su F5 vive en `appkit`.
- BUILD_DESTINATIONS: M rompe todos los módulos, igual que su D18/D19; conviene hacerlo en la misma pasada.
