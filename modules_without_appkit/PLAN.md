# Módulos sin `appkit` y `appkit` sin concretos — plan

> Implementa D62, D63, D65–D69 de [ARCHITECTURE_MASTER_PLAN.md](../ARCHITECTURE_MASTER_PLAN.md). Se ejecuta **en local**
> por [EXECUTION_ORDER paso 17](../EXECUTION_ORDER_MASTER_PLAN.md) (su protocolo: marca, `replace`, `gotest`, cerrar la
> fila). Nada se publica antes de la puerta 🔵 18. Esta carpeta se borra en esa puerta.

**Estado: §3 se reescribe paso a paso cuando el dueño responda P55 y P56 del master.** Los archivos de etapa (código exacto, compilado) se escriben
después del 🔵: si una forma cambia, no se reescriben etapas.

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
- **Nuevo:** nada. Ningún repo nuevo, ninguna función nueva en `model`, `rpc`, `mcp`, `orm`.

## Design gate

### 1. Prior art (enlaces no leídos en esta sesión: «sin verificar»)

Uber Fx (https://pkg.go.dev/go.uber.org/fx) y Google Wire (https://github.com/google/wire): las librerías exportan
constructores; la **app** decide las implementaciones y las conecta. Django (`INSTALLED_APPS`,
https://docs.djangoproject.com/en/stable/ref/applications/): la app lista sus piezas en orden y ninguna pieza importa al
proyecto. Seguimos la regla común; diferimos en que conectamos a mano en `config/`, sin reflexión ni genéricos (TinyGo,
skill **api-harness**).

### 2. Nombres (prueba del novato)

| Nombre nuevo | Se lee como | Reemplaza |
|---|---|---|
| `appkit.FindBuiltModule(d, name)` | «busca el módulo ya construido con ese nombre» | `appkit.Find[T]` (D69) |
| `appkit.App.Name` | el nombre de la app | `appkit.App.Brand` / `view.Brand` (D66) |
| `appkit.App.SignIn` + `appkit.SignInByTrustedIP` | «cómo entra la gente: por IP de confianza» | `access.Module` en `App.Modules` (D68) |
| `appkit.AppServer.NewIDs`, `.Events`, `.Transports` | lo que la app elige para su servidor | `unixid`, `eventsmock`, `rpc`, `mcp` dentro de `appkit` (D67) |
| `devicemanager.TenantDevices` | «los equipos de un tenant» | `IPLocator` + el tipo escondido `served` (D63) |

### 3. Forma propuesta — lo que el dueño revisa

**Lo que escribe la app** (`mjosefa-cms`), todo junto, para ver que se lee de corrido:

```go
// config/app.go — lo mismo para servidor y navegador
var App = appkit.App{
	TenantID: "mjosefa-cms",
	Name:     "Mjosefa",                // D66: el logo sale de docs/logo/logo.png (PNG)
	Timezone: "America/Santiago",
	SignIn:   appkit.SignInByTrustedIP, // D68: cero = nadie entra, toda operación se niega (cerrado por defecto)
	Modules: []appkit.Module{           // D65: el orden importa: quien provee va antes
		{Name: servicecatalog.Definition.Name, Server: catalogServer, Browser: catalogScreens},
		{Name: devicemanager.Definition.Name, Server: devicesServer, Browser: devicesScreens},
		{Name: staffmanager.Definition.Name, Server: staffServer, Browser: staffScreens},
		// … calendar, rooms, patients, booking, encounters
	},
	Server: server,
	Log:    func(a ...any) { fmt.Println(a...) },
}

// config/server.go (!browser) — D67: la app elige cada concreto
var server = appkit.AppServer{
	Database:   postgres.Open,                                                      // ya existe hoy
	NewIDs:     func() (model.IDGenerator, error) { return unixid.NewUnixID() },
	Events:     &eventsmock.Broker{},                                               // el broker en proceso que hoy appkit elige solo
	Transports: []appkit.Transport{jsonOverHTTP, agentTools},                       // abajo
	MCPTools:   MCPTools(),
}

// config/transports.go (!browser) — el JSON por HTTP (rpc) y las herramientas para agentes (mcp)
func jsonOverHTTP(r router.Router, o appkit.Operations) error {
	return rpc.Mount(r, rpc.DefaultPrefix, o.Log, o.Public, o.Events, o.Modules...)
}
func agentTools(r router.Router, o appkit.Operations) error { /* mcp.HarvestOps + mcp.NewServer + MountAPI, movido de appkit/server.go:47-58 */ }

// config/modules_server.go (!browser) — D65 + D69 + D63
var devicesServer = appkit.ModuleServer{Build: func(d appkit.Deps) (model.ModuleNaming, error) {
	return devicemanager.New(d.DB, devicemanager.Deps{IDs: d.IDs, Publisher: d.Events, TenantID: d.TenantID})
}}
var staffServer = appkit.ModuleServer{Build: func(d appkit.Deps) (model.ModuleNaming, error) {
	m, err := appkit.FindBuiltModule(d, devicemanager.Definition.Name)
	if err != nil {
		return nil, err
	}
	dm, ok := m.(*devicemanager.Module)
	if !ok {
		return nil, errDevicesType // error escrito en config: «device_manager no construyó *devicemanager.Module»
	}
	devices := devicemanager.TenantDevices{Devices: dm, TenantID: d.TenantID}
	return staffmanager.New(d.DB, staffmanager.Deps{IDs: d.IDs, Publisher: d.Events, TenantID: d.TenantID,
		ValidateRUT: input.NormalizeRUT, Devices: devices, Registrar: devices})
}}

// config/modules_browser.go (browser)
var devicesScreens = view.ModuleBrowser{Screens: func(d view.BrowserDeps) []view.Screen { return devicesui.Screens(d.Caller) }}
```

**Lo que escribe el `main` generado** (`server/maingen.go`; la app no lo toca). El `main` crea el servidor HTTP y lee
`DATABASE_URL` y los argumentos; `appkit` solo construye y monta:

```go
// web/server/main.go (generado)
built, err := appkit.Build(config.App, os.Getenv("DATABASE_URL"), config.Schema) // módulos, ingreso, colas
srv := httpd.New(httpd.Config{Port: port, PublicDir: publicDir, Gzip: true, Health: true, Logger: config.App.Log,
	Authn: built.Authn, Authorize: built.Authorize, Policy: built.Policy})
err = built.Mount(srv.Router(), devtools.Server(config.Schema)...) // Transports + Infra + rutas de la app
log.Fatal(srv.ListenAndServe())

// web/browser/main.go (una línea, la escribe la app)
func main() { appkit.Browser(config.App, rpc.NewCaller(""), layoutapp.New(devtools.Menu())) }
```

**Puntos a confirmar** (cada uno cambia lo que se escribe arriba):
1. **`externalcalls` y `backoff` no son concretos**: son lógica sobre la base que pasó la app, como `auth` y `rbac`; se
   quedan dentro de `appkit`. (D67 los listó; si el dueño los quiere elegibles, se agrega `AppServer.NewCallQueue`.)
2. **El servidor HTTP lo crea el `main` generado**, en dos pasos (`appkit.Build` → `httpd.New` → `built.Mount`), porque
   `httpd.Config` necesita `Authn`/`Authorize`/`Policy`, que salen del ingreso que arma `appkit`.
3. **Los transportes los escribe la app** (`jsonOverHTTP`, `agentTools`: unas 15 líneas por app), porque `appkit` no
   puede nombrar `rpc` ni `mcp`, y `rpc`/`mcp` no pueden nombrar `appkit.Operations` (las importa la app, no al revés).
   Alternativa: que el `main` generado los escriba (la app no los ve, pero el generador elige `rpc`/`mcp` por ella).
4. **`appkit.SignInByTrustedIP` es un valor de un conjunto cerrado** (`type SignIn int`, un valor por método de
   ingreso); el cero niega todo. El ingreso pide los equipos del personal y el registro del primer administrador buscando
   **por interfaz** entre los módulos construidos, dentro de `appkit` (no exportado).
5. **La pantalla de ingreso del navegador** (`access/module/browser.go:86-151`) pasa a un archivo `browser` de `appkit`;
   deja de crear su propio generador de ids (`browser.go:91`, `unixid` + `panic`): usa el que trae la sesión.

### 4. ¿Cuál evita el acoplamiento y es más mantenible? ¿Dónde va cada cosa?

- **Módulos:** dependen solo de sus interfaces, `model`, `orm`, `view`. Sin `module/`, sin `appkit` (D62, D65).
- **`access`:** solo el puente usuarios ↔ roles por interfaces (`Deps.Users`, `Deps.Roles`). Sin `appkit` (D68).
- **`appkit`:** el armado común a toda app (orden de módulos, ingreso, montaje, colas de llamadas externas), sin elegir
  concretos: los recibe (D67). Así ninguna librería lo importa y ninguna app repite el ingreso.
- **App (`config/` + `main` generado):** elige base, ids, eventos, transportes y servidor HTTP; conecta módulos entre sí.
- Lo que se descartó y por qué está en las decisiones del master (P49 B, P51 A/B, P54 B).

### 5. Qué se borra

`module/` en 8 módulos y en `access` · `webtyp.com/appkit` de 9 `go.mod` · `appkit.Find`, `ErrManyProviders` ·
`appkit` deja de importar `unixid`, `events/mock`, `server/httpd`, `rpc`, `mcp`, `env` · `appkit.Server`, `BuildServer`,
`MountServer`, `ServerConfig`, `ServerEnv`, `processEnv` (los reemplazan `Build` + `Mount`) · `view.Brand` y
`App.Brand` · `devicemanager.IPLocator` y el tipo `served` · el helper `optional` de `clinical_encounter` · los nombres
viejos `appkit.ModuleBrowser|BrowserDeps|EntryScreens|Brand` que hoy rompen el navegador de 3 repos.

### Cuentas

```
Conceptos que aprende el desarrollador   −3 (module/, Brand, Find[T]) / +3 (FindBuiltModule, SignIn, Transport)
Archivos que toca para sumar un módulo   −3 en el módulo (module/*.go) / +2 líneas en la app (Server y Browser)
Maneras de hacer lo mismo                −2 (buscar un módulo: Find[T] en módulo y en app → FindBuiltModule solo en app;
                                             logo: Brand.LogoMark y docs/logo/logo.png → solo el archivo)
```

## Etapas (índice; los archivos se escriben tras el 🔵)

| # | Repo | Qué | Depende de |
|---|---|---|---|
| 1 | `view`, `layout`, `appkit` | D66: `Brand` → `App.Name`; el logo desde `docs/logo/logo.png` | — |
| 2 | `appkit` | D69 (`FindBuiltModule`) y D67 (`AppServer.NewIDs`, `Events`, `Transports`; `Build` + `Mount`; fuera `httpd`, `rpc`, `mcp`, `unixid`, `env`) | 1 |
| 3 | `server` | `maingen.go`: el `main` generado crea `httpd`, lee `DATABASE_URL`, llama `Build` y `Mount` | 2 |
| 4 | `appkit`, `access` | D68: el ingreso entra a `appkit` (`SignIn`); `access/module/` se borra — junto con «access 1b» | 2 |
| 5 | 8 módulos, uno por vez | D65 + D63: borrar `module/`, quitar `appkit`; `device_manager` exporta `TenantDevices` | — (no depende de `appkit`) |
| 6 | `mjosefa-cms` | escribe lo de §3 (junto con A5) | 3, 4, 5 |
| 7 | docs y skills | **app-modules**, **module-layout**, **app-server-main**, **isomorphic-libraries**, **project-layout**; quitar las líneas STATUS | 6 |

Verificación al cerrar (D64, sin compuerta): `grep -l webtyp.com/appkit veltylabs/modules/*/go.mod webtyp/access/go.mod`
→ vacío; `grep -n "unixid\|httpd\|webtyp.com/rpc\|webtyp.com/mcp\|events/mock" webtyp/appkit/*.go` → vacío.

**Puerta 18 — orden de publicación** (de abajo hacia arriba): `country` → `view` → `layout` → `appkit` → `server` →
8 módulos → `access` → `mjosefa-cms`.
