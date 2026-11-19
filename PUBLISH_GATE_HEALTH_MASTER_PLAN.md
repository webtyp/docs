# Gate de publicación en rojo — master plan
> **Status:** IN PROGRESS · 2026-11-19 — 7 de 10 publicados; los 3 restantes (`iam`, `misitio`, `sitetheme`) quedan bloqueados por `site_content`/`site_manager` aún en `router v0.4.16` (incompatible con `layout v0.3.134` → `router v0.5.7`) y cambios sin guardar en `iam`.

## Preguntas abiertas
Ninguna.

## Cerrado (publicado, sin pendientes)
- `webtyp`: `access v0.0.72` · `agenteval v0.4.107` · `avatar v0.2.16` · `pwa v0.2.24` · `goflare-demo v0.0.221`
- `veltylabs`: `appointment_booking v0.3.103` · `payment_manager v0.4.63`

## Fases
Una fila por repo pendiente; carpeta entre paréntesis.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| G3 | `iam` (`veltylabs/iam`) | Bloqueado por cambios locales sin guardar (7 archivos en `config/`, `modules/panel/`, `tests/`) y por `IAM_APPKIT_MASTER_PLAN` / `ARCHITECTURE_MASTER_PLAN`. Al cerrar su trabajo, subir `orm`, `storage`, `auth`, `form`, `server`, `layout`. | ⏸ bloqueado |
| G4 | `misitio` (`veltylabs/misitio`) | Importa `iam`, `site_content`, `site_manager` y `sitetheme`; bloqueado hasta que esos cuatro migren a `router v0.5`. | ⏸ bloqueado |
| G5 | `sitetheme` (`veltylabs/sitetheme`) | `layout v0.3.134` exige `router v0.5.7`, pero `site_content` (último tag `v0.2.57`) aún usa `router.OperationModule` (`router v0.4.16`). Bloqueado hasta migrar `site_content` a `router v0.5`. | ⏸ bloqueado |

## Bloqueos
- 2026-11-19 · `veltylabs/site_content` (v0.2.57) y `veltylabs/site_manager` aún usan `router v0.4.16` (`router.OperationModule`, `reg.Operation("get", ...)`), que no compila con `router v0.5.7` (requerido por `layout v0.3.134`). Bloquea `sitetheme` (G5) y `misitio` (G4).
- 2026-11-19 · `veltylabs/iam` tiene 7 archivos modificados sin commitear y está bloqueado por `ARCHITECTURE_MASTER_PLAN` / `IAM_APPKIT_MASTER_PLAN`. Bloquea G3 y G4.
- 2026-11-19 · El cascade original saltó otros repos por cambios sin guardar o sesión de `codejob` activa (`mjosefa-cms`, `app`, `appkit`, `auth/tests`, `dbsync`, `clinical_encounter`). No son errores de gate.

## Relación con otras olas
- `IAM_APPKIT_MASTER_PLAN` y `ARCHITECTURE_MASTER_PLAN`: desbloquean `iam` (G3) y la migración de módulos viejos (`site_content`, `site_manager`) a `router v0.5` / `crud`.
- `CODE_GUARD_MASTER_PLAN` y `STDLIB_MIGRATION_MASTER_PLAN`: cubren los demás repos con `style ❌` de corridas anteriores.
