# Pruebas del dominio — master plan
> **Status:** PENDING · 2026-11-05 · esperan P1–P5 (nomenclatura para la skill **testing**); la limpieza de repos va al final (mantenibilidad)

Cada agente agrega pruebas sin criterio de dominio: `mjosefa-cms/tests/` tiene 34 pruebas en 20 archivos y la
mayoría prueba el framework, no la clínica (`TestMCPClosedByDefault`, `TestDefaultDeny_NoAuthorizeConfigured`,
`TestHealth`, `TestPWAConfig`, `TestProjectLayout`, `TestIntrospectionListsRPCRoutes`, `TestSplitIPs`…). Una prueba
que sería idéntica en cualquier otra app pertenece a la librería que dueña ese comportamiento (responsabilidad única).
Meta: abrir un repo y saber, en un documento, qué reglas de su negocio están protegidas y dónde.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | `docs/TESTS.md` por repo: una sección por área, diagrama si hay flujo, filas «regla → enlace a la prueba» | S3, S4 |
| D2 (P2) | Nombre `Test<Área>_<Regla>` | S3, S4 |
| D3 (P3) | Un archivo por área: `tests/<área>_test.go`; ayudantes en `tests/setup_test.go` | S3, S4 |
| D4 (P4) | Una app prueba solo su negocio + humo de arranque; lo del framework va a la librería dueña | S3, S4 |
| D5 (P5) | `gotest` falla si una prueba no está en `docs/TESTS.md` o una fila apunta a una prueba inexistente | S2 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S1 | `devskills` (**testing**, **plan-authoring**, **plan-review**) | reglas D1–D5 (sección «Domain tests») | 🟢 hecho 2026-11-05 |
| S2 | `devflow` (`gotest`) | control de D5 (sin `docs/TESTS.md` → solo aviso; con él → falla) | 🟢 hecho 2026-11-05 (`v0.4.165`) |
| S3 | `mjosefa-cms` | `docs/TESTS.md`; mover las pruebas del framework a su librería; renombrar | espera: cierre de DATA_IMPORT (repo con cambios sin commit y `replace` locales) |
| S4 | librerías y módulos | lo mismo, repo por repo, al tocarse | al final |

## Bloqueos

- S3: `mjosefa-cms` tiene trabajo sin commit (DATA_IMPORT) y dependencias con `replace` local; no se despacha hasta que quede limpio.

## Relación con otras olas

- DATA_IMPORT (`mjosefa-cms`): sus pruebas nuevas (`pastdb`) ya siguen la regla cuando S1 cierre.
