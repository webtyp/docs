# Stage 1 — new repo `webtyp.com/appmodule`

← [Plan](PLAN.md) | Next → [Stage 2](STAGE_2_APPKIT.md)

## 1.1 Create the repo

```bash
cd ~/Dev/Project/webtyp
gonew appmodule "Contract between a domain module and the app that lists it: Module, ModuleServer, Deps, Find" --owner webtyp
cd appmodule
go get webtyp.com/view@v0.6.86 webtyp.com/orm@v0.12.42 webtyp.com/crud@v0.0.17 webtyp.com/country@v0.0.2 \
       webtyp.com/events@v0.1.14 webtyp.com/time@v0.6.7 webtyp.com/model@v0.2.24 webtyp.com/router@v0.5.0
```

Never `git init` / `go mod init` by hand. Never write `go.sum` lines by hand.

## 1.2 Files (verbatim — compiled and tested on 2026-11-16, server and `GOOS=js GOARCH=wasm -tags browser`)

### `module.go`
```go
// Package appmodule is the contract between a domain module and the app that lists it: the module declares a
// Module value, the app (through appkit) builds it. It holds no implementation, so a module that imports it
// never pulls a database driver, an HTTP server or an MCP server into its go.mod.
package appmodule

import "webtyp.com/view"

// Module is one domain module as an app lists it in App.Modules, with its server part and its screens.
//
//	var Module = appmodule.Module{Name: "service_catalog", Server: server, Browser: browser}
type Module struct {
	Name    string             // identity: migration step name, error prefix
	Server  ModuleServer       // the module fills it from a !browser file; zero in the browser
	Browser view.ModuleBrowser // the module fills it from a browser file; zero on the server
}
```

### `module_server_browser.go`
```go
//go:build browser

package appmodule

// ModuleServer is empty in the browser: what only the server needs lives in module_server.go.
type ModuleServer struct{}
```

### `module_server.go`
```go
//go:build !browser

package appmodule

import (
	"errors"

	"webtyp.com/country"
	"webtyp.com/crud"
	"webtyp.com/events"
	"webtyp.com/model"
	"webtyp.com/orm"
	"webtyp.com/router"
	"webtyp.com/time"
)

// ErrNoProvider and ErrManyProviders are what Find returns when the modules
// built so far hold zero or more than one value of the requested type.
var (
	ErrNoProvider    = errors.New("appmodule: no module provides it — list the module that does before the one that needs it in App.Modules")
	ErrManyProviders = errors.New("appmodule: more than one module provides it — keep one in App.Modules")
)

// ModuleServer is how the server builds the module.
//
//	var server = appmodule.ModuleServer{Build: build}
type ModuleServer struct {
	Build         func(d Deps) (model.ModuleNaming, error)
	DefaultGrants []model.Grant // what any signed-in person may do with this module (granted to rbac.RoleAuthenticated)
}

// Infra is a module with its own HTTP routes (login, logout, file upload): it gets the whole Router, so the app mounts
// it only because it lists it in AppServer.Infra. It is built after App.Modules, so it can Find what they built.
//
//	var Infra = appmodule.Infra{Name: "auth", Build: func(d appmodule.Deps) (router.APIModule, error) { return appmodule.Find[*authority.Module](d) }}
type Infra struct {
	Name  string // identity in error messages
	Build func(d Deps) (router.APIModule, error)
}

// Deps is what every module gets when the app builds it: the database, ids, events, the
// installation's tenant and time zone, and Find over the modules built before it.
type Deps struct {
	DB              *orm.DB
	IDs             model.IDGenerator
	Events          events.Publisher
	TenantID        string
	Zone            time.Zone       // the installation's zone (App.Timezone), loaded once: every local date and hour comes from it
	DefaultDocument string          // the document type a new record starts with (App.DefaultDocument)
	Countries       country.Catalog // the installation's countries (App.Countries): the default country and which come first
	Log             func(...any)    // the app's log (App.Log); never nil when appkit builds the Deps
	Queue           crud.CallQueue
	// BuiltModules returns the modules the app has built until now, in App.Modules order. appkit sets it;
	// a module never calls it: it asks with Find. nil in a Deps written by hand (tests): Find finds nothing.
	BuiltModules func() []model.ModuleNaming
}

// Find returns the one module built so far that satisfies T, so a module asks
// for what it needs by interface and never imports the module that has it.
//
//	cats, err := appmodule.Find[roomlayout.CategoryReader](d)
func Find[T any](d Deps) (T, error) {
	found, n := providers[T](d)
	switch n {
	case 0:
		return found, ErrNoProvider
	case 1:
		return found, nil
	default:
		var zero T
		return zero, ErrManyProviders
	}
}

// FindOptional is Find for a contract the app may leave out: no provider returns the zero value and no error;
// more than one is still ErrManyProviders.
//
//	hours, err := appmodule.FindOptional[roomlayout.HoursReader](d)
func FindOptional[T any](d Deps) (T, error) {
	found, n := providers[T](d)
	if n > 1 {
		var zero T
		return zero, ErrManyProviders
	}
	return found, nil
}

// providers returns the last built module that satisfies T and how many do.
func providers[T any](d Deps) (found T, n int) {
	if d.BuiltModules == nil {
		return found, 0
	}
	for _, m := range d.BuiltModules() {
		if v, ok := m.(T); ok {
			found = v
			n++
		}
	}
	return found, n
}
```

