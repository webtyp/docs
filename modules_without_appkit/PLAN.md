# Modules without `appkit` (ARCHITECTURE D62) — plan

> Read the `AGENTS.md` of each repo before changing it. Executed **locally** through
> [EXECUTION_ORDER step 17](../EXECUTION_ORDER_MASTER_PLAN.md): follow its executor protocol (mark, `replace`, `gotest`,
> close the row). Nothing is published until gate 🔵 18. This file is deleted at that gate.

**Owner review (🔵) required before step 17 starts.** What the owner confirms (each marked **[confirm]** below):
1. New repo `webtyp.com/appmodule` holds `Module`, `ModuleServer`, `Deps`, `Find`, `FindOptional`, `Infra` (not `view`).
2. The gate «only an app requires appkit» goes in `gotest` now (`gocheck` is an empty stub), and moves with CODE_GUARD D10.
3. `Module.Name` stays a string here; it moves into `model.ModuleDefinition` in step 20.
4. `access` (a 9th importer D62 did not list) and the app `mjosefa-cms` migrate with their own pending plans, outside gate 18.

## Why

Eight veltylabs modules and `webtyp/access` import `webtyp.com/appkit` only to write `appkit.Module`, `appkit.ModuleServer`,
`appkit.Deps` and `appkit.Find` in their `module/` folder. `appkit` imports every implementation (`sqlite`, `server`, `mcp`,
`rpc`, `httpd`, `externalcalls`…), so each module's `go.mod` drags all of them, and a module cannot be tested or reused
without the app kit. D62: **only the app imports `appkit`**; modules import a contract with no implementation.

`access` also imports `appkit` (D62 lists 8) **[confirm 4]**. Found while writing the plan (fixed by it): `access/module/browser.go`, `appointment_booking/module/browser.go` and
`staff_manager/module/browser.go` still name `appkit.ModuleBrowser`, `appkit.BrowserDeps`, `appkit.EntryScreens`,
`appkit.Brand`, which no published `appkit` has (they moved to `view`): their browser build does not compile today.

## Ecosystem search

- `ls webtyp veltylabs/modules`; grep `appmodule|ModuleServer|BuiltModules|FindOptional` over both trees: no repo holds a
  module construction contract; the only one is inside `appkit` (`module_server.go:15-71`).
- `view` already owns the **browser** half (`view.ModuleBrowser`, `view.BrowserDeps`, `view/browser.go:14-25`) → **reuse**.
- `orm` cannot host the server half: its concern is mapping records, and `Module` needs `view.ModuleBrowser`.
  `crud` cannot name `*orm.DB` (`orm` imports `crud`). `model` cannot name `view` (cycle).
- `gocheck` is an empty stub (`gocheck/gocheck.go`, 7 lines; CODE_GUARD phase 1 not started) → the gate cannot live there yet;
  `gotest` already runs repo checks (`devflow/gotest_testsindex.go`) → **extend** `devflow`.
- `server.HasApp(rootDir)` (`server/maingen.go:262`) already answers «is this repo an app» → **reuse** in the gate.
- **New**: repo `webtyp.com/appmodule` (contract only), created with `gonew`.

## Design gate

