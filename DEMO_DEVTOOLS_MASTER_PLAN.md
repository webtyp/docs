# Herramientas de desarrollo en el demo de un módulo — master plan
> **Status:** IN PROGRESS · 2026-10-30 · sin preguntas; X1b y X1c cerradas; en Jules X1a (`router`) y X1d (`appkit`); sigue X2 (`layout`)

**Disparador:** en el demo de un módulo (`webtyp` en la raíz del módulo) el menú muestra «Explorador de API» y «Base
de datos», pero salen vacíos: piden `/_routes` y `/_schema` (`layout/apiexplorer/explorer.go:58`, `erd.SchemaPath`) a un
servidor que no existe, porque el demo corre entero en el navegador (`loopback` + base en memoria,
`access/web/browser/main.go`) → 404. Pasa en **todos** los demos, y ahí se revisa cada módulo antes de entrar a una app
(D23 de MVP_DEPLOYMENT_ROADMAP). Decidido (P39-A del roadmap): el demo los muestra con lo que ya corre en el navegador.

**Lo que ya existe (verificado):** la tabla de rutas sale de `Routes()` de cualquier `router.Router`
(`router.MountIntrospection`, `router/introspection.go:56`), no de HTTP; el diagrama sale de `ddl.Capture(pasos)` +
`erd.ReadApp(grupos...)` (`appkit/server.go:58-60`, `appkit/schema.go:10`), tampoco de HTTP. Lo que falta: `loopback`
descarta `Requires`/`Public`/`RequiresMFA` (`noopRoute`, `router/loopback/loopback.go:106`), así que hoy no sabe qué
permiso pide cada operación; y las dos pantallas solo saben leer por `fetch`.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P39 del roadmap) | El demo de un módulo muestra su API y su base de datos con lo que corre en el navegador; una app real sigue leyéndolas del servidor | según P1 |
| D2 (P1) | **Fuentes explícitas**: las dos pantallas leen de una fuente; una app con servidor les da la de HTTP (como hoy), un demo les da la tabla y el diagrama armados en el navegador con las mismas piezas del servidor: `rpc.Routes` + `router.Table` y `erd.ReadSteps` | X1, X2 |
| D3 (P1, regla del dev) | **Las herramientas aparecen solo si hay algo que mostrar**: sin API ni base, no hay grupo «Herramientas de desarrollo»; con solo API, solo el explorador; con solo base, solo el diagrama. Vale también para una app 100 % sin servidor. `BuildMenu` recibe las herramientas como parámetro **obligatorio** (`Tools{}` = ninguna), así cada app y cada demo declara lo que tiene y ninguna queda mostrando pantallas vacías por omisión | X2, X3 |

## Fases

Ecosistema (Step 0): no existe una fuente local de rutas ni de diagrama; `router.MountIntrospection`, `ddl.Capture` y
`erd.ReadApp` se **reutilizan**; `loopback`, `layout/apiexplorer`, `layout/schemaview` y `layout/app` se **extienden**.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| X1a | `router` | `router.Table(routes, policy)`: un solo armador de la tabla, el mismo de `/_routes`; se borran los escritores privados | [Jules · 2026-10-30 16:00] despachado |
| X1b | `rpc` | `rpc.Routes(prefix, mods...)` | cerrada (publicada) |
| X1c | `erd` | `erd.ReadSteps(steps...)` | cerrada v0.0.14 |
| X1d | `appkit` | Usa `erd.ReadSteps` y borra `groupsOf` | [Jules · 2026-10-30 17:30] despachado |
| X2 | `layout` (`apiexplorer`, `schemaview`, `app`) | Las pantallas leen de una fuente; `BuildMenu(..., tools Tools)` obligatorio; el grupo y cada herramienta aparecen solo con fuente (D3) | espera X1a–X1c |
| X3 | cada app y cada demo que llama `BuildMenu` | Declara sus herramientas: HTTP en una app con servidor, locales en un demo, ninguna sin API ni base; captura del explorador y del diagrama | espera X2 |

## Bloqueos

- Ninguno fuera de P1.

## Relación con otras olas

- MVP_DEPLOYMENT_ROADMAP (`mjosefa-cms`): D23 revisa cada módulo en su demo; P39 de allí abrió esta ola.
- APP_COMPOSITION: `appkit.MountServer` es la composición del servidor que esta ola reutiliza.
