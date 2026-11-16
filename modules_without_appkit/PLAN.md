# Módulos sin `appkit` y `appkit` sin concretos — plan

> Implementa D62, D63, D65–D71 de [ARCHITECTURE_MASTER_PLAN.md](../ARCHITECTURE_MASTER_PLAN.md). Se ejecuta **en local**
> por [EXECUTION_ORDER paso 17](../EXECUTION_ORDER_MASTER_PLAN.md) (su protocolo: marca, `replace`, `gotest`, cerrar la
> fila). Nada se publica antes de la puerta 🔵 18. Esta carpeta se borra en esa puerta.

**Estado: 🔵 aprobado por el dueño (2026-11-16).** Antes de escribir las etapas con código se responden P57, P58 y P59 del master
(tocan §3). Revisión de coherencia hecha el mismo día: lo corregido está en «Correcciones de la revisión», al final.

## Qué cambia, en una frase

Solo la app importa `appkit`; `appkit` arma la app (módulos, ingreso, operaciones) con lo que la app le pasa y no elige
ningún concreto; los módulos y `access` ya no saben que `appkit` existe.

## Búsqueda en el ecosistema

- La firma del módulo ya está en `model`: `model.ModuleDefinition` (`model/module_definition.go:7`) → **reusar** (D65).
- Cada módulo ya exporta `New(db *orm.DB, d Deps)`, `Deps`, sus interfaces y `ui.Screens` → **reusar**; `module/` sobra.
- `staff_manager` ya declara lo que necesita (`staff_manager/interfaces.go:3`, `DeviceReader`, `DeviceRegistrar`);
  `devicemanager.IPLocator` (`device_manager/locator.go:8`) ya busca por IP → **renombrar y extender** a `TenantDevices` (D63).
- Contrato de eventos: `events.Broker` (`events/events.go:65`); implementación en proceso: `events/mock.Broker` → la app la pasa.
- Ids: `model.IDGenerator`; implementación `unixid.NewUnixID() (*UnixID, error)` → la app la pasa.
- Servidor HTTP: la skill **isomorphic-libraries** ya dice «un kit nunca crea el servidor HTTP: lo crea el `main`»; hoy
  `appkit/serve.go:207` lo crea igual → se corrige, el `main` generado (`server/maingen.go`) lo crea.
- Transporte: `rpc.Mount(r, prefix, log, public, pub, modules...)` (`rpc/mount.go:178`), `rpc.NewCaller(origin)`
  (`rpc/caller.go:22`), `mcp.NewServer` → la app los elige.
- **Nuevo:** ningún repo. Dos tipos chicos: `rpc.JSON` y `mcp.Tools` (D70), cada uno cumple una interfaz que declara `appkit`.
  Nada nuevo en `model` ni `orm`.

## Design gate

### 1. Prior art (enlaces no leídos en esta sesión: «sin verificar»)

