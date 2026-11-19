# Señal de errores, TUI compacta y `logs.log` crítico — master plan
> **Status:** IN PROGRESS · 2026-11-19 · D1–D7 cerradas (sin preguntas abiertas); listo para ejecutar L1–L5 en local con `replace`

Problema que se ve:
- En `webtyp dev`, los errores se duplican (`ASSETS`/`SERVER` + `WATCH`), se repiten por línea (`undefined: X` × 8) y `go.mod`/`go.sum` desfasado detiene el hot-reload (D1–D3).
- Los manejadores de `BUILD` (`SERVER`, `CLIENT`, `BROWSER`, `ASSETS`, `SSR`, `WATCH`, `DEFINITIONC`, `IMAGE`, `LANG`, `DDL`, `SVG_LEAK`, `CONFIG`, `GOMOD`) acumulan líneas por cada guardado en vez de mantener 1 línea viva con `OperationID` + `tui.LogOpen`/`tui.LogClose` (D4–D5).
- Al cerrar la app, los eventos críticos de seguridad/alertas (`appkit/backend/security_watch.go:105`) se pierden si no hay canal configurado; falta persistir solo lo crítico en `logs.log` con poda de 7 días / 1 MB (D6–D7).

Reglas: **console-logs**, **api-design**, **bug-fix-flow**, **zero-debt**, **isomorphic-libraries**.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1-A) | **Sin eco duplicado en `WATCH`:** cuando un handler (`ASSETS`, `SERVER`, `CLIENT`) ya logueó su error, devuelve `devwatch.ErrReported` y `WATCH` no repite el texto | `devwatch`, `app` |
| D2 (P2-A) | **Compactar errores repetidos de compilación:** agrupar por archivo + mensaje con ruta `~` (`~/Dev/…/file.go:181 (+7 líneas: …): msg`) | `gobuild`, `sitec`, `app` |
| D3 (P3-A) | **Auto `go mod tidy` en `webtyp dev`:** al detectar `updates to go.mod needed` / `missing go.sum entry`, ejecutar `go mod tidy` en el módulo afectado (máx. 1 reintento por cambio) y loguear `GOMOD: ran go mod tidy in <dir>` | `devflow`, `app` |
| D4 (P4-A) | **1 línea fija en TUI para todos los manejadores de `BUILD`** (salvo `SECURITY`, donde cada incidente es discreto): `HeadlessTUI` asigna `OperationID` por defecto (`"build:" + handlerName`) si el handler no trae uno propio, y los manejadores con pasos (`SERVER`, `CLIENT`, `ASSETS`/`SSR`, `BROWSER`, `DEFINITIONC`) usan `tui.LogOpen`/`tui.LogClose` para animar la misma línea | `server`, `devbrowser`, `app` |
| D5 (P5-A) | **Errores resueltos desaparecen de la TUI:** al volver a verde (`tui.LogClose`), la línea única del manejador reemplaza el error anterior; lo crítico persiste en `logs.log` y el historial reciente sigue en `app_get_logs` | `server`, `app` |
| D6 (P6-A) | **Suscriptor por defecto en `appkit/backend` (`!wasm`):** escucha `notify.TopicAlert` y escribe siempre en `<root>/logs.log` (dev y prod self-host), haya o no email/WhatsApp configurado | `appkit` |
| D7 (P7-A) | **Retención y formato de `logs.log`:** solo `Warning`/`Error` críticos, poda automática a últimos **7 días** y tope duro de **1 MB**, 1 línea plana ISO-8601 con campos `clave=valor` | `appkit` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| L1 | `devwatch`, `app` | D1: sentinel `devwatch.ErrReported` para evitar eco duplicado entre `ASSETS`/`SERVER`/`CLIENT` y `WATCH` | 🔴 listo para ejecutar |
| L2 | `gobuild`, `sitec`, `app` | D2: compactador de errores de compilación por archivo + mensaje con `~` y líneas agrupadas | 🔴 listo para ejecutar |
| L3 | `devflow`, `app` | D3: detección y `go mod tidy` automático (1 reintento) en `webtyp dev` | 🔴 listo para ejecutar |
| L4 | `server`, `devbrowser`, `app` | D4–D5: `OperationID` fijo para todos los manejadores de `BUILD` (excepto `SECURITY`) + `tui.LogOpen`/`tui.LogClose` en hot-reload | 🔴 listo para ejecutar |
| L5 | `appkit` | D6–D7: escritor crítico `logs.log` (`!wasm`) sobre `notify.TopicAlert` con poda 7 días / 1 MB | 🔴 listo para ejecutar |

## Bloqueos

Ninguno.

## Relación con otras olas

- Complementa `TUI_SIGNAL` (cerrado), `SECURITY_OBSERVABILITY` (cerrado 2026-11-19) y la skill **console-logs**.
