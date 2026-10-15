# Destinos de compilación — master plan

> **Status:** PENDING · 2026-10-15 · D1–D14 cerradas; P5 abierta (tests por plataforma). **Cambio sin retrocompatibilidad en todo webtyp.**

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
| D1 | **El destino es el nombre del archivo y su etiqueta de build.** `web/browser.go` lleva `//go:build browser`, `web/edge.go` lleva `//go:build edge`, `web/server.go` lleva `//go:build server`. | `web/client.go` → `web/browser.go`. La carpeta `edge/` desaparece de todos los proyectos: su `main.go` pasa a `web/edge.go`. |
| D2 | **El código de aplicación deja de usar `wasm`/`!wasm`.** | Todo `//go:build wasm`/`!wasm` en apps y en los archivos de módulos que usan las apps se reemplaza por etiquetas de destino (D4–D7). |
| D3 | **Cambio sin retrocompatibilidad**, en todo webtyp. | Sin período de convivencia: cada repo pasa al esquema nuevo en su plan, y las herramientas dejan de entender el viejo en el mismo cambio. |

### Derivadas (2026-10-15)

| # | Decisión | Por qué |
|---|---|---|
| D4 | **Cada compilación pasa exactamente una etiqueta de destino**: cliente → `-tags browser`; Worker (`goflare`) → `-tags edge`; servidor nativo → `-tags server`. `dev` se suma en desarrollo (`-tags server,dev`, `-tags browser,dev`), como hoy (`app/client_builder_init.go:33`). | Una herramienta, una etiqueta: no hay combinación que aprender. |
| D5 | **El vocabulario completo** en código de aplicación: `browser`, `edge`, `server`, y `!browser` para la lógica de servidor que comparten el Worker y el servidor nativo. Un archivo sin etiqueta va a los tres. | `!browser` es la única negación: "todo lo que no es el navegador". |
| D6 | **Módulos** (skill **app-modules**): `server.go` → `//go:build !browser`; `browser.go` → `//go:build browser`; `module.go` sin etiqueta. | Hoy `server.go` es `!wasm` y por eso no entra al Worker; con `!browser` entra al Worker y al servidor, y sale del navegador. |
| D7 | **Archivos que lee `sitec` en tiempo de build** (`css.go`, `svg.go`, `html.go`, `js.go`, `favicon.go`): `//go:build server`. `sitec` compila con `-tags server`. | Hoy son `!wasm`; con `server` siguen fuera del navegador **y** fuera del Worker (con `!browser` entrarían al Worker y pesarían en su límite de 1 MB). |
| D8 | **`gotest` corre una pasada por cada etiqueta de destino presente** en el módulo (más la de `dev`, que ya existe en `devflow` v0.4.132), cada una en la plataforma que decida P5. | Los archivos `browser` dejan de compilar en `go test` simple; sin esto, los tests de vista (`tests/<m>_view_test.go`) dejarían de correr. |
| D9 | **La declaración de despliegue sigue en `config/deploy.go` con `//go:build deploy`** (D34 de DEPLOY_WIZARD sin cambios). | Ya cumple D1: el nombre del archivo es su etiqueta, y ningún binario lo compila. |
| D10 | **`layoutscan` (`webtyp/router`) verifica**: cada `main` de `web/` lleva la etiqueta de su nombre; no queda `//go:build wasm`/`!wasm` en código de aplicación; no existe `edge/`. | El compilador no puede ver estas reglas; el chequeo estático sí. |

**Límite conocido:** el editor (`gopls`) analiza una sola combinación de etiquetas a la vez.
`webtyp/app` ya escribe la configuración del IDE (`app/client_vscode_config.go`): fijará
`-tags server` como vista por defecto; los archivos `browser`/`edge` se verifican con `gotest` y
`go vet` por destino (D8). Es el mismo costo que hoy tiene cualquier archivo `wasm` en el editor.


### Cerradas al consolidar (2026-10-15)

| # | Decisión | Consecuencia |
|---|---|---|
| D11 (cierra P1) | **Un solo `web/server.go`; "desarrollo" es un modo (`-tags server,dev`), no un destino.** En un proyecto cloud (existe `web/edge.go`) el despliegue es el Worker y `web/server.go` solo sirve para probar en local, componiendo los mismos módulos. | Sin `web/dev.go` ni `web/local.go`. |
| D12 (cierra P2) | **Una sola carpeta de salida: `web/`**, igual en desarrollo y en despliegue; **no existe `.build/`** en ningún proyecto. Lo que se sirve va a `web/public/`. Los binarios quedan junto a su `main`: `web/server` (ya es así) y `web/edge.js` + `web/edge.wasm`. | Implicación: el Worker **no** va a `web/public/`, porque todo lo de `public` se sirve a cualquiera y expondría el código del servidor. `goflare` cambia `OutputDir` por defecto de `.build/` a `web/`; desaparecen `.build/` y `/edge.wasm` de la raíz (misitio los tiene en `.gitignore`). `.gitignore` de cada app: `/web/public/`, `/web/server`, `/web/edge.*`. |
| D13 (cierra P3) | **Sin adaptadores locales nuevos**: los contratos de almacenamiento ya existen; el desorden venía de `config/` mezclando código de servidor y navegador, y D1–D7 lo resuelven. | Se elimina el paquete `cloudflare/local` del alcance. `web/server.go` de un proyecto cloud compone con los adaptadores que ya existen. |
| D14 (cierra P4) | **Todo webtyp migra, librerías incluidas: no queda `wasm`/`!wasm` en ningún archivo.** Cada etiqueta se reemplaza por el **destino donde ese archivo realmente corre**: solo navegador → `browser` (p. ej. `dom`, `keyring/browser`); solo Worker → `edge` (`cloudflare/edge/edge.go`, `cloudflare/d1/adapter.go`); navegador y Worker → `browser \|\| edge` (`env/env_wasm.go`, `fetch/client_wasm.go`: llaman `syscall/js` en los dos); solo nativo → `server` (`env/env_native.go`, `fetch/client_stdlib.go`); servidor nativo y Worker → `!browser`. | El reemplazo literal (`wasm`→`browser`, `!wasm`→`!browser`) rompe el Worker: `cloudflare/edge/edge.go` saldría del build `edge` y `env/env_native.go` (usa `os`) entraría. Hoy hay 197 archivos `!wasm` y 100 `wasm` en `webtyp/`: cada plan de librería clasifica los suyos con esta tabla. |

