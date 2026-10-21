# Destinos de compilación — master plan

> **Status:** IN PROGRESS · 2026-10-21 · D1–D16 cerradas (dos ejes: `wasm` de plataforma + `browser` de destino; `web/{browser,edge,server}.go` → `{browser.wasm,edge.wasm,server}`); todos los gates cerrados (`js`, `html`, `sitec`, `devskills`, `router`, `devflow` v0.4.146); siguientes: planes de `app`, `goflare` y módulos. **Cambio sin retrocompatibilidad en todo webtyp.**

**Disparador:** misitio es un proyecto **100 % cloud** (Cloudflare Worker) que se prueba en local
con `web/server.go`. `//go:build wasm`/`!wasm` no distingue el Worker del navegador (los dos son
wasm), así que `config/` terminó mezclando servidor y navegador, aparecieron `api/` y `routes/`
como parches, y `client.wasm` llegó a 682 KB gzip contra un presupuesto de 512 KB (2026-10-15).
Fallas medidas: M1–M8 en «Plan siguiente de misitio» de
[DEPLOY_WIZARD_MASTER_PLAN.md](DEPLOY_WIZARD_MASTER_PLAN.md).

**Objetivo:** que el **nombre del archivo diga a qué binario va**, igual en proyectos nativos y
cloud, y que cada binario compile solo lo suyo.

**Relación con otras olas:** reemplaza las reglas de etiquetas de las skills **app-modules**,
**project-layout**, **app-server-main**, **app-rest-edge**, **wasm** y la línea de etiquetas del
bloque base de **agents-md**. Confirma D34 de DEPLOY_WIZARD (`config/deploy.go` con
`//go:build deploy` ya sigue la regla). misitio es el primer caso; su plan siguiente espera esta ola.

---

## Decisiones

### Fijadas por el usuario (2026-10-15)

| # | Decisión | Consecuencia |
|---|---|---|
| D1 | **El destino es el nombre del archivo y de su binario**; su etiqueta es la del vocabulario de D15: `web/browser.go` → `//go:build browser` → `web/public/browser.wasm` (`browser.<hash>.wasm` en release), `web/edge.go` → `//go:build wasm && !browser` → `web/edge.wasm` (+ `web/edge.js`), `web/server.go` → `//go:build !wasm` → `web/server`. | `web/client.go` → `web/browser.go`; `client.wasm` → `browser.wasm` (`js.DefaultWasmURL = "/browser.wasm"`). La carpeta `edge/` desaparece de todos los proyectos: su `main.go` pasa a `web/edge.go`. |
| D2 | **En código de aplicación `wasm` deja de significar "navegador".** Lo que solo corre en el navegador lleva `browser`; la lógica de servidor que comparten Worker y servidor nativo lleva `!browser`. `wasm`/`!wasm` quedan solo para lo que depende de la plataforma (D15). | Cada `//go:build wasm` de una app que en realidad es "navegador" → `browser`; cada `!wasm` que es "lógica de servidor" → `!browser`. |
| D3 | **Cambio sin retrocompatibilidad**, en todo webtyp. | Sin período de convivencia: cada repo pasa al esquema nuevo en su plan, y las herramientas dejan de entender el viejo en el mismo cambio. |

### Derivadas (2026-10-15)

