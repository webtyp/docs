# Conformidad de las operaciones — master plan
> **Status:** PENDING · 2026-11-18 · sin preguntas; lo ejecuta **Antigravity**: C1 lista para tomar; C2 espera que Claude publique `server`

Problema que se ve: al correr las pruebas de `server` (o `cloudflare`) con `go test -v` aparecen cuatro avisos («implementation does not support Operation yet», «router does not implement OperationRegistry»). No son fallas: son pruebas omitidas (`t.Skip`). Causa: `router/conformance.Run` mezcla dos contratos en una suite:
- **`router.Router`** (rutas, métodos, permisos de un servidor HTTP): lo implementan `server/httpd`, `cloudflare` y `router/mock`.
- **`router.OperationRegistry`** (registrar una operación por su nombre): ningún servidor HTTP lo implementa, a propósito. De lo exportado, solo `router/mock` cumple los dos.

Las cuatro cláusulas `op_route_*` (`router/conformance/conformance.go:189-195`) se omiten siempre en `httpd` y `cloudflare`. Las garantías de las operaciones ya se prueban donde corresponde: `mcp` en sus propias pruebas (`mcp/tests/rbac_test.go`, `harvest_plaintext_error_test.go`), `rpc` delega el permiso en el `Router` (que `Run` cubre), y el loopback de `crud` no es una frontera de seguridad (corre en el mismo proceso «actuando como» alguien; su `Requires` no hace nada a propósito). El defecto es solo de forma: una suite que no separa contratos y deja ruido en cada servidor HTTP.

