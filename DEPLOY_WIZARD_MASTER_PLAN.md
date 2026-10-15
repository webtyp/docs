# Despliegue guiado — master plan

> **Status:** PENDING · 2026-10-15 · D1–D22 cerradas (P1–P14 aceptadas + MCP + idioma); P15–P20 abiertas antes de escribir los planes por librería.

**Disparador:** desplegar `veltylabs/misitio` exige hoy pasos manuales repartidos en tres documentos
(`misitio/docs/DEPLOY.md`, `CI_TOKEN.md`, `PLAN_INFRA_BLOQUEADA.md`): crear D1 y R2 a mano, crear un
token de Cloudflare a mano, `gh secret set` cuatro veces, cargar secretos del Worker en el dashboard.
El último intento falló porque el token hecho a mano no tenía `Workers Scripts:Edit`
(`PUT workers/scripts/misitio → 403`), y nada lo detectó antes del deploy.

**Objetivo:** cada vez que se quiere desplegar, `webtyp/app` abre un wizard que **sabe qué falta**,
lo pide (los secretos en `webtyp/keyring`), lo crea donde hay API, lo entrega a cada destino
(CI, Worker) y despliega. El agente ve lo mismo por MCP: sabe qué falta, guía al desarrollador paso a
paso y despliega cuando no falta nada. El desarrollador nunca edita `.env`, nunca corre
`gh secret set`, nunca crea un token a mano.

**Alcance:** `webtyp/wizard`, `webtyp/deploy`, `webtyp/goflare`, `webtyp/github` (nuevo),
`webtyp/selfhost` (nuevo, condicional, D16), `webtyp/env`, `webtyp/keyring`, `webtyp/app`, y
`veltylabs/misitio` como caso de aceptación.

**Relación con otras olas:** independiente de UI_CONSISTENCY, API_SURFACE, NO_REFLECTION y
SOURCE_SELECTION. Consume KEYRING_DEPENDENCY_FREE (keyring inyectable, ya publicado). Es el
mecanismo con el que SECURITY_BY_DEFAULT §9 (misitio) entregará `IAM_CLIENT_PRIVATE_KEY` al Worker;
no la bloquea: §9 puede salir antes con su paso documentado y este wizard lo reemplaza después.

---

## Decisiones

### Fijadas por el usuario (2026-10-15)

| # | Decisión | Consecuencia |
|---|---|---|
| D1 | **Un secreto vive en dos lugares y en ningún otro**: en producción, en el almacén del **destino** (secretos de GitHub Actions o del Worker); en la máquina del desarrollador, en **`webtyp/keyring`**, que es la fuente desde la que el wizard alimenta los destinos. Nunca en `.env`, repo, plan, shell ni chat. Los destinos son de solo escritura: si el keyring no lo tiene, el wizard lo vuelve a crear (mintea o genera y reemplaza) o lo pide; nunca lo trae de la nube. Flujo: [diagrams/DEPLOY_SECRETS_FLOW.md](diagrams/DEPLOY_SECRETS_FLOW.md). | Skill **secrets** (devskills v0.0.14). Se borra todo doc que diga `gh secret set X` o "pon X en `.env`". |
| D2 | **`webtyp/app` es el orquestador**: no sabe *cómo* se despliega nada; recibe inyectadas las formas de desplegar y las configura. | `app` es la raíz de composición: único que importa `goflare` y `github`. |
| D3 | **`webtyp/deploy` no depende de ninguna forma de desplegar.** Solo define el contrato. | Se borran de `deploy`: `providers/cloudflare/`, `pusher_cloudflare.go`, el registro global `RegisterPusher`/`init()` (`pusher.go:21-27`), las claves `CF_*` y `goflare/<proyecto>` de `store.go`, el mapeo heredado `"cloudflare"→"cloudflarePages"` (`wizard.go:48`). `deploy/go.mod` deja de requerir `goflare`. |
| D4 | **Cada proyecto declara cómo se despliega y qué necesita.** | Formato: D7. |
| D5 | **Una sola interfaz común** para todas las herramientas involucradas (skill **api-design**). | Forma: D13. |
| D6 | **El wizard dice qué falta** antes de desplegar. | Diagnóstico: D17. |

### Cerradas al consolidar (P1–P14 → recomendación aceptada)

