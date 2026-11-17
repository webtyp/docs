# Backend y front por paquetes (`appkit`, `server`, `mjosefa-cms`) — plan

> Implementa D21, D23 y la parte de archivos de D24 de [BUILD_DESTINATIONS_MASTER_PLAN.md](../BUILD_DESTINATIONS_MASTER_PLAN.md).
> Se ejecuta **en local** por [EXECUTION_ORDER paso 17c](../EXECUTION_ORDER_MASTER_PLAN.md) (su protocolo: marca, `replace`,
> `gotest`, cerrar la fila). Nada se publica antes de la puerta 18 (regla 8 de ese protocolo). Esta carpeta se borra en esa puerta.

**Estado: autorizado por la orden del dueño del 2026-11-17 (ejecutar EXECUTION_ORDER completo sin revisión previa); el dueño lo revisa al final.** La etapa 1 está probada: aplicada a una copia de `appkit`
el 2026-11-17, compila en nativo y en wasm y `gotest` da `vet ✅, style ✅, race ✅, tests ✅, tests index ✅`. Las etapas 2 y 3
no se probaron: su código está escrito leyendo `server/maingen.go` y `mjosefa-cms/config/` del árbol local de ese día.

## Qué cambia, en una frase

Lo que es de servidor y lo que es de navegador dejan de vivir en el mismo paquete separados por la etiqueta `browser`:
cada lado es un paquete, y quien lo importa decide qué entra a su binario.

| Hoy | Después |
|---|---|
| `appkit` (un paquete, archivos `browser` / `!browser`) | `appkit` (solo `App`), `appkit/backend`, `appkit/browser` |
| `config` de la app (un paquete, archivos `browser` / `!browser`) | `config` (solo `App`), `config/services`, `config/screens` |
| `appkit.Build(config.App, config.Backend, config.Server, dsn, config.Schema)` | `backend.Build(config.App, services.Backend, services.Server, dsn, services.Schema)` |
| `appkit.RunBrowser(config.App, config.Browser, layout)` | `browser.Run(config.App, screens.Browser, layout)` |

`appkit/backend` y `appkit/browser` son paquetes del **mismo módulo** `webtyp.com/appkit`: no hay repo ni `go.mod` nuevo
y el orden de publicación de la puerta 18 no cambia.

## Búsqueda en el ecosistema

- Los tres consumidores de `appkit` son `webtyp/server` (genera los `main`), `veltylabs/mjosefa-cms` y los tests de ambos
  (`grep -rl '"webtyp.com/appkit"'`). Ningún módulo lo importa (ARCHITECTURE D62).
- Ya existe una prueba de imports en `appkit/tests/browser_build_test.go` → **reusar** (la etapa 1 la apunta al paquete nuevo).
- Los archivos de navegador de `appkit` compilan en nativo (`dom`, `form`, `fetch`, `view` tienen su variante nativa):
  `appkit/browser` no necesita ninguna etiqueta.
- **Nuevo:** ningún repo, ningún tipo. Solo cambian de paquete y de nombre los que lista el «Design gate».

## Design gate

1. **Prior art** (sin verificar en esta sesión): la stdlib de Go separa por paquete y no por etiqueta lo que es de un
   lado (`net/http` y `net/http/httptest`, `database/sql` y `database/sql/driver`); Django separa `settings`, `urls` y
   `views` en módulos del proyecto. Diferimos en que el lado compartido (`config`) solo tiene un valor, `App`.
