# Despliegue guiado — master plan

> **Status:** CLOSED · 2026-10-20 · todas las librerías de la ola publicadas (`deploy` v0.2.111, `wizard` v0.0.31, `goflare` v0.5.42, `github` v0.0.3, `env` v0.1.1, `keyring` v0.2.9) y `app` **v0.7.0** (pestaña DEPLOY, herramientas MCP `deploy_*`, SECRETS desde la declaración, dependencias de archivos generados); comprobado con una ejecución real de `deployment()` sobre un proyecto de prueba. Lo que queda no es de esta ola: la **aceptación con cuenta real de Cloudflare** (§Aceptación, la hace el usuario) y el re-layout y la declaración de `misitio`, que pasan a [BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md).

**Disparador:** desplegar `veltylabs/misitio` exige hoy pasos manuales repartidos en tres documentos
(`misitio/docs/DEPLOY.md`, `CI_TOKEN.md`, `PLAN_INFRA_BLOQUEADA.md`): crear D1 y R2 a mano, crear un
token de Cloudflare a mano, `gh secret set` cuatro veces, cargar secretos del Worker en el dashboard.
El último intento falló porque el token hecho a mano no tenía `Workers Scripts:Edit`
(`PUT workers/scripts/misitio → 403`), y nada lo detectó antes del deploy.

**Objetivo:** cada vez que se quiere desplegar, `webtyp/app` abre un wizard que **sabe qué falta**,
lo pide, lo crea donde hay API, lo entrega a cada destino (CI, Worker) y despliega. El agente ve lo
mismo por MCP: sabe qué falta, guía al desarrollador paso a paso y despliega cuando no falta nada.
Funciona para un **equipo**: cualquier persona autorizada aprovisiona y despliega sin recibir nada de
otra. Nadie edita `.env`, nadie corre `gh secret set`, nadie crea un token a mano.

**Alcance:** `webtyp/wizard`, `webtyp/deploy`, `webtyp/goflare`, `webtyp/github` (nuevo, creado
2026-10-15), `webtyp/env`, `webtyp/keyring`, `webtyp/app`, y `veltylabs/misitio` como caso de
aceptación. `veltylabs/iam` recibe un ajuste de su workflow (D38).

**Relación con otras olas:** independiente de UI_CONSISTENCY, API_SURFACE, NO_REFLECTION y
SOURCE_SELECTION. Consume KEYRING_DEPENDENCY_FREE (keyring inyectable, ya publicado). Es el
mecanismo con el que SECURITY_BY_DEFAULT §9 (misitio) entregará `IAM_CLIENT_PRIVATE_KEY` al Worker;
no la bloquea: §9 puede salir antes con su paso documentado y este wizard lo reemplaza después.

---

## Decisiones

### Fijadas por el usuario (2026-10-15)

| # | Decisión | Consecuencia |
|---|---|---|
| D1 (reescrita por D29) | **Un secreto de producción vive solo en el almacén de su destino** (secretos de GitHub Actions o del Worker). **Ningún desarrollador tiene una copia.** Se crea (mintado, generado o escrito en el wizard) y se entrega directo al destino; recuperar es siempre rotar. El keyring de cada persona guarda solo **lo suyo**: su token de arranque de Cloudflare y los secretos de **su** entorno de desarrollo. Nunca en `.env`, repo, plan, shell ni chat. Flujo: [diagrams/DEPLOY_SECRETS_FLOW.md](diagrams/DEPLOY_SECRETS_FLOW.md). | Skill **secrets** reescrita con este modelo. Se borra todo doc que diga `gh secret set X` o "pon X en `.env`". |
| D2 | **`webtyp/app` es el orquestador**: no sabe *cómo* se despliega nada; recibe inyectadas las formas de desplegar y las configura. | `app` es la raíz de composición: único que importa `goflare` y `github`. |
| D3 | **`webtyp/deploy` no depende de ninguna forma de desplegar.** Solo define el contrato. | Se borran de `deploy`: `providers/cloudflare/`, `pusher_cloudflare.go`, el registro global `RegisterPusher`/`init()` (`pusher.go:21-27`), las claves `CF_*` y `goflare/<proyecto>` de `store.go`, el mapeo heredado `"cloudflare"→"cloudflarePages"` (`wizard.go:48`). `deploy/go.mod` deja de requerir `goflare`. |
| D4 | **Cada proyecto declara cómo se despliega y qué necesita.** | Formato: D7 y D34. |
| D5 | **Una sola interfaz común** para todas las herramientas involucradas (skill **api-design**). | Forma: D13 y D35. |
| D6 | **El wizard dice qué falta** antes de desplegar. | Diagnóstico: D17. |

### Cerradas en la primera consolidación (P1–P14)