| # | Decisión | Consecuencia |
|---|---|---|
| D7 (cierra P1) | La declaración es **`config/deploy.go`**, un valor literal `deploy.Plan` **leído por AST** (sin ejecutar el proyecto), como `config/routes.go` y `RenderImages()`. | `.github/workflows/deploy.yml` pasa a ser **generado** desde la declaración con `ghaction`; se borran `deploy.writeGHAWorkflow` (`deploy/wizard.go:113`), `deploy.CreateDefaultConfig` y `deploy.yaml` para este uso. |
| D8 (cierra P2) | **Solo CI despliega** (push a `main` o `workflow run`). | El "Desplegar" del wizard y de MCP dispara el workflow y sigue el run; no hay deploy local. Se borra `deploy.Provider.Build/Deploy` como camino local. |
| D9 (cierra P3) | Los secretos se **copian desde el keyring** a cada destino declarado mediante su API, solo cuando cambian (huella guardada en el keyring). | Implicación: un secreto de **runtime** (`To: Runtime`, p. ej. `IAM_CLIENT_PRIVATE_KEY`) va **directo al Worker**, nunca a GitHub; CI solo recibe el token de deploy. Un `PUT` de script conserva los bindings `secret_text` (`misitio/docs/DEPLOY.md:80-81`), así que el deploy no los borra. |
| D10 (cierra P4) | **Token de arranque de Cloudflare** por cuenta (plantilla "Create additional tokens"), en el keyring global; el wizard **crea un token por proyecto** con exactamente los permisos que implica la declaración. | `goflare` deja de exigir un token hecho a mano; en CI sigue leyendo `CLOUDFLARE_API_TOKEN` (`goflare/auth.go:12`), ahora mintado. Vida del token: P18. |
| D11 (cierra P5) | GitHub usa la sesión de **`gh`** (`gh auth token`); si falta, el wizard lanza `gh auth login`. | Sin PAT propio. |
| D12 (cierra P6) | **GCP no es parte del deploy de misitio** (login vía iam, commit `ff120d0`). Para iam, el cliente OAuth de Google es un **paso guiado**: URL de consola + redirect URI calculado + validación contra el endpoint de token de Google. La IAP OAuth Admin API está obsoleta desde 2025-01-22: no hay API. | Se borra la tabla `GOOGLE_*` de `misitio/docs/DEPLOY.md:74-76`. Sin dependencia de `gcloud`. |
| D13 (cierra P7) | `webtyp/deploy` = **solo contratos**: `deploy.Plan`, `deploy.Secret{Key, What, Source, To}`, `deploy.Target` (`Check`, `Provision`, `Release`), `deploy.Destination` (`Has`, `Put`), `deploy.Missing`. | Se borran `deploy.Provider`, `deploy.Pusher` y `deploy.Puller` del paquete raíz. Los textos de `Secret` y `Missing` son `lang.Text` (D22). |
| D14 (cierra P8) | **Un solo almacén**: keyring por proyecto (`keyring/auto.OpenForModule`) + una entrada global por cuenta para el token de arranque. | Se borran `deploy.SecureStore`, `sensitiveKeys` y `KeyringServiceName` (`deploy/store.go`). |
| D15 (cierra P9) | La declaración es la **única lista de secretos**; `.env` solo guarda preferencias no secretas. `webtyp/env` resuelve una clave declarada secreta desde el keyring. | Se borran `keyring.Scheme`/`IsReference` (`keyring/reference.go`) y la lectura de `.env` en `app.MissingSecrets` (`app/missing_secrets.go`), que pasa a leer la declaración. |
| D16 (cierra P10) | Puller de Windows, SSH y webhook salen de `deploy` a **`webtyp/selfhost`**, que implementa `deploy.Target`, **solo si tienen un consumidor real**; si no, se borran (quedan en el historial de git). | Verificar consumidores en `veltylabs/*` y `webtyp/*` antes de despachar `deploy`. |
| D17 (cierra P11) | La pestaña **DEPLOY** corre el diagnóstico al abrir el proyecto: checklist ✅/❌ con el texto de cada `deploy.Missing`; cada ❌ con solución automática se resuelve con un paso; "Desplegar" se habilita sin ❌; el run de CI se ve en vivo. | El agente ve la misma checklist por MCP (D21). |
| D18 (cierra P12) | El `Target` **crea D1, R2, dominio y regla de rate limit si faltan** (idempotente) con los nombres de la declaración. | `D1_DATABASE_ID` y `R2_BUCKET_ID` dejan de ser secretos de GitHub: se resuelven por nombre. |
| D19 (cierra P13) | Repo nuevo **`webtyp/github`**: secretos de Actions (`deploy.Destination`), `workflow run`, seguimiento del run. | `ghaction` sigue siendo solo generación de YAML. |
| D20 (cierra P14) | `DEV_AUTOLOGIN` es un secreto declarado más: keyring + wizard. | Se actualiza la skill **dev-autologin** (`SKILL.md:24-26`) en el mismo cambio en que `env` lo resuelve. |

