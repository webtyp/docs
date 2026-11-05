# Migración a la stdlib medida — master plan
> **Status:** PENDING · 2026-11-05 · sin preguntas; listo para que otro agente ejecute S1 en local (nada se publica sin revisión del dueño)

Viene de CODE_GUARD (decisiones P9, P11, P12 de allí). Las mediciones con TinyGo que justifican todo están en
`webtyp/gocheck/docs/STDLIB_WEIGHT.md` (se reproducen con `gocheck/benchmark/weight/measure.sh`). Las reglas que luego
lo vigilan (`nostdlib`, `ifaceeq`, `nomaps`) son de CODE_GUARD; aquí solo se cambia el código.
Ejecución **local** (skill **master-plan** → «Local execution»): cada consumidor con `replace` a la copia local, `gotest`
verde en cada repo, nada de `gopush` ni tags hasta que el dueño revise; luego publicar de abajo hacia arriba.

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | `webtyp/fmt` borra `Contains`, `HasPrefix`, `HasSuffix`, `Index`, `LastIndex`, `Split`, `TrimSpace`, `TrimPrefix`, `TrimSuffix`, `ToLower`, `ToUpper`, `Repeat`, `Count`, `ReplaceAll`, `JoinSlice` y usa `strings` por dentro. `fmt.Err` se queda (mensajes traducibles con langc). Medido: la app que usa texto ahorra 4,8 KB; `fmt` solo no baja (TinyGo ya descarta lo no usado) | `fmt` |
| D2 | `fmt.ErrType` crea errores con **tipo** propio, revisables con `errors.AsType`, no centinelas para `errors.Is` (comparar dos interfaces suma +27 KB en TinyGo) | `fmt` |
| D3 | Los centinelas declarados en webtyp/veltylabs (`keyring.ErrNotFound`, `selfupdate.ErrUnhealthy`, `deploy.ErrNoPlan`, `ormc.ErrNoModelsFound`, `appkit.ErrNoProvider`…) pasan a errores con tipo. Los de la stdlib o externos (`os.ErrNotExist`, `io.EOF`, `pgx.ErrNoRows`) se siguen comparando con `errors.Is` | cada librería con centinelas |
| D4 | Migrar los ~360 usos de las funciones de D1 a `strings` y los `errors.Is`/`== ErrX` de D3 a `errors.AsType`, todo de una vez | todos los repos de webtyp y veltylabs |
| D5 | No se crea `webtyp/errors`: `errors` de la stdlib se usa tal cual (`New` y `AsType` pesan 0) | — |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S1 | `fmt` | D1 + D2; volver a correr `measure.sh` y anotar el peso antes/después en `STDLIB_WEIGHT.md` | pendiente |
| S2 | librerías con centinelas | D3, de abajo hacia arriba en el grafo de dependencias | espera S1 |
| S3 | todos | D4: `grep` de cada función de D1 y de cada centinela de D3; `gotest` verde en cada repo | espera S2 |
| S4 | `mjosefa-cms` | Correr la app de referencia y sus tests con todo lo migrado | espera S3 |
| S5 | todos | Revisión del dueño; publicar de abajo hacia arriba, quitar `replace` | espera S4 |

## Bloqueos
- Ninguno.

## Relación con otras olas
- CODE_GUARD: sus reglas `nostdlib`/`ifaceeq`/`nomaps` marcarán en `gotest` lo que esta ola no haya migrado.