2. **Nombres (prueba del novato; el dueño los revisa al final):**

   | Hoy | Después | Qué ve quien lo lee |
   |---|---|---|
   | `appkit.Backend`, `appkit.Server`, `appkit.Deps`, `appkit.Built`, `appkit.Build`, `appkit.BuildModules`, `appkit.FindBuiltModule` | `backend.` + el mismo nombre | lo que arma el servidor |
   | `appkit.BackendModule` | `backend.Module` | cómo se construye un módulo en el servidor (antes repetía «backend») |
   | `appkit.Browser`, `appkit.NewSession`, `appkit.Screens` | `browser.` + el mismo nombre | lo que corre en el navegador |
   | `appkit.BrowserModule` | `browser.Module` | las pantallas de un módulo |
   | `appkit.RunBrowser` | `browser.Run` | el cuerpo de `web/browser/main.go` |
   | `appkit.App`, `appkit.SignInByTrustedIP`, `appkit.ErrNoLog`, `appkit.ErrNoTimezone`, `appkit.ErrUnknownSignIn` | sin cambio | la app, igual para los dos lados |
   | `config.Backend`, `config.Server`, `config.Schema`, `config.Migrations`, `config.Removed`, `config.MCPTools` | `services.` + el mismo nombre | lo que la app ofrece desde el servidor |
   | `config.Browser` | `screens.Browser` | las pantallas de la app |

   `backend.Module` y `browser.Module` no están en D23: los propone este plan para no escribir `backend.BackendModule`.
3. **Complejidad:** se borran dos etiquetas (`browser`, `!browser`) de 10 archivos; se agregan 4 carpetas. Ninguna API nueva.
4. **¿Dónde vive?** En `appkit` (el kit) y en `config/` de la app: los únicos dos lugares que hoy mezclan los lados.
5. **Qué borra:** la regla «`backend.go` es `!browser`, `browser.go` es `browser`» y el aviso en rojo del editor en esos archivos.

## Herramientas y entorno

```bash
export PATH="$PATH:$(go env GOPATH)/bin"      # gotest, goimports, langc
go install golang.org/x/tools/cmd/goimports@latest
```
`gotest` se corre como `GOWORK=off gotest` en cada repo; en `mjosefa-cms` además `GOFLAGS=-mod=mod`. Los `replace` locales
que ya existen se dejan como están. No correr `git reset`, `git restore`, `git checkout -- <archivo>` ni `git stash`: los
tres repos tienen trabajo del paso 17 sin commit. Commit después de cada etapa, sin `push`.

## Etapa 1 — `webtyp/appkit` (probada)

Desde `webtyp/appkit`, tal cual:

```bash
mkdir backend browser
git mv backend.go build.go external_calls.go backend/
git mv signin_server.go backend/signin.go
git mv browser.go browser/browser.go
git mv signin_browser.go browser/signin.go
for f in backend/*.go browser/*.go; do sed -i '1{/^\/\/go:build/d}' "$f"; sed -i '1{/^$/d}' "$f"; done
sed -i 's/^package appkit$/package backend/' backend/*.go
sed -i 's/^package appkit$/package browser/' browser/*.go
for f in backend/*.go browser/*.go; do
  for s in App SignInByTrustedIP SignInMethod ErrNoLog ErrNoTimezone ErrUnknownSignIn; do gofmt -r "$s -> appkit.$s" -w "$f"; done
done
gofmt -r 'BackendModule -> Module' -w backend/*.go
gofmt -r 'BrowserModule -> Module' -w browser/*.go
gofmt -r 'RunBrowser -> Run' -w browser/*.go
goimports -w backend/*.go browser/*.go

cd tests
for s in Deps Server Backend Build FindBuiltModule BuildModules StopCallExecutor NewCallRunner StartCallExecutor NewCallQueue Built \
  ErrUnlistedPart ErrNotCrudModule ErrNoPart ErrNoOpen ErrNoModuleBuilt ErrNoIDs ErrNoEvents ErrNoDatabase ErrNoBuildFunc ErrNoAPI; do
  gofmt -r "appkit.$s -> backend.$s" -w *.go
done
gofmt -r 'appkit.BackendModule -> backend.Module' -w *.go
for s in Browser NewSession ErrUnlistedScreens ErrNoCaller ErrNoBrowserPart ErrNoBrowserIDs; do gofmt -r "appkit.$s -> browser.$s" -w *.go; done
gofmt -r 'appkit.BrowserModule -> browser.Module' -w *.go
sed -i '1s#^//go:build !wasm && !browser$#//go:build !wasm#' *.go
sed -i '1{/^\/\/go:build browser$/d}' session_test.go; sed -i '1{/^$/d}' session_test.go
goimports -w *.go
cd ..
```