### Agregadas por el usuario al consolidar

| # | Decisión | Consecuencia |
|---|---|---|
| D21 | **El deploy es herramienta MCP de webtyp.** El agente puede ver qué falta, guiar al desarrollador paso a paso (incluido el paso de consola de Google de D12) y desplegar. **Los valores secretos solo se escriben en el wizard de la TUI**: ninguna herramienta MCP recibe ni devuelve un secreto (D1). | Herramientas en `app`, como `mcp.ToolProvider` en `buildProjectProviders` (`app/mcp_registry.go:25`): `deploy_status` (lectura: checklist sin valores); `deploy_guide` (lectura: pasos de un ítem que falta, con URLs y valores no secretos ya calculados, p. ej. el redirect URI); `deploy_provision` (escritura: corre lo automático que no necesita un secreto nuevo: crear recursos, mintar tokens, copiar secretos ya presentes); `deploy_release` (escritura: dispara el workflow y devuelve el estado del run). Las cuatro y la TUI usan el mismo diagnóstico (una sola implementación). |
| D22 | **El wizard habla inglés y español.** | `wizard` no traduce hoy (no importa `webtyp/lang`). `Step.LabelText`, `deploy.Secret.What/Source` y `deploy.Missing` pasan a `lang.Text`; `wizard`, `deploy`, `goflare` y `github` publican su `Dictionary()` con la columna `es` completa, y `app/tui_lang.go:19` los carga junto a `devtui`, `devbrowser`, `server` y `sitec`. El idioma es el que ya elige la TUI (`TUI_LANG`). Las claves siguen la skill **translations** (inglés, sustantivo antes que adjetivo, sin género). La salida de MCP queda en inglés: el agente la transmite en el idioma del desarrollador. |

---

## Preguntas abiertas

Responde inline con `respuesta: X` (o texto libre) y luego «consolida el master».

### P15 — ¿Puede el agente desplegar a producción sin confirmación en la TUI?

- **A) (recomendada)** Sí: `deploy_release` es una herramienta MCP de escritura y el permiso lo da el
  harness del agente (Claude Code pide aprobación para herramientas de escritura salvo que el dev la
  haya autorizado). Un deploy es reversible (D8: CI; versiones del Worker en Cloudflare).
- B) `deploy_release` solo deja el deploy "listo" y el dev confirma con una tecla en la TUI. Más
  seguro, pero el agente no puede cerrar el ciclo solo, que es lo que pide D21.

### P16 — Idiomas del diccionario más allá de inglés y español

Los diccionarios del ecosistema tienen 8 columnas (`es zh hi ar pt fr de ru`, p. ej.
`devtui/lang.json`).

- **A) (recomendada)** `es` obligatorio en esta ola; las otras 7 columnas pueden quedar vacías (se
  muestra la clave inglesa) y `lang_missing` las lista para completarlas después.
- B) Las 8 completas en esta ola. Más trabajo por plan sin que nadie las use hoy.

### P17 — Secretos que hoy existen solo en GitHub

Los secretos de GitHub son de solo escritura: los valores actuales de misitio e iam no se pueden
leer para pasarlos al keyring.

- **A) (recomendada)** **Rotarlos**: el wizard mintea tokens nuevos (D10), genera claves nuevas
  donde la fuente es `Generated`, y pide de nuevo solo lo que viene de fuera (p. ej. el cliente
  OAuth de Google de iam). El token de misitio está mal escopeado de todos modos.
- B) Pedirle al dev que vuelva a pegar cada valor viejo. Requiere que los tenga guardados en algún
  lado, que es justo lo que D1 prohíbe.

### P18 — Vida del token de arranque de Cloudflare

Cloudflare recomienda limitarlo con TTL o filtro de IP.

- **A) (recomendada)** TTL de 90 días; el diagnóstico (D17) avisa 7 días antes y el wizard pide uno
  nuevo. El filtro por IP no sirve con IP doméstica dinámica.
