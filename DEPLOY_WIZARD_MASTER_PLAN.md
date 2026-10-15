# Despliegue guiado — master plan

> **Status:** IN PROGRESS · 2026-10-15 · D1–D42 cerradas; sin preguntas abiertas. `wizard`, `deploy`, `env` running; los demás planes escritos en disco, esperando sus prerequisitos.

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

Ninguna. Las que surjan al ejecutar los planes van aquí como P27 en adelante.

## Agregados enviados a agentes en ejecución (textual)

Regla (skill **master-plan**): todo lo que se le pide a un agente después de despacharlo queda aquí.

1. `webtyp/deploy`, sesión 7552631315228362006, 2026-10-15:
   > Addition to Stage 4 (same plan, no other change): export func LocalMissing(p Plan, local Destination) ([]Missing, error) in deployment.go. It validates p, then returns exactly the Missing items of the ToLocal places (same construction as Check: Random -> FixAutomatic, Typed -> FixInput with Accept putting into local). It never touches a Target or CI, and a nil local is ErrNilTool. Deployment.Check must build its ToLocal items by calling LocalMissing (one implementation). Test: LM1 a plan with ToRuntime and ToLocal secrets, none present -> LocalMissing returns only the ToLocal ones; LM2 nil local -> ErrNilTool. Reason: webtyp/app blocks the local dev server until local secrets exist and must not call GitHub/Cloudflare for that.
2. `webtyp/deploy`, misma sesión, 2026-10-15 (D42):
   > Second addition (decision P26 of the master plan), same plan: (1) plan.go: add to Plan the field 'Vars []Var' and the type 'type Var struct { Key, Value string }' with doc comment: non-secret runtime variables, value written in config/deploy.go, delivered by the Target as plain-text bindings on every deploy. (2) Scan: Vars is a composite literal of Var elements whose Key and Value are string literals (same rules as other string fields). (3) validate: each Var has non-empty Key and Value; a Var Key must not repeat and must not equal any Secret Key -> new errors ErrIncompleteVar, ErrDuplicateVar ('<KEY> declared twice, as a Var or Secret'). (4) README example: add Vars: []deploy.Var{{Key: "IAM_BASE_URL", Value: "https://iam.velty.cl"}}. Tests: S1 includes Vars; V-tests for both new errors. Nothing else changes (Deployment does not deliver Vars: the Target puts them in its Job).

Al revisar el PR de `deploy` (skill **plan-review**), verificar que ambos estén implementados.

## Tareas que no son planes de código (instrucciones exactas)

- **Skill `dev-autologin`** (`webtyp/devskills/skills/dev-autologin/SKILL.md`), **después** de que
  `env` publique su plan de esta ola: reemplazar la sección "Configuration via `.env`" (líneas
  24-41 hoy) por "Configuration": declarar `DEV_AUTOLOGIN` en `config/deploy.go` como
  `deploy.Secret{Key: "DEV_AUTOLOGIN", What: "Credentials replayed to open a dev session automatically", Source: deploy.Typed, Guide: []lang.Text{"Type the JSON payload of your authenticator, e.g. {\"code\":\"12345678-5\"}"}, To: []deploy.Place{deploy.ToLocal}}`;
  la persona lo escribe en la pestaña SECRETS de webtyp; `env` lo lee del keyring del proyecto.
  Conservar los dos ejemplos de JSON como ejemplos de **valor**, sin `.env`. Sincronizar con
  `devskills -f` y publicar devskills.
- **Despacho de planes con `docs/PLAN.md` ocupado:** iam (`PLAN.md` = seguridad, espera `auth`) y
  misitio (`PLAN.md` = `cmd/migrate`, running). Al despachar, si `docs/PLAN.md` sigue ocupado,
  convertirlo en cola (skill **agents-workflow** → «execution queue») con `PLAN_DEPLOY_WIZARD.md`
  primero en iam (rompe deploys mientras no se aplique) y último en misitio.
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
| `webtyp/wizard` | `StepSource` tipado; `Step.NextFn` (pasos en tiempo de ejecución); `LabelText lang.Text`; `Dictionary()` | **gate** de `app` | [wizard/docs/PLAN.md](https://github.com/webtyp/wizard/blob/main/docs/PLAN.md) · **running** (18053327749468295567) |
| `webtyp/deploy` | solo contratos + `Scan` + `Deployment` (D34–D36); borra todo lo demás (D3, D14, D16); `Dictionary()` | **gate** de goflare, github, app | [deploy/docs/PLAN.md](https://github.com/webtyp/deploy/blob/main/docs/PLAN.md) · **running** (7552631315228362006) |
| `webtyp/goflare` | `deploy.Target` `goflare.Worker`: token de arranque (D27, D30, D40), token por proyecto, D1/R2 por nombre (D18, D38), secretos del Worker (D9), `Job` para CI (D37); `Dictionary()` | paralelo, tras `deploy` | [goflare/docs/PLAN.md](https://github.com/webtyp/goflare/blob/main/docs/PLAN.md) · escrito, espera tag de `deploy` |
| `webtyp/github` | `deploy.CI` `github.Actions`: diagnóstico de `gh`, workflow generado, secretos de Actions, `Release` + estado del run (D19, D39); `Dictionary()` | paralelo, tras `deploy` | [github/docs/PLAN.md](https://github.com/webtyp/github/blob/main/docs/PLAN.md) · escrito, espera tag de `deploy` |
| `webtyp/env` | entorno → `.env` → keyring del proyecto (D15); deja de usar `keyring.Scheme` | paralelo | [env/docs/PLAN.md](https://github.com/webtyp/env/blob/main/docs/PLAN.md) · **running** (7810215043722152745) |
| `webtyp/keyring` | borra `Scheme`/`IsReference` | tras publicar `env` | [keyring/docs/PLAN.md](https://github.com/webtyp/keyring/blob/main/docs/PLAN.md) · escrito |
| `webtyp/app` | pestaña DEPLOY (D17), MCP `deploy_*` (D21, D23, D26), SECRETS desde la declaración, diccionarios (D22), compilación del Worker con `goflare` directo | tras todos los anteriores | [app/docs/PLAN.md](https://github.com/webtyp/app/blob/main/docs/PLAN.md) · escrito, espera todos los anteriores |
| `veltylabs/iam` | workflow con `d1-name` (D38) y `config/deploy.go` con `Vars` (D42) | **inmediatamente** tras publicar `goflare` (sin él, iam despliega sin D1) | [iam/docs/PLAN_DEPLOY_WIZARD.md](https://github.com/veltylabs/iam/blob/main/docs/PLAN_DEPLOY_WIZARD.md) · escrito |
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