| # | Decisión | Consecuencia |
|---|---|---|
| D7 (cierra P1) | La declaración es **`config/deploy.go`**, un valor literal `deploy.Plan` **leído por AST** (sin ejecutar el proyecto), como `config/routes.go` y `RenderImages()`. | `.github/workflows/deploy.yml` pasa a ser **generado** desde la declaración; se borran `deploy.writeGHAWorkflow` (`deploy/wizard.go:113`), `deploy.CreateDefaultConfig` y `deploy.yaml`. |
| D8 (cierra P2) | **Solo CI despliega** (push a `main` o `workflow run`). | El "Desplegar" del wizard y de MCP dispara el workflow y sigue el run; no hay deploy local. |
| D9 (reescrita por D29) | Los secretos se entregan al destino **por su API**, en el momento en que se crean. Un secreto de **runtime** va **directo al Worker**, nunca a GitHub; CI solo recibe el token de deploy y el ID de la cuenta. Si el destino ya lo tiene, no se toca; cambiarlo es rotar (D26). | Un `PUT` de script conserva los bindings `secret_text` (`misitio/docs/DEPLOY.md:80-81`), así que el deploy no los borra. |
| D10 (cierra P4) | **Token de arranque de Cloudflare** (plantilla "Create additional tokens") en el keyring de la persona; el wizard **crea un token por proyecto** con exactamente los permisos que implica la declaración. | `goflare` deja de exigir un token hecho a mano; en CI sigue leyendo `CLOUDFLARE_API_TOKEN` (`goflare/auth.go:12`), ahora mintado. Dueño y vida: D27, D30. |
| D11 (cierra P5) | GitHub usa la sesión de **`gh`**; si falta, el diagnóstico pide `gh auth login`. | Sin PAT propio. |
| D12 (cierra P6) | **GCP no es parte del deploy de misitio** (login vía iam, commit `ff120d0`). Para iam, el cliente OAuth de Google es un **paso guiado** (`Source: Typed` con `Guide`): la IAP OAuth Admin API está obsoleta desde 2025-01-22, no hay API. | Se borra la tabla `GOOGLE_*` de `misitio/docs/DEPLOY.md:74-76`. Sin dependencia de `gcloud`. |
| D13 (cierra P7) | `webtyp/deploy` = **solo contratos** (forma exacta: D35). | Se borran `deploy.Provider`, `deploy.Pusher`, `deploy.Puller`, `deploy.Daemon`. Los textos son `lang.Text` (D22). |
| D14 (reescrita por D29) | **Un solo almacén local**: el keyring. Por proyecto (`keyring/auto.OpenForModule`) para los secretos de desarrollo; una entrada personal global para el token de arranque. Ningún secreto de producción entra al keyring. | Se borran `deploy.SecureStore`, `sensitiveKeys` y `KeyringServiceName` (`deploy/store.go`). |
| D15 (cierra P9) | La declaración es la **única lista de secretos**; `.env` solo guarda preferencias no secretas. `webtyp/env` busca una clave en el entorno, luego en `.env`, luego en el keyring del proyecto. | Se borran `keyring.Scheme`/`IsReference` (`keyring/reference.go`) y su uso en `env` y `app.MissingSecrets`, que pasa a leer la declaración. |
| D16 (cierra P10) | Puller de Windows, SSH y webhook **se borran**: no tienen consumidor. mjosefa-cms, el único proyecto con servidor propio, dice explícitamente que no usa `webtyp/deploy` (`mjosefa-cms/docs/DESPLIEGUE_WINDOWS.md:9`). `webtyp/selfhost` **no se crea**; si un día hace falta, implementará `deploy.Target`. | Se borran `puller.go`, `handler.go`, `checker.go`, `downloader.go`, `hmac.go`, `webhook.go`, `ssh.go`, `pusher*.go`, `manager*.go`, `shortcut_*.go`, `config.go`, `cmd/puller/`. |
| D17 (cierra P11) | La pestaña **DEPLOY** corre el diagnóstico al abrir el proyecto: checklist ✅/❌; cada ❌ que pide un valor es un paso del wizard; lo automático lo resuelve "Aprovisionar"; "Desplegar" se habilita sin ❌; el run de CI se ve en vivo. | El agente ve la misma checklist por MCP (D21). |
| D18 (cierra P12) | El `Target` **crea D1, R2, dominio y regla de rate limit si faltan** (idempotente) con los nombres de la declaración. | `D1_DATABASE_ID` y `R2_BUCKET_ID` dejan de ser secretos de GitHub: `goflare` los resuelve por nombre (D38). |
| D19 (cierra P13) | Repo nuevo **`webtyp/github`** (creado, `webtyp.com/github` v0.0.1): secretos de Actions, workflow generado, `workflow run` y seguimiento del run. | `ghaction` sigue siendo solo generación de YAML; `github` lo usa. |
| D20 (cierra P14) | `DEV_AUTOLOGIN` es un secreto de desarrollo declarado (`To: ToLocal`): keyring + wizard. | Se actualiza la skill **dev-autologin** (`SKILL.md:24-26`) cuando `env` publique D15. |
| D21 | **El deploy es herramienta MCP de webtyp.** El agente ve qué falta, guía al desarrollador paso a paso y despliega. **Los valores secretos solo se escriben en el wizard de la TUI**: ninguna herramienta MCP recibe ni devuelve un secreto. | En `app`: `deploy_status` (lectura), `deploy_guide` (lectura), `deploy_provision`, `deploy_rotate` y `deploy_release` (escritura). Las cuatro y la TUI usan `deploy.Deployment` (una sola implementación). |
| D22 | **El wizard habla inglés y español.** | `Step.LabelText` y los textos de `deploy` son `lang.Text`; `wizard`, `deploy`, `goflare` y `github` publican `Dictionary()`; `app/tui_lang.go:19` los carga. Idioma: `TUI_LANG`. Salida MCP en inglés. |

### Cerradas en la segunda consolidación (P15–P25)

