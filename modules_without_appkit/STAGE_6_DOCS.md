# Stage 6 — docs and skills (Opus, right after gate 18)

← [Stage 5](STAGE_5_APP.md) | [Plan](PLAN.md)

Skills are the only source of how to work (`webtyp/devskills/skills`); run the `devskills` sync after editing.

| Where | Change |
|---|---|
| skill **app-modules** | `module/` pair writes `appmodule.Module`, `appmodule.ModuleServer`, `appmodule.Deps`, `appmodule.Find` / `FindOptional`; a module never imports `appkit`; `_ :=` on a Find error is forbidden |
| skill **module-layout** | `module/` imports `webtyp.com/appmodule` and `view`, never `appkit` |
| skill **project-layout** | invariant: only an app (`config/` with `var App`) requires `appkit`; `gotest` fails otherwise |
| skill **isomorphic-libraries** | `appmodule` is the module/app contract; `appkit` is the composition root |
| skill **app-server-main** | `App.Modules []appmodule.Module`; `AppServer.Infra []appmodule.Infra` |
| `webtyp/appkit/docs/ARCHITECTURE.md`, `docs/diagrams/*` | the module contract lives in `appmodule`; appkit fills `Deps.BuiltModules` |
| `webtyp/docs/ARCHITECTURE_MASTER_PLAN.md` | D62 row leaves; `appmodule vX` · `appkit vX` · modules join «Cerrado» |
| `webtyp/docs/CODE_GUARD_MASTER_PLAN.md` | D10: «the appkit-only-in-apps check of `gotest` moves into `gocheck` with the others» |
| `webtyp/docs/CONFIGURABLE_MODULES_MASTER_PLAN.md` | its module declaration (rubros, íconos, maneras de uso) is `model.ModuleDefinition`, referenced by `appmodule.Module` — moving `Module.Name` there is step 20 |
| this folder | deleted |
