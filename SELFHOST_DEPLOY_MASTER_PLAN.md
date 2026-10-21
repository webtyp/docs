# Despliegue continuo en servidor propio — master plan
> **Status:** PENDING · 2026-10-21 · espera tus respuestas a P1–P8; P4 (sistema operativo del servidor) bloquea todo lo demás. Nada despachado.

**Disparador:** `mjosefa-cms` corre en un servidor propio dentro de la red del consultorio y depende de ~11 módulos
`veltylabs/*` y ~40 librerías `webtyp/*`. Se quiere que, cuando un módulo (p. ej. `service_catalog`) publique en su
rama `release`, el proceso estándar de webtyp con GitHub Actions compile, pruebe y despliegue la app.
[DEPLOY_WIZARD_MASTER_PLAN.md](DEPLOY_WIZARD_MASTER_PLAN.md) **no cubre este caso**: su único `Target` es
`goflare.Worker` (nube), D16 borró a propósito el puller/SSH/webhook ("sin consumidor") y dejó previsto que un servidor
propio sería un `deploy.Target` nuevo (`webtyp/selfhost`, no creado). `mjosefa-cms/docs/DESPLIEGUE_WINDOWS.md:9` dice
que no usa `webtyp/deploy`; hoy el despliegue es manual (`cmd/build` aún no existe, `cmd/migrate` a mano, NSSM).

**Lo que sí se reutiliza:** `deploy.Plan` + `Deployment.Check/Provision/Release/Status` (el "dice qué falta"),
`github.Actions` (workflow generado, `workflow_dispatch`, seguimiento del run), la pestaña DEPLOY y las herramientas
MCP `deploy_*` de `app`, y la regla D1/D29 (un secreto vive solo en el almacén de su destino).

**Lo que falta (verificado en el código):**
1. **Un `Target` de servidor propio** (`Job` hoy es `uses: action@v1`; `deploy/interfaces.go:15`) y su action.
2. **Alcanzar la red local.** Un runner de GitHub en la nube no llega al servidor. Hace falta un runner propio
   (solo conexiones salientes) o un agente que descargue.
3. **Rama de disparo.** D8 fija "push a `main`"; `Plan` no tiene rama (`deploy/plan.go:16`) ni `runs-on` ni `environment`.
4. **Propagación módulo → app.** Hoy `gopush` (local, del desarrollador) etiqueta y actualiza dependencias; ningún
   workflow avisa a `mjosefa-cms`. El `release.yml` de los módulos solo existe en `core` y `goflare`.
5. **Migrar con respaldo y revertir.** `DESPLIEGUE_WINDOWS.md:§1,§5`: migrar es manual y sin rollback.
6. **`config/deploy.go` de `mjosefa-cms`** (no existe) y `cmd/build` (no existe).

## Preguntas abiertas

**P1 — ¿Qué es "release" en un módulo?** Hoy `gopush` etiqueta desde cualquier rama. A) `gopush` solo publica tag
semver desde `release`; otras ramas no publican (o solo `-rc`). B) tag desde cualquier rama y el consumidor decide.
**Recomiendo A**: Go no guarda la rama de un tag, así que la única forma de que "solo `release`" signifique algo es que
la rama sea la única que etiqueta. Respuesta:

**P2 — ¿Cómo se entera `mjosefa-cms` de que un módulo cambió?** A) *pull*: un workflow programado (cada hora) en la app
compara las versiones de `go.mod` con el último tag de cada dependencia, sube las versiones en su rama `release` y
despliega. B) *push*: el workflow de cada módulo envía `repository_dispatch` a la app (token entre repos, como
`RELEASE_TOKEN` de `core/.github/workflows/release.yml:28`; hay que instalar el workflow en ~50 repos). C) `gopush`
local sube `go.mod` de la app (sin Actions en medio). **Recomiendo A** con `workflow_dispatch` para forzarlo: cero
configuración en los módulos, la app declara qué vigila (su `go.mod`), y la demora de minutos es irrelevante para un
consultorio. B se agrega después solo si molesta la demora. Respuesta:

**P3 — ¿Cómo llega el artefacto al servidor?** Compilar y probar siempre en un runner de GitHub (reproducible, con
`GOPRIVATE`); el *deploy* es otro job. A) runner **autoalojado** en el servidor (solo salientes, sin abrir puertos).
B) runner en otra PC de la LAN que copia por SSH/WinRM. C) agente "puller" en el servidor que baja el Release (revive
lo que D16 borró). **Recomiendo A**: es lo único que usa el mismo workflow generado y deja el run visible en la pestaña
DEPLOY; C duplica un cliente de GitHub. Depende de P4. Respuesta:

**P4 — ¿Qué sistema operativo tendrá el servidor?** `DESPLIEGUE_WINDOWS.md` apunta a Windows Server 2012, pero
`mjosefa-cms/go.mod` pide Go 1.26.8 y Go exige Windows 10/Server 2016 desde Go 1.21 (el último que corre en 2012 es
Go 1.20); además el runner de Actions no soporta 2012. **Si esto es así, el binario no arranca ahí** y la frase "Windows
Server 2012 no es un límite" (`:131`) es falsa. A) Windows Server 2016+ . B) Linux (mini-PC; `systemd` en vez de NSSM).
C) mantener 2012 (imposible con este Go). **Recomiendo B si el hardware lo permite, si no A**: Linux cierra runner,
servicio, `pg_dump` y TLS con piezas estándar. Verificar con `go.dev/wiki/MinimumRequirements` antes de contestar. Respuesta:

**P5 — ¿Cuándo se despliega?** A) cualquier momento. B) solo en ventana (p. ej. 22:00–06:00, hora de Santiago) y
manual fuera de ella. C) siempre manual (`workflow_dispatch`). **Recomiendo B**: un reinicio con la clínica atendiendo
cuesta; offline-first (OFFLINE_FIRST_MASTER_PLAN de mjosefa-cms) aún está en diseño y no cubre la caída. Respuesta:

**P6 — ¿Aprobación humana?** D32: permiso de despliegue = permiso de GitHub; el entorno `production` con revisores
solo existe en repos privados con Enterprise. A) sin aprobación: el tag en `release` es la aprobación. B) revisores
(requiere plan de pago). C) `release` protegida: solo se entra por PR aprobado. **Recomiendo C**: la aprobación vive
donde ya hay permisos (D32) y no cuesta plan. Respuesta:

**P7 — Base de datos en cada despliegue.** `cmd/migrate` es obligatorio en cada despliegue (`MASTER.md` §6, B2). A) el
job hace `pg_dump` → migra → arranca → `/health`; si algo falla, restaura el binario anterior (y la base solo si la
migración fue destructiva). B) migración manual aprobada. **Recomiendo A, con migraciones solo *aditivas*** (expandir y
luego contraer en otra versión), porque así volver al binario anterior es seguro sin tocar datos. Respuesta:

**P8 — ¿Dónde viven los secretos de producción (`DATABASE_URL`, clave del router)?** El servidor no tiene API; D9 prohíbe
pasarlos por GitHub *en la nube*. A) secretos del entorno `production` de GitHub; el job escribe el `.env` del servidor
en cada despliegue (rotar = cambiar el secreto). B) solo en el `.env` del servidor, escrito a mano una vez (`Check` no
puede saber si falta). **Recomiendo A**: es la única vía sin conexión entrante y D29 se cumple (el estado se lee de
nombres y `updated_at` de GitHub). Es una excepción documentada a D9. Respuesta:

## Decisiones por implementar

Ninguna todavía. Heredadas (no se reabren): D1/D29 secretos, D2 `app` orquesta, D8 solo CI despliega, D16 el servidor
propio sería un `deploy.Target`, D34 `config/deploy.go` con `//go:build deploy`, D39 `workflow_dispatch`.

## Fases

Ecosistema (Step 0): no existe `selfhost`; `goflare/target_native.go` es el `Target` de Cloudflare compilado `!wasm`,
no un servidor propio → **nuevo `webtyp/selfhost`**, reusando `deploy.Target`, `github` y el patrón de action de goflare.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S0 | docs | Contestar P1–P8; consolidar | pendiente (tú) |
| S1 | `deploy` | `Plan.Branch` (def. `main`), `Job.RunsOn`, `Job.Environment`; `Scan`/validate; D8 pasa a "push a `Plan.Branch`" | espera S0 |
| S2 | `github` | Workflow generado con rama, `runs-on`, `concurrency`, ventana (P5) y job programado de P2 | espera S1 |
| S3 | `selfhost` (nuevo) | `Target`: `Check` (runner en línea, servicio, Postgres alcanzable, disco, tarea de respaldo, CA de TLS), `Provision` (token de registro del runner, servicio), `Job`; action `webtyp/selfhost@v1`: bajar artefacto → `pg_dump` → parar → cambiar → migrar → iniciar → `/health` → revertir | espera S1, P3, P4 |
| S4 | `sitec` / `mjosefa-cms` | `cmd/build` (3 líneas, `DESPLIEGUE_WINDOWS.md:§2`) empaqueta `web/server` + `web/public/` como un artefacto con versión; el servidor expone la versión | espera BUILD_DESTINATIONS |
| S5 | `app` | Registrar el `Target` en `deployment()`; sin cambios en la pestaña DEPLOY ni en `deploy_*` | espera S3 |
| S6 | devflow (`gopush`) | Etiquetar solo desde `release` (P1); `go.mod` de la app lo sube el job de P2 | espera P1, P2 |
| S7 | `mjosefa-cms` | `config/deploy.go`; rama `release`; borrar `DESPLIEGUE_WINDOWS.md` y B2 de `MASTER.md` al aceptar | espera S2–S5, B3, B4 |
| S8 | `devskills` | Skill de despliegue en servidor propio; ajustar **secrets** (excepción P8) | con S3 |

**Aceptación:** (1) un tag nuevo de `service_catalog` en `release` → en la ventana, el servidor corre la versión nueva
sin tocarlo; (2) una migración que falla o un `/health` rojo → queda corriendo la versión anterior y el run sale rojo;
(3) sin cambios de dependencias → no despliega; (4) la pestaña DEPLOY lista qué falta en un servidor sin preparar.

## Bloqueos

- 2026-10-21 · P4: si el servidor sigue siendo Windows Server 2012, ni el binario (Go ≥ 1.21) ni el runner corren; tuyo.
- 2026-10-21 · `MASTER.md` de `mjosefa-cms` B3 (datos reales) y B4 (producción): no se despliega vacío; tuyo.
- 2026-10-21 · no leí `webtyp/github`, `app` ni `devflow` (fuera de esta sesión); S2, S5 y S6 se afinan al leerlos.

## Relación con otras olas

- Extiende DEPLOY_WIZARD (cerrado): añade un `Target`, no cambia su contrato salvo S1.
- BUILD_DESTINATIONS define el artefacto `server` + `browser.wasm` que empaqueta S4.
- OFFLINE_FIRST (mjosefa-cms) reduce el costo de una caída durante el despliegue; no la sustituye.
- SECURITY_BY_DEFAULT: el `/health` y la versión no exponen datos; el runner autoalojado no acepta PR de forks.