Después, a mano:

1. `tests/browser_build_test.go`: la línea de `exec.Command` queda
   `cmd := exec.Command("go", "list", "-f", "{{join .Imports \"\\n\"}}", ".", "./browser")` (sin `-tags browser`: revisa la raíz y `appkit/browser`).
2. Comentarios y mensajes que nombran lo viejo (`gofmt -r` no toca comentarios ni textos). En `app.go`, `render_ssr.go`,
   `backend/*.go`, `browser/*.go`, `README.md`, `docs/ARCHITECTURE.md`, `docs/TESTS.md`, `AGENTS.md` reemplazar:
   `appkit.Build(` → `backend.Build(` · `appkit.BuildModules(` → `backend.BuildModules(` · `appkit.FindBuiltModule(` → `backend.FindBuiltModule(` ·
   `appkit.Deps` → `backend.Deps` · `appkit.Backend{` → `backend.Backend{` · `appkit.Server{` → `backend.Server{` ·
   `appkit.BackendModule` y `BackendModule` → `backend.Module` · `appkit.Browser{` → `browser.Browser{` ·
   `appkit.BrowserModule` y `BrowserModule` → `browser.Module` · `appkit.RunBrowser(` → `browser.Run(` · `appkit.NewSession(` → `browser.NewSession(` ·
   `config.Backend` → `services.Backend` · `config.Server` → `services.Server` · `config.Schema` → `services.Schema` · `config.Browser` → `screens.Browser`.
   El texto de los errores (`"appkit: Backend.Events is nil — …"`) **no** cambia: empieza con el nombre del módulo, no del paquete.
3. `render_ssr.go` conserva `//go:build !wasm` (es de plataforma, D24 lo permite) y se queda en la raíz: es un método de `App`.
4. `langc sync .` → sin claves nuevas ni avisos (los textos no cambian, solo de carpeta).

Criterios (cada uno debe dar vacío o lo que se indica):

```bash
grep -rn "go:build.*browser" --include=*.go .                 # vacío
grep -rn "BackendModule\|BrowserModule\|RunBrowser" .         # vacío (salvo docs/LAST_PLAN_EXECUTED.md)
ls *.go                                                       # app.go render_ssr.go
GOWORK=off go list -deps . | grep webtyp.com                  # solo fmt, lang, country, model, appkit
GOWORK=off GOOS=js GOARCH=wasm go list -deps ./browser | grep -E "appkit/backend|postgres|sqlite|httpd|/mcp|authority|externalcalls|backoff"   # vacío
GOWORK=off gotest                                             # todo ✅
```

## Etapa 2 — `webtyp/server` (el generador de los `main`)

Todo en `server/maingen.go` y `server/tests/maingen_test.go`. `replace webtyp.com/appkit => ../appkit` si los tests lo compilan.

1. **Constantes nuevas** (junto a `appVarName`) y el import `"errors"` (hoy `maingen.go` no lo importa). Se llaman `dir…` porque `maingen.go:299` y `:608` ya tienen una variable local `configDir`; esas dos líneas pasan a `configDir := filepath.Join(rootDir, dirConfig)`:

```go
// Where an app declares itself: config (App, both sides), config/services (the backend) and config/screens (the browser).
const (
	dirConfig         = "config"
	dirServices = "services"
	dirScreens  = "screens"
	packageConfig     = "package config"
	packageServices   = "package services"
)

// ErrNoServicesPackage: an app declared as one appkit.App must split its config by side before its mains can be generated.
var ErrNoServicesPackage = errors.New("server: config/services or config/screens is missing — move Backend, Server and MCPTools to config/services (package services) and Browser to config/screens (package screens): https://github.com/webtyp/appkit/blob/main/docs/ARCHITECTURE.md")
```

2. **Plantillas** (solo cambian estas líneas; el resto de cada plantilla queda igual):