**1. Prior art** (links not read in this session: «unverified»).
NestJS `@Module({providers, imports})` — the module declares what it provides and needs, the container wires it at startup
(unverified, https://docs.nestjs.com/modules). Uber Fx `fx.Provide` / `fx.Invoke` — constructors declared by type, resolved
by the app (unverified, https://pkg.go.dev/go.uber.org/fx). Django `AppConfig` in `INSTALLED_APPS` — the app lists its apps
in order, each app knows nothing of the project (unverified, https://docs.djangoproject.com/en/stable/ref/applications/).
All three keep the **declaration** in the module and the **container** in the app; we follow that. We differ from Fx/Nest in
resolving by **explicit order** (`App.Modules`) and `Find[T]` instead of reflection: no reflection is allowed (TinyGo), and
the order is readable in `config/app.go`.

**2. Novice-name test.**
- `appmodule` — «the module as the app plugs it in» **[confirm 1]**. Rejected: `module` (clashes with every module's own
  `package module`), `modulekit` (`kit` reads as «implementations», the thing D62 removes), `plugin` (stdlib name).
- `appmodule.Module`, `appmodule.ModuleServer`, `appmodule.Deps`, `appmodule.Find`, `appmodule.Infra`: the same words modules write today,
  only the package changes (one lookup less for anyone who knows `appkit`).
- `FindOptional[T]` — «find it if the app has it». Replaces three different hand-written forms (below).
- `Deps.BuiltModules func() []model.ModuleNaming` — «the modules built so far». A function, not a slice, because `access`
  keeps its `Deps` and calls `Find` at sign-in time, after later modules were built (`access/module/server.go:141-150`).

**3. Complexity ledger.**
```
Concepts the developer must learn   +1 (package appmodule) / −0   (same four names, new package)
Files they must touch to do X       ±0 (module/ keeps module.go, server.go, browser.go)
Lines at the call site              −9 (clinical_encounter `optional` helper deleted)
Ways to do the same thing           −2 (optional contract: `x, _ := Find`, the `optional` helper, `errors.Is(err, ErrNoProvider)` → FindOptional)
```
`hours, _ := appkit.Find[...]` in `appointment_booking` and `room_layout` also **hid** `ErrManyProviders` (a silent fallback,
**zero-debt**); `FindOptional` returns it.

**4. Where it belongs.**
- `webtyp.com/appmodule`: one concern, the contract between a module and its app. Imports only contracts and isomorphic
  libraries (`model`, `crud`, `orm`, `router`, `events`, `country`, `time`, `view`); `orm` only under `!browser`
  (`module_server.go`), as `appkit` does today. **[confirm 1]** location (alternative B: put `ModuleServer`/`Deps` in `view`
  next to `ModuleBrowser` — no new repo, but `view` would import `orm` and mix screens with server construction; rejected).
- `appkit` keeps what only the composition root needs: `App`, `AppServer`, `Entry`, `Parts` (asserted by appkit,
  implemented structurally by `access`; no module names them), `BuildModules`, `BuildServer`.
- `Infra` moves to `appmodule` too: a **module** declares it (`appkit/app_server.go:30` shows `access.Infra`; access's
  pending plan `access/docs/PLAN_STAGE_4_access_infra.md` writes `var Infra = appkit.Infra{…}` inside access, which D62
  forbids). Stage 4 corrects that plan's text.
- The gate goes in `gotest` (`devflow`), not `gocheck` **[confirm 2]**: `gocheck` has no engine yet. When CODE_GUARD D10 moves
  `gotest`'s own checks into `gocheck`, this one moves with them (one line added to CODE_GUARD at gate 18).
- `Module.Name` stays a string in this plan **[confirm 3]**: moving it into `model.ModuleDefinition` (D62's «data of the
  module») is done in EXECUTION_ORDER step 20 with the rest of that data (rubros, íconos, maneras de uso);
  `appointment_booking` has no `var Definition` yet.

**5. What it deletes.** From `appkit`: `Module`, `ModuleServer` (both files), `Deps`, `Find`, `Infra`, `ErrNoProvider`,
`ErrManyProviders`, the unexported `built` pointer. From the 9 modules: `webtyp.com/appkit` in `go.mod` and every import of
it; `clinical_encounter/module/server.go` `optional`. Stale `appkit.ModuleBrowser`/`BrowserDeps`/`EntryScreens`/`Brand` names.

## APIs used

| Symbol | Signature | Tag | File |
|---|---|---|---|
| `view.ModuleBrowser` | `struct{ Screens func(d BrowserDeps) []Screen; Entry func(d BrowserDeps) EntryScreens }` | `view` v0.6.86 | `view/browser.go:22` |
| `view.BrowserDeps` | `struct{ Caller router.Caller; Zone time.Zone; DefaultDocument string; Countries country.Catalog }` | local `view` (step 15) | `view/browser.go:14` |
| `view.EntryScreens`, `view.Brand` | structs | `view` v0.6.86 | `view/browser.go:29`, `view/session.go:11` |
| `model.ModuleNaming` | `interface{ ModelName() string }` | `model` v0.2.24 | `model/interface.go:42` |
| `model.IDGenerator` | `interface{ NewID() string }` | `model` v0.2.24 | `model/interface.go:90` |
| `model.Grant` | `struct{ Resource Resource; Actions Action }` | `model` v0.2.24 | `model/rbac.go:247` |
| `events.Publisher`, `crud.CallQueue`, `country.Catalog`, `time.Zone`, `*orm.DB` | as in `appkit/module_server.go:36-48` | `events` v0.1.14 · `crud` v0.0.17 · `country` v0.0.2 · `time` v0.6.7 · `orm` v0.12.42 | — |
| `router.APIModule` | interface (the routes an Infra mounts) | `router` v0.5.0 | `router` |
| `server.HasApp` | `func HasApp(rootDir string) bool` | `server` v0.2.127 | `server/maingen.go:262` |
| `(*devflow.Go).HasDependency` | `func (g *Go) HasDependency(gomodPath, modulePath string) bool` | local `devflow` | `devflow/go_mod.go:723` |

If a signature differs from this table, stop and write it in «Bloqueos» of EXECUTION_ORDER; never adapt it by guessing.

## Stages

| # | Repo | File | gotest green after |
|---|---|---|---|
| 1 | `appmodule` (new) | [STAGE_1_APPMODULE.md](STAGE_1_APPMODULE.md) | yes |
| 2 | `appkit` | [STAGE_2_APPKIT.md](STAGE_2_APPKIT.md) | yes (`replace` → `../appmodule`) |
| 3 | `devflow` | [STAGE_3_GATE.md](STAGE_3_GATE.md) | yes |
| 4 | the 8 modules, one per pass; `access` with its own plan | [STAGE_4_MODULES.md](STAGE_4_MODULES.md) | yes, per repo |
| 5 | `mjosefa-cms` (with A5, after `access`) | [STAGE_5_APP.md](STAGE_5_APP.md) | yes |
| 6 | docs and skills (Opus, after gate 18) | [STAGE_6_DOCS.md](STAGE_6_DOCS.md) | — |

## Gate 18 — publish order (bottom-up; if a `go.mod` disagrees, the `go.mod` wins)

`country` → `view` → `layout` (step 15) → `appmodule` → `appkit` (steps 14, 15 and this plan) → `devflow` →
`business_calendar` · `device_manager` · `staff_manager` · `patient_directory` · `appointment_booking` ·
`clinical_encounter` · `service_catalog` · `room_layout` (no module imports another). Then install the new `gotest`.
Then: no `go.mod` keeps a `replace`; `grep -l webtyp.com/appkit veltylabs/modules/*/go.mod` → empty.

**Not in gate 18:** `access` and `mjosefa-cms` land with their own pending work («access 1b», A5; STAGE_4 §4.6 and
STAGE_5). Until then `gotest` in `access` reports «appkit only in apps» red — the truth, not a regression.
