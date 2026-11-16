# Stage 4 — the modules drop `appkit`

← [Stage 3](STAGE_3_GATE.md) | Next → [Stage 5](STAGE_5_APP.md)

**One repo per pass** (mark the row of EXECUTION_ORDER step 17 with the repo you are on). Each pass leaves `gotest`
green in that repo. Order (simple ones first; the order does not matter for the build — no module imports another):

| # | Repo | Extra beyond the recipe | Depends on |
|---|---|---|---|
| a | `business_calendar` | — | stage 2 |
| b | `device_manager` | — | stage 2 |
| c | `staff_manager` | `browser.go` used stale `appkit.ModuleBrowser`/`BrowserDeps` (the recipe fixes it) | stage 2 |
| d | `patient_directory` | `go.mod` drops `replace webtyp.com/appkit` (step 15); README mentions `appkit.Module` | stage 2 |
| e | `appointment_booking` | §4.3 (two optional contracts); stale browser names; comment in `tests/booking_ops_test.go:138` (`appkit.BrowserDeps` → `view.BrowserDeps`) | stage 2 |
| f | `clinical_encounter` | §4.4 (`optional` helper deleted) — verified 2026-11-16: `go vet` server and browser green | stage 2 |
| g | `service_catalog` | — | stage 2 **and** EXECUTION_ORDER step 27 closed (its `crud` is older than `appmodule` requires; `go mod tidy` would break it) |
| h | `room_layout` | §4.5 (optional hours) | stage 2 **and** EXECUTION_ORDER step 28 closed (same reason; verified: today its root package fails on `crud` v0.0.17) |
| i | `access` | §4.6 — **not mechanical today** | its own pending plan «access 1b» (ARCHITECTURE capa 8); see §4.6 |

## 4.1 The recipe (every repo)

Paths: veltylabs modules live at `~/Dev/Project/veltylabs/modules/<repo>` → `replace` to `../../../webtyp/appmodule`;
`access` → `../appmodule`.

```bash
cd ~/Dev/Project/veltylabs/modules/<repo>
go mod edit -replace webtyp.com/appmodule=../../../webtyp/appmodule
sed -i -E 's/appkit\.(ModuleServer|Deps|Find|Module|Infra|ErrNoProvider|ErrManyProviders)\b/appmodule.\1/g; s/appkit\.(ModuleBrowser|BrowserDeps|EntryScreens|Brand)\b/view.\1/g; s#"webtyp.com/appkit"#"webtyp.com/appmodule"#' module/*.go
go mod edit -droprequire webtyp.com/appkit -dropreplace webtyp.com/appkit
go mod tidy
```

Then by hand:
- Every `x, _ := appmodule.Find[...]` becomes `FindOptional` with its error checked (§4.3, §4.5). Never `_` on a Find error.
- `module/module.go`: the comment example says `Modules: []appmodule.Module{<alias>.Module}`.
- `grep -rn "appkit" . --include=*.go --include=*.md` → only lines inside `docs/PLAN*.md` of other plans may remain; fix
  README, `docs/ARCHITECTURE.md`, diagrams and code comments (Spanish in veltylabs: «appmodule.Module», «appmodule.Find»).

`module/module.go` of `service_catalog` after the recipe (the README example of stage 1 copies this):

```go
package module

import "webtyp.com/appmodule"

// Module es service_catalog para la app: va una sola vez en App.Modules.
//
// Ejemplo: Modules: []appmodule.Module{catalog.Module}
var Module = appmodule.Module{Name: "service_catalog", Server: server, Browser: browser}
```

## 4.2 Done for each repo

```bash
grep -n "webtyp.com/appkit" go.mod        # → empty
grep -rn "appkit\." --include=*.go .      # → empty
gotest                                     # green, server and browser destinations
"$TMPDIR/gotest-gate"                      # the stage 3 binary: «appkit only in apps» passes (silent)
```

## 4.3 `appointment_booking/module/server.go` — replace the two `_` lookups

```go
	hours, err := appmodule.FindOptional[ab.OpeningHours](d)
	if err != nil {
		return nil, err
	}
	payments, err := appmodule.FindOptional[ab.PaymentGateway](d)
	if err != nil {
		return nil, err
	}
```

## 4.4 `clinical_encounter/module/server.go` (verified)

The three `optional[clinicalencounter.X](d)` calls become `appmodule.FindOptional[clinicalencounter.X](d)`; the function
`optional` and its comment are deleted, and `"errors"` leaves the imports. `grep -n "func optional" module/` → empty.

## 4.5 `room_layout/module/server.go`

```go
	hours, err := appmodule.FindOptional[roomlayout.HoursReader](d)
	if err != nil {
		return nil, err
	}
```

## 4.6 `access` — merged into its own pending plan

Verified 2026-11-16: after the recipe, `access/module` still fails for reasons that are not D62 — `auth` v0.0.112 does not
compile against `crud` v0.0.17 (`authority/definition_gen.go:554`), and `module/browser.go:110` reads `d.Brand`, which
`view.BrowserDeps` no longer has (access plan stage 5, «entry screens are data»). So `access` changes in the same pass as
its pending plan «access 1b» (`access/docs/PLAN.md`, stages 4–6):

- **Before** that plan runs, Opus updates its text (one edit, after the owner's 🔵 on this plan): in
  `PLAN_STAGE_4_access_infra.md` `appkit.Infra|Find|Deps|AppServer` → `appmodule.Infra|Find|Deps` and `appkit.AppServer`
  only in the app example; in `PLAN.md`/`PLAN_PART2.md` `## Tools` `webtyp.com/appkit@v0.2.36` → `webtyp.com/appmodule@<tag
  of gate 18>`; and a last stage «drop appkit» = the recipe + this:

```go
	reg, err := appmodule.FindOptional[FirstAdminRegistrar](d)
	if err != nil {
		return err
	}
	if reg != nil {
		if err := reg.RegisterFirstAdmin(user.Id, code, name, ip); err != nil {
			return err
		}
	}
```

  (replaces the `switch … errors.Is(err, appkit.ErrNoProvider)` of `module/server.go:64-71`). `lazyTrustedStore` keeps
  `appmodule.Find` — its `Deps` copy sees later modules through `BuiltModules` (appkit test of stage 2.3).
- `access` therefore does **not** block gate 18: the published `gotest` will report it red until «access 1b» lands, which
  is the truth.