| # | Decisión | Por qué |
|---|---|---|
| D4 | **Solo el compilador del cliente pasa etiqueta de destino**: `-tags browser` (`-tags browser,dev` en desarrollo, como hoy en `app/client_builder_init.go:33`). `goflare` (Worker) y el servidor nativo no pasan etiqueta de destino; el servidor solo suma `dev` en desarrollo. | `wasm` lo fija el compilador por `GOARCH=wasm`; con eso y `browser` los tres destinos quedan distinguidos (D15). |
| D5 | **El vocabulario completo** (D15): `browser`, `wasm && !browser`, `!wasm`, `wasm`, `!browser`, y sin etiqueta. | Cubre las 6 combinaciones posibles de 3 destinos con una sola etiqueta nueva. |
| D6 | **Módulos** (skill **app-modules**): `server.go` → `//go:build !browser`; `browser.go` → `//go:build browser`; `module.go` sin etiqueta. | Hoy `server.go` es `!wasm` y por eso no entra al Worker; con `!browser` entra al Worker y al servidor, y sale del navegador. |
| D7 | **Archivos que lee `sitec` en tiempo de build** (`css.go`, `svg.go`, `html.go`, `js.go`, `favicon.go`) **siguen con `//go:build !wasm`**. En `sitec` solo cambian la entrada/salida por defecto del navegador (`web/browser.go` → `browser.wasm` compilado con `-tags browser`) y el sondeo de paquetes alcanzables (`-tags=browser` en `js/wasm`). | `!wasm` ya deja los productores de assets fuera del navegador **y** del Worker (con `!browser` entrarían al Worker y pesarían en su límite de 1 MB); el sondeo con `-tags=browser` evita perder el CSS de paquetes importados solo por archivos `browser`. |
| D8 | **`gotest` corre una pasada por destino presente** en el módulo: nativo sin etiqueta (servidor), `GOOS=js GOARCH=wasm` sin etiqueta (Worker y wasm compartido) y `GOOS=js GOARCH=wasm -tags browser` (navegador) con Chrome headless (`wasmbrowsertest`), más la de `dev` que ya existe en `devflow` v0.4.132 (D16). | Los archivos `browser` no compilan en `go test` simple; sin esto, los tests de vista (`tests/<m>_view_test.go`) dejarían de correr. |
| D9 | **La declaración de despliegue sigue en `config/deploy.go` con `//go:build deploy`** (D34 de DEPLOY_WIZARD sin cambios). | Ya cumple D1: el nombre del archivo es su etiqueta, y ningún binario lo compila. |
| D10 | **`layoutscan` (`webtyp/router`) verifica**: cada `main` de `web/` lleva exactamente la etiqueta de D1; `browser` nunca se combina con `!wasm`; no existe `edge/`. **`goflare` rechaza `-tags browser`.** | El Worker se define por ausencia de `browser` (D15): estos chequeos impiden que un build olvidado o mal etiquetado salga en silencio como otro destino. |

**Límite conocido:** el editor (`gopls`) analiza una sola combinación de etiquetas a la vez.
`webtyp/app` ya escribe la configuración del IDE (`app/client_vscode_config.go`): la vista por
defecto es nativa sin etiquetas (servidor); los archivos `browser`/wasm se verifican con `gotest` y
`go vet` por destino (D8). Es el mismo costo que hoy tiene cualquier archivo `wasm` en el editor.


### Cerradas al consolidar (2026-10-15)