| # | Decisión | Consecuencia |
|---|---|---|
| D23 (cierra P15) | El agente **puede desplegar a producción** con `deploy_release` sin confirmación en la TUI; el permiso lo da el harness del agente. | `deploy_release` es una herramienta de escritura. |
| D24 (cierra P16) | Columna `es` obligatoria en esta ola; las otras 7 pueden quedar vacías (`lang_missing` las lista). | Criterio de aceptación de cada plan con diccionario. |
| D25 (cierra P17) | Los secretos que hoy existen solo en GitHub **se rotan**, no se recuperan. | El primer aprovisionamiento de misitio crea todo de nuevo; los secretos viejos `D1_DATABASE_ID`, `R2_BUCKET_ID` se borran de GitHub. |
| D26 | **Rotar es una acción explícita** de la TUI (botón) y de MCP (`deploy_rotate`), nunca un efecto lateral del diagnóstico. | Sin ella, recuperar (D1) no tendría camino. |
| D27 (cierra P18) | El token de arranque dura **90 días**; el diagnóstico avisa 7 días antes y pide uno nuevo. | `goflare` lee `expires_on` de `GET /user/tokens/verify`. |
| D28 (cierra P19) | Deploys de vista previa por PR: **siguiente ola**. | Fuera de alcance. |
| D29 (cierra P21) | **Ningún secreto de producción se guarda en el keyring de nadie** (reescribe D1, D9, D14). Que un secreto "esté al día" se lee de los **metadatos no secretos del destino**: GitHub devuelve los nombres y `updated_at` de los secretos de Actions; Cloudflare lista los nombres de los secretos del Worker. | Sin huellas locales. Dos personas no se pisan: nadie tiene valores viejos que reescribir. |
| D30 (cierra P22) | **Cada persona tiene su propio token de arranque** (miembro de la cuenta de Cloudflare con su rol). Los tokens por proyecto son **del ámbito de la cuenta** (`POST /accounts/{id}/tokens`): sobreviven a que su creador deje el equipo. | La baja de una persona = quitarla de Cloudflare y de la organización de GitHub; no se rota nada. |
| D31 (cierra P23) | Secretos de desarrollo de terceros: **credenciales propias por persona**. Un cofre compartido (`deploy.Vault`) solo en una ola futura, con un caso real. | Fuera de alcance. |
| D32 (cierra P24) | **Permiso de despliegue = permiso de GitHub.** webtyp no tiene roles propios. Entorno `production` con revisores solo donde el plan de GitHub lo permita (en repos privados: Enterprise). | misitio (privado, GitHub Free): quien tiene escritura en el repo despliega. |
| D33 (cierra P25) | **Aceptación con dos personas**: una segunda máquina, con otro usuario de GitHub y otro miembro de Cloudflare, aprovisiona y despliega misitio sin recibir ningún valor de la primera; la primera sigue desplegando después. | Ver «Aceptación». |
| D41 (cierra P20) | Orden: `wizard` ‖ `deploy` ‖ `env` → `goflare` ‖ `github` ‖ `keyring` → `iam` (justo tras `goflare`) → `app` → `misitio`. | Tabla de librerías. |
| D42 (cierra P26) | **Variables de runtime no secretas** se declaran en `deploy.Plan.Vars []deploy.Var{Key, Value}` con su valor literal de producción. `goflare` las pasa en el `Job` (input `vars`) y las envía como bindings `plain_text` en **cada** deploy, así un `PUT` de script nunca las borra. En desarrollo siguen en `.env`. | misitio declara `IAM_BASE_URL`, `MISITIO_BASE_URL`; iam declara `GOOGLE_REDIRECT_URL`, `IAM_PANEL_ORIGIN`. |

| D43 (cierra P27) | **Rechazada la etiqueta `edge` como parche**; el usuario decidió el cambio de fondo en [BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md) (D1, D15): dos ejes con **una sola etiqueta nueva, `browser`** (`browser`/`!browser` de destino + `wasm`/`!wasm` de plataforma: `web/browser.go` → `//go:build browser` → `browser.wasm`, `web/edge.go` → `//go:build wasm && !browser` → `edge.wasm`, `web/server.go` → `//go:build !wasm` → `server`; en módulos `browser.go` → `browser` y `server.go` → `!browser`, sin etiquetas `edge` ni `server`). Su D9 confirma D34 (`config/deploy.go` con `//go:build deploy`). | Los planes de esta ola que crean o editan `edge/main.go`, `web/client.go` (→ `web/browser.go`) o etiquetas `wasm` (`app`, `goflare`, misitio, iam) se ajustan cuando esa ola cierre sus preguntas: sus repos tendrán un plan pendiente donde fusionarlo (skill **agents-workflow**). |

### Diseño derivado al revisar el código (2026-10-15)

Salió de leer el código al escribir los planes. Si alguno no te convence, dilo antes de despachar.

| # | Decisión | Por qué |
|---|---|---|
| D34 | `config/deploy.go` empieza con `//go:build deploy` (así ni el Worker, ni el cliente, ni el servidor enlazan `goflare`/`github`; `app` comprueba que compila con `go vet -tags deploy ./config/`) y declara `func Deploy() deploy.Plan` con **solo literales**. `Target` y `CI` se escriben como selector de una constante exportada del paquete que implementa (`goflare.Worker`, `github.Actions`); `deploy.Scan` lo resuelve por la ruta de import a `"webtyp.com/goflare.Worker"`, y la constante vale exactamente ese texto. Cualquier otra expresión (llamada, variable) es un error de `Scan` con archivo y línea. | Leer por AST no puede evaluar código; un selector a constante es tipado al compilar y legible sin ejecutar. Mismo enfoque que routescan. |
| D35 | Contrato de `deploy`: `Plan{Name, Domain, Target, CI, Databases, Buckets, Secrets}`, `Database{Binding, Name}`, `Bucket{Binding, Name}`, `Secret{Key, What, Source, Guide, To}`, `Source` (`Typed`, `Random`), `Place` (`ToRuntime`, `ToCI`, `ToLocal`), `Missing{Item, Why, Steps, Fix, Secret, Accept}`, `Fix` (`FixAutomatic`, `FixInput`, `FixManual`), interfaces `Target`, `CI`, `Destination`, y `Deployment{Plan, Target, CI}` con `Check`, `Provision`, `Rotate`, `Release`. | Es la "variable común" de D5: nombre, dominio, bases y buckets son conceptos de cualquier hosting; lo propio de cada uno queda dentro de su implementación. Firma exacta: plan de `deploy`. |
| D36 | Un ítem que pide un valor (`FixInput`) trae su propio `Accept(value string) error`: quien lo pide decide dónde va (keyring personal, Worker directo, keyring del proyecto). `app` solo arma el paso `Sensitive` del wizard y llama `Accept`. | `app` no sabe dónde vive cada secreto (D2); el valor nunca pasa por MCP (D21). |
| D37 | Si existe `cmd/migrate/main.go`, `goflare` agrega `pre-deploy: go run ./cmd/migrate` al workflow generado. Convención, no campo de la declaración. | Skill **app-rest-edge** ya fija esa ruta; iam la usa y misitio la está incorporando. |
| D38 | `goflare` resuelve D1 y R2 **por nombre** y renombra campos engañosos: `D1DatabaseName` (era el binding) → `D1Binding`; `D1DatabaseID` → borrado (se resuelve por `D1Name`); `R2BucketName` → `R2Binding`; `R2BucketID` (era el nombre del bucket) → `R2Bucket`. Antes de `pre-deploy` exporta `D1_DATABASE_ID` ya resuelto, así `cmd/migrate` de iam y misitio no cambia. | `goflare/cloudflare.go:233-253`, `config.go:88-133`. iam deja de pasar `D1_DATABASE_ID` como secreto: ajuste de su workflow en `iam/docs/PLAN_DEPLOY_WIZARD.md`, despachado justo tras publicar `goflare`. |
| D39 | `github` usa la CLI `gh` (sesión del desarrollador, D11) a través de un ejecutor de comandos inyectable; los valores van por **stdin**, nunca por argumentos. El workflow generado incluye `workflow_dispatch` para que `Release` pueda dispararlo. | `gh secret set` cifra con la clave del repo (lo que exige la API); stdin evita que el valor quede en el historial o en `ps`. |
| D40 | Si el token de arranque ve más de una cuenta de Cloudflare, el diagnóstico pide crearlo limitado a una sola (`FixManual`). La cuenta no se guarda en ningún lado: sale del token. | Evita una clave más en `.env` o en la declaración. |

