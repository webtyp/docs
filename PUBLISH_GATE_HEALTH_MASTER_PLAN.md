# Gate de publicación en rojo — master plan
> **Status:** PENDING · 2026-11-19 — sin preguntas; 10 repos fallan el gate de `gopush` por causas anteriores a `widget` v0.6.50 / `layout` v0.3.134 (que pasaron todo); 0 de 10 corregidos; espera un agente que los ejecute.

## Preguntas abiertas
Ninguna.

## Qué pasó
Al publicar `widget` v0.6.50 y `layout` v0.3.134 (ScrollCue del menú), el cascade de `gopush` intentó actualizar a los dependientes y **10 no pasaron su gate** (`vet`, `style`, `tests`). Ninguno falla por el cambio publicado: tenían el error antes. Mientras sigan así, `gopush` no los actualiza y se quedan en versiones viejas de `widget` y `layout`. Los logs del gate están en `/tmp/gopush-gate-<repo>.log` (se borran al reiniciar; el detalle de abajo basta para reproducir con `gotest` dentro del repo).

## Cómo trabajar cada repo (receta común)
1. Entra al repo, corre `gotest` y confirma que ves el mismo error de la tabla.
2. **Causa A (versiones fijadas viejas):** sube la dependencia al último tag publicado (`go get webtyp.com/<dep>@latest`, `go mod tidy`) y corrige lo que deje de compilar. Si el `go.mod` de una librería fija un tag que ya no compila con otra que fija ella misma, el defecto está en esa librería: publícala corregida primero (**bug-fix-flow**), no lo parches en el consumidor.
3. **Causa B (`U1000`, código sin uso):** borra el código muerto (**zero-debt**: ni `_ =` ni comentarios para silenciar). **Antes de borrar, comprueba con las mismas etiquetas de build que usa el gate** (`wasm`, `browser`, `dev`): algo sin uso en una etiqueta puede usarse en otra. Si de verdad lo usa otra etiqueta, el defecto es del gate, no del repo: anótalo en `Bloqueos`.
4. **Causa C (reglas de estilo):** aplica la corrección que el aviso indica; sin `//nolint`.
5. `gotest` verde → `gopush` según **codejob** («gopush and local `replace`»). Un repo publicado sale de la tabla y pasa al párrafo de cerrados.

## Cerrado (publicado, sin pendientes)
Ninguno todavía.

## Fases
Una fila por repo; carpeta entre paréntesis. Las filas A no compilan; las B y C compilan y solo fallan `style`. Tags actuales a los que subir: `orm` v0.12.42 · `server` v0.2.133 · `storage` v0.2.8 · `auth` v0.0.123 (releer con `git describe --tags` en cada repo antes de fijar). Las fases son independientes entre sí.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| G1 | `access` (`webtyp/access`) | A: fija `auth v0.0.112`, que no compila con el `crud`/`storage` actuales (`authority/definition_gen.go:554`: `cannot use func(a crud.CurrentUser, filter model.Model, text string) ([]storage.Condition, error)`). `auth` ya está sano en **v0.0.123**: basta subir el pin. B: `errors.go:29-42` sin uso (`msgDecode`, `decodeError`, `writeError`, `statusOf`) y `tests/setup_test.go:113-147` (`mockOpRegistry`, `mockRoute`). También falla `browser`. | 🔴 |
| G3 | `iam` (`veltylabs/iam`) | A: fija `orm v0.12.26` (`db.go:73,173`: `have (storage.Result, error) want (error)`), `storage v0.1.18`, `auth v0.0.80`, `form v0.4.52`; y `server v0.2.115` (`httpd/adapter.go:16`: `*httpContext` no implementa `router.Context`, falta `RawQuery`). Subir `orm`, `storage`, `auth`, `form`, `server` a sus últimos tags. | 🔴 |
| G4 | `misitio` (`veltylabs/misitio`) | A: mismos dos errores que `iam` (`orm v0.12.26`, `server v0.2.115`); fija además `auth v0.0.111`, `form v0.4.61`, `storage v0.1.18`. | 🔴 |
| G5 | `sitetheme` (`veltylabs/sitetheme`) | A: `orm v0.12.26` y `storage v0.1.19` como indirectas; mismo error de `orm`. Subir. | 🔴 |
| G6 | `appointment_booking` | B: `tests/mocks_test.go:145` `zoneSantiago`; `ui/staffpicker.go:88` `refreshAfterFilterChange`, `:114` `nameOf`; `ui/step_slot.go:49` `show`. | 🔴 |
| G7 | `payment_manager` | B: `facts.go:76` `refundFact`; `module.go:44` campo `retURL`, `:97` `isCounterMethod`, `:383` `listPaymentsPaged`, `:393` `create`; `repository.go:42` `paymentAnyTenant`, `:61` `paymentsPaged`, `:97` `refundAnyTenant`; `ui/paymentstep.go:38` `clsStepInput`. | 🔴 |
| G8 | `agenteval` | B: `tests/scenario_test.go:184` `bytesContains`, `:188` `findSubstr`; `ui/panel.go:146` `(*Panel).onEvent`. | 🔴 |
| G9 | `avatar` | C: `engine.go:126` y `:148` usar `switch` con etiqueta; `check.go:42,44,76` usar `fmt.Fprintf` en vez de `WriteString(fmt.Sprintf(...))`; `tests/avatarc_test.go:9` reemplazar `io/ioutil` por `io`/`os`. | 🔴 |
| G10 | `pwa` | C: `naming.go:50` simplificar con De Morgan; `tests/naming_test.go:21` expresiones idénticas a ambos lados de `!=` (la comprobación no comprueba nada: revisar qué debía comparar). | 🔴 |
| G11 | `goflare-demo` | B: `modules/contact/sendEmail.go:10` `sendEmail` sin uso. | 🔴 |

## Bloqueos
- 2026-11-19 · El cascade **saltó** otros repos por tener cambios sin guardar o una sesión de `codejob` activa (`mjosefa-cms`, `app`, `appkit`, `auth/tests`, `dbsync`, `clinical_encounter`; con commit de solo dependencias: `mjosefa-cote`, `room_layout`, `dbsync/tests`). No son errores de gate: se actualizarán solos en el próximo `gopush` de `widget`/`layout` cuando su dueño cierre su trabajo. **No tocar** los que tienen sesión de `codejob`.
- 2026-11-19 · Además de estos 10, el directorio `/tmp` guarda logs de gates de otros ~50 repos con `style ❌`, pero son de corridas anteriores; no se sabe cuáles siguen rojos. Lo cubre `STDLIB_MIGRATION` S2 (D6, errores sin traducir) y `CODE_GUARD`; este master solo trata los 10 de arriba.

## Relación con otras olas
- `CODE_GUARD_MASTER_PLAN`: hace cumplir las skills por línea y deja `gotest` en rojo mientras no se migre; la causa C puede solaparse con sus reglas.
- `STDLIB_MIGRATION_MASTER_PLAN`: S2 migra todos los repos; si toca uno de estos, coordina para no editarlo dos veces.
- `ARCHITECTURE_MASTER_PLAN`: ningún defecto de librería encontrado (todo es pin viejo o código sin uso en los consumidores); si al subir pines aparece una rotura real de contrato, usa **bug-fix-flow** y anótalo allí.
