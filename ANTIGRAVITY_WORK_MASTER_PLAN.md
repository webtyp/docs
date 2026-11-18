# Trabajo delegado a Antigravity — master plan
> **Status:** IN PROGRESS · 2026-11-18 · fases G1 y G2 listas para Antigravity; G3 espera que Claude publique el inicio de sesión (ARCHITECTURE D75–D82)

Reparto (dueño, 2026-11-18): Claude hace lo de arquitectura (diseño, contratos, decisiones); Antigravity hace el trabajo largo que no necesita decisiones de diseño. Este master es la hoja de ruta de Antigravity: cada fase dice exactamente qué hacer, cómo verificarlo y qué no tocar.

## Protocolo para Antigravity (leer antes de cada fase)

1. **Lee las skills** antes de tocar un repo: [framework-map](https://github.com/webtyp/devskills/blob/main/skills/framework-map/SKILL.md) (qué hace cada repo) y las que la fase nombra. Las skills son la única fuente de cómo se trabaja.
2. **Marca la fase** que tomas en su celda `Estado`: `[Antigravity · AAAA-MM-DD HH:MM]` (regla de **master-plan** «Varios agentes»); actualízala en cada pasada y bórrala al cerrar.
3. **Nunca toques los repos reservados** (abajo): Claude tiene cambios sin publicar en ellos.
4. **Nada se publica sin revisión del dueño.** Trabaja en local, deja `gotest` en verde en cada repo y marca la fila 🟢 «listo para revisar» con la lista de repos tocados. Cuando el dueño apruebe, publica **un repo por vez con `gopush --no-cascade 'mensaje'`**: la cascada entraría en los repos reservados y subiría sus `go.mod` con `replace` locales. Nunca `git commit`/`git push`/`git tag` a mano.
5. **Si algo pide una decisión** (un texto ambiguo, una regla que no aplica, un error que no es de traducción), no lo decidas: escríbelo en «Preguntas abiertas» de este master, con el repo y la línea, y sigue con el siguiente repo.
6. **El mapa al día:** si un cambio altera qué hace un repo o sus imports, actualiza su fila en `framework-map` y la skill de su dominio en el mismo cambio. Una traducción o un documento no los altera.

**Repos reservados (no tocar hasta que G3 se desbloquee):** `model`, `router`, `view`, `auth`, `appkit`, `appkitc`, `layout`, `devskills`, `server`, `crud`, `notify`, `securityevent`, `veltylabs/mjosefa-cms`.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| G-D1 (dueño, 2026-11-18) | Claude hace arquitectura; Antigravity, el trabajo largo sin decisiones de diseño, con este master como hoja de ruta | este master |

## Fases

| Fase | Repos | Qué (exacto) | Estado |
|---|---|---|---|
| G1 | todos los de `webtyp/*` y `veltylabs/modules/*` **menos los reservados** | **Traducciones completas.** En cada repo que tenga `lang.json` (o textos con `fmt.Err`/`lang.Text`): (1) `langc sync .`; (2) completa cada valor vacío `""` en todos los idiomas de su `languages`, con frases reales (nunca la clave copiada en sí misma); (3) corrige cada aviso de reutilización como dice el propio aviso (compón con `fmt.Err("<parte común>", resto)`, o cambia la redacción para que no comparta 2+ palabras) y cada error de «placeholder» o «unchanged» (pasa los códigos y ejemplos como dato, una variable, no como literal); (4) repite `langc sync .` hasta que no imprima nada; (5) `gotest` en verde. Reglas: **translations**, **langc**, **api-harness** «Messages» (mensajes de una línea, sin nombres de código como palabras). Empieza por los que `webtyp dev` ya reporta: `veltylabs/modules/service_catalog` (12), `webtyp/externalcalls` (7), `webtyp/access` (1). Registra en esta fila cada repo terminado | 🔴 lista para tomar |
| G2 | `webtyp/docs`, repos con planes viejos | **Documentación que miente.** (1) Reescribe [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md) desde el código actual, en inglés (regla de **documentation**: contexto, `flowchart TD`, recorrido numerado): hoy describe `router/loopback`, `MountAPI` desde módulos y `ops.go`, que ya no existen; el recorrido real está en el diagrama de [framework-map](https://github.com/webtyp/devskills/blob/main/skills/framework-map/SKILL.md) y en `crud/mount.go`, `rpc/`, `server/httpd/enforce.go` (léelos, no los cambies). (2) Busca planes obsoletos en los repos no reservados: `find . -path '*/docs/PLAN*.md' -not -name 'LAST_PLAN_EXECUTED.md'`; por cada uno, comprueba contra el código si su trabajo ya está hecho o fue reemplazado (por ejemplo `access/docs/PLAN_STAGE_5_entry_screens_are_data.md` nombra `view.EntryScreens`, que ya no existe); si está obsoleto, bórralo; si no, anótalo aquí como pregunta. Regla: **agents-workflow** «Obsolete plan files» | 🔴 lista para tomar |
| G3 | los 29 repos de STDLIB D6 | **Errores traducibles** (STDLIB_MIGRATION D6, junto con S1/S2). Espera: que Claude publique los repos reservados y que STDLIB S1 (`fmt`) tenga plan | ⏸ bloqueada |

## Bloqueos

- 2026-11-18 · G3 espera la publicación del inicio de sesión (ARCHITECTURE D75–D82) y el plan de STDLIB S1.

## Relación con otras olas

- ARCHITECTURE: Claude publica D75–D82 tras la revisión del dueño; entonces se liberan los repos reservados.
- SECURITY_OBSERVABILITY: Claude hace O3, O4 y O3b (arquitectura).
- STDLIB_MIGRATION: G3 ejecuta su D6 cuando S1 tenga plan.