---

## Por qué romper todo webtyp (justificación)

1. **La causa raíz es de nombres, no de un proyecto.** `wasm`/`!wasm` describe la **plataforma**
   y se usó como si describiera el **destino**. Coincide en proyectos nativos y falla en cuanto
   aparecen dos destinos wasm (navegador + Worker). misitio no está "mal hecho": siguió la regla y
   la regla no alcanzaba.
2. **El nombre del archivo ya es la convención de Go para el destino**: `env_js.go`,
   `file_windows.go`, `foo_wasm.go` se compilan solo para su `GOOS`/`GOARCH` sin escribir nada más.
   D1 extiende esa idea a los destinos de webtyp. Precedentes fuera de Go: Remix y React Router
   (`.server.ts`/`.client.ts`), React Server Components (`.server.js`/`.client.js`), Flutter
   (carpetas por plataforma).
3. **Sin retrocompatibilidad** porque convivir dos esquemas es tener dos formas de decir lo mismo
   (balance de **api-design**: nunca positivo), y el ecosistema no tiene usuarios externos que se
   rompan: el costo de renombrar solo crece.
4. **Balance de complejidad:**
   ```
   Conceptos a aprender         +3 (browser, edge, server)  −3 (wasm/!wasm como destino, api/, routes/)
   Carpetas raíz por proyecto   −1 (edge/)
   Formas de separar destinos   −1 (etiqueta de plataforma + paquetes parche → solo etiqueta de destino)
   ```

---

## Preguntas abiertas

Responde inline con `respuesta: X` (o texto libre) y luego «consolida el master».

### P5 — ¿Dónde corren los tests de lo que va al navegador y al Worker?

Hoy `browser.go` de un módulo no lleva etiqueta para que su test de vista corra en Go nativo
(skill **app-modules**), y los archivos que llaman `syscall/js` quedan fuera de los tests nativos
porque `wasm` es la arquitectura. Con D1/D14 un archivo `browser` o `edge` que importa
`syscall/js` **no compila en nativo** (`syscall/js` solo existe en `js/wasm`), así que
`go test -tags browser` nativo falla. D8 ("una pasada por etiqueta") necesita decir en qué
plataforma corre cada pasada.

- **A) (recomendada)** **Cada destino se prueba en su plataforma**: `server` en nativo; `browser`
  con `GOOS=js GOARCH=wasm` en navegador headless (`webtyp/wasmbrowsertest`, que el ecosistema
  ya tiene y `gotest` ya sabe invocar para wasm); `edge` compila y pasa `vet` con
  `GOOS=js GOARCH=wasm -tags edge` (más el test de tamaño con TinyGo que ya existe). Es la prueba
  honesta: el binario real de esos destinos es wasm. Contra: los tests de vista dejan de ser
  "stdlib rápidos" y piden Chrome headless (ya requerido por los tests wasm actuales).
- B) Mantener los tests de vista en nativo: los archivos `browser` de las apps no pueden importar
  `syscall/js` (solo librerías), y las librerías `browser` traen un doble nativo para tests.
  Duplica código en cada librería de navegador.

---

## Repos afectados (los planes se escriben al cerrar P5)

| Repo | Cambio | Tipo |
|---|---|---|
| devskills | **project-layout**, **app-modules**, **app-server-main**, **app-rest-edge**, **wasm**, bloque base de **agents-md**: vocabulario D4–D7 y D14, `web/{browser,edge,server}.go`, salida única en `web/` (D12) | primero (lo leen los agentes de todos los planes) |
| `webtyp/devflow` | `gotest`: una pasada por etiqueta de destino, cada una en su plataforma (D8, P5); plantillas de `gonew` | **gate** |
| `webtyp/router` | `layoutscan` (D10) | **gate** |
| `webtyp/app` | cliente `-tags browser`, servidor `-tags server`; `web/browser.go`; configuración del IDE | tras gates |
| `webtyp/server` | generador de `web/server.go` con `//go:build server` | tras gates |
| `webtyp/goflare` | compila `./web` con `-tags edge` (deja de leer `edge/`); `OutputDir` por defecto `web/` (D12) | tras gates; se fusiona con su plan pendiente (D41 de DEPLOY_WIZARD) |
| `webtyp/sitec` | compila productores con `-tags server` (D7) | tras gates |
| **todas las librerías de `webtyp/` y `veltylabs/`** con `wasm`/`!wasm` (297 archivos en `webtyp/`) | etiquetas según la tabla de D14; `css.go`/`svg.go` → `server` | tras gates, en paralelo, un plan por repo |
| `veltylabs/misitio`, `veltylabs/iam`, `veltylabs/mjosefa-cms`, demás apps | migración: `edge/` → `web/edge.go`, `client.go` → `browser.go`, etiquetas, sin `.build/` ni `/edge.wasm` | al final; misitio primero (re-layout M1–M8) |
