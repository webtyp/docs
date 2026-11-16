# Stage 5 — the app `mjosefa-cms` lists `appmodule.Module`s

← [Stage 4](STAGE_4_MODULES.md) | Next → [Stage 6](STAGE_6_DOCS.md)

The app is the one place that keeps `webtyp.com/appkit` (`config/` declares `var App`, so the stage 3 gate passes).

**Depends on stage 4 row i (`access`)**: `config/app.go` lists `access.Module`, which is an `appkit.Module` until access
migrates, so the app cannot compile against `[]appmodule.Module` before it. The app also still names symbols no published
`appkit` has (`appkit.Brand`, `appkit.LoginByRUT`, `appkit.RunBrowser`): that is ARCHITECTURE A5 (`mjosefa-cms`, waits for
`access` and `appointment_booking`), not this plan — do this stage in the same pass as A5.

## 5.1 Recipe

```bash
cd ~/Dev/Project/veltylabs/mjosefa-cms
go mod edit -replace webtyp.com/appmodule=../../webtyp/appmodule
grep -rl "appkit\.\(Deps\|Find\|Module\b\|ModuleServer\|Infra\)" --include=*.go . | xargs sed -i -E 's/appkit\.(Deps|Find|Module|ModuleServer|Infra)\b/appmodule.\1/g'
go mod tidy
```

Add `"webtyp.com/appmodule"` to the imports of each touched file (`goimports -w` on them). Today's count: `appkit.Deps` 42,
`appkit.Find` 24, `appkit.Module` 3, `appkit.ModuleServer` 1 — mostly `migrate/pastdb/*.go`
(`dataimport.Plan[appmodule.Deps]`, `dataimport.Session[appmodule.Deps]`) and `tests/app_helpers_test.go:44`.

`config/app.go` after it:

```go
	// Orden: quien entrega un contrato va antes de quien lo busca (appmodule.Find).
	Modules: []appmodule.Module{
		access.Module, catalog.Module, devices.Module, staff.Module, calendar.Module, rooms.Module,
		patients.Module, booking.Module, encounters.Module,
	},
```

## Done when

`grep -rn "appkit\.\(Deps\|Find\|Module\b\|ModuleServer\|Infra\)" --include=*.go .` → empty; `gotest` green; `webtyp`
MCP `restart_development` of the app boots and the access screen opens (screenshot at 972 px).