| # | Decisión | Consecuencia |
|---|---|---|
| D11 (cierra P1) | **Un solo `web/server.go`; "desarrollo" es un modo (`-tags dev`), no un destino.** En un proyecto cloud (existe `web/edge.go`) el despliegue es el Worker y `web/server.go` solo sirve para probar en local, componiendo los mismos módulos. | Sin `web/dev.go` ni `web/local.go`. |
| D12 (cierra P2) | **Una sola carpeta de salida: `web/`**, igual en desarrollo y en despliegue; **no existe `.build/`** en ningún proyecto. Lo que se sirve va a `web/public/` (`web/public/browser.wasm`). Los binarios que no son públicos quedan junto a su `main`: `web/server` (ya es así) y `web/edge.js` + `web/edge.wasm`. | Implicación: el Worker **no** va a `web/public/`, porque todo lo de `public` se sirve a cualquiera y expondría el código del servidor. `goflare` cambia `OutputDir` por defecto de `.build/` a `web/`; desaparecen `.build/` y `/edge.wasm` de la raíz (misitio los tiene en `.gitignore`). `.gitignore` de cada app: `/web/public/`, `/web/server`, `/web/edge.*`. |
| D13 (cierra P3) | **Sin adaptadores locales nuevos**: los contratos de almacenamiento ya existen; el desorden venía de `config/` mezclando código de servidor y navegador, y D1–D7 lo resuelven. | Se elimina el paquete `cloudflare/local` del alcance. `web/server.go` de un proyecto cloud compone con los adaptadores que ya existen. |
| D14 (cierra P4, reescrita por D15) | **Las librerías conservan `wasm`/`!wasm`** cuando la separación se debe a una API de plataforma (`syscall/js`, `os`, `net`): `env`, `fetch`, `dom`, `keyring/browser`, `cloudflare/*`, `json`, `time`, `fmt`… **no se tocan.** Solo suman `browser`/`!browser` los archivos de un paquete que importan **a la vez** el navegador y el Worker y deben comportarse distinto en cada uno. | En una librería la importación ya decide el destino: `dom` lleva `wasm` y nunca entra al Worker porque el Worker no lo importa. La migración de librerías baja de 297 archivos a los módulos (D6) y esos pocos casos mixtos. |
| D15 (cierra P6) | **Dos ejes, una sola etiqueta nueva.** Plataforma: `wasm`/`!wasm` (automática, la fija el compilador). Destino: `browser`/`!browser` (la pasa solo el compilador del cliente). Tabla completa: solo navegador → `browser`; solo Worker → `wasm && !browser`; solo servidor nativo → `!wasm`; navegador + Worker → `wasm`; servidor + Worker → `!browser`; los tres → sin etiqueta. Regla para el agente: *"¿por qué se separa? Si la API no existe en esa plataforma → `wasm`/`!wasm`; si pertenece a otro binario → `browser`/`!browser`."* | Se eliminan las etiquetas `edge` y `server` del diseño anterior. `wasm && !browser` aparece en la práctica solo en `web/edge.go` (las librerías del Worker ya llevan `wasm` y el navegador no las importa). **Costos aceptados:** (1) el Worker se define por ausencia de `browser`: lo cubren D4 (una sola herramienta pasa la etiqueta) y D10; (2) vale mientras exista un solo destino wasm además del navegador: un tercer runtime wasm agregaría su propia etiqueta en ese momento. **Nombre `browser`:** precedente directo en las condiciones de `exports`/campo `browser` de `package.json` (`browser`, `node`, `workerd`), que separan exactamente navegador de Worker; coincide con el nombre de archivo ya usado (`browser.go` de módulos, `web/browser.go`); `js` está tomada por `GOOS=js` (navegador **y** Worker), y `client` choca con `fetch/client_*.go` y es ambiguo (el Worker también es cliente HTTP). |

---

## Por qué romper todo webtyp (justificación)

1. **La causa raíz es de nombres, no de un proyecto.** `wasm`/`!wasm` describe la **plataforma**
   y se usó como si describiera el **destino**. Coincide en proyectos nativos y falla en cuanto
   aparecen dos destinos wasm (navegador + Worker). misitio no está "mal hecho": siguió la regla y
   la regla no alcanzaba. La corrección no es borrar el eje de plataforma sino sumarle el único
   dato que le faltaba: si el wasm es el navegador (D15).
2. **El nombre del archivo ya es la convención de Go para el destino**: `env_js.go`,
   `file_windows.go`, `foo_wasm.go` se compilan solo para su `GOOS`/`GOARCH` sin escribir nada más.
   D1 extiende esa idea a los destinos de webtyp. Precedentes fuera de Go: Remix y React Router
   (`.server.ts`/`.client.ts`), React Server Components (`.server.js`/`.client.js`), condiciones
   `browser`/`workerd` de `package.json`, Flutter (carpetas por plataforma).
3. **Sin retrocompatibilidad** porque convivir dos esquemas es tener dos formas de decir lo mismo
   (balance de **api-design**: nunca positivo), y el ecosistema no tiene usuarios externos que se
   rompan: el costo de renombrar solo crece.
4. **Balance de complejidad:**
   ```
   Conceptos a aprender         +1 (browser)  −3 (wasm/!wasm como destino, api/, routes/)
   Carpetas raíz por proyecto   −1 (edge/)
   Formas de separar destinos   −1 (etiqueta de plataforma + paquetes parche → plataforma + browser)
   Herramientas que pasan tags  1 (solo el compilador del cliente)
   ```

---

## Preguntas abiertas

Ninguna.

### D16 (cierra P5) — Dónde corren los tests de lo que va al navegador y al Worker

**Cada destino se prueba en su plataforma**: servidor en nativo; navegador con
`GOOS=js GOARCH=wasm -tags browser` en navegador headless (`webtyp/wasmbrowsertest`, que el
ecosistema ya tiene y `gotest` ya sabe invocar para wasm); Worker compila y pasa `vet` con
`GOOS=js GOARCH=wasm` sin etiqueta (más el test de tamaño con TinyGo que ya existe). Es la prueba
honesta: el binario real de esos destinos es wasm. Costo aceptado: los tests de vista dejan de ser
"stdlib rápidos" y piden Chrome headless (ya requerido por los tests wasm actuales).

