# Subida de archivos (D46) — diseño de implementación y preguntas

> **Estado:** DECIDIDO · plan 1 (`router`) despachado 2026-11-10 · P1 = A (`Context.RawQuery`), P2 = A (un archivo por tabla), P3 = A (la descarga usa `Read`). Siguiente: un plan por repo, en etapas, en el orden de la tabla. Master: `ARCHITECTURE_MASTER_PLAN.md`, fila A6.12.

## Lo que encontré en el código (hoy)

| Pieza | Estado | Dónde |
|---|---|---|
| Detección por bytes y lista permitida | existe: `filetype.Detect`, `filetype.Allowlist.Validate`, `filetype.Images` (SVG y HTML detectados para rechazarlos) | `filetype/filetype.go` |
| Subida de un archivo crudo con límite por `Content-Length` | existe pero **aparte de `crud`**: `cloudflare/files.Store.Mount` (`PUT prefix`, clave minteada por el servidor, 201 + clave) | `cloudflare/files/files.go` |
| Contrato de almacén de archivos | existe: `files.Store` = `ReadFile`, `WriteFile`, `AppendFile`, `RemoveFile`; implementaciones `mem`, `disk`, `opfs` | `files/files.go` |
| Un campo que valida bytes | **no existe**: `model.Kind.Validate(value string)` valida texto | `model/kind.go` |
| Leer el query de la petición | **existe a medias**: `router.QueryOf(path)` y `router.QueryReader(query)` ya leen un query como un registro, y `Decode` los usa; pero `Context.Path()` **no significa lo mismo en todos los transportes**: en `httpd` y en `router/mock` incluye el `?query`, en `cloudflare/edge` es solo la ruta. Un manejador no puede leer el query igual en todos | `router/query_codec.go`, `server/httpd/adapter.go:42`, `cloudflare/edge/edge.go:78` |
| Un `Create` de `crud` con cuerpo que no es JSON | **no existe**: `stored_entity_create.go` decodifica el cuerpo con `Decode` a la lista de registros | `crud/stored_entity_create.go` |

## Diseño (cada pieza en el repo que ya es su dueño)

```go
// router — contract change (P1 = A, corregido): Path() is the path only, in every transport; the query has its own method.
type Context interface {
	// … lo de hoy …
	RawQuery() string // "a=b&c=d" without the "?", "" when none; read it with router.QueryReader(ctx.RawQuery())
}

// input — a field kind for a file. The record keeps only the KEY of the file (a text column).
// Example: logo := input.File(filetype.Images, 2<<20)
func File(allowed filetype.Allowlist, maxBytes int) Input // implements FileKind
type FileKind interface {
	model.Kind                                   // Validate(key string): the stored key is well-formed
	ValidateFile(data []byte) (filetype.Type, error) // the same code in the browser (before sending) and in crud (on arrival)
	MaxBytes() int
}

// model — a table declares at most ONE file field; the others are metadata.
// In definition.go:  {Name: "logo", Type: input.File(filetype.Images, 2<<20)}
// definitionc writes the columns <name>, <name>_type, <name>_size (type and size are ServerOnly: they come from the bytes).

// crud — StoredEntity gains the destination; the create handler gets a second path.
type StoredEntity struct {
	// … lo de hoy …
	Files files.Store // nil unless the entity has a file field; MountOperations refuses a file entity without it
}
// Create of a file entity: POST /api/<module>/<entity>?<field>=<value>…
//   0. the record's fields: `rec.DecodeFields(router.QueryReader(ctx.RawQuery()))` — the reader that already exists.
//   1. Content-Length > MaxBytes → 413 before reading the body.
//   2. data := ctx.Body(); ValidateFile(data) → 415 with the reason; never the client's Content-Type.
//   3. the rule runs as for any Create over the record read in step 0;
//   4. key := new id + type.Ext; Files.WriteFile(key, data); the record gets key, type and size;
//   5. the transaction fails → Files.RemoveFile(key) (no orphan); the commit succeeds → the response is the record.
// Reading the file: a read model of the entity answers the key; the bytes are served by the route below.
```

