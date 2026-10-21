# Despliegue continuo en servidor propio — master plan
> **Status:** IN PROGRESS · 2026-10-21 · D1–D10 fijadas; esperan tus respuestas P1–P6; P3 (sistema operativo del servidor) bloquea S3–S5. Nada despachado.

**Disparador:** `mjosefa-cms` corre en un servidor propio dentro de la red del consultorio y depende de ~11 módulos
`veltylabs/*` y ~40 librerías `webtyp/*`. Cuando un módulo publica en su rama `release`, la app debe llegar sola al
servidor. [DEPLOY_WIZARD_MASTER_PLAN.md](DEPLOY_WIZARD_MASTER_PLAN.md) no lo cubre: su único `Target` es `goflare.Worker`
(nube) y D16 borró el puller/SSH/webhook por falta de consumidor, dejando previsto un `deploy.Target` de servidor propio.

**Enfoque (fijado por el usuario):** la nube **compila, prueba, firma y publica**; el **servidor pregunta** (sale él a
GitHub, sin puertos de entrada ni runner en la LAN). La actualización es **un módulo más de la app**, sin TUI. Los
clientes son PWA: con service worker la app abre aunque el servidor esté caído (OF-10 de OFFLINE_FIRST) y
`pwa/ui.UpdatePrompt` nunca recarga a mitad de una atención (D-PWA-8).

**Lo que se reutiliza:** `deploy.Plan` + `Check/Provision/Release` (el "dice qué falta"), `github.Actions` (workflow
generado), `webtyp.com/update` (descarga, checksum, `Swap`, `Rollback`; hoy lo usa `core/update.go:77` para `webtyp
--update`), `webtyp/pwa`, el RBAC de la app y los archivos generados con dependencias propias (`core/docs/DEPLOY.md`).

**Lo que falta (verificado):** (1) un `Target` de servidor propio — `Job` es solo `uses: accion@v1`
(`deploy/interfaces.go:15`); (2) `Plan` no tiene rama (`deploy/plan.go:16`) y D8 fija "push a `main`"; (3) ningún
workflow avisa a `mjosefa-cms` cuando un módulo etiqueta (solo `core` y `goflare` tienen `release.yml`); (4) el módulo
de actualización del servidor, con su política de reinicio; (5) migrar con respaldo y revertir (`DESPLIEGUE_WINDOWS.md`
§1 y §5: manual y sin rollback); (6) `config/deploy.go` y `cmd/build` en `mjosefa-cms`, que no existen.

## Preguntas abiertas

**P1 — ¿Cuál es la segunda ocasión de revisión?** Dijiste "solo en 2 ocasiones": la primera es **al arrancar** (asíncrona,
sin bloquear). Propongo la segunda: A) **cuando se desconecta el último cliente** (es justo cuando puede aplicarse sola) +
botón "Buscar ahora" del administrador. B) cada N horas. C) solo al arrancar + el botón. **Recomiendo A**: cero
consultas inútiles con la clínica atendiendo, y coincide con tu regla automática. Respuesta:

**P2 — ¿Qué es "cliente conectado"?** El módulo no puede depender del chat. A) sesión con latido reciente (`chat_room`
CR-6: 30 s, en línea < 90 s) o stream SSE abierto, dado por una función que la app inyecta (`ActiveClients() int`).
B) cualquier petición en los últimos N minutos. **Recomiendo A**: ya existe, y una PWA abierta sin servidor no cuenta
(correcto: no se interrumpe). Respuesta:

**P3 — ¿Qué sistema operativo tendrá el servidor?** `DESPLIEGUE_WINDOWS.md` apunta a Windows Server 2012, pero
`go.mod` pide Go 1.26.8 y Go exige Windows 10/Server 2016 desde Go 1.21 (el último que corre en 2012 es 1.20): **el binario
no arrancaría** y la frase "2012 no es un límite" (`:131`) sería falsa. Con este enfoque ya no hace falta runner, así que
solo importa el binario. A) Windows Server 2016+. B) Linux (mini-PC, `systemd`). C) 2012 (imposible con este Go).
**Recomiendo B si el hardware alcanza, si no A.** Verificar en `go.dev/wiki/MinimumRequirements`. Respuesta:

**P4 — ¿Cómo confía el servidor en lo que descarga y ejecuta?** `update` hoy verifica checksum (protege de corrupción, no
de un Release falso). A) además **firma ed25519**: la clave privada es secreto de Actions, la pública va compilada en la
app (`config/update_key.go`, archivo generado por `Provision`). B) solo checksum. **Recomiendo A**: un binario que se
instala solo es el mejor objetivo de un ataque a la cuenta de GitHub. El repo es privado: el servidor necesita un token
de **solo lectura de Releases**, tecleado una vez por el administrador en la pantalla de actualización. Respuesta:

**P5 — Migraciones al aplicar la versión.** El binario nuevo corre `cmd/migrate` antes de servir. A) `pg_dump` previo →
migrar → `/health`; si falla, vuelve el binario anterior (la base solo se restaura si la migración fue destructiva), y
las migraciones son **solo aditivas** (expandir; contraer en una versión posterior). B) con migraciones pendientes la
actualización **nunca** es automática: exige al administrador. **Recomiendo A para lo aditivo y B para lo destructivo**:
tras un corte de luz no debe borrarse nada sin que alguien lo apruebe. Respuesta:

**P6 — El bot de D3 y la rama protegida.** El commit del bot a `release` necesita un token con escritura, y una rama
protegida con PR obligatorio lo bloquea. A) `release` sin protección; entra el bot y las personas con escritura.
B) protegida, con el bot como excepción. C) el bot abre un PR que se fusiona solo si pasan los tests. **Recomiendo B**:
las personas pasan por PR y el bot solo sube `go.mod`/`go.sum`. Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1 anterior) | **`release` es una rama como `main`.** `Plan.Branch` (def. `main`); el workflow generado corre en push a esa rama; el tag lo publica ese workflow/`gopush` desde ella. | S1, S2, S6 |
| D2 | **El servidor pregunta, la nube construye.** Nada entra a la LAN; el artefacto es un Release de GitHub de `mjosefa-cms` (`server` + `web/public/` + checksums + firma). | S3, S4 |
| D3 | **Módulo → app**: la action de la app (por push a `release`, horario o `workflow_dispatch`) sube a su última versión publicada todo `veltylabs/*` y `webtyp.com/*` de su `go.mod` (sin `-rc`), corre `go mod tidy` y los tests de todos los destinos; si fallan, no publica y el servidor conserva su versión; si pasan, un commit del bot guarda `go.mod` en `release` y publica el Release firmado. Sin cambios, no publica. Sin tocar ~50 repos. | S2 |
| D4 | **La actualización es un módulo integrado** (`webtyp/selfupdate`, sin TUI): pantalla "Actualizaciones" para el administrador, recurso RBAC propio y contador en el menú cuando hay una pendiente. | S4, S7 |
| D5 | **Política de reinicio:** con una actualización pendiente y clientes conectados, **solo el administrador** la aplica (botón). Con **cero clientes** se aplica **sola**: caso típico, el servidor se reinició por un corte de luz y arrancó con una versión pendiente; mientras se instala, los clientes PWA esperan y reintentan. | S4 |
| D6 | **Revisión asíncrona y nunca bloqueante**: al arrancar, y en la segunda ocasión (P1). Un fallo de red es una línea de log, nunca un error para el usuario (como `core/update.go:23`). | S4 |
| D7 | **Reusar `webtyp.com/update`** (`Swap`/`Rollback`/checksum); `selfupdate` añade política, firma, estado persistente y vigilancia. Un supervisor (NSSM o `systemd`) relanza el proceso; la versión nueva vive en su carpeta y se apunta a ella (Windows no deja reemplazar un `.exe` en uso). | S4 |
| D8 | **Vigilancia tras reiniciar:** si `/health` no responde en N segundos, se vuelve a la versión anterior y se marca la nueva como mala (no se reintenta sola). | S4 |
| D9 | **Secretos de producción solo en el servidor** (`DATABASE_URL`, `ROUTER_URL`, token de lectura): sin API ni conexión entrante, D1/D29 de DEPLOY_WIZARD se cumplen tal cual. La app informa lo que falta al arrancar (mismo texto de `Missing`). | S4, S7 |
| D10 | **Un binario por sistema operativo, un solo cliente.** La action compila `server` para cada `GOOS/GOARCH` listado (como `core/.github/workflows/release.yml`) y publica `web/public/` una vez; el servidor baja el que corresponde a su plataforma (`core/update.go:40`). Cambiar de Windows a Linux no cambia el flujo. Compilar en la nube no resuelve el mínimo de Go (P3). | S2, S4 |