Reglas: **api-harness** (regla 8: interfaces estrechas; «Publication»), **testing**, **zero-debt**. Protocolo de trabajo: el de [ANTIGRAVITY_WORK](ANTIGRAVITY_WORK_MASTER_PLAN.md) (marca de fase, repos reservados, nada se publica sin revisión del dueño, `gopush --no-cascade`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| C-D1 (dueño, 2026-11-18) | **Una suite por contrato, en `router/conformance`.** `Run(t, f)` prueba solo el contrato `Router` (se le quitan las cuatro cláusulas `op_route_*`). Nueva `RunOperations(t, f)` prueba `OperationRegistry` con la misma `Factory`: exige `f.New` y `f.ServeOp` (`t.Fatal` si falta, nunca `t.Skip`) y que el `Router` de `f.New` cumpla `router.OperationRegistry` (`t.Fatal` si no). Solo la corre quien implementa los dos contratos: hoy `router/mock` | `router` |

## Fases

| Fase | Repo | Qué (exacto) | Estado |
|---|---|---|---|
| C1 | `router` | Los cambios de abajo, «C1 paso a paso»; `gotest` en verde; marcar 🟢 para revisión | 🔴 lista para tomar (Antigravity) |
| C2 | `server`, `cloudflare` | Tras publicar C1: en cada uno, `go get webtyp.com/router@<versión de C1>`, `go mod tidy`, `gotest`, y comprobar que `go test -v -run Conformance ./...` ya no muestra ninguna línea `op_route_`. Sin cambios de código: ninguno usa `ServeOp` (verificado) | ⏸ `server` está reservado (trabajo sin publicar de Claude): esperar a que salga de la lista de ANTIGRAVITY_WORK; `cloudflare` se puede hacer tras C1 |

### C1 paso a paso (`webtyp/router`)

1. **`conformance/conformance.go`, `Run`:** borra estas cuatro líneas (hoy 189 y 193-195):
   ```go
   t.Run("op_route_surfaces_plain_text_error_body", func(t *testing.T) { opRouteSurfacesPlainTextErrorBody(t, f) })
   t.Run("op_route_reports_args_schema", func(t *testing.T) { opRouteReportsArgsSchema(t, f) })
   t.Run("op_route_is_invoked_by_name", func(t *testing.T) { opRouteIsInvokedByName(t, f) })
   t.Run("op_route_enforces_rbac", func(t *testing.T) { opRouteEnforcesRBAC(t, f) })
   ```
2. **Agrega `RunOperations`** justo después de `Run`:
   ```go
   // RunOperations executes every clause of the OperationRegistry contract: an operation registered by name is
   // served by that name, reports its argument schema, passes the same access gate as any route, and surfaces a
   // plain-text error body. Only an implementation that is both a Router and an OperationRegistry runs it (today
   // router/mock); an HTTP server runs Run alone.
   //
   //	conformance.RunOperations(t, factory) // factory.ServeOp is required here
   func RunOperations(t *testing.T, f Factory) {
   	if f.New == nil {
   		t.Fatal("conformance: Factory.New is required")
   	}
   	if f.ServeOp == nil {
   		t.Fatal("conformance: Factory.ServeOp is required by RunOperations")
   	}
   	t.Run("op_route_surfaces_plain_text_error_body", func(t *testing.T) { opRouteSurfacesPlainTextErrorBody(t, f) })
   	t.Run("op_route_reports_args_schema", func(t *testing.T) { opRouteReportsArgsSchema(t, f) })
   	t.Run("op_route_is_invoked_by_name", func(t *testing.T) { opRouteIsInvokedByName(t, f) })
   	t.Run("op_route_enforces_rbac", func(t *testing.T) { opRouteEnforcesRBAC(t, f) })
   }
   ```
3. **`opReg`** (hoy línea 611): cambia `t.Skip("router does not implement OperationRegistry")` por `t.Fatal("conformance: RunOperations needs a Router that is also a router.OperationRegistry")`.
4. **Borra las tres guardas** `if f.ServeOp == nil { t.Skip("implementation does not support Operation yet") }` (hoy líneas 641, 657 y 861, con sus tres líneas cada una): `RunOperations` ya lo comprobó.
5. **Comentario de `Factory.ServeOp`** (hoy líneas 153-159): reemplaza el párrafo «Optional. An implementation that does not yet implement Operation leaves this nil, and the Operation clauses skip with a loud reason instead of failing to compile.» por «Required by RunOperations and ignored by Run: an HTTP server leaves it nil.». Revisa también el comentario de `opReg` (líneas 602-605, «these clauses skip loudly if it does not») y déjalo diciendo que solo `RunOperations` las corre.
6. **`tests/conformance_test.go`:** saca la `Factory` de `TestMockConformance` a una función `mockFactory() conformance.Factory` (mismo contenido, `ServeOp` incluido), y deja:
   ```go
   func TestMockConformance(t *testing.T)           { conformance.Run(t, mockFactory()) }
   func TestMockOperationsConformance(t *testing.T) { conformance.RunOperations(t, mockFactory()) }
   ```
   Conserva el comentario que explica por qué el mock cumple el contrato y el de `ServeOp`.
7. **`docs/TESTS.md`:** agrega la fila `| [TestMockOperationsConformance](../tests/conformance_test.go) | the mock passes the OperationRegistry contract: an operation is served by name, reports its argument schema, passes the access gate and surfaces a plain-text error body |` y cambia la de `TestMockConformance` a `| the mock passes the Router contract |`.
8. **Documentación:** `grep -rn "ServeOp\|op_route\|conformance.Run" --include=*.md .` en `router`; actualiza el README o `docs/ARCHITECTURE.md` si nombran la suite (los `docs/PLAN_STAGE_*.md` viejos los trata la fase G2 de ANTIGRAVITY_WORK, no esta).
9. **Verificación:** `gotest` en verde; `go test -v -run 'TestMock' ./tests/` muestra las cuatro `op_route_*` como PASS (no SKIP); `grep -n "t.Skip" conformance/conformance.go` no muestra ninguna línea de operaciones.
10. **Publicación (solo tras aprobación del dueño):** `gopush --no-cascade 'feat!: router/conformance splits RunOperations from Run'`, y apunta la versión en la fila C1.

## Bloqueos

- 2026-11-18 · C2 en `server` espera que Claude publique su trabajo del inicio de sesión y de la observabilidad.

## Relación con otras olas

- ANTIGRAVITY_WORK: mismo protocolo; `router` ya no está reservado (publicado v0.5.6).
- ARCHITECTURE / SECURITY_OBSERVABILITY: `server` sale de la lista reservada cuando Claude lo publique; entonces C2.
