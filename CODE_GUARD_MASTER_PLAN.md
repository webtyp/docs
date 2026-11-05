# Guardián de código webtyp — master plan
> **Status:** IN PROGRESS · 2026-11-05 · espera P13 (gotest en rojo mientras no se migre); fase 0 y fase 1 (`gocheck`) pueden empezar, en local

Objetivo: **una sola herramienta**, `gocheck`, que dice con el mínimo de tokens **qué línea corregir y por qué**. La usan
igual el dev (warning en la TUI de `webtyp dev` al guardar), el agente (MCP de webtyp y `gotest`) y la terminal.

**Búsqueda en el ecosistema:** `devflow/gotest` corre `go vet` y parsea su texto (`devflow/gotest.go:111-153`); `langc`,
`app/routescan`, `app/layoutscan` y `depfind` parsean con `go/ast` cada uno por su lado; `app` ya vigila archivos y tiene
TUI+MCP (se reusa). Ningún repo usa `golang.org/x/tools`. Nada valida las reglas de webtyp → **nuevo** (`gocheck`).
**Prior art** (leer en fase 0; hasta entonces «unverified»): `go/analysis` https://pkg.go.dev/golang.org/x/tools/go/analysis ·
staticcheck https://staticcheck.dev/docs/checks · golangci-lint https://golangci-lint.run · revive/ruleguard.
**Peso medido de cada paquete:** `gocheck/docs/STDLIB_WEIGHT.md`.

## Preguntas abiertas

**P13 — Mientras STDLIB_MIGRATION no termine, `gotest` fallará en casi todos los repos.** Con D4 (gotest corre gocheck y
falla si hay hallazgos), `nostdlib`, `ifaceeq` y `nomaps` marcarán cientos de líneas (461 archivos con `fmt`, 178 `errors.Is`)
que la otra ola todavía no migró. A) Fase 2 (`gotest`) espera a que STDLIB_MIGRATION cierre; mientras, `gocheck` corre suelto y
en la TUI. B) Fase 2 ya, y todo repo queda en rojo hasta migrarse (presión para migrar, pero bloquea cualquier otro trabajo).
**Recomiendo A**: las demás olas abiertas no pueden quedar bloqueadas por esta; el agente igual ve los hallazgos en la TUI/MCP. Respuesta:

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Sin motor propio: reglas como `*analysis.Analyzer` de `go/analysis`; reusar los existentes (vet, `unusedfunc`…) y escribir solo las reglas de webtyp | `gocheck` |
| D3 (P1) | `app` la consume: al guardar un `.go` revisa ese paquete y muestra warning en la TUI con línea y motivo; lo mismo por el MCP | `app` |
| D4 (P4) | `gotest` sin argumentos corre siempre vet → gocheck → tests y falla si hay hallazgos; sin flag | `devflow` |
| D5 (P6) | Sin excepciones en el código: si una regla se equivoca, se arregla la regla | `gocheck` |
| D6 (P1) | README de `gocheck`: en palabras simples cada herramienta de Go que corre y cada regla webtyp: qué detecta, por qué ayuda, ejemplo malo/bueno | `gocheck` |
| D7 | Salida: una línea `ruta:línea regla mensaje → acción` vía `fmt.Err`/`lang.Text`; `-json` para MCP; cada regla enlaza su skill | `gocheck` |
| D8 | Reglas iniciales: `unused`, `perf` (concat en WriteString), `jsonlyindom` (`syscall/js` solo en `dom`/`js`), `interfacesfile`, `modelfile`, `nostdlib` (D16), `ifaceeq` (D20), `nomaps` (D23) | `gocheck` |
| D9 | Rendimiento: una carga por paquete, analizadores en paralelo, caché por hash de archivo | `gocheck` |
| D10 | Elimina el parseo de texto de vet en `gotest.go:127-153` y los ítems manuales equivalentes de **plan-review** | `devflow`, devskills |
| D16 (P8,P10) | `nostdlib` en **todo** el código (navegador, Worker `wasm` con límite de tamaño, servidor): una sola forma, sin código doble. Prohibidos (medido con TinyGo): `fmt`, `encoding/json`, `reflect`, `strconv`. Permitidos: `strings`, `errors` (salvo D20). **No** se crea `webtyp/errors` | `gocheck` |
| D20 (P9) | Regla `ifaceeq` en todo el código: prohíbe comparar dos interfaces (`err == X`) y `errors.Is`; acción: `errors.AsType`. Solo contra centinelas declarados en webtyp/veltylabs (D24). El cambio de `fmt.ErrType` y la migración: STDLIB_MIGRATION | `gocheck` |
| D24 (P12) | `ifaceeq` mira el paquete del centinela: los de la stdlib o externos (`os.ErrNotExist`, `io.EOF`, `pgx.ErrNoRows`) se pueden comparar; los nuestros (`keyring.ErrNotFound`…) se marcan | `gocheck` |
| D23 | Regla `nomaps` en todo el código: ni tipos `map` ni el paquete `maps` (TinyGo los maneja mal y suman runtime); acción: slice de pares o `switch` | `gocheck` |
| D17 | Toda regla de peso cita su medición en `docs/STDLIB_WEIGHT.md`; un paquete nuevo entra a la lista solo con su programa en `benchmark/weight/` | `gocheck` |
| D18 | `gonew` crea en `cdvelop` si no recibe `-owner`: debe deducir el dueño de los repos vecinos, como ya deduce la ruta del módulo (`gonew.go:206`) | `devflow` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 0 | — | Leer los 4 enlaces de prior art | pendiente |
| W | `gocheck` | Medición de peso (`benchmark/weight/`, `docs/STDLIB_WEIGHT.md`) | hecha local, sin publicar |
| 1 | `gocheck` | D1, D5–D9, D16, D17, D20, D23 + tests con paquetes de ejemplo | espera fase 0 |
| 2 | `devflow` | D4, D10, D18 + herramienta MCP de gotest | espera 1 |
| 3 | `app` | D3 (TUI al guardar + MCP) | espera 1 |
| 4 | devskills | `wasm`: lista medida de D16 y la regla D20; `agents-md`/`plan-review`/`testing`: «gotest verde» incluye gocheck | espera 1 |
| 5 | `mjosefa-cms` | Correr sobre la app de referencia, limpiar hallazgos, ver el warning en la TUI | espera 3 |

## Bloqueos
- 2026-11-05 · P13 espera al usuario (solo afecta a la fase 2).

## Relación con otras olas
- STDLIB_MIGRATION ([STDLIB_MIGRATION_MASTER_PLAN.md](STDLIB_MIGRATION_MASTER_PLAN.md)): cambia el código que las reglas de peso marcan; ver P13.
- Todas las olas abiertas se benefician (el agente ejecutor deja de dejar basura).
