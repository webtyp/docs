# Herramientas de desarrollo en el demo de un módulo — master plan
> **Status:** PENDING · 2026-10-30 · espera tu P1 (cómo llegan al demo la tabla de rutas y el diagrama)

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

**P1 — ¿Cómo llegan al demo la tabla de rutas y el diagrama?** A) **Fuentes explícitas**: las dos pantallas reciben de
dónde leer; en una app es HTTP (como hoy), en un demo es lo que el demo le pasa a `BuildMenu`, armado con las mismas
piezas del servidor (`loopback` aprende a guardar el permiso de cada operación y devuelve su tabla; el diagrama, con
`ddl.Capture` + `erd.ReadApp` sobre los pasos de migración del demo). B) **Servidor dentro del navegador**: el demo monta
sus módulos en un router en memoria con el mismo `appkit.MountServer` del servidor, y `fetch` responde `/_routes`,
`/_schema` y `/api/...` ahí mismo, sin red. **Recomiendo A**: el compilador obliga a cada demo a pasar sus fuentes y no
hay que interceptar `fetch` (B cambia `fetch`, `rpc` y `loopback` a la vez, y un error ahí rompe también las apps
reales); A reutiliza igual las piezas del servidor, sin su transporte.
```go
// hoy (access/web/browser/main.go): el explorador pide /_routes a un servidor que no existe
menu, err := layout_app.BuildMenu(ui.Screens(caller), ids, allowAll)
// A: el demo pasa lo que ya tiene; una app real no cambia (sin Tools = HTTP, como hoy)
menu, err := layout_app.BuildMenu(ui.Screens(caller), ids, allowAll,
	layout_app.LocalTools(loopback.Table(accessMod), migrate.Migrate))
// B: el demo arma su «servidor» en memoria y todo pasa por fetch
srv := appkit.InBrowser(appkit.ServerConfig{Operations: ops, Migrations: steps})
menu, err := layout_app.BuildMenu(ui.Screens(srv.Caller()), ids, allowAll)
```
Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P39 del roadmap) | El demo de un módulo muestra su API y su base de datos con lo que corre en el navegador; una app real sigue leyéndolas del servidor | según P1 |

## Fases

Ecosistema (Step 0): no existe una fuente local de rutas ni de diagrama; `router.MountIntrospection`, `ddl.Capture` y
`erd.ReadApp` se **reutilizan**; `loopback`, `layout/apiexplorer`, `layout/schemaview` y `layout/app` se **extienden**.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| X0 | docs | Contestar P1; consolidar | pendiente (tú) |
| X1 | `router` (`loopback`) | Guardar permiso, público y MFA de cada operación y devolver su tabla (según P1) | espera P1 |
| X2 | `layout` (`apiexplorer`, `schemaview`, `app`) | Las pantallas leen de una fuente; `BuildMenu` recibe las del demo | espera X1 |
| X3 | cada módulo con demo | El demo pasa sus fuentes; captura del explorador y del diagrama | espera X2 |

## Bloqueos

- Ninguno fuera de P1.

## Relación con otras olas

- MVP_DEPLOYMENT_ROADMAP (`mjosefa-cms`): D23 revisa cada módulo en su demo; P39 de allí abrió esta ola.
- APP_COMPOSITION: `appkit.MountServer` es la composición del servidor que esta ola reutiliza.