---

## Preguntas abiertas

Ninguna en esta ola. El layout de misitio se decide en [BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md).

## Agregados enviados a agentes en ejecución (textual)

Regla (skill **master-plan**): todo lo que se le pide a un agente después de despacharlo queda aquí.

1. `webtyp/deploy`, sesión 7552631315228362006, 2026-10-15:
   > Addition to Stage 4 (same plan, no other change): export func LocalMissing(p Plan, local Destination) ([]Missing, error) in deployment.go. It validates p, then returns exactly the Missing items of the ToLocal places (same construction as Check: Random -> FixAutomatic, Typed -> FixInput with Accept putting into local). It never touches a Target or CI, and a nil local is ErrNilTool. Deployment.Check must build its ToLocal items by calling LocalMissing (one implementation). Test: LM1 a plan with ToRuntime and ToLocal secrets, none present -> LocalMissing returns only the ToLocal ones; LM2 nil local -> ErrNilTool. Reason: webtyp/app blocks the local dev server until local secrets exist and must not call GitHub/Cloudflare for that.
2. `webtyp/deploy`, misma sesión, 2026-10-15 (D42):
   > Second addition (decision P26 of the master plan), same plan: (1) plan.go: add to Plan the field 'Vars []Var' and the type 'type Var struct { Key, Value string }' with doc comment: non-secret runtime variables, value written in config/deploy.go, delivered by the Target as plain-text bindings on every deploy. (2) Scan: Vars is a composite literal of Var elements whose Key and Value are string literals (same rules as other string fields). (3) validate: each Var has non-empty Key and Value; a Var Key must not repeat and must not equal any Secret Key -> new errors ErrIncompleteVar, ErrDuplicateVar ('<KEY> declared twice, as a Var or Secret'). (4) README example: add Vars: []deploy.Var{{Key: "IAM_BASE_URL", Value: "https://iam.velty.cl"}}. Tests: S1 includes Vars; V-tests for both new errors. Nothing else changes (Deployment does not deliver Vars: the Target puts them in its Job).

Revisado en el PR #7 (2026-10-15): ambos implementados. La revisión corrigió localmente (commit `0ba9fb6`): `Scan` guardaba `""` ante variables o llamadas en vez de `ErrOnlyLiterals` con `archivo:línea`; el build tag se buscaba en cualquier comentario y no en la primera línea; `Source`/`To` aceptaban cualquier paquete; la construcción de `Missing` estaba triplicada (`Check`, `LocalMissing`, `Rotate`). Se agregó `ErrUnknownImport` (selector que no nombra un import del archivo). `Check` lista los `ToLocal` al final, por construirlos con `LocalMissing`.

**Consecuencia para `app`:** desde `deploy` v0.2.110, `app` no compila (`deploy.Daemon`, `deploy.NewDaemon`, `deploy.DaemonConfig` en `handler.go:60`, `section-deploy.go:20,39`); su plan ya los elimina, y `gopush` no actualiza `deploy` en `app` hasta entonces.

