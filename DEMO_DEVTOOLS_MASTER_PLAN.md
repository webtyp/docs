# Herramientas de desarrollo en el demo de un módulo — master plan
> **Status:** IN PROGRESS · 2026-10-30 · sin preguntas; X1–X2 cerradas; X3: `access` (v0.0.41), `agreement_manager` (v0.3.28), `appointment_booking` (v0.3.67) y `mjosefa-cms` (v0.0.100) cerradas; 2 en Jules (`chat_room`, `clinical_encounter`)

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
| X1a | `router` | `router.Table(routes, policy)` | cerrada v0.4.17 |
| X1b | `rpc` | `rpc.Routes(prefix, mods...)` | cerrada (publicada) |
| X1c | `erd` | `erd.ReadSteps(steps...)` | cerrada v0.0.14 |
| X1d | `appkit` | Usa `erd.ReadSteps` y borra `groupsOf` | cerrada |
| X2 | `layout` | `app.Tools`, `LocalTools`, fuentes, «Probar» solo con servidor | cerrada v0.3.99 |
| X3 | `mjosefa-cms` (con M5) y los demos de `access`, `agreement_manager`, `appointment_booking`, `chat_room`, `clinical_encounter` | Declaran sus herramientas; captura del explorador y del diagrama en la revisión | `access` v0.0.41, `agreement_manager` v0.3.28, `appointment_booking` v0.3.67, `mjosefa-cms` v0.0.100 cerradas; 2 repos en Jules |

## Bloqueos

- Ninguno fuera de P1.

## Relación con otras olas

- MVP_DEPLOYMENT_ROADMAP (`mjosefa-cms`): D23 revisa cada módulo en su demo; P39 de allí abrió esta ola.
- APP_COMPOSITION: `appkit.MountServer` es la composición del servidor que esta ola reutiliza.
