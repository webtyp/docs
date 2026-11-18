# Mapa del framework — master plan
> **Status:** PENDING · 2026-11-18 · espera respuestas del dueño a P1–P5; nada se construye antes · bloquea D75 de [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md) (inicio de sesión), que espera la skill de su dominio

Problema que se ve: un agente llega sin saber qué hace cada uno de los 184 repos (147 `webtyp`, 37 `veltylabs`), ni en qué orden dependen, ni qué skill es dueña de cada uno. Entonces supone, pone código en la pieza equivocada y el dueño tiene que revisarlo todo. Caso real (2026-11-18): el modo de entrada `trusted_ip` quedó dentro de `appkit` (D68), que debía solo conectar piezas, y ninguna skill dice que el dominio «inicio de sesión y permisos» lo forman `user`, `auth`, `rbac` y `access`.
Objetivo: que el agente sepa, desde la skill del dominio donde trabaja, qué repos lo forman, la única responsabilidad de cada uno y de qué depende. Que eso **no se escriba a mano**, para que no envejezca, y que una pieza nueva no se publique sin declarar dónde va.
Reglas: **devskills** (una skill = un dominio, ≤150 líneas), **documentation**, **api-design** (puerta 4), **zero-debt**.

## Preguntas abiertas

**P1 — Dónde lee el agente el mapa.**
A) Una skill índice corta, `framework-map`, con dominio → skill dueña y el orden de capas. Además, cada skill de dominio lleva su tabla de piezas (repo | única responsabilidad | depende de), generada entre marcas, como el bloque base de **agents-md**.
B) Un solo archivo generado, `webtyp/docs/FRAMEWORK_MAP.md`, con los 184 repos, y una skill corta que lo enlaza.
C) Las tablas escritas a mano en las skills.
¿Cuál evita el acoplamiento y es más mantenible? A: cada dominio ve solo sus repos, la skill que el agente ya cargó le trae su mapa, y el generador es el único que escribe. B obliga a leer 184 filas para encontrar 5. C envejece en semanas, que es lo que pasó. **Recomiendo A.**
```markdown
<!-- webtyp:pieces — generado por `devskills map`; no editar -->
| Repo | Única responsabilidad | Depende de (webtyp) |
| `auth` | los modos de entrada (`auth.Authenticator`), la sesión y las identidades | `user`, `orm`, `crud`, `webauthn` |
<!-- /webtyp:pieces -->
```
Respuesta:

**P2 — De dónde sale «la única responsabilidad» de cada repo.** Hoy solo 20 de 184 README tienen «Why This Exists».
A) La primera frase de la sección `## Why This Exists` del README, que la plantilla de **documentation** ya exige. Se escribe en los 164 repos que no la tienen, y el gate la exige.
B) Un archivo nuevo por repo (`docs/PIECE.md`, una línea).
C) La descripción del repo en GitHub.
¿Cuál evita el acoplamiento y es más mantenible? A: no agrega archivos, y el README ya es el primer documento que se lee. B es un segundo lugar que contradice al README. C vive fuera del repo y sin gate. **Recomiendo A**: una frase con «hace X», que deja ver si un repo tiene dos responsabilidades. Respuesta:

**P3 — Dónde se declara a qué dominio pertenece un repo** (la única decisión humana del mapa).
A) En la skill del dominio: una lista de sus repos. El generador falla si un repo no aparece en ninguna skill o aparece en dos.
B) En cada repo, con una línea en su README que diga «Dominio: sign-in».
¿Cuál evita el acoplamiento y es más mantenible? A: los repos no conocen las skills, y la pertenencia se revisa en un solo lugar por dominio. B acopla cada repo a los nombres de las skills. **Recomiendo A.** Respuesta:

**P4 — Qué herramienta genera y dónde se hace cumplir.**
A) `devskills map` (en `devskills`, dueño de las skills) genera las tablas con el grafo que ya existe en `devflow/cascade.go:57` (`BuildDependentGraph`, `topologicalSort`), exportado sin copiarlo. `devskills -f` se niega a sincronizar si una tabla quedó vieja, y `gonew` pide el dominio al crear un repo.
B) Un comando nuevo en `devflow`, y que `gopush` revise el mapa en cada publicación.
¿Cuál evita el acoplamiento y es más mantenible? A: escribe las skills quien es dueño de las skills, y el grafo vive una sola vez en `devflow`. B hace que cada `gopush` de cualquier repo dependa del árbol de skills y lo vuelve más lento. **Recomiendo A.** Respuesta:

**P5 — Qué cuenta como «depende de».** Hoy el grafo lee el `go.mod` completo, con las indirectas: `access` aparecería dependiendo de 46 repos.
A) Solo los imports directos del código que no es de prueba (`go list -deps=false`, filtrado a `webtyp.com/` y `github.com/veltylabs/`).
B) El `require` directo del `go.mod` (sin `// indirect`).
¿Cuál evita el acoplamiento y es más mantenible? A muestra lo que el código usa de verdad, y una dependencia que sobra salta a la vista. B incluye lo que solo usan las pruebas. **Recomiendo A.** Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (dueño, 2026-11-18) | `appkit` es el único lugar donde una app configura, pero no implementa ninguna pieza: arma lo que la app eligió. Lo que hoy implementa por su cuenta (el modo `trusted_ip`) sale a la pieza dueña | ARCHITECTURE D75 |
| D2 (dueño, 2026-11-18) | El mapa se genera, nunca se escribe a mano, y cada fase se revisa antes de construir la siguiente: el dueño no confía en el criterio del agente para esto | este master |

## Fases

Cada fase termina con 🔵 del dueño antes de la siguiente. Nada se publica sin su revisión.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M0 | `docs` (este master) | 🧠 Propuesta de la lista de dominios y la asignación de cada uno de los 184 repos, en una tabla que revisa el dueño. Los dominios salen de las skills que existen; los que falten se proponen con nombre | 🔴 espera P1–P5 |
| M1 | `devflow` | Exportar el grafo (P5) desde `cascade.go`, sin duplicarlo | 🔴 |
| M2 | `devskills` | `devskills map`: lee la lista de cada skill (P3), la frase del README (P2) y el grafo (M1), y escribe las tablas (P1). Falla con el nombre del repo huérfano, duplicado o sin frase | 🔴 |
| M3 | 164 repos | Escribir la frase de responsabilidad que falta (P2). Por capa, de abajo hacia arriba; un repo cuya frase necesite «y» se anota como candidato a dividirse en ARCHITECTURE, no se arregla aquí | 🔴 |
| M4 | `devskills` | La skill `framework-map` y las tablas en cada skill de dominio; skills nuevas para los dominios sin dueño (primero `sign-in`: `user`, `auth`, `rbac`, `access`) | 🔴 |
| M5 | `devskills`, `devflow` (`gonew`) | Los gates de P4 | 🔴 |

## Bloqueos

- 2026-11-18 · Todo espera P1–P5 del dueño.

## Relación con otras olas

- ARCHITECTURE: D75 (inicio de sesión como una pieza) espera la skill `sign-in` (M4). Los repos con dos responsabilidades que encuentre M3 pasan allí como preguntas.
- CODE_GUARD: sus gates por línea son la misma idea a nivel de código; no se mezclan.