| Plantilla | Antes | Después |
|---|---|---|
| `appServerTemplateText`, imports | `"webtyp.com/appkit"` | `"webtyp.com/appkit/backend"` |
| `appServerTemplateText`, imports | `"{{.ModulePath}}/config"` | esa línea y debajo `"{{.ModulePath}}/config/services"` |
| `appServerTemplateText` | `appkit.Build(config.App, config.Backend, config.Server, env.Get(envDatabase), config.Schema)` | `backend.Build(config.App, services.Backend, services.Server, env.Get(envDatabase), services.Schema)` |
| `appServerTemplateText` | `devtools.Server(config.Schema)...` | `devtools.Server(services.Schema)...` |
| `migrateTemplateText`, imports | `"{{.ModulePath}}/config"` | `"{{.ModulePath}}/config/services"` |
| `migrateTemplateText` | `migrate.Drop(config.Server.Database, config.Schema, *drop)` | `migrate.Drop(services.Server.Database, services.Schema, *drop)` |
| `migrateTemplateText` | `migrate.App(config.Server.Database, log.Println, config.Schema)` | `migrate.App(services.Server.Database, log.Println, services.Schema)` |
| `removedTemplateText` | `package config` | `package services` |
| `migrationsTemplateText` | `package config` | `package services` |
| `appBrowserTemplateText`, imports | `"webtyp.com/appkit"` | `"webtyp.com/appkit/browser"` |
| `appBrowserTemplateText`, imports | `"{{.ModulePath}}/config"` | esa línea y debajo `"{{.ModulePath}}/config/screens"` |
| `appBrowserTemplateText` | `appkit.RunBrowser(config.App, config.Browser, layoutapp.New(devtools.Menu()))` | `browser.Run(config.App, screens.Browser, layoutapp.New(devtools.Menu()))` |

   Las tres plantillas conservan su etiqueta de plataforma (`//go:build !wasm`, `//go:build wasm`). En los comentarios de
   cabecera, `config/removed.go` → `config/services/removed.go` y `config/migrations.go` → `config/services/migrations.go`.

3. **Rutas** en `GenerateMain` y `generateMigrationsFile`: las tres que hoy son `filepath.Join(rootDir, "config", "migrations.go")`
   (dos veces) y `filepath.Join(rootDir, "config", GeneratedRemovedFile)` pasan a
   `filepath.Join(rootDir, dirConfig, dirServices, "migrations.go")` y `filepath.Join(rootDir, dirConfig, dirServices, GeneratedRemovedFile)`.
   `generateMigrationsFile` **sigue leyendo** `App` de la carpeta `config` (su variable local `configDir`): `App` no se mueve.

4. **Función nueva** y su llamada. En `GenerateMain`, primera línea dentro del segundo `if isApp {`:
   `if err := moveToServices(rootDir); err != nil { return "", err }`.

```go
// moveToServices refuses an app whose config is not split by side, and carries the two generated files of the old
// layout to config/services: removed.go keeps the developer's answers, migrations.go is rewritten right after.
func moveToServices(rootDir string) error {
	services := filepath.Join(rootDir, dirConfig, dirServices)
	for _, dir := range []string{services, filepath.Join(rootDir, dirConfig, dirScreens)} {
		if info, err := os.Stat(dir); err != nil || !info.IsDir() {
			return ErrNoServicesPackage
		}
	}
	if err := os.Remove(filepath.Join(rootDir, dirConfig, "migrations.go")); err != nil && !errors.Is(err, os.ErrNotExist) {
		return err
	}
	old := filepath.Join(rootDir, dirConfig, GeneratedRemovedFile)
	content, err := os.ReadFile(old)
	if errors.Is(err, os.ErrNotExist) {
		return nil
	}
	if err != nil {
		return err
	}
	next := filepath.Join(services, GeneratedRemovedFile)
	if _, err := os.Stat(next); errors.Is(err, os.ErrNotExist) {
		moved := strings.Replace(string(content), packageConfig, packageServices, 1)
		if err := os.WriteFile(next, []byte(moved), filePerm); err != nil {
			return err
		}
	}
	return os.Remove(old)
}
```