Descarga: `GET /api/<module>/<entity>/file?id=…` (la misma comprobación de permiso `Read` de la entidad; `Content-Type` = el guardado, `X-Content-Type-Options: nosniff`, `Content-Disposition: inline`). Una entidad pública con archivo (el logo de un sitio) la abre la app en `AppServer.Public` con `Read`.

## Prueba de conformidad (una sola forma para todo el framework)

`router/conformance` ya es el contrato ejecutable de `router.Router` y lo pasan `server/httpd` y `cloudflare/edge`. Ahí es donde se escribió la regla que hoy falta: **`Path()` es solo la ruta y `RawQuery()` es el query, igual en todos**. Dos cambios:

1. Cláusulas nuevas en `conformance.Run` (se sirve una petición `POST /x?a=1&b=dos` por el transporte real): `Path()` = `/x`; `RawQuery()` = `a=1&b=dos`; sin query → `""`; `router.QueryReader(RawQuery())` lee los campos igual; el cuerpo crudo llega intacto con su `Content-Length`.
2. Una suite hermana `conformance.RunContext(t, new func(method, target string, body []byte) router.Context)` para los contextos que se construyen **desde una dirección** y no pasan por un `Router` (hoy `router/mock`). Los de `crud` en `loopback` y `mcp/harvest.go` no tienen URL (se llama una operación por nombre): `Path()` y `RawQuery()` responden vacío y cada uno lo prueba con una prueba propia, no con `RunContext`. Un transporte nuevo que no la pase no entra al framework.

Este plan va primero (plan 1 de abajo) y los otros repos lo llaman al migrar su transporte.

## Orden de los planes (de abajo hacia arriba; cada uno se publica antes del siguiente)

| # | Repo | Qué | Depende de |
|---|---|---|---|
| 1 | `router` + los 4 transportes (`server/httpd`, `cloudflare/edge`, `router/mock`, el loopback de `crud`) | `Context.Query` | — |
| 2 | `input` | `input.File`, `FileKind` | `filetype` |
| 3 | `model`, `definitionc`, `definitioncheck` | campo de archivo: columnas `<name>_type`/`_size`, un solo archivo por tabla, SVG/HTML rechazados al declarar | 2 |
| 4 | `crud` | camino de `Create` de archivo, ruta de descarga, limpieza | 1, 2, 3 |
| 5 | `appkit` | entrega `files.Store` a cada entidad con archivo, desde `config/` | 4 |
| 6 | `cloudflare/r2` | adaptador R2 al contrato `files.Store` (hoy `r2.Bucket` es `Put`/`Get`) | — |
| 7 | `view` | widget de subida (valida con `ValidateFile`, comprime con `image/browser`, envía el archivo crudo) | 2, 4 |
| 8 | `misitio` | migra a `crud`: desaparecen `UploadAsset` y `files.Store.Mount`; logos y favicon como PNG/JPG/WebP | 4–7 |

## Preguntas para ti

**P1 (decidido: A; corregida el 2026-11-10) — el query no se lee igual en todos los transportes.** Al armar el plan vi que `router` ya sabe leer un query (`QueryReader`), pero `Context.Path()` lo incluye en `httpd` y no en `edge`: el mismo método significa dos cosas, y `cloudflare/files` ya depende de que sea solo la ruta (`ctx.Path()[len(prefix):]`, que con `?x=1` en `httpd` daría una clave equivocada). Corrección: `Path()` es solo la ruta en los 4 transportes, y se agrega `RawQuery() string` (nombre de Go: `URL.RawQuery`). Un plan pequeño por transporte; no se inventa un lector nuevo. Mi propuesta anterior, `Query(name)`, era innecesaria: el lector por registro ya existe.

**P2 (decidido: A) — ¿Un archivo por tabla o varios?** A) uno solo (la petición lleva un archivo crudo; una ficha con 2 archivos son 2 tablas). B) varios, cada uno en su petición contra el mismo registro. **Recomiendo A**: es lo que cabe en «un archivo crudo por petición» sin inventar un protocolo de varias partes; los casos conocidos (logo, favicon, foto de ficha) son una tabla cada uno. Respuesta:

**P3 (decidido: A) — ¿Quién puede descargar?** La descarga usa el permiso `Read` de la entidad. Para el logo del sitio (público) la app lo abre con `Public`. A) así, sin otra regla. B) además una regla `Download` por entidad (por ejemplo «solo el dueño de la ficha»). **Recomiendo A**: la regla `Search` de la entidad ya filtra qué filas ve cada persona y la descarga la reutiliza. Respuesta:

## Hallazgos al preparar el plan de `definitionc` y `crud` (2026-11-11) — dos preguntas nuevas

**Estado:** `router` v0.5.0, los 4 transportes, `input.File` (`input` v0.1.0) y `r2.Files` están publicados o en cierre. Lo que sigue (`model`, `definitionc`, `crud`) choca con dos hechos del código:

1. `definitionc` copia la expresión de `Type:` tal cual al código generado y la **sondea compilándola** (`probe.go`): solo califica el paquete del constructor (`input`), no el de los argumentos. `input.File(filetype.Images, 2<<20)` fallaría dos veces: el sondeo no importa `filetype`, y `2<<20` (una expresión binaria) no se copia. Arreglarlo es cambiar el generador.
2. `crud` no puede importar `input` (las capas van `model` ← `input`). Para saber que un campo es un archivo, `crud` necesita una interfaz en `model`.

**P4 — ¿cómo se declara un campo de archivo?** A) constructores con nombre y solo un número, sin tocar `definitionc`: `Type: input.ImageFile(2097152)` (solo imágenes) e `input.DocumentFile(2097152)` (imágenes y PDF); `input.File(allow, max)` queda para código y pruebas. B) cambiar `definitionc` para aceptar `input.File(filetype.Images, 2<<20)` (el sondeo importa los paquetes de los argumentos y se copian expresiones binarias). **Recomiendo A**: es lo que lee un junior sin saber de `filetype`, no cambia el generador y cubre logo, favicon y foto de ficha (todas imágenes); un caso nuevo agrega un constructor con nombre. Respuesta:

**P5 — ¿dónde vive la interfaz que `crud` reconoce?** `model.FileKind { Kind; ValidateFile(data []byte) (filetype.Type, error); MaxBytes() int }` en `model` (que pasa a importar `filetype`, isomórfico y sin dependencias) y `input.FileKind` la incluye. La alternativa es una interfaz local en `crud` con tipos simples (`ValidateFile(data []byte) (mime string, ext string, err error)`), sin tocar `model`. **Recomiendo la de `crud`** (`crud` declara lo que necesita con tipos simples; `input.File` ya la cumple si su método devuelve texto en vez de `filetype.Type`, un cambio pequeño en `input` v0.1.1). Respuesta:

### Respuestas (2026-11-11)

**P4 = A**, ampliada: constructores con nombre y un solo número, uno por familia de activos de un negocio, tipados por los bytes: `input.ImageFile(max)`, `input.DocumentFile(max)` (imágenes y PDF), `input.VideoFile(max)`, `input.AudioFile(max)`. Un tipo nuevo (por ejemplo, otro formato de video) es una firma más en `filetype` y, si hace falta, un constructor más en `input`: se publican esas dos librerías y se recompilan solo las apps que usan el tipo nuevo. **Límite de tamaño:** `Context.Body()` entrega el cuerpo entero en memoria (`router.go`; `httpd` hace `io.ReadAll`), así que un video grande no cabe en un Worker (unos 128 MB de memoria) ni conviene en un servidor chico: `VideoFile` sirve para videos cortos (el plan fija un máximo prudente y lo documenta); los largos necesitan otro camino (subida directa al almacén con una dirección firmada), que es un diseño aparte y queda fuera de D46.
**P5 = interfaz en `crud`**, y **no** en `model`: `model` no depende de `filetype` (tú lo dijiste: la dependencia va al revés). `crud` ya maneja bytes y almacén, así que declara lo que necesita, `type fileField interface { ValidateFile(data []byte) (filetype.Type, error); MaxBytes() int }`, y `input.FileKind` la cumple sin que `crud` importe `input`. No cambia `model` ni `input` v0.1.0.

Planes: `filetype` v0.1.0 con video, audio y las listas `Videos`, `Audio`, `Documents` (despachado); después `input` v0.2.0 con los cuatro constructores; luego `crud` (camino de archivo), `appkit`, `view` y `misitio`. `definitionc` no cambia.