**Revisión de `wizard` (PR #2, 2026-10-15):** implementado según el plan; la revisión solo corrigió `go.mod` (`webtyp.com/lang` marcado `// indirect` siendo directo). El plan afirmaba que `devflow.CodeJob` no necesitaba cambios, pero su `LabelText` concatena un enlace en tiempo de ejecución (`devflow/codejob.go:387`) y dejó de compilar contra `lang.Text`: corregido en `devflow` con `lang.Text(...)` explícito. `app` queda roto donde el plan ya lo anunciaba (`app/section-secrets.go:27`, `LabelText: key`); lo resuelve el plan de `app`.

**Revisión de `github` (PR #1, 2026-10-15):** implementado; la revisión corrigió localmente antes de cerrar (v0.0.3): borrado `release_test_export.go` (exportaba `SetClockForTest`/`ResetClockForTest`/`SetTimeoutForTest` bajo `!production`, violando la regla de no exportar símbolos solo para tests) y reemplazado por `release_internal_test.go` justificado en la raíz sobre las costuras no exportadas `timeNow`/`timeSleep`; completadas las 14 traducciones obligatorias `es` en `lang.json` (estaban vacías) y el test `TestLang` (L1, dejado como stub); cubiertas las filas G3–G5 de `Check` y todos los estados de `Status`; movido `Provision` a `workflow.go`; `go mod tidy` eliminó 25 dependencias indirectas sin uso.

**Revisión de `keyring` (PR #2, 2026-10-15):** implementado (`reference.go` y `reference_test.go` borrados); la revisión agregó la línea sobre `config/deploy.go` y `auto.OpenForModule` en `README.md` y limpió bloques vacíos que el agente dejó en `docs/LAST_PLAN_EXECUTED.md` antes de cerrar (v0.2.9).

3. `webtyp/goflare`, sesión 4620029912345869226, 2026-10-15:
   > Proceed through all remaining stages (Stage 1 step 5 through Stage 9) and open the PR without stopping between stages. Make sure: (1) lang.json has the Spanish ('es') column populated for every key (no empty strings in column 0) and langc sync is run; (2) no symbols are exported only for tests (tests in tests/ must use package goflare_test and the exported API); (3) go mod tidy and gotest ./... pass before opening the PR.
4. `webtyp/goflare`, misma sesión, 2026-10-15:
   > 1. Yes: add unexported 'func (c *CfClient) delete(path string) ([]byte, error) { return c.do(http.MethodDelete, path, nil) }' to CfClient in cloudflare.go (or call c.do(http.MethodDelete, path, nil) directly since target_provision.go and target_check.go are in package goflare). 2. Add the delete helper first, then fix tests/target_check_test.go (including the K8 permission_groups/tokens mocks and K9 ci.Has mocks) and complete all remaining Stage 8 tests and Stage 9 docs. Ensure lang.json has Spanish ('es') translations populated, run langc sync, go mod tidy, and gotest ./..., and open the PR without stopping again.
5. `veltylabs/iam`, sesión 12558430203965191498, 2026-10-15:
   > Never mock library types locally. webtyp.com/auth v0.0.75 is now published with SessionPolicy, Principal, cookie.Options, passkey, and clientassertion. Run: GOPROXY=https://proxy.golang.org,direct go get webtyp.com/auth@v0.0.75 webtyp.com/router@v0.4.3 webtyp.com/server@v0.2.86 webtyp.com/cloudflare@v0.0.39 webtyp.com/jwt@v0.1.24 webtyp.com/webauthn@v0.1.6 webtyp.com/crypto@v0.0.28 webtyp.com/model@v0.2.3 webtyp.com/deploy@v0.2.110 webtyp.com/github@v0.0.3. Note for Stage 5: to check HasPasskey in AdminMeResponse, check what webtyp.com/auth@v0.0.75/passkey exposes (e.g. passkey.Module.CountByUser or ListByUser or ReadAllPasskeyCredential). Proceed through Stages 4–10 now; when you reach Stage 11 (config/deploy.go), run go get webtyp.com/goflare@latest for goflare.Worker (and if goflare's new tag is not live yet at that moment, pause at Stage 11 and ask).
6. `webtyp/goflare`, misma sesión, 2026-10-16:
   > Yes, that aligns completely. Run gotest ./..., finish implementing tests/target_provision_test.go and tests/target_misc_test.go (Stage 8) and the documentation updates (Stage 9), verify lang.json has column 0 ('es') populated for every key and run langc sync, run go mod tidy and gotest ./..., and open the PR immediately without pausing again.
7. `webtyp/goflare`, misma sesión, 2026-10-16:
   > Everything looks good. Finish updating docs/D1.md and docs/QUICK_REFERENCE.md (and any remaining docs so the grep checks in the plan return empty), ensure lang.json has column 0 ('es') populated for every key, run langc sync, go mod tidy, and gotest ./..., and open the PR immediately.
8. `veltylabs/iam`, misma sesión, 2026-10-16:
   > 1. Yes: delete VerifyProjectSecret, GenerateClientSecret, hashProjectSecret, projectSecretKey, projectSecretKeyLabel, clientSecretBytes, clientSecretPrefix, RegenerateProjectSecret, and the pepper env constant from config/projects.go, as specified in Stage 6 step 2. Machine authentication is handled by clientassertion.New(db, ProjectKeys{DB: db}, b.Origin, authMod.Notify) registered via authMod.Enable(ca). 2. asym.VerifyCompact is in package webtyp.com/crypto/asym (and base64url decoding is in webtyp.com/crypto/encoding.DecodeBase64URL), both in webtyp.com/crypto@v0.0.28 which has no extra dependencies. Proceed through Stages 6–12 and open the PR without pausing again.
9. `webtyp/goflare`, sesión 4620029912345869226, 2026-10-16:
   > Yes, proceed with all three fixes immediately: (1) use t.TempDir() / t.Chdir(t.TempDir()) in TestConfig_C1_C2_V1_V2 (and delete any stray edge/ directory in the repo working tree) and set t.Setenv("CLOUDFLARE_API_TOKEN", "test-token") where Deploy is called; (2) in TestProvision_P3, Provision creates resources (D1, R2, custom domain, rate limit, runtime secrets) before minting the project token, so the mock for P3 must allow the resource endpoints (or test P3 with a plan that has no databases/buckets/domain) and return permission_groups missing one of the required group names when minting the token; (3) in K1, Accept validates the bootstrap token via /user/tokens/verify and /accounts before storing it in personal keyring, so a token for which /user/tokens/verify returns 401/inactive must return an error and not call personal.Put. Make gotest ./... 100% green and open the PR immediately without stopping again.
10. `webtyp/goflare`, misma sesión, 2026-10-16:
   > Yes! Two critical reminders for that fix: (1) parseCFResponse already extracts and returns payload.Result, so in target_provision.go and target_check.go unmarshal the bytes returned by client.get/post/put/delete directly into the inner struct/slice (e.g. struct{ ID string; Value string } for token creation), never into a wrapper struct{ Result ... }; (2) verify that project token endpoints in target_check.go, target_provision.go, and fakeCFForProvision use the account-scoped paths (/accounts/{a}/tokens, /accounts/{a}/tokens/permission_groups, DELETE /accounts/{a}/tokens/{id}) per Stage 3 K8 and Stage 4 (only bootstrap token verification uses /user/tokens/verify). Fix that now, run gotest ./... to confirm 0 failures, and open the PR immediately.

## Plan siguiente de misitio

Espera [BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md). Fallas medidas
(2026-10-15), que ese plan debe resolver:

**Qué falla en misitio** (medido contra las skills **project-layout**, **app-modules**,
**app-rest-edge**):

| # | Falla | Dónde |
|---|---|---|
| M1 | `config/` importa paquetes del propio repo (`api`, `routes`, `authctx`, `config/panel`): la invariante 2 lo prohíbe | `config/api.go`, `app.go`, `assets.go`, `config.go`, `iamauth.go`, `policy.go`, `svg.go` |
| M2 | Existen `api/` y `routes/`, patrones que webtyp ya eliminó: las rutas van en `config/routes.go` (`Register`) y los handlers en `modules/<m>/server.go` | `api/`, `routes/` |
| M3 | `config/panel/` es un subpaquete de UI; el único subpaquete permitido es `config/layouts/` | `config/panel/{admin,constants,modules,preview}.go` |
| M4 | `modules/` no sigue el par tipado: `admin`, `app`, `images`, `support` son archivos wasm sueltos, sin `server.go`/`browser.go`/`module.go`, sin `modules/server.go`/`modules/browser.go` | `modules/*` |
| M5 | Duplicados del cliente de iam | `config/iamauth.go`, `config/authzcache.go`, `authctx/` (§9 de seguridad los borra) |
| M6 | La composición del servidor está en archivos **sin build tag** porque el Worker también es wasm, así que entra a `browser.wasm` (hoy `client.wasm`) | `config/app.go`, `backend.go`, `store.go` |
| M7 | `sitecontent.New` sin `Members`: las operaciones de `site_content` deniegan | `config/store.go` |
| M8 | `config/favicon.go` generado sin su dependencia | etapa 8 del plan de `app` |

Cuando esa ola cierre sus preguntas se escribe **un** plan nuevo, `misitio/docs/PLAN_NEXT.md`
(se renombra a `PLAN.md` cuando cierre `cmd/migrate`), con: el re-layout; el cliente de iam con
`private_key_jwt` (§9 de seguridad, si iam ya publicó); y la declaración de despliegue
(`docs/PLAN_DEPLOY_WIZARD.md`, que se borra al fusionar). `PLAN_ARQUITECTURA_MODULOS.md` y
`PLAN_RUTAS_PARAMETRIZADAS.md` se borran al escribirlo (usan nombres prohibidos y editan `routes/`,
que desaparece), recuperando solo los problemas medidos que sigan vigentes. `PLAN_STAGE_5/6/7` y
`PLAN_INFRA_BLOQUEADA.md` no entran (bloqueados por infraestructura).

## Tareas que no son planes de código (instrucciones exactas)

- **Skill `dev-autologin`** (`webtyp/devskills/skills/dev-autologin/SKILL.md`): **publicado**
  (`devskills` v0.0.21, 2026-10-15) con `DEV_AUTOLOGIN` declarado en `config/deploy.go` (`ToLocal`)
  y leído del keyring del proyecto por `webtyp/env`.
- **iam:** fusionado en su `docs/PLAN.md` (etapas 10–12) y publicado (iam v0.0.24); planes
  obsoletos `PLAN_SECURITY_HARDENING.md`, `PLAN_STAGE_1..6` borrados.
- **Planes obsoletos borrados el 2026-10-15:** `goflare/docs/PLAN_STAGE_1..7` (size gate,
  diagnóstico, subcomando tinygo, skew, generador de action: implementados), `app/docs/PLAN_UPDATE.md`
  (auto-actualización: implementada en `b78e68b`, evolucionó a `CheckAndInstall`/`UpdateAction`).
- **Ventana de misitio:** cuando `goflare` publique el renombre de D1/R2, el workflow a mano de
  misitio deja de crear sus bindings; no importa hoy porque cada deploy de misitio ya falla con 403.
  Lo arregla `app` al regenerar el workflow durante la aceptación.

---

## Evidencia — estado actual

| Hecho | Dónde |
|---|---|
| `deploy` importa `goflare` y tiene un proveedor Cloudflare dentro | `deploy/go.mod:8`, `deploy/providers/cloudflare/provider.go` |
| `deploy.Daemon` también compila el Worker en desarrollo y `app` lo registra en el watcher | `deploy/composer.go:51,70-80`, `app/section-deploy.go:29-30` |
| Registro global de pushers en `init()` | `deploy/pusher.go:21-27`, `deploy/pusher_cloudflare.go:5` |
| `wizard.New(onComplete, items ...any)` con aserción de tipo; sin pasos dinámicos; sin `lang` | `wizard/wizard.go:39-48`, `wizard/step.go:7-11` |
| Wizard de deploy con menú impreso a mano y 5 pasos "proxy" fijos | `deploy/wizard.go:28-104` |
| La TUI ya carga diccionarios por librería y guarda el idioma | `app/tui_lang.go:19-35` |
| Las herramientas MCP del proyecto se registran como `mcp.ToolProvider` | `app/mcp_registry.go:25`, `app/mcp-tools.go:20` |
| Lista fija de qué es secreto; el token de arranque no está | `deploy/store.go:30-42` |
| `env` resuelve `KEY=keyring://` desde `.env` | `env/env_native.go:19-35`, `env/env_wasm.go:28` |
| `goflare` lee su configuración de `.env`/entorno; nombres de D1/R2 engañosos | `goflare/config.go:50-133`, `goflare/cloudflare.go:233-253` |
| Los IDs de D1/R2 se copian a mano como "secretos" de GitHub | `misitio/docs/DEPLOY.md:54-57,107-108` |
| Token de Cloudflare mal escopeado → 403 en cada deploy desde v0.0.7 | run 38746286153; `misitio/docs/PLAN_INFRA_BLOQUEADA.md` |
| Puller sin consumidor | `mjosefa-cms/docs/DESPLIEGUE_WINDOWS.md:9`; `grep webtyp.com/deploy` → solo `app` |

## Industria — cómo despliegan otros

| Herramienta | Declaración | Secretos | Aprovisiona | Diagnóstico |
|---|---|---|---|---|
| Vercel / Netlify | detecta el framework; `vercel.json` opcional | en la plataforma; el equipo accede por rol | sí (proyecto) | log del build |
| Fly.io | `fly.toml` generado por `fly launch` | `fly secrets set`, cifrados en la plataforma, nunca en archivo | sí (app, Postgres, IPs) | `fly doctor` |
| Kamal 2 (Rails) | `config/deploy.yml` | cofre compartido (1Password/Bitwarden) con adaptadores | no (servidores propios) | `kamal config` |
| SST / Encore | **código** (`sst.config.ts`; infra inferida del código Go en Encore) | gestor del proveedor de nube | sí, todo | errores tipados al compilar |
| Cloudflare Wrangler | `wrangler.toml` | `wrangler secret put` | parcial | — |

Patrón común: login una vez; un comando que detecta, pregunta solo lo que falta y crea; secretos de
producción **solo en la plataforma**; diagnóstico antes de desplegar. Donde webtyp difiere:
Cloudflare no ofrece OAuth a terceros (D10); la declaración es Go leído por AST; y el agente es un
usuario de primera clase por MCP (D21).

## Sugerencias (no bloquean esta ola)

1. **Docs de despliegue generados desde la declaración** (`deploy.Secret.What/Guide`), para que
   `DEPLOY.md` no vuelva a quedar desactualizado.
2. **Rollback de un paso** desde la TUI y por MCP (Cloudflare guarda las versiones del Worker).
3. **Vista previa por PR** (D28).
4. **Cofre compartido** `deploy.Vault` (D31), cuando aparezca un secreto de desarrollo que no se pueda
   tener por persona.

## Puerta api-design

- **Precedentes:** SST, Encore, Fly, Kamal, Vercel (tabla anterior).
- **Nombres:** `deploy.Plan`, `deploy.Deployment`, `deploy.Target`, `deploy.CI`,
  `deploy.Destination`, `deploy.Secret`, `deploy.Missing`; herramientas `deploy_status`,
  `deploy_guide`, `deploy_provision`, `deploy_release`. Se descartan `Sink`, `Pusher`, `Puller`.
- **Balance de complejidad** (misitio):
  ```
  Conceptos a aprender            +2 (Plan, Secret)  / −4 (Pusher, Puller, SecureStore, keyring://)
  Archivos a tocar para desplegar +1 (config/deploy.go) / −2 (deploy.yml a mano, DEPLOY.md a mano)
  Pasos manuales                  0 / −9 (crear D1, R2, token, 4× gh secret set, 2× secretos de Worker)
  Formas de hacer lo mismo        −3 (dos almacenes de secretos, dos listas de secretos, deploy local vs CI)
  ```
- **Dónde va:** contratos en `deploy`; Cloudflare en `goflare`; GitHub en `github`; composición,
  TUI y MCP en `app`.
- **Qué borra:** D3, D7, D13, D14, D15, D16, D38, y en misitio las secciones manuales de
  `DEPLOY.md`, `CI_TOKEN.md` y `PLAN_INFRA_BLOQUEADA.md`.

## Librerías

| Repo | Cambio | Tipo | Plan |
|---|---|---|---|
| `webtyp/wizard` | `StepSource` tipado; `Step.NextFn` (pasos en tiempo de ejecución); `LabelText lang.Text`; `Dictionary()` | **gate** de `app` | PR [#2](https://github.com/webtyp/wizard/pull/2) · **publicado v0.0.30** (2026-10-15) |
| `webtyp/deploy` | solo contratos + `Scan` + `Deployment` (D34–D36); borra todo lo demás (D3, D14, D16); `Dictionary()` | **gate** de goflare, github, app | PR [#7](https://github.com/webtyp/deploy/pull/7) · **publicado v0.2.110** (2026-10-15) |
| `webtyp/goflare` | `deploy.Target` `goflare.Worker`: token de arranque (D27, D30, D40), token por proyecto, D1/R2 por nombre (D18, D38), secretos del Worker (D9), `Job` para CI (D37); `Dictionary()` | paralelo, tras `deploy` | **publicado v0.5.42** (`NewTarget`, `Worker`, `Dictionary`) |
| `webtyp/github` | `deploy.CI` `github.Actions`: diagnóstico de `gh`, workflow generado, secretos de Actions, `Release` + estado del run (D19, D39); `Dictionary()` | paralelo, tras `deploy` | PR [#1](https://github.com/webtyp/github/pull/1) · **publicado v0.0.3** (2026-10-15) |
| `webtyp/env` | entorno → `.env` → keyring del proyecto (D15); deja de usar `keyring.Scheme` | paralelo | publicado: ✅ Tag: v0.1.0 |
| `webtyp/keyring` | borra `Scheme`/`IsReference` | tras publicar `env` | PR [#2](https://github.com/webtyp/keyring/pull/2) · **publicado v0.2.9** (2026-10-15) |
| `webtyp/app` | pestaña DEPLOY (D17), MCP `deploy_*` (D21, D23, D26), SECRETS desde la declaración, diccionarios (D22), compilación del Worker con `goflare` directo | tras todos los anteriores | **publicado v0.7.0** (2026-10-20): el segundo intento de Jules no compilaba y se reescribió localmente contra las APIs reales (ver «Revisión del PR de `app`») |
| `veltylabs/iam` | workflow con `d1-name` (D38) y `config/deploy.go` con `Vars` (D42) | **inmediatamente** tras publicar `goflare` (sin él, iam despliega sin D1) | publicado como v0.5.42 |
| `veltylabs/misitio` | `config/deploy.go` con `Vars` (D42); borra pasos manuales de sus docs; rota secretos (D25) | último, aceptación | [misitio/docs/PLAN_DEPLOY_WIZARD.md](https://github.com/veltylabs/misitio/blob/main/docs/PLAN_DEPLOY_WIZARD.md) · escrito, espera `app` |

## Aceptación de la ola

Dos máquinas, dos personas (D33). En cada una, solo `gh auth login` hecho y su propio token de
arranque de Cloudflare en su keyring:

1. Un agente llama `deploy_status` sobre misitio y recibe la lista de lo que falta, sin valores.
2. Con `deploy_guide` guía al desarrollador, que escribe cada valor solo en el wizard de la TUI, en
   español o inglés según `TUI_LANG`.
3. `deploy_provision` y `deploy_release` dejan el Worker respondiendo en `misitio.velty.cl`.
4. La segunda persona repite 1–3 sin recibir nada de la primera; la primera vuelve a desplegar
   después sin errores.
5. Ningún secreto en `.env`, en el repo, en el keyring de nadie (salvo tokens de arranque y
   secretos de desarrollo) ni en el chat.

## Adición enviada a un agente en ejecución (`codejob reply`, 2026-10-19, `app`, sesión 17048424785949401293)

Jules avisó que las etapas 1–6 estaban hechas pero que no podía ejecutar las pruebas por un error del «private proxy». Respuesta textual: «Good progress; continue with Stage 7, Stage 8 and Stage 9 exactly as the plan says. One thing must change: do not deliver code you could not run. The private-proxy error is an environment setting, not a code problem. In every shell that runs go, set 'export GOPRIVATE=webtyp.com/*,github.com/veltylabs/*' and 'export GOFLAGS=-mod=mod', then run 'go mod tidy' and 'gotest' (install it with 'go install webtyp.com/devflow/cmd/gotest@latest' under the same variables). The Delivery check of the plan (gotest green, the greps, git diff --stat main) must pass before you open the PR, and the PR must contain the files of every stage: the previous attempt of this plan ended with an empty PR. If a module still cannot be fetched after this, stop and name the exact module and error in the PR instead of guessing types. Commit after each stage.»

Segunda adición (2026-10-19, `app`, misma sesión): Jules perdió su trabajo con un `git reset --hard` en su sandbox y pidió confirmar si rehacía las etapas 1–9. Respuesta textual: «Yes: re-apply Stage 1 through Stage 9 now, using the solutions you already have in context, with GOPRIVATE and GOFLAGS set as indicated. To avoid losing work again: never run 'git reset --hard', 'git checkout -- .' or 'git clean' on this repository; commit after each stage ('git add -A && git commit') so every stage survives; run gotest after Stage 4, Stage 7 and Stage 8. Open the pull request when the last stage is done, with the Delivery check of the plan passing. A session that ends without a pull request is a failed delivery.»

## Revisión del PR de `app` (2026-10-19, `webtyp/core` PR #30)

El PR de Jules (segundo intento) **no compilaba**: escribió contra firmas inventadas de `deploy`, `goflare`, `github`, `keyring/auto` y `mcp`, sin `go.mod` actualizado, sin tests (los quitó), sin documentación y con un archivo suelto (`PR_DESCRIPTION.md`). Se corrigió **localmente en la rama del PR** (commit `aba1a60`) contra las APIs reales; `gotest` (vet, race, tests) en verde. Desviaciones del plan, juzgadas por mérito:

- **`deploy_guide` sin argumento `item`**: devuelve los pasos de **todos** los ítems con la numeración de `deploy_status`. Un argumento exigía un modelo generado con `ormc` y la herramienta instalada no lo generaba para `client_models.go`; el agente ve todos los pasos y elige.
- **Rotar**: en vez de un botón, un asistente que pide escribir `ROTATE` (reemplaza secretos de producción) y luego pide el valor nuevo de cada secreto tipeado. Se hizo así porque **no se puede registrar un manejador después de iniciar la TUI** (`UpdateAction` lo documenta: la lista de manejadores no tiene candado) y `Rotate()` devuelve ítems que hay que preguntar.
- **Tests**: un archivo de raíz (`deploy_flow_test.go`, con su justificación) en vez de cuatro en `tests/`, porque ejercitan costuras no exportadas (el constructor de `Deployment`, los pasos del asistente, `ensureImports` con un ejecutor inyectado).
- **`Handler.EdgeWorker`** (el `goflare` del Worker) reemplaza a `DeployManager`: el test de archivos no observados sigue necesitando acceso.
- El criterio de aceptación `grep NewDaemon` coincide con `NewDaemonToolProvider` (anterior y ajeno); el plan debió acotarlo a `deploy.NewDaemon`.

## Cierre de la ola (2026-10-20)

**Comprobación real, no solo pruebas:** sobre un proyecto de prueba con `config/deploy.go` (`goflare.Worker`, `github.Actions`), `app` construyó el `Deployment` de verdad (`go vet -tags deploy`, `deploy.Scan`, llavero, `goflare.NewTarget`, `github.New`) y `Check()` devolvió la lista correcta: repositorio de GitHub (manual), workflow de despliegue (automático), workflow desactualizado (manual) y token de arranque de Cloudflare (se escribe en el asistente, con tres pasos). Sin fallas.

**No verificado, y por qué:** la TUI real (los botones y asistentes se registran antes de iniciarla, y las pruebas usan una TUI simulada) y los pasos 1–5 de «Aceptación», que necesitan una cuenta de Cloudflare, `gh auth login` y dos personas.

**Si la aceptación falla en la máquina del usuario:** anotar el paso y el mensaje exacto aquí y abrir un plan nuevo en `app` con la **firma exacta** de cada API que toque (regla agregada a la skill **plan-authoring** tras esta ola: el agente inventó firmas cuando el plan solo las nombraba).

**Lo que sigue en otras olas:** `misitio` (re-layout M1–M8 y declaración de despliegue `misitio/docs/PLAN_DEPLOY_WIZARD.md`) depende de BUILD_DESTINATIONS, no de esta ola; M8 (favicon sin dependencia) quedó resuelto en `app` v0.7.0.