5. **Tests** (`server/tests/maingen_test.go`): cada prueba que escribe `config/app.go` con `appkit.App` crea además las
   carpetas `config/services` y `config/screens` (`os.MkdirAll`, en el mismo ayudante que escribe `app.go`); los textos
   esperados cambian según la tabla del punto 2 (líneas 277–303) y las rutas según el punto 3. Pruebas nuevas, con su fila en `docs/TESTS.md`:

| Test | Preparación → llamada → esperado |
|---|---|
| `TestMaingen_RefusesAppWithoutServicesPackage` | `config/app.go` con `var App = appkit.App{}` y **sin** `config/services` → `GenerateMain` → `errors.Is(err, server.ErrNoServicesPackage)` y no existe `web/browser/main.go` |
| `TestMaingen_MovesRemovedFileToServices` | además `config/removed.go` con `package config` y `var Removed = []ddl.Removed{{Table: "t", Column: "c"}}` → `GenerateMain` → no existe `config/removed.go`; `config/services/removed.go` contiene `package services` y `Table: "t"` |
| `TestMaingen_DeletesOldMigrationsFile` | además `config/migrations.go` → `GenerateMain` → no existe `config/migrations.go`; existe `config/services/migrations.go` con `package services` |

6. Comentarios de otros repos que muestran la llamada vieja (solo el comentario, ningún código): `webtyp/view/session.go:22`,
   `webtyp/layout/app/browser.go:91`, `webtyp/devtools/server.go:17`, `webtyp/devtools/menu.go:13`, `webtyp/migrate/migrate.go:60,93`,
   `webtyp/ddl/capture.go:19`. Escribir la forma nueva de la tabla del punto 2.

Criterios: `grep -n 'config\.\(Backend\|Server\|Schema\|Browser\)\|appkit\.\(Build\|RunBrowser\)' maingen.go` → vacío; `GOWORK=off gotest` → todo ✅.

## Etapa 3 — `veltylabs/mjosefa-cms`

```bash
mkdir -p config/services config/screens
git mv config/backend.go config/mcp.go config/errors.go config/server.go config/migrations.go config/removed.go config/services/
git mv config/browser.go config/screens/browser.go
sed -i '1{/^\/\/go:build !browser$/d}' config/services/backend.go config/services/mcp.go
sed -i '1{/^$/d}' config/services/backend.go config/services/mcp.go
sed -i '1{/^\/\/go:build browser$/d}' config/screens/browser.go; sed -i '1{/^$/d}' config/screens/browser.go
sed -i 's/^package config$/package services/' config/services/*.go
sed -i 's/^package config$/package screens/' config/screens/browser.go
for s in Deps Backend Server Build BuildModules FindBuiltModule; do gofmt -r "appkit.$s -> backend.$s" -w config/services/*.go migrate/pastdb/*.go migrate/pastdb/rules/*.go tests/*.go; done
gofmt -r 'appkit.BackendModule -> backend.Module' -w config/services/*.go
gofmt -r 'appkit.Browser -> browser.Browser' -w config/screens/browser.go
gofmt -r 'appkit.BrowserModule -> browser.Module' -w config/screens/browser.go
for s in Backend Server Schema; do gofmt -r "config.$s -> services.$s" -w migrate/pastdb/*.go migrate/pastdb/rules/*.go tests/*.go; done
gofmt -r 'config.Browser -> screens.Browser' -w tests/*.go
goimports -w config/services/*.go config/screens/*.go migrate/pastdb/*.go migrate/pastdb/rules/*.go tests/*.go
rm -f web/public/client.wasm
```

- `config/services/server.go`, `migrations.go` y `removed.go` **conservan** `//go:build !wasm` (Postgres no existe en wasm: es de plataforma).
- `config/app.go`, `css.go`, `pwa.go`, `deploy.go`, `lang.json` se quedan en `config`. `config/app.go` no cambia.
- `migrate/pastdb/reservations.go` y `encounters.go` tienen una variable local `services`: si `goimports` agrega ahí el
  import de `config/services`, renombrar **la variable** a `serviceIndex` en ese archivo (nunca un alias de import).
