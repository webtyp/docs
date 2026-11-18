# Conformidad de las operaciones — master plan
> **Status:** PENDING · 2026-11-18 · sin preguntas; sigue C1 (`router`, Claude); C3–C5 esperan C1 publicado

Problema que se ve: al correr las pruebas de `server` con `-v` aparecen cuatro avisos («implementation does not support Operation yet», «router does not implement OperationRegistry»). No son fallas: son pruebas omitidas. Causa: `router/conformance.Run` mezcla dos contratos en una sola suite:
- **`router.Router`** (rutas, métodos, permisos de un servidor HTTP): lo implementan `server/httpd`, `cloudflare` y `router/mock`.
- **`router.OperationRegistry`** (registrar una operación por su nombre): ningún servidor HTTP lo implementa, a propósito; lo implementan los transportes que montan operaciones: `rpc` (`rpc/mount.go:134`), `mcp` (`mcp/harvest.go:97`), el loopback de `crud` (`crud/loopback.go:85`) y `router/mock` (`mock/router.go:145`).

Por eso las cuatro cláusulas `op_route_*` (`router/conformance/conformance.go:190-196`) siempre se omiten en `httpd` y `cloudflare`, y **solo se ejecutan contra `router/mock`**. Los transportes que llevan operaciones a producción nunca prueban que una operación protegida rechace al anónimo, ni que el cuerpo de error en texto plano llegue intacto. Este último defecto ya ocurrió en `mcp` sin que nada lo detectara (lo cuenta el comentario de `opRouteSurfacesPlainTextErrorBody`). La regla es que un contrato con varias implementaciones tenga una suite que todas pasen (**api-harness**, «Publication»): hoy la pasa solo la implementación falsa.

Reglas: **api-harness** (reglas 8 y publicación), **testing**, **zero-debt**, **framework-map**.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| C-D1 (dueño, 2026-11-18) | **Una suite por contrato.** `conformance.Run(t, Factory)` prueba solo el contrato `Router`: se quitan de ella las cuatro cláusulas `op_route_*` y el campo `Factory.ServeOp`. Nueva `conformance.RunOperations(t, OperationsFactory)` prueba el contrato `OperationRegistry`, sin omisiones: `OperationsFactory.New(t, Setup) (router.OperationRegistry, ServeOpFunc)` es obligatorio, con `ServeOpFunc func(op router.Operation, body []byte, who Identity) Response`, la misma forma que `ServeFunc` pero por nombre. Sus cláusulas: el esquema de argumentos (`Accepts`), la invocación por nombre, el permiso (protegida → 403 al anónimo, autorizada → 200) y el cuerpo de error en texto plano intacto | `router` |
| C-D2 | **La corren todas las implementaciones que atienden operaciones:** `router/mock`, `rpc`, `mcp` y el loopback de `crud`. `dbsync` (`client.go:373`) queda fuera: su `harvester` solo recolecta los nombres para sincronizar y descarta el manejador (`Operation(op, _ router.HandlerFunc)`): nunca atiende una operación. Si una implementación falla una cláusula, es un defecto real: se corrige en ese repo, nunca se omite la cláusula | `router`, `rpc`, `mcp`, `crud` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| C1 | `router` | C-D1: separar la suite; `router/tests` corre `Run` y `RunOperations` sobre `mock`; `docs/TESTS.md` y la descripción de la suite al día. Publicar con `gopush --no-cascade` (la cascada entraría en repos con trabajo sin publicar) | 🔴 Claude |
| C2 | `server`, `cloudflare` | Ninguno usa `ServeOp` (verificado): solo comprobar que `go test -v` ya no muestra omisiones `op_route_*` | 🔴 espera C1; `server` va con la publicación del inicio de sesión |
| C3 | `rpc` | `tests/operation_conformance_test.go`: `RunOperations` con el registro activo de `rpc` montado sobre `router/mock` (`ServeOp` = POST a la ruta de la operación) | 🔴 espera C1 |
| C4 | `mcp` | `RunOperations` con su `opRegistry` (`ServeOp` = `tools/call` de la herramienta) | 🔴 espera C1 |
| C5 | `crud` | `RunOperations` con el loopback (`ServeOp` = `Caller.Call` actuando como `who`) | 🔴 espera C1; `crud` está en el trabajo local de observabilidad: va con él |
| C6 | `devskills` | `framework-map` y la skill del dominio de transporte cuando exista: qué implementación corre qué suite | 🔴 espera C3–C5 |

## Bloqueos

- 2026-11-18 · `server` y `crud` tienen cambios sin publicar (inicio de sesión y observabilidad): C2 y C5 se hacen sobre esas copias y se publican con ellas.

## Relación con otras olas

- ARCHITECTURE: el inicio de sesión (D75–D82) y SECURITY_OBSERVABILITY O3 tocan `server` y `crud`; esta corrección se suma a su publicación.
- ANTIGRAVITY_WORK: `rpc` y `mcp` no están reservados, pero C3 y C4 los hace Claude: si una cláusula falla, corregirlo es arquitectura.
