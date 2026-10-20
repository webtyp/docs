# Unidades de tiempo explícitas — master plan
> **Status:** IN PROGRESS · 2026-10-20 · ola 2a y 2b cerradas; `appointment_booking` en curso (Jules); `misitio` bloqueada por otra ola de `iam`

Por qué y cómo: `webtyp/time/docs/ARCHITECTURE.md` (los dos tipos y por qué `time` importa `calendar`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1–D4 | Relojes `NowNano`/`NowUnix`, unidad en el tipo (`time.Unix`, `time.Nano`, `model.UnixTime/NanoTime`), funciones con tipo, duraciones `time.Minute/Hour/Day` | hecho en `time`, `model`, `ormc`; falta en los consumidores (ola 2) |
| D6 | **Sin migrar datos:** cada campo se tipa con la unidad que ya guarda. Por eso `updated_at` es `time.Unix` en `room_layout`, `payment_manager` e `iam` (guardan segundos) y `time.Nano` en el resto | ola 2 |
| D7 | Un `int64` sobrevive solo en el borde de un formato externo, escrito una vez: `exp` de JWT, `Max-Age` de HTTP, valores de formulario `input.Number()`, `outbox` e `idempotency` (hasta que se tipen) | ola 2 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| T0 | `webtyp/date` → `webtyp/calendar` | renombre | cerrada (`calendar` v0.1.0) |
| T1–T3 | `time`, `model`, `ormc` | tipos, kinds, generación | cerradas (`time` v0.6.2, `model` v0.2.4, `ormc` v0.1.23) |
| T4a | librerías de `webtyp` | `unixid`, `jwt`, `agentcontext`, `pdf`, `stock`, `agent`, `agentmemory`, `auth`, `components`, `layout`, `devtui`, `dbsync`, `money` | cerradas (hechas en local: bloqueaban a los módulos) |
| 2a | 13 módulos | plan por repo | **cerrada**: `site_manager` v0.4.10, `editorial` v0.0.23, `patient_directory` v0.1.32, `service_catalog` v0.5.3, `device_manager` v0.2.54, `supply_manager` v0.2.3, `iam` v0.0.28, `business_calendar` v0.4.86, `payment_manager` v0.1.20, `network_manager` v0.1.55, `pharmacy_manager` v0.1.3, `chat_room` v0.2.76, `room_layout` v0.3.40 (atascado 4h sin PR, terminado en local; corrigió además `isShiftCancelled`, que comparaba `time.Unix` sin convertir a `int64` en la condición del ORM — el mismo patrón que `chat_room` ya usaba) |
| T4b | `webtyp/outbox`, `webtyp/idempotency` | tipar su reloj y sus instantes (`time.Nano`), ruptura de API | cerradas (`outbox` v0.1.6, `idempotency` v0.1.1) |
| T4c | `webtyp/dbsync` | quitar las conversiones `int64(...)` hacia `outbox` e `idempotency` | cerrada (`dbsync` v0.3.6) |
| 2b | `staff_manager` | ídem | cerrada v0.1.119 |
| 2c | `appointment_booking` | ídem (dos relojes duplicados, 15 campos de fecha, constantes `OnlineHoldSeconds`/`PaymentWindowSeconds` tipadas sin renombrar) | `[Antigravity · 2026-10-20 15:27]` en curso (Jules ejecutando etapa 3 y suite) |
| 2d | `clinical_encounter` | ídem | `[Antigravity · 2026-10-20 15:27]` en espera de cierre de 2c |
| 2e | `mjosefa-cms`, `misitio`, `mjosefa-cote`, `alert_manager` (local) | ídem o en local si son 2 archivos o menos | tras todo lo anterior |
| T5 | `devskills` | regla en **module-layout** | cerrada |

## Bloqueos

- `misitio`: su campo `config/authzcache.go` necesita `webtyp.com/time` v0.6.2, que solo trae `github.com/veltylabs/iam` v0.0.28, pero entre v0.0.23 y v0.0.28 `iam` reescribió su API de cliente por otra ola ajena (passkeys, RFC 7523: `client.FetchAuthzToken` desapareció, `ClientSecret` ya no existe). `misitio/routes/admin.go` no compila contra la nueva API. No se improvisa esa adaptación aquí: queda pendiente de que esa ola de `iam` tenga su propio plan hacia `misitio`, o de que el usuario decida. `misitio` quedó revertido, sin cambios.

## Relación con otras olas

- CATALOG_SPLIT: los `nowSeconds()` locales de farmacia e insumos se borran en 2a; el `At` de los hechos de costo pasaba nanosegundos donde `money` espera segundos (error real, lo corrige el plan de 2a).