---

## Repos afectados

| Repo | Cambio | Tipo | Plan |
|---|---|---|---|
| devskills | **project-layout**, **app-modules**, **app-server-main**, **app-rest-edge**, **wasm**, bloque base de **agents-md**: dos ejes y tabla de D15, D6, `web/{browser,edge,server}.go` → `{browser.wasm,edge.wasm,server}`, salida única en `web/` (D12) | primero (lo leen los agentes de todos los planes) | **cerrado** v0.0.23 |
| `webtyp/js` | `DefaultWasmURL = "/browser.wasm"` (D1); docs y tests (`browser.wasm` / `browser.<hash>.wasm`) | **gate** de `sitec` y `app` | **cerrado** v0.1.3 |
| `webtyp/html` | `scaffold/client.go` → `scaffold/browser.go` con `//go:build browser` (`tests/scaffold_test.go` compila con `-tags browser` hacia `browser.wasm`, D1/D4) | **gate** de `app` | **cerrado** v0.0.28 |
| `webtyp/sitec` | `WasmBuildOptions` por defecto `web/browser.go` → `browser.wasm` (`browser.<hash>.wasm` en release) pasando `-tags browser` cuando compila `browser.go` (D1, D4, D7); `build.go` detecta `web/browser.go`; `reach.go` pasa `-tags=browser` en el sondeo `js/wasm` | **gate** de `goflare` y `app` (tras `js`) | **cerrado** v0.2.66 |
| `webtyp/devflow` | `gotest`: una pasada por destino, cada una en su plataforma (D8, D16); `gonew` no tiene plantillas afectadas | **gate** | **cerrado** v0.4.146 |
| `webtyp/router` | `layoutscan` (D10) | **gate** | **cerrado** v0.4.5 |
| `webtyp/app` | cliente `-tags browser` (`web/browser.go` → `web/public/browser.wasm`, `scaffold/browser.go`); servidor sin etiqueta de destino; configuración del IDE | tras gates (`js`, `html`, `sitec`, `devflow`, `router`) | [Antigravity · 2026-10-21 17:19] despachado (sesión 13590018183934190997) |
| `webtyp/server` | generador de `web/server.go` con `//go:build !wasm` | — | **cerrado**: `maingen.go` ya lo emite |
| `webtyp/goflare` | compila `web/edge.go` sin etiqueta de destino (deja de leer `edge/`) y rechaza `-tags browser` (D10); `OutputDir` por defecto `web/` (D12) | tras gates (`sitec`) | [Antigravity · 2026-10-21 17:17] despachado (sesión 6212008180401272509) |
| `veltylabs/modules/appointment_booking` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.3.14 |
| `veltylabs/modules/pharmacy_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.12 |
| `veltylabs/modules/supply_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.2.12 |
| `veltylabs/modules/service_catalog` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.5.12 |
| `veltylabs/modules/patient_directory` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.41 |
| `veltylabs/modules/staff_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.129 |
| `veltylabs/modules/business_calendar` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.4.95 |
| `veltylabs/modules/payment_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.26 |
| `veltylabs/modules/cash_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.2 |
| `veltylabs/modules/room_layout` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | [Antigravity · 2026-10-21 18:04] despachado (sesión 378554101422698965) |
| `veltylabs/modules/chat_room` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.2.84 |
| `veltylabs/modules/device_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.2.63 |
| `veltylabs/modules/network_manager` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.1.63 |
| `veltylabs/modules/clinical_encounter` | D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | módulos | **cerrado** v0.2.20 |
| demás módulos (`veltylabs/modules/*`) | D6 y D14/D17: `web/browser.go` y `ui/browser.go` con `//go:build browser` | tras gates, en paralelo, un plan por repo | por despachar |
| `veltylabs/misitio`, `veltylabs/iam`, `veltylabs/mjosefa-cms`, demás apps | migración: `edge/` → `web/edge.go` (`edge.wasm`), `web/client.go` → `web/browser.go` (`browser.wasm`), `config/client.go` → `config/browser.go`, etiquetas según D1/D2/D15, sin `.build/` ni `/edge.wasm` | al final; misitio primero (re-layout M1–M8) | plan por escribir |