- B) Sin TTL. Un token que puede crear tokens con cualquier permiso no debería ser eterno.

### P19 — Deploys de vista previa por PR

Versiones de Worker con URL propia por PR, como Vercel/Netlify: ver un PR de Jules funcionando sin
mergearlo.

- **A) (recomendada)** Siguiente ola. Necesita decidir qué D1/R2 usa una vista previa (¿copia?,
  ¿la de producción en solo lectura?), que es una pregunta propia.
- B) En esta ola.

### P20 — Orden de los planes

- **A) (recomendada)** `wizard` (pasos dinámicos + `lang.Text`) y `deploy` (contratos) primero, en
  paralelo; luego `goflare`, `github`, `env`+`keyring` y `selfhost` en paralelo; luego `app`; al
  final `misitio` como aceptación. Antes de despachar `deploy`, verificar los consumidores del
  Puller (D16).
- B) Un solo plan grande en `app`. Viola D2/D3: `app` terminaría con el conocimiento de Cloudflare
  y GitHub adentro.

---

## Evidencia — estado actual

| Hecho | Dónde |
|---|---|
| `deploy` importa `goflare` y tiene un proveedor Cloudflare dentro | `deploy/go.mod:8`, `deploy/providers/cloudflare/provider.go` |
| Registro global de pushers en `init()` | `deploy/pusher.go:21-27`, `deploy/pusher_cloudflare.go:5` |
| Wizard con menú impreso a mano y 5 pasos "proxy" fijos porque `wizard` no admite pasos dinámicos | `deploy/wizard.go:28-104`, `wizard/step.go:7-11` |
| `wizard` no traduce: `LabelText` es `string` y el paquete no importa `webtyp/lang` | `wizard/step.go` |
| La TUI ya carga diccionarios por librería y guarda el idioma elegido | `app/tui_lang.go:19-35` |
| Las herramientas MCP del proyecto se registran como `mcp.ToolProvider` | `app/mcp_registry.go:25`, `app/mcp-tools.go:20` (`lang_missing`) |
| Lista fija de qué es secreto; el token de arranque no está | `deploy/store.go:30-42` |
| Dos almacenes de secretos: `deploy.SecureStore` y la pestaña SECRETS | `deploy/store.go`, `app/section-secrets.go`, `app/missing_secrets.go` |
| `goflare` solo lee el token desde la variable de entorno (CI) | `goflare/auth.go:12-24` |
| `goflare` ya resuelve dominio y rate limit; no crea D1/R2 | `goflare/cloudflare.go:306,360,426` |
| Los IDs de D1/R2 se copian a mano como "secretos" de GitHub | `misitio/docs/DEPLOY.md:54-57,107-108` |
| Token de Cloudflare mal escopeado → 403 en cada deploy desde v0.0.7 | run 38746286153; `misitio/docs/PLAN_INFRA_BLOQUEADA.md` |
| `GOOGLE_*` documentados para misitio, sin uso en el código | `misitio/docs/DEPLOY.md:74-76` |

## Industria — cómo despliegan otros

| Herramienta | Declaración | Secretos | Aprovisiona | Diagnóstico |
|---|---|---|---|---|
| Vercel / Netlify | detecta el framework; `vercel.json` opcional | UI o `vercel env add`, guardados en la plataforma | sí (proyecto) | log del build |
| Fly.io | `fly.toml` generado por `fly launch` | `fly secrets set`, cifrados en la plataforma, nunca en archivo | sí (app, Postgres, IPs) | `fly doctor` |
| Kamal 2 (Rails) | `config/deploy.yml` | `.kamal/secrets` obtiene valores de 1Password/Bitwarden con adaptadores | no (servidores propios) | `kamal config` |
| SST / Encore | **código** (`sst.config.ts`; infra inferida del código Go en Encore) | `sst secret set` → gestor del proveedor de nube | sí, todo | errores tipados al compilar |
| Cloudflare Wrangler | `wrangler.toml` | `wrangler secret put` | parcial (`d1 create` a mano) | — |

Patrón común: (1) **login una vez** con el proveedor, (2) un comando que **detecta, pregunta solo lo
que falta y crea**, (3) secretos guardados en el almacén del destino, **nunca en archivos**, (4)
diagnóstico antes de desplegar. Donde webtyp difiere: Cloudflare no ofrece OAuth a terceros (de ahí
D10); la declaración es Go leído por AST (como SST/Encore, sin ejecutar el proyecto); y el agente es
un usuario de primera clase por MCP (D21), cosa que ninguna de las anteriores ofrece.