- `goimports` no adivina los imports del propio proyecto si el nombre es ambiguo: verificar que cada archivo que usa
  `services.` o `screens.` importe `github.com/veltylabs/mjosefa-cms/config/services` o `…/config/screens`, y que los que usan
  `backend.` o `browser.` importen `webtyp.com/appkit/backend` o `webtyp.com/appkit/browser`.
- Comentarios `// Ejemplo:` de `config/app.go`, `config/services/*.go`, `config/screens/browser.go`: la forma nueva de la tabla de la etapa 2.
- Los tres `main` generados se reescriben **a mano con el texto exacto de las plantillas de la etapa 2**, con
  `{{.ModulePath}}` = `github.com/veltylabs/mjosefa-cms` (el demonio instalado genera el texto viejo): `web/server/main.go`,
  `web/browser/main.go` (primera línea de etiqueta `//go:build wasm`, hoy dice `browser`) y `migrate/main.go`.
- `go.mod`: `replace webtyp.com/server => ../../webtyp/server` ya existe; no tocar `go.sum` a mano (`go mod tidy`).

Criterios:

```bash
grep -rn "go:build.*browser" --include=*.go .                          # vacío
ls config/*.go                                                         # app.go css.go deploy.go pwa.go
GOFLAGS=-mod=mod GOWORK=off GOOS=js GOARCH=wasm go list -deps ./web/browser | grep -E "config/services|appkit/backend|postgres|sqlite|server/httpd|webtyp.com/mcp|auth/authority"   # vacío
GOFLAGS=-mod=mod GOWORK=off go list -deps ./web/server | grep -E "config/screens|appkit/browser"   # vacío
GOFLAGS=-mod=mod GOWORK=off gotest                                     # todo ✅
```

Pantalla (protocolo, paso 5): `restart_development` de `mjosefa-cms`, captura a 972 px y a 320 px del ingreso y de la
primera pantalla; debe verse igual que antes del cambio.

## Etapa 4 — documentación y skills

1. `appkit/README.md` y `appkit/docs/ARCHITECTURE.md`: los ejemplos con los tres paquetes y un párrafo «por qué tres paquetes»
   (el import decide qué entra al binario; ninguna etiqueta propia). `appkit/docs/TESTS.md`: la fila de
   `TestBrowserBuild_AppkitImportsNoServerCode` dice «la raíz de `appkit` y `appkit/browser` no importan código de servidor».
2. Skills (`webtyp/devskills/skills`, luego `cd webtyp/devskills && go install ./cmd/devskills && devskills -f`):
   **project-layout** (el árbol: `config/app.go`, `config/services/{backend,server,mcp,migrations,removed}.go`,
   `config/screens/browser.go`; la línea `STATUS … BUILD_DESTINATIONS` se acorta a lo que falta: la etiqueta `browser` aún la
   pasan las herramientas), **app-server-main** (las llamadas nuevas), **isomorphic-libraries** (un kit separa sus lados
   por paquete), **app-modules** (título y líneas 10–36: ya no existe el par `server.go`/`browser.go` por etiqueta).
3. `mjosefa-cms/docs/`: donde nombre `config/backend.go`, `config/browser.go` o `config.Backend`.

## Entrega

`gotest` verde en `appkit`, `server` y `mjosefa-cms`; los criterios `grep` de cada etapa; las dos capturas. Una etapa que
no compila o cuyas pruebas no corren no está entregada: regla 9 del protocolo de EXECUTION_ORDER (línea en «Bloqueos», el paso queda 🚫). Nada de
`gopush` ni tags dentro de este plan: se publica en la puerta 18.

| Etapa | Repo | Qué | Verificación |
|---|---|---|---|
| 1 | `webtyp/appkit` | tres paquetes, sin etiqueta `browser` | criterios + `gotest` |
| 2 | `webtyp/server` | plantillas, rutas, `moveToServices`, 3 pruebas nuevas | criterios + `gotest` |
| 3 | `veltylabs/mjosefa-cms` | `config/services`, `config/screens`, los 3 `main` | criterios + `gotest` + pantalla |
| 4 | `appkit`, `devskills`, `mjosefa-cms` | documentación y skills | `devskills -f` sin error |