### `tests/find_test.go`
```go
//go:build !browser

package tests

import (
	"errors"
	"testing"

	"webtyp.com/appmodule"
	"webtyp.com/model"
)

type hoursReader interface{ OpensAt(day int) int }

type calendarModule struct{ name string }

func (c calendarModule) ModelName() string    { return c.name }
func (calendarModule) OpensAt(day int) int    { return 9 }

type otherModule struct{}

func (otherModule) ModelName() string { return "other" }

func depsWith(built ...model.ModuleNaming) appmodule.Deps {
	return appmodule.Deps{BuiltModules: func() []model.ModuleNaming { return built }}
}

func TestFind_OneProvider(t *testing.T) {
	got, err := appmodule.Find[hoursReader](depsWith(otherModule{}, calendarModule{"calendar"}))
	if err != nil {
		t.Fatalf("Find: %v", err)
	}
	if got.OpensAt(1) != 9 {
		t.Fatalf("OpensAt = %d, want 9", got.OpensAt(1))
	}
}

func TestFind_NoProvider(t *testing.T) {
	if _, err := appmodule.Find[hoursReader](depsWith(otherModule{})); !errors.Is(err, appmodule.ErrNoProvider) {
		t.Fatalf("err = %v, want ErrNoProvider", err)
	}
}

func TestFind_SeesModulesBuiltLater(t *testing.T) {
	var built []model.ModuleNaming
	d := appmodule.Deps{BuiltModules: func() []model.ModuleNaming { return built }}
	built = append(built, calendarModule{"calendar"})
	if _, err := appmodule.Find[hoursReader](d); err != nil {
		t.Fatalf("Find after a later build: %v", err)
	}
}

func TestFindOptional_NoProvider(t *testing.T) {
	got, err := appmodule.FindOptional[hoursReader](depsWith(otherModule{}))
	if err != nil || got != nil {
		t.Fatalf("FindOptional = (%v, %v), want (nil, nil)", got, err)
	}
}

func TestFindOptional_ManyProviders(t *testing.T) {
	_, err := appmodule.FindOptional[hoursReader](depsWith(calendarModule{"a"}, calendarModule{"b"}))
	if !errors.Is(err, appmodule.ErrManyProviders) {
		t.Fatalf("err = %v, want ErrManyProviders", err)
	}
}
```

Add one more test, same file: `TestFind_ManyProviders` — `depsWith(calendarModule{"a"}, calendarModule{"b"})` →
`errors.Is(err, appmodule.ErrManyProviders)`; and `TestFind_NilBuiltModules` — `appmodule.Find[hoursReader](appmodule.Deps{})`
→ `ErrNoProvider`.

### `docs/TESTS.md`
```markdown
# Tests

| Test | Rule |
|---|---|
| [TestFind_OneProvider](../tests/find_test.go) | one built module satisfies T → it is returned |
| [TestFind_NoProvider](../tests/find_test.go) | none satisfies T → ErrNoProvider |
| [TestFind_ManyProviders](../tests/find_test.go) | two satisfy T → ErrManyProviders |
| [TestFind_NilBuiltModules](../tests/find_test.go) | a hand-written Deps finds nothing |
| [TestFind_SeesModulesBuiltLater](../tests/find_test.go) | a Deps kept by a module sees modules built after it |
| [TestFindOptional_NoProvider](../tests/find_test.go) | optional contract absent → zero value, no error |
| [TestFindOptional_ManyProviders](../tests/find_test.go) | optional contract twice → ErrManyProviders |
```

## 1.3 README.md (≤ 150 lines, template of skill **documentation**)

Four sections in this order:
1. **Why This Exists** — a module must not import `appkit` (which carries every implementation). Without/With:
   `import "webtyp.com/appkit"` + `appkit.Module{...}` → `import "webtyp.com/appmodule"` + `appmodule.Module{...}`.
2. **Before You Start** — the app lists modules in `App.Modules` (appkit); a module never imports appkit; `gotest` fails a
   non-app repo whose `go.mod` requires appkit.
3. **Step-by-Step Guide** — the full `module/module.go`, `module/server.go`, `module/browser.go` of `service_catalog`
   after stage 4 (copy from STAGE_4 §4.2), then a `Find` / `FindOptional` example from `room_layout`. Link
   [module_server.go](module_server.go) for the errors instead of copying them.
4. **Documentation Map** — `module.go`, `module_server.go`, `docs/TESTS.md`.

## Done when

`gotest` green in `appmodule`; `grep -rn "map\[" *.go` → empty; the browser destination of `gotest` passes.
