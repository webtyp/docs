# Pantalla «Importar» — master plan
> **Status:** PENDING · 2026-11-05 · sin fecha: la importación de Pa100T se hace por consola (`migrate/pastdb`); se abre cuando el dueño lo pida

Una pantalla «Importar» dentro de «Base de datos» para correr una importación (`webtyp.com/dataimport`) desde la
interfaz: elegir el respaldo, ensayar, ver el informe y aplicar. Hoy lo mismo se hace por consola
(`veltylabs/mjosefa-cms/migrate/pastdb/README.md`).

## Preguntas abiertas

Ninguna todavía (al abrir la ola: maqueta de la pantalla y dónde vive, ahora que «Base de datos» es solo de
desarrollo — `webtyp/devtools`).

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| I1 | `dataimport`, `devtools`, `layout` | maqueta y pantalla «Importar» | por diseñar |

## Bloqueos

- Ninguno.

## Relación con otras olas

- DATA_IMPORT (`mjosefa-cms`): de ahí sale; aquella importa por consola.
- DB_DIAGRAM: la pantalla «Base de datos» donde iría.
