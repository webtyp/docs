# Stage 2 — `appkit` builds `appmodule.Module`s

← [Stage 1](STAGE_1_APPMODULE.md) | Next → [Stage 3](STAGE_3_GATE.md)

`appkit` keeps the unpublished changes of EXECUTION_ORDER steps 14 and 15 (`Deps.Countries`, `DefaultDocument`, `replace`
to `country` and `view`). Do not revert them.

## 2.1 Point at the local contract

```bash
cd ~/Dev/Project/webtyp/appkit
go mod edit -replace webtyp.com/appmodule=../appmodule
go get webtyp.com/appmodule
```

## 2.2 Apply the verified patch

[stage2_appkit.patch](stage2_appkit.patch) was produced on 2026-11-16 against the working tree of `appkit` with steps 14–15
applied, and passed `go vet ./...`, `go test ./...` and `GOOS=js GOARCH=wasm go vet -tags browser .`:

```bash
git apply -p1 ../docs/modules_without_appkit/stage2_appkit.patch
go mod tidy
```

If `git apply` refuses (the tree changed since), make the same edits by hand; they are:

| File | Change |
|---|---|
| `module_server_browser.go` | **deleted** |
| `module_server.go` | keeps only `Entry` and `Parts`; `ModuleServer`, `Deps`, `Find` deleted; imports `model`, `router` only |
| `app.go` | `ErrNoProvider`, `ErrManyProviders` and `type Module` deleted; `App.Modules []appmodule.Module`; comments say `appmodule.Deps.Zone` etc. |
| `serve.go` | `buildModules(app App, d *appmodule.Deps)` with the body below; `buildInfra(app App, d appmodule.Deps)`; `d := appmodule.Deps{…}`; `appmodule.Find[Entry](d)`; `isErr(err, appmodule.ErrNoProvider)` |
| `build_modules.go` | `BuildModules(app App, db *orm.DB) (appmodule.Deps, error)`; doc example uses `dataimport.Plan[appmodule.Deps]` and `appmodule.Find` |
| `app_server.go` | `type Infra` deleted; `AppServer.Infra []appmodule.Infra` (comment: «a module declares it (access.Infra), the app lists it») |
| `tests/*.go` | `appkit.Find|Deps|Module|ModuleServer|Infra|ErrNoProvider|ErrManyProviders` → `appmodule.…`, import `webtyp.com/appmodule` |

```go
// buildModules builds app.Modules in order on d.DB and points d.BuiltModules at the list it fills, so later modules
// Find the earlier ones — also a module that keeps its Deps and calls Find after the whole list is built.
// It returns the crud modules among them, for BuildServer to mount.
func buildModules(app App, d *appmodule.Deps) (ops []crud.Module, err error) {
	built := make([]model.ModuleNaming, 0, len(app.Modules))
	d.BuiltModules = func() []model.ModuleNaming { return built }
	appendBuilt := func(v model.ModuleNaming) {
		built = append(built, v)
		if o, ok := v.(crud.Module); ok {
			ops = append(ops, o)
		}
	}
	// … the loop over app.Modules is unchanged …
```

## 2.3 New test (TDD for the lazy lookup `access` relies on)

`tests/build_modules_test.go`, `TestBuildModules_KeptDepsSeesLaterModules`: module A's `Build` stores its `appmodule.Deps`
in a package variable and returns `dummyMod{}`; module B (listed after A) returns `partItem{"later"}` wrapped in a type that
also implements `crud.Module` (copy `dummyMod`'s three methods). After `appkit.BuildModules`, `appmodule.Find[laterMarker](kept)`
returns no error. Add its row to `docs/TESTS.md`.

## Deletions (each grep → empty, run in `appkit/`)

```bash
grep -n "type Infra\|type ModuleServer\|type Deps \|func Find\|ErrNoProvider *=\|ErrManyProviders *=\|type Module struct\|d.built" *.go
grep -rn "appkit\.\(Find\|Deps\|Infra\|Module\b\|ModuleServer\|ErrNoProvider\|ErrManyProviders\)" . --include=*.go --include=*.md
```

## Done when

`gotest` green in `appkit` (server and browser destinations); README and `docs/` mention `appmodule.Module` where they
said `appkit.Module`.
