# Destinos de compilación — master plan
> **Status:** IN PROGRESS · 2026-11-17 · sin preguntas abiertas; D21–D24 decididas y sin construir; la fase 3 la ejecuta Antigravity por EXECUTION_ORDER paso 17c (orden del dueño: sin revisión previa); las fases 4 y 5 (herramientas) tienen su plan por escribir, después de la puerta C de EXECUTION_ORDER. El tamaño del wasm pasó a WASM_SIZE_MASTER_PLAN.

**Qué es:** que la **carpeta de `web/` diga a qué binario va** (`web/browser/`, `web/server/`, `web/edge/`) y que cada binario compile solo lo suyo, sin que backend y front se mezclen. Sale de `misitio` (Worker de Cloudflare probado en local): `wasm` no distingue Worker de navegador y `client.wasm` llegó a 682 KB gzip contra un presupuesto de 512 KB.

**Las reglas viven en los skills, no aquí:** un `main` por carpeta de `web/` sin etiqueta de destino, salida única en `web/`, rutas relativas al directorio de trabajo, `wasm`/`!wasm` para la plataforma y `browser`/`!browser` hasta que se construya D24, que la borra (D1–D20, cerradas). Cada destino se prueba en su plataforma con `gotest` (D16).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D19 | `ui/` de un módulo sin etiqueta (los productores de assets siguen `!wasm`) | cada módulo, al migrarlo (D62) |
| D21 (P10, P8) | **Backend y front se separan por paquete, nunca por etiqueta dentro de un paquete**; los imports deciden qué entra al wasm. Vale para `appkit`, para `config/` de la app y para los módulos propios de una app (`modules/<m>/ui/`, mismo molde que un módulo reutilizable). Reemplaza D6 (`server.go` `!browser` / `browser.go` `browser`) | `appkit`, `config` de las apps |
| D21 revisión | Una prueba automática de `gotest` (antes la llamé «candado»): falla, nombrando el import, si el `main` del navegador o del Worker importa un paquete que solo es de servidor (`postgres`, `sqlite`, `server/httpd`, `mcp`, `auth/authority`, `appkit/backend`). No lista `orm`, `crud`, `storage`, `events`, `rbac`, `ddl`: corren en los dos lados. **Sin presupuesto de tamaño** (dueño, 2026-11-17: complica) | `router` (`layoutscan`), `devflow` (`gotest`) |
| D21 limpieza | Lo que hoy **enseña** la regla vieja: `router/layoutscan/layoutscan.go:199` (el mensaje manda a crear `server.go` `!browser` y `browser.go` `browser`) y `:227` (lo exige); el skill `app-modules` (líneas 10–36, contradice a `project-layout`); D65 de ARCHITECTURE (`modules_server.go` `!browser` / `modules_browser.go` `browser`). En `mjosefa-cms`: `web/public/client.wasm` (8 de octubre, 7,5 MB) huérfano junto a `browser.wasm` | `router`, `devskills`, ARCHITECTURE_MASTER_PLAN, `mjosefa-cms` |
| D22 (P7) | **Un solo lugar compila `browser.wasm`** con los flags de producción; lo usan `app`, `sitec` y el release. Hoy hay tres juegos de flags: `app/client_builder_init.go:63` (`-opt=z -no-debug -panic=trap`), `sitec/wasm_builder_exec.go:131` (`-opt=2 -no-debug`) y `misitio/tests/client_size_test.go` (sin flags). `sitec` sigue existiendo (no hay reemplazo: fue un error del chat) | `gobuild` o `tinygo` (paso 0 de `api-design` en el plan), `app`, `sitec` |
| D23 (P11) | Nombres de D21: el kit es `appkit` (solo `App`), `appkit/backend` (`Backend`, `Server`, `Build`, `Deps`, el ingreso) y `appkit/browser` (`Browser`, `Run`, `NewSession`); la app es `config` (solo `App`), `config/services` (`Backend`, `Server`, `Schema`, `MCPTools`) y `config/screens` (`Browser`). Se escribe `backend.Build(config.App, services.Backend, services.Server, dsn, services.Schema)` y `browser.Run(config.App, screens.Browser, layout)` | `appkit`, `mjosefa-cms`, `server` (genera los `main`) |
| D24 (P12) | **Se borra la etiqueta `browser`**: queda un solo eje, `wasm`/`!wasm`, y `dev` como única etiqueta propia. `app` deja de pasar `-tags browser`, `gotest` falla si un archivo la lleva, `goflare` ya no necesita rechazarla. Hoy la llevan: 6 archivos de `appkit`, `config/` y `web/browser/main.go` de `mjosefa-cms`, `clinical_encounter/view.go`, `clinical_encounter/ui/browser.go`, `supply_manager/web/browser.go` | `appkit`, `app`, `devflow`, `goflare`, 2 módulos, `mjosefa-cms`, skills |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | `goflare` | `action.yml` deja de hornear la versión (`goflare/docs/PLAN.md`); la guardia de `-tags browser` sale del plan (D24) | ⬜ corregir el plan y despachar |
| 2 | `veltylabs/misitio` | Migrar a D18: `edge/main.go` → `web/edge/main.go`, `web/client.go` → `web/browser/main.go`, `web/server.go` → `web/server/main.go`, sin `.build/` (re-layout M1–M8 de DEPLOY_WIZARD); su `config/panel` (paquete de navegador sin etiqueta) ya cumple D21 | ⬜ plan por escribir |
| 3 | `appkit`, `server`, `mjosefa-cms`, `devskills` | D21 + D23 + archivos de D24 (ejecutado en EXECUTION_ORDER paso 17c) | ✅ publicado 2026-11-17 (puerta C): `appkit` v0.2.37 · `server` v0.2.128 · `clinical_encounter` v0.2.81 · `devskills` v0.0.51 · `mjosefa-cms` v0.0.125 |
| 4 | `router`, `devflow`, `app`, `sitec`, `lang`, `goflare`, `devskills` | D21 revisión y limpieza, y D24 en las herramientas que conocen la etiqueta: `devflow/buildtag.go:21` y `gotest_destinations.go:34`, `app/client_builder_init.go:33,48,63`, `sitec/wasm_builder_exec.go:19` y `reach.go:61`, `lang/langc/scan.go:34`, `goflare/config.go:215`, `router/layoutscan/layoutscan.go:51,199,227`. Ya existe `gobuild/depguard.go` (`CheckForbiddenImports`): reusar para la revisión de imports | ⬜ 🧠 plan por escribir, después de la puerta C (hasta entonces pasar `-tags browser` no hace daño: ningún archivo la lleva) |
| 5 | `gobuild` o `tinygo`, `app`, `sitec` | D22: un solo compilador de `browser.wasm` con un solo juego de flags | ⬜ plan por escribir |
| 7 | `service_catalog`, `supply_manager`, `alert_manager` | El demo de un módulo es `seed/demo.go` (`seed.Demo`), no `web/`: borrar `service_catalog/web/` y `supply_manager/web/` (sobras), y `supply_manager` y `alert_manager` no tienen `seed/demo.go` | ⬜ por hacer |

## Bloqueos
- 2026-11-17 · ARCHITECTURE D16 dice que `appkit.Browser` pasa a `layout/app`; D23 lo deja en `appkit/browser`.

## Relación con otras olas

- **WASM_SIZE**: recibe lo que es tamaño del binario (antes P13/D25 de aquí); su D2 se construye en el compilador único de D22.
- **ARCHITECTURE (D62–D75)** y **EXECUTION_ORDER** (paso 17): D21 + D23 cambian lo que publica el paso 17; **DEPLOY_WIZARD** (re-layout de `misitio`, D34 `config/deploy.go` con `//go:build deploy` sin cambios); **TIME_UNITS** ola 2e (apps) y **SELFHOST_DEPLOY** (migrar al arrancar, antes P9: hoy lo genera `migrate/main.go` en la raíz de la app, `project-layout`) esperan esta ola.