## Sugerencias (no bloquean esta ola)

1. **Docs de despliegue generados desde la declaración.** `DEPLOY.md` de misitio quedó
   desactualizado (`GOOGLE_*`); si la tabla de secretos sale de `deploy.Secret.What/Source`, no
   puede divergir. Candidata a decisión al escribir el plan de `app`.
2. **Rollback de un paso** desde la TUI y por MCP (Cloudflare guarda las versiones del Worker).
3. **Vista previa por PR**: P19.

## Puerta api-design

- **Precedentes:** SST, Encore, Fly, Kamal, Vercel (tabla anterior).
- **Nombres:** `deploy.Plan`, `deploy.Target`, `deploy.Destination`, `deploy.Secret`,
  `deploy.Missing`; herramientas `deploy_status`, `deploy_guide`, `deploy_provision`,
  `deploy_release`. Se leen como frases; se descartan `Sink`, `Pusher`, `Puller` (jerga).
- **Balance de complejidad** (misitio):
  ```
  Conceptos a aprender            +2 (Plan, Secret)  / −4 (Pusher, Puller, SecureStore, keyring://)
  Archivos a tocar para desplegar +1 (config/deploy.go) / −2 (deploy.yml a mano, DEPLOY.md a mano)
  Pasos manuales                  0 / −9 (crear D1, R2, token, 4× gh secret set, 2× secretos de Worker)
  Formas de hacer lo mismo        −3 (dos almacenes de secretos, dos listas de secretos, deploy local vs CI)
  ```
- **Dónde va:** contratos en `deploy`; Cloudflare en `goflare`; GitHub en `webtyp/github`;
  composición, TUI y MCP en `app`.
- **Qué borra:** D3, D7, D8, D13, D14, D15, D16, y en misitio las secciones manuales de
  `DEPLOY.md`, `CI_TOKEN.md` y `PLAN_INFRA_BLOQUEADA.md` que el wizard reemplaza.

## Librerías

| Repo | Cambio | Tipo | Plan |
|---|---|---|---|
| `webtyp/wizard` | pasos agregados en tiempo de ejecución (elimina los 5 proxy de `deploy/wizard.go`); `LabelText lang.Text`; `Dictionary()` | **gate** de `app` | pendiente (P20) |
| `webtyp/deploy` | solo contratos (D13); borra implementaciones y almacén (D3, D14, D16); `Dictionary()` | **gate** de goflare, github, selfhost, app | pendiente |
| `webtyp/goflare` | `deploy.Target` (Worker); token por proyecto desde el de arranque (D10); D1/R2 si faltan (D18); `deploy.Destination` para secretos del Worker (D9); `Dictionary()` | paralelo | pendiente |
| `webtyp/github` (nuevo) | `deploy.Destination` para secretos de Actions; `workflow run` + seguimiento (D19); `Dictionary()` | paralelo | pendiente |
| `webtyp/selfhost` (nuevo, condicional) | Puller/SSH/webhook como `deploy.Target`, solo si hay consumidor (D16) | paralelo | pendiente |
| `webtyp/env` | resuelve claves declaradas secretas desde el keyring (D15) | paralelo | pendiente |
| `webtyp/keyring` | borra `Scheme`/`IsReference` (D15) | con `env` | pendiente |
| `webtyp/app` | pestaña DEPLOY: diagnóstico + wizard + run en vivo (D17); MCP `deploy_*` (D21); carga de diccionarios (D22); inyecta goflare y github | tras los anteriores | pendiente |
| `veltylabs/misitio` | `config/deploy.go`; borra pasos manuales de sus docs; rota secretos (P17) | último, aceptación | pendiente |

**Aceptación de la ola:** en una máquina con solo `gh auth login` hecho y el token de arranque de
Cloudflare en el keyring: (1) un agente llama `deploy_status` sobre misitio y recibe la lista de lo
que falta, sin valores; (2) con `deploy_guide` guía al dev, que escribe cada valor solo en el wizard
de la TUI, en español o inglés según `TUI_LANG`; (3) `deploy_provision` y `deploy_release` dejan el
Worker respondiendo en `misitio.velty.cl`; (4) ningún secreto queda en `.env`, en el repo ni en el
chat (`grep` de los valores en repo y transcripción → vacío).
