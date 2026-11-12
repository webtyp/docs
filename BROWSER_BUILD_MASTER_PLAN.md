# Master plan — compilar Go y webtyp en el navegador, sin servidor y sin kernel Linux

> PENDING · 2026-11-12 — decisiones D1–D4 tomadas; fase 0 (prueba de viabilidad) sin despachar. Sin código en este documento: solo alcance, piezas, enlaces y orden.

## Objetivo

Que el dev abra una página, edite un proyecto webtyp y vea el resultado en segundos, con la compilación **ejecutándose dentro del navegador**: sin servidor de build, sin instalar nada y sin depender de JavaScript escrito a mano que haya que mantener (la parte de JS se limita a un arranque mínimo y fijo; ver D5).

No se busca un Linux. Se busca **la mínima superficie de sistema operativo** que necesitan `compile` y `link` de Go: archivos, variables de entorno, argumentos, reloj y memoria.

## Respuesta corta a «¿se puede ejecutar Linux en el navegador?»

Sí, de tres formas. Ninguna es la más eficiente para este caso:

| Enfoque | Qué es | Por qué no es la base |
|---|---|---|
| Emular un PC completo | [v86](https://github.com/copy/v86) emula x86 en Wasm/JS y arranca un Linux real | 32 bits, lento para compilar, arranque de segundos y memoria alta |
| Emular el sistema de llamadas de Linux | [CheerpX / WebVM](https://labs.leaningtech.com/cheerpx) compila x86 a Wasm en caliente (JIT) y emula syscalls; [WebVM](https://github.com/leaningtech/webvm) trae un Debian | Rápido para una VM, pero es un motor externo, comercial fuera de FOSS/personal, y cerrado |
| Convertir un contenedor a Wasm | [container2wasm](https://github.com/ktock/container2wasm) empaqueta una imagen (p. ej. Alpine) con un emulador de CPU en Wasm; [demo](https://ktock.github.io/container2wasm-demo) | Emulas CPU + kernel + `go`: tres capas para ejecutar un programa que ya es Go |

Como el compilador de Go está escrito en Go, la ruta eficiente es **compilar el propio compilador a Wasm** y ejecutarlo sobre un sistema de archivos virtual. Se elimina la CPU emulada y el kernel.

## Decisiones tomadas

| ID | Decisión | Razón |
|---|---|---|
| D1 | Compilador del navegador: **Go gc (`compile` + `link`) como Wasm**; **TinyGo queda diferido** (fase 6) | gc no necesita LLVM. TinyGo sí (LLVM, clang, wasm-ld) y no hay precedente público de TinyGo-compilador en el navegador que se haya encontrado. La vista previa del dev tolera el binario grande (~2 MB vs ~100 KB de TinyGo, cifras de `BUILD_PIPELINE_MASTER_PLAN.md`) |
| D2 | Runtime: **WASI + sistema de archivos virtual propio**, sin kernel. Linux real (container2wasm) solo como respaldo opcional (fase 7) | Menos capas, menos memoria, mantenible. Permite probar el mismo toolchain en CI con [wazero](https://wazero.io) |
| D3 | Dependencias: **paquetes precompilados** (stdlib y `tinywasm/*`) publicados por CI y guardados en OPFS; el navegador compila solo el código del usuario | Builds incrementales y offline. Compilar la stdlib en cada sesión sería inviable |
| D4 | Plataforma: **COOP/COEP** (aislamiento entre orígenes) + Chromium, Firefox y Safari recientes; OPFS como almacenamiento | Hilos y memoria compartida para paralelizar; OPFS estable en los tres motores |

## Decisiones abiertas (resolver antes de despachar la fase que las usa)

| ID | Pregunta | Sugerencia y justificación | Bloquea |
|---|---|---|---|
| D5 | ¿Cómo se ejecuta un módulo Wasm desde Wasm? Un Wasm no puede instanciar otro sin la API del navegador | Mantener **un único archivo de arranque fijo y mínimo** (no se edita nunca) que expone «instanciar módulo WASI» y lo demás en Go sobre `tinywasm/jsvalue`. Alternativa: usar [browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim) tal cual y escribir solo el orquestador en Go. Medir ambas en fase 0 | F1 |
| D6 | ¿Módulos de terceros en `go.mod` (no `tinywasm/*`)? | Fase inicial: solo un conjunto cerrado (stdlib + `tinywasm/*` + lista blanca). Luego, descargar fuente vía proxy con CORS y compilar en el cliente. Evita el problema de red en el MVP | F3 |
| D7 | ¿Versión de Go fijada? | Fijar **una** versión del toolchain por release del sandbox; la stdlib precompilada debe ser byte-compatible con el compilador. Subir de versión = regenerar todo en CI | F2 |
| D8 | Anfitrión del compilador: `GOOS=wasip1` o `GOOS=js` | Sugerencia: **wasip1**. Es el contrato estándar ([blog de Go](https://go.dev/blog/wasi)), corre en wazero para pruebas nativas y no arrastra `syscall/js`. El **destino** del usuario sigue siendo `js/wasm` | F0 |
| D9 | Memoria: Wasm de 32 bits limita a 4 GB y el runtime wasip1 de Go es de un solo hilo | Paralelizar **por paquete en varios Workers**, no dentro del compilador. Medir pico de RAM en fase 0 con un proyecto webtyp real | F0 |

## Proyectos y referencias existentes

- [static-go-playground](https://github.com/Yeicor/static-go-playground): el compilador completo de Go en el navegador, sitio estático. Es la prueba de que la idea funciona. Soporta Go 1.13–1.18 y no se actualiza desde 2022 (verificar estado actual).
- [TinyGo playground](https://play.tinygo.org/) / [código](https://github.com/tinygo-org/playground): compila en un servidor; no es compilación en el cliente.
- [LiveCodes](https://github.com/live-codes/livecodes): ejecuta Go en el navegador con Yaegi (intérprete) o TinyGo (verificar si compila en cliente).
- [Yaegi](https://github.com/traefik/yaegi): intérprete de Go; alternativa de fallback para vista previa instantánea sin compilar (sin parity con wasm real).
- [Go y WebAssembly (wiki)](https://go.dev/wiki/WebAssembly) y [notas de Eli Bendersky](https://eli.thegreenplace.net/2024/notes-on-running-go-in-the-browser-with-webassembly/): detalles de `js/wasm` y `wasip1`.
- [wasmer-js](https://github.com/wasmerio/wasmer-js): runtime WASI para navegador con sistema de archivos virtual; referencia de diseño.
- [OPFS (MDN)](https://developer.mozilla.org/docs/Web/API/File_System_API/Origin_private_file_system), [COOP/COEP (web.dev)](https://web.dev/articles/coop-coep) y [coi-serviceworker](https://github.com/gzuidhof/coi-serviceworker): almacenamiento y aislamiento para hosting estático.
- [Clang en el navegador (cib)](https://github.com/tbfleming/cib): único antecedente que conozco de clang como Wasm; referencia para fase 6 (verificar vigencia).

## Librerías necesarias

Marca el estado real en la columna «Tengo» (los nombres son propuestos; las que ya aparecen en `docs/` están como «existe»).

| Librería (propuesta) | Qué haría | Tengo | Fase |
|---|---|---|---|
| `tinywasm/jsvalue` | Puente tipado a la API del navegador (Workers, OPFS, instanciar Wasm) | existe | F1 |
| `tinywasm/indexdb` | Almacén para metadatos y manifiestos (no para binarios grandes) | existe | F2 |
| `tinywasm/await` | Espera de promesas sin JS manual | existe | F1 |
| `tinywasm/fmt`, `tinywasm/base64` | Utilidades sin dependencias de la stdlib pesada | existe | todas |
| `tinywasm/client`, `tinywasm/sitec`, `tinywasm/svg`, `tinywasm/app` | Ensamblador del sitio y salida wasm; lo que el sandbox debe poder ejecutar | existe | F5 |
| `tinywasm/opfs` | Acceso a OPFS con API de archivos (lectura, escritura, directorios, bloqueo); base del sistema de archivos | falta | F1 |
| `tinywasm/vfs` | Sistema de archivos virtual en capas: memoria (rápida) sobre OPFS (persistente) y capa de solo lectura del GOROOT precompilado; semántica POSIX mínima | falta | F1 |
| `tinywasm/wasi` | Implementación del contrato WASI preview 1 sobre `vfs`: argumentos, entorno, reloj, aleatorio, descriptores y salida estándar | falta | F1 |
| `tinywasm/wasihost` | Instancia y conecta módulos WASI desde el navegador; mapea stdout/stderr a un canal tipado | falta | F1 |
| `tinywasm/proc` | «Procesos»: lanza módulos en Workers, tuberías entre ellos, código de salida, cancelación | falta | F1 |
| `tinywasm/workerpool` | Pool de Workers con memoria compartida para compilar paquetes en paralelo | falta | F3 |
| `tinywasm/goroot` | Empaqueta y versiona el GOROOT precompilado: `compile`, `link`, stdlib `.a` para `js/wasm`, tabla de hashes | falta | F2 |
| `tinywasm/gorootgen` | Herramienta de CI que genera el GOROOT precompilado y lo publica como archivos estáticos | falta | F2 |
| `tinywasm/pkgcache` | Descarga, verifica y guarda en OPFS los paquetes precompilados y su manifiesto; invalida por versión | falta | F2 |
| `tinywasm/gobuild` | Sustituto mínimo de `go build` para el sandbox: resuelve imports, orden topológico, configuración de importación, invoca `compile` y `link`; caché por contenido | falta | F3 |
| `tinywasm/modresolve` | Resuelve `go.mod`, `replace` y lista blanca de módulos; fuente desde el VFS o proxy | falta | F3 |
| `tinywasm/diag` | Parsea la salida del compilador a diagnósticos tipados (archivo, línea, mensaje) para mostrarlos al dev | falta | F3 |
| `tinywasm/preview` | Service Worker en Go que sirve el proyecto compilado desde el VFS y recarga al terminar un build | falta | F4 |
| `tinywasm/sandboxui` | Pantalla mínima: árbol de archivos, editor, consola, iframe de vista previa (reutiliza `dom`/`form`) | falta | F4 |
| `tinywasm/isolate` | Detección y habilitación de COOP/COEP (service worker o cabeceras) y avisos si el navegador no cumple | falta | F4 |
| `tinywasm/sandbox` | Fachada pública: `Open(proyecto)`, `Build`, `Preview`, eventos; la única API que consume `app`/`sitec` | falta | F5 |
| `tinywasm/tinygo-remote` | Cliente opcional que delega TinyGo a CI/servidor y cachea el resultado (fase 6) | falta | F6 |
| `tinywasm/c2w` | Respaldo opcional: ejecutar una imagen Alpine vía container2wasm para `git`/`make` | falta | F7 |

## Hoja de ruta

| Fase | Entregable | Criterio de cierre |
|---|---|---|
| **F0** Viabilidad | Compilar `compile` y `link` como `wasip1` y ejecutarlos en navegador y en wazero; compilar un «hola» `js/wasm` con VFS en memoria | Mide: tiempo, pico de RAM, tamaño de descarga. Resuelve D5, D8, D9. Se decide seguir o ajustar |
| **F1** Runtime | `opfs`, `vfs`, `wasi`, `wasihost`, `proc` | Un módulo WASI cualquiera corre con archivos persistentes y salida capturada |
| **F2** Toolchain empaquetado | `goroot`, `gorootgen`, `pkgcache` | Primera carga descarga el GOROOT; segunda carga es offline; el hash invalida correctamente |
| **F3** Build | `gobuild`, `modresolve`, `workerpool`, `diag` | Compilar un módulo `tinywasm/*` completo; segunda compilación incremental en una fracción del tiempo |
| **F4** Vista previa | `preview`, `sandboxui`, `isolate` | Editar → compilar → ver en el iframe, con errores visibles, en Chromium, Firefox y Safari |
| **F5** Integración webtyp | `sandbox` consumido por `app`/`sitec`; el pipeline extract → wasm → route → flush corre dentro del sandbox | Un proyecto webtyp real (p. ej. el de `BUILD_PIPELINE_MASTER_PLAN.md`) se construye en el navegador con la misma salida que el build local, salvo tamaño |
| **F6** TinyGo | Opción A: delegar a CI/servidor con caché. Opción B: investigar clang + wasm-ld como Wasm (precedente: cib) | Decidir A o B con medición. La paridad de tamaño de release no bloquea F0–F5 |
| **F7** Respaldo Linux | `c2w` opcional | Solo si aparece un caso de uso real (git, make). No se hace por adelantado |
| **F8** Endurecimiento | Límites de memoria, cancelación, versionado del GOROOT, actualización sin romper cachés, pruebas en CI con wazero | El sandbox se actualiza de versión de Go sin pérdidas de caché inconsistentes |

Grafo: F0 → F1 → F2 → F3 → F4 → F5; F6 y F7 salen de F5; F8 acompaña cada fase.

## Riesgos principales

1. **Memoria y tiempo del compilador en Wasm** (D9): se mide en F0 antes de comprometer el resto. Si falla en proyectos medianos, la alternativa es dividir en paquetes pequeños o limitar la vista previa a módulos.
2. **Compatibilidad exacta entre compilador y stdlib precompilada** (D7): una versión de Go por release del sandbox, generada por una sola herramienta.
3. **Safari y COOP/COEP**: requiere hosting con cabeceras o un service worker que las inyecte; verificar en F4 con cada navegador.
4. **Paridad con release**: la vista previa usa Go gc; el release usa TinyGo (tamaño y APIs distintas). Se mitiga con las etiquetas de `BUILD_DESTINATIONS_MASTER_PLAN.md` y la prueba de F5.
5. **JavaScript residual** (D5): se limita a un archivo fijo; cualquier crecimiento se considera un defecto del diseño.

## Relación con otras olas

- [BUILD_PIPELINE_MASTER_PLAN.md](BUILD_PIPELINE_MASTER_PLAN.md): el ensamblador único de `sitec` es lo que F5 ejecuta dentro del sandbox; conviene que esa ola cierre antes.
- [BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md): etiquetas `wasm`/`browser` que definen qué se compila para cada destino.
- [NO_REFLECTION_MASTER_PLAN.md](NO_REFLECTION_MASTER_PLAN.md): tamaño del wasm de cliente; relevante para F6.
- [PWA_ARTIFACTS_MASTER_PLAN.md](PWA_ARTIFACTS_MASTER_PLAN.md): artefactos pesados y service worker; reutilizable en `pkgcache` y `preview`.