## Fases

Ecosistema (Step 0): no existe `selfupdate`; `goflare/target_native.go` es el `Target` de Cloudflare compilado `!wasm`,
no un servidor propio → **nuevo `webtyp/selfupdate`** encima de `webtyp/update` y `webtyp/pwa`; **nuevo `Target`
`selfhost`** (en el mismo repo, sin importar `goflare`) que solo prepara la nube. No leí `webtyp/update`, `github`, `app`
ni `devflow` (fuera de esta sesión): sus filas se afinan al leerlos.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S0 | docs | Contestar P1–P6; consolidar | pendiente (tú) |
| S1 | `deploy` | `Plan.Branch`; D8 de DEPLOY_WIZARD pasa a "push a `Plan.Branch`"; `Scan`/validate | pendiente |
| S2 | `github` | Workflow generado: rama, `go get` de dependencias, prueba, compila por plataforma (D10), firma, publica Release; disparos de D3 | espera S1 |
| S3 | `selfupdate` (`Target`) | `Check`: secreto de firma en Actions, workflow, último Release; `Provision`: par ed25519 (privada a Actions, pública en `config/update_key.go`) | espera S1, P4 |
| S4 | `selfupdate` (módulo) | Estado `idle→disponible→descargada→aplicando`, política D5 con `ActiveClients()`, firma, intercambio de versión (D7), vigilancia (D8), migración (P5), pantalla y RBAC | espera P1–P3, P5 |
| S5 | `sitec` / `mjosefa-cms` | `cmd/build` (3 líneas) empaqueta `web/server` + `web/public/` con versión; el servidor expone su versión y `/health` | espera BUILD_DESTINATIONS |
| S6 | devflow (`gopush`) | Etiquetar desde la rama de `Plan.Branch`; sin cambios para el resto | espera S1 |
| S7 | `mjosefa-cms` | `config/deploy.go`, rama `release`, `selfupdate` en `modules/`, supervisor; al aceptar borrar `DESPLIEGUE_WINDOWS.md` §5 y B2 de `MASTER.md` | espera S2–S5, B3, B4 |
| S8 | `pwa` / `devskills` | Cliente: ante reinicio del servidor muestra "actualizando" y reintenta sin perder la cola (`outbox`); skill de despliegue en servidor propio y excepción D9 en **secrets** | con S4 |

**Aceptación:** (1) un tag nuevo de `service_catalog` en `release` → la nube publica el Release → el servidor lo detecta;
con clientes conectados queda pendiente y el administrador lo aplica; (2) apagar y encender el servidor con una versión
pendiente → se instala sola antes de recibir clientes y estos esperan sin perder datos; (3) migración fallida o `/health`
rojo → queda la versión anterior; (4) un Release sin firma válida se rechaza; (5) sin cambios de dependencias no se publica.

## Bloqueos

- 2026-10-21 · P3: si el servidor sigue siendo Windows Server 2012, el binario (Go ≥ 1.21) no corre; tuyo.
- 2026-10-21 · `MASTER.md` de `mjosefa-cms` B3 (datos reales) y B4 (producción): no se despliega vacío; tuyo.

## Relación con otras olas

- Extiende DEPLOY_WIZARD (cerrado): añade un `Target` y `Plan.Branch`; no cambia el resto de su contrato.
- BUILD_DESTINATIONS define el artefacto `server` + `browser.wasm` que empaqueta S5.
- OFFLINE_FIRST (mjosefa-cms): sin él los clientes no sobreviven al reinicio; S8 depende de `outbox` y del service worker.
- SECURITY_BY_DEFAULT: el módulo exige permiso explícito, no expone datos en `/health` y verifica firma antes de ejecutar.
