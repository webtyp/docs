# `iam` sobre `appkit`: la pieza «aplicación» para un Worker con login por correo — master plan
> **Status:** PENDING · 2026-11-03 · esperando P1–P3 del dueño (diseño)

**De dónde viene:** APP_COMPOSITION (cerrada 2026-11-03) dejó `mjosefa-cms` como una declaración
(`config.App`, `appkit` v0.2.2). `iam` no pudo seguir: tiene 2.027 líneas en `config/` y es otro tipo de app.

## Preguntas abiertas

**P1 — `appkit` para un Worker.** `iam` corre como Worker de Cloudflare (`iam/edge/main.go`); `appkit.Server` es un
servidor `httpd` (`appkit/serve.go`, `!wasm`). A) `appkit.Edge(config.App)` que monta lo mismo sobre `edge.Serve`
· B) `iam` sigue con su `edge/main.go` y solo usa `appkit.Module`/`Find`. Respuesta:

**P2 — Login por correo.** `iam` entra por Google + passkey con una cookie compartida entre subdominios
(`iam/config/auth.go`: `SSOCookieDomain = ".velty.cl"`). A) `appkit.LoginByEmail{…}` junto a `LoginByRUT` · B) otro
diseño. Respuesta:

**P3 — La UI fuera del Worker.** `appkit` arrastra `dom`, `form` y `html` (`appkit/browser.go` sin etiqueta y
`view.Screen.Component dom.Component`, `view/screen.go:43`); `iam/tests/layering_test.go:56` lo prohíbe en el Worker.
A) mover la UI de `appkit` a un subpaquete y tipar `Component` sin `dom` · B) otra separación. Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| Z | `iam` | `config.App`; captura de cada pantalla; prueba que recorre el menú como admin y sin permisos | ⚪ tras P1–P3 |

## Bloqueos

- 2026-10-29 · `view` importa `dom` desde R0c: todo binario que importe `view` arrastra `dom` (P3).

## Relación con otras olas

- APP_COMPOSITION (cerrada): `appkit.App`, `Module{Server,Browser}`, `LoginByRUT`, `migrate/main.go` generado.
- SELFHOST_DEPLOY: D14 ya usa el binario `migrate`.