Uber Fx (https://pkg.go.dev/go.uber.org/fx) y Google Wire (https://github.com/google/wire): las librerías exportan
constructores; la **app** decide las implementaciones y las conecta. Django (`INSTALLED_APPS`,
https://docs.djangoproject.com/en/stable/ref/applications/): la app lista sus piezas en orden y ninguna pieza importa al
proyecto. Seguimos la regla común; diferimos en que conectamos a mano en `config/`, sin reflexión ni genéricos (TinyGo,
skill **api-harness**).

### 2. Nombres (prueba del novato)

| Nombre | Se lee como | Reemplaza |
|---|---|---|
| `config/app.go` · `config/backend.go` · `config/browser.go` | qué app es · lo que corre en el servidor y el Worker · lo que corre en el navegador (la etiqueta se llama `browser`) | `config/server.go`, `config/browser.go` (vacío) de hoy |
| `appkit.App{Name, TenantID, Timezone, Countries, SignIn, Modules, Log}` (`DefaultDocument`: P59) | la app, igual en los dos lados | `App.Brand`, `App.Server`, `[]appkit.Module` |
| `appkit.Backend{Database, NewIDs, Events, API, Agents, Public, Routes, Modules}` | lo que elige el backend y cómo se construye cada módulo | `appkit.AppServer`, `ModuleServer` |
| `appkit.BackendModule{Name, Build, DefaultGrants}` · `appkit.BrowserModule{Name, Screens}` | la parte de backend · la parte de navegador de un módulo | `appkit.Module{Server, Browser}` |
| `appkit.Browser{Caller, NewIDs, Modules}` · `appkit.RunBrowser` | lo que usa el navegador · arrancar el navegador | `appkit.Browser(app, layout)` |
| `appkit.SignInByTrustedIP` | «la gente entra por IP de confianza» | `access.Module` en la lista (D68) |
| `appkit.FindBuiltModule(d, name)` | «busca el módulo ya construido con ese nombre» | `appkit.Find[T]` (D69) |
| `rpc.JSON{}` · `mcp.Tools{Allowed}` | las operaciones como JSON por HTTP · las que un agente puede llamar | `rpc.Mount`/`mcp.NewServer` dentro de `appkit` (D70) |
| `devicemanager.TenantDevices` | «los equipos de un tenant» | `IPLocator` + el tipo escondido `served` (D63) |

### 3. Montar una app, paso a paso — lo que el dueño revisa

Cada parte va en el archivo de su etiqueta; la lista de módulos se escribe **una vez**; ningún valor existe vacío en la
otra etiqueta (D71). Ejemplo: `mjosefa-cms`.

**Paso 1 — Qué app es** · `config/app.go` (sin etiqueta: lo leen el servidor, el Worker y el navegador)

```go
var App = appkit.App{
	Name:     "Mjosefa",                // el logo es docs/logo/logo.png (PNG), no un campo (D66)
	TenantID: "mjosefa-cms",
	Timezone: "America/Santiago",
	Countries:       country.Catalog{ /* el país por defecto y los que van primero (paso 14) */ },
	SignIn:   appkit.SignInByTrustedIP, // cómo entra la gente; sin esto nadie entra y toda operación se niega (D68)
	Modules: []model.ModuleDefinition{  // los módulos, una vez y en orden: quien provee a otro va antes (D65, D71)
		servicecatalog.Definition, devicemanager.Definition, staffmanager.Definition, businesscalendar.Definition,
		roomlayout.Definition, patientdirectory.Definition, appointmentbooking.Definition, clinicalencounter.Definition,
	},
	Log: func(a ...any) { fmt.Println(a...) },
}
```

**Paso 2 — Lo que corre en el servidor** · `config/backend.go` (`//go:build !browser`: servidor nativo y Worker)

```go
var Backend = appkit.Backend{
	Database: postgres.Open,                                                  // qué base de datos
	NewIDs:   func() (model.IDGenerator, error) { return unixid.NewUnixID() }, // cómo se crean los ids
	Events:   &eventsmock.Broker{},                                           // los avisos entre módulos, en el mismo proceso
	API:      rpc.JSON{},                                                     // las operaciones como JSON por HTTP (D70)
	Agents:   mcp.Tools{Allowed: MCPTools()},                                 // las que un agente puede llamar (D70)
	// Public (acciones sin sesión) y Routes (rutas propias de la app) siguen como hoy; mjosefa-cms no usa ninguna
	Modules: []appkit.BackendModule{                                          // cómo se construye cada módulo
		{Name: devicemanager.Definition.Name, Build: func(d appkit.Deps) (model.ModuleNaming, error) {
			return devicemanager.New(d.DB, devicemanager.Deps{IDs: d.IDs, Publisher: d.Events, TenantID: d.TenantID})
		}},
		{Name: staffmanager.Definition.Name, Build: buildStaff}, // necesita los equipos: abajo
		// … un elemento por módulo de App.Modules; si falta uno, appkit lo dice al arrancar
	},
}

// buildStaff le pasa a staff_manager los equipos de device_manager, sin que uno importe al otro (D63, D69).
func buildStaff(d appkit.Deps) (model.ModuleNaming, error) {
	m, err := appkit.FindBuiltModule(d, devicemanager.Definition.Name)
	if err != nil {
		return nil, err
	}
	dm, ok := m.(*devicemanager.Module)
	if !ok {
		return nil, errDevicesType // config/errors.go: «device_manager no construyó un *devicemanager.Module»
	}
	devices := devicemanager.TenantDevices{Devices: dm, TenantID: d.TenantID}
	return staffmanager.New(d.DB, staffmanager.Deps{IDs: d.IDs, Publisher: d.Events, TenantID: d.TenantID,
		ValidateRUT: input.NormalizeRUT, Devices: devices, Registrar: devices})
}
```

**Paso 3 — Lo que corre en el navegador** · `config/browser.go` (`//go:build browser`)

```go
var Browser = appkit.Browser{
	Caller: rpc.NewCaller(""),                                                // cómo habla el navegador con el servidor
	NewIDs: func() (model.IDGenerator, error) { return unixid.NewUnixID() },
	Modules: []appkit.BrowserModule{ // las pantallas de cada módulo; toda ui.Screens recibe view.BrowserDeps, que suma IDs (D71)
		{Name: devicemanager.Definition.Name, Screens: devicesui.Screens},
		{Name: staffmanager.Definition.Name, Screens: staffui.Screens},
		// …
	},
}
```

**Paso 4 — Arrancar** · cada `main` recibe la parte de su etiqueta

```go
// web/browser/main.go — una línea, la escribe la app
func main() { appkit.RunBrowser(config.App, config.Browser, layoutapp.New(devtools.Menu())) }

// web/server/main.go — generado por server/maingen.go, la app no lo toca: crea el servidor HTTP y lee DATABASE_URL
built, err := appkit.Build(config.App, config.Backend, os.Getenv("DATABASE_URL"), config.Schema)
srv := httpd.New(httpd.Config{Port: port, PublicDir: publicDir, Gzip: true, Health: true, Logger: config.App.Log,
	Authn: built.Authn, Authorize: built.Authorize, Policy: built.Policy})
err = built.Mount(srv.Router(), devtools.Server(config.Schema)...)
log.Fatal(srv.ListenAndServe())
```

**Puntos a confirmar:**
1. `externalcalls` y `backoff` **no** son concretos (lógica sobre la base, como `auth`/`rbac`): se quedan en `appkit`.
2. El servidor HTTP lo crea el `main` generado en dos pasos (`Build` → `httpd.New` → `Mount`), porque `httpd.Config`
   necesita `Authn`/`Authorize`/`Policy`, que salen del ingreso que arma `appkit`.
3. `NewIDs` aparece en backend y en navegador: cada lado crea sus ids (hoy la pantalla de ingreso crea los suyos con
   `unixid` y `panic`, `access/module/browser.go:91`).
4. El Worker (`web/edge/`) y la etiqueta `!browser`: ver P57 del master.

### 4. ¿Cuál evita el acoplamiento y es más mantenible? ¿Dónde va cada cosa?

- **Módulos:** dependen solo de sus interfaces, `model`, `orm`, `view`. Sin `module/`, sin `appkit` (D62, D65).
- **`access`:** solo el puente usuarios ↔ roles por interfaces (`Deps.Users`, `Deps.Roles`). Sin `appkit` (D68).
- **`appkit`:** el armado común a toda app (orden de módulos, ingreso, montaje, colas de llamadas externas), sin elegir
  concretos: los recibe (D67). Así ninguna librería lo importa y ninguna app repite el ingreso.
- **App (`config/` + `main` generado):** elige base, ids, eventos, transportes y servidor HTTP; conecta módulos entre sí.
- **Etiquetas:** lo compartido sin etiqueta, el backend en `!browser`, el navegador en `browser`; el código de backend nunca alcanza el WASM (D71, skill **wasm**).
- Lo que se descartó y por qué está en las decisiones del master (P49 B, P51 A/B, P54 B, P55 A, P56 B/C).

### 5. Qué se borra

`module/` en 8 módulos y en `access` · `webtyp.com/appkit` de 9 `go.mod` · `appkit.Find`, `ErrManyProviders`,
`appkit.Module`, `ModuleServer`, `AppServer`, `Server`, `BuildServer`, `MountServer`, `ServerConfig`, `ServerEnv`,
`processEnv`, `Infra` (su único uso era el ingreso, D68), `AppServer.Grants` (`appkit` arma `rbac` él mismo, D68) · `appkit` deja de importar `unixid`, `events/mock`, `server/httpd`, `rpc`, `mcp`, `env` · `view.Brand` y
`App.Brand` · `config/server.go` y el `config/browser.go` vacío de las apps · `devicemanager.IPLocator` y `served` · el
helper `optional` de `clinical_encounter` · los nombres viejos `appkit.ModuleBrowser|BrowserDeps|EntryScreens|Brand`.

### Cuentas

```
Conceptos que aprende quien monta una app   −4 (module/, Brand, Find[T], AppServer) / +3 (Backend, Browser, SignIn)
Archivos de config/                          3, uno por etiqueta: app.go, backend.go, browser.go
Maneras de hacer lo mismo                    −2 (buscar un módulo: solo FindBuiltModule en la app; logo: solo el archivo)
```

## Etapas (índice; los archivos con código se escriben tras el 🔵)

| # | Repo | Qué | Depende de |
|---|---|---|---|
| 1 | `view`, `layout`, `appkit` | D66: sin `Brand`; `App.Name`; el logo desde `docs/logo/logo.png`; `view.BrowserDeps` suma `IDs` | — |
| 2 | `rpc`, `mcp` | D70: `rpc.JSON{}` y `mcp.Tools{Allowed}`, cada uno cumple la interfaz chica de `appkit` | — |
| 3 | `appkit` | D67, D69, D71: `App`/`Backend`/`Browser`, `BackendModule`/`BrowserModule`, `FindBuiltModule`, `Build` + `Mount`, `RunBrowser`; fuera `httpd`, `rpc`, `mcp`, `unixid`, `env` | 1, 2 |
| 4 | `server`, `migrate` | `maingen.go`: el `main` generado del paso 4 **y** el `migrate/main.go` generado (hoy lee `config.App.Server.Database`, `maingen.go:158,163`; pasa a `config.Backend.Database`); ejemplos de `migrate/migrate.go:60,93` | 3 |
| 5 | `appkit`, `access` | D68: el ingreso entra a `appkit` (`SignIn`); `access/module/` se borra — junto con «access 1b» | 3 |
| 6 | 8 módulos, uno por vez | D65, D63, D71: borrar `module/`, quitar `appkit`, `ui.Screens(d view.BrowserDeps)`; `device_manager` exporta `TenantDevices` (en `queries.go`, skill **module-layout**). `appointment_booking` no tiene `var Definition`: va después de su migración a `crud` (ARCHITECTURE capa 8), que la crea | 1 (`BrowserDeps.IDs`) |
| 7 | `mjosefa-cms` | pasos 1–4 de §3 (junto con A5); `migrate/pastdb/plan.go:18` pasa a `appkit.BuildModules(config.App, config.Backend, tx)` | 4, 5, 6 |
| 8 | docs y skills | **app-modules**, **module-layout**, **app-server-main**, **isomorphic-libraries**, **project-layout**; quitar las líneas STATUS | 7 |

Verificación al cerrar (D64, sin compuerta): `grep -l webtyp.com/appkit veltylabs/modules/*/go.mod webtyp/access/go.mod`
→ vacío; `grep -n "unixid\|httpd\|webtyp.com/rpc\|webtyp.com/mcp\|events/mock" webtyp/appkit/*.go` → vacío.

**Puerta 18 — orden de publicación** (de abajo hacia arriba): `country` → `view` → `layout` → `rpc` · `mcp` → `appkit` →
`server` → 8 módulos → `access` → `mjosefa-cms`.

## Correcciones de la revisión (2026-11-16)

1. «Nuevo: nada» contradecía D70: `rpc.JSON` y `mcp.Tools` son tipos nuevos.
2. `App` no traía `Countries` (paso 14); `DefaultDocument` (paso 15) es un dato de un solo módulo: P59.
3. `Backend` perdía en silencio `Public` y `Routes` de `AppServer`; se mantienen. `Infra` y `Grants` se borran a
   propósito (solo servían al ingreso, que ahora arma `appkit`). `BackendModule` conserva `DefaultGrants`.
4. `appointment_booking/ui.Screens` necesita ids: `view.BrowserDeps` suma `IDs`, y la etapa 6 depende de la 1.
5. `appointment_booking` no tiene `var Definition`: no puede ir en `App.Modules` hasta su migración a `crud`.
6. El `migrate/main.go` generado y la importación de `mjosefa-cms` leen `config.App.Server.Database` y
   `appkit.BuildModules(app, tx)`: entran a las etapas 4 y 7.
7. El navegador recibía su transporte como argumento suelto y el backend como campo: ahora los dos son campos
   (`Browser.Caller`, `Backend.API`), una sola forma.
8. El punto «el Worker recibe `config.Backend`» era falso: ver P57.
