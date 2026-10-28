# Administración de acceso reutilizable — master plan
> **Status:** IN PROGRESS · 2026-10-23 · P1–P2 cerradas; R1 cerrada (rbac v0.0.21); A1a y A1b cerradas (access v0.0.3); A2 cerrada (iam v0.0.54); A3 desbloqueada (M2 ejecutada, falta publicarla)

**Disparador:** `mjosefa-cms` necesita lo mismo que ya hace `veltylabs/iam`: crear roles, asignar y quitar
un rol a un usuario, dar de alta a los administradores por correo y ver la auditoría. Esa lógica vivía
**dentro de la app** `iam`, así que la única forma de tenerla en otra app era reescribirla. **Regla del
dueño:** `webtyp` entrega la pieza lista; cada app solo la **configura**.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | `webtyp` entrega la administración de acceso lista; las apps solo la configuran (regla del dueño, 2026-10-23) | A1a, A1b |
| D2 | `auth` y `rbac` siguen sin conocerse; lo que los cruza vive en `webtyp/access`, que habla con ellos por dos interfaces estrechas (`Users`, `Roles`) | A1a |
| D3 | `veltylabs/iam` y `mjosefa-cms` pasan a ser consumidores; borran sus copias en el mismo cambio que adoptan la pieza | A2, A3 |
| D4 (P1) | Repo nuevo **`webtyp/access`** (no `webtyp/iam`, para no chocar con `veltylabs/iam`). Se porta el **comportamiento** de iam al patrón vigente de módulo (`OperationModule` + formatos de `crudview`), no su forma (rutas REST y DOM a mano). Un `Module` = un proyecto de `rbac`; la exigencia de MFA es configuración (`Deps.Assurance`) | A1a |
| D5 (P2) | **Sale a `access`:** roles (guardar, borrar, límite de sesión), asignar y quitar rol por correo, alta de admins por correo (`EnsureRole`), auditoría (`Record`, `list_audit`) y, en A1b, sus pantallas. **Se queda en `iam`:** proyectos y sus claves, `/api/token` (SSO), `resolve_user` y el origen del panel; sus propias acciones de auditoría las escribe con `access.Record` | A1a, A2 |
| D6 | Lo que faltaba en la base se arregla en la base: `rbac` no sabía listar roles (iam leía su tabla directo) → `rbac.ListRoles` | R1 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| R1 | `webtyp/rbac` | `Service.ListRoles(projectID)` | 🟢 cerrada · v0.0.21 publicada |
| A1a | `webtyp/access` | Dominio + 7 operaciones + `migrate` + pruebas con `auth` y `rbac` reales | 🟢 cerrada · v0.0.2 publicada |
| A1b | `webtyp/access` | Pantallas Roles, Asignaciones y Auditoría con `crudview` (`ui/`) + demo `web/browser/main.go` | 🟢 cerrada · v0.0.3 publicada |
| A2 | `veltylabs/iam` | Consume `access`; borra `admin_handlers.go`, `admin_backend.go`, `bootstrap.go`, `audit.go` y lo que corresponda de `panelapi.go` y `modules/panel` | 🟢 cerrada · v0.0.54 publicada |
| A3 | `mjosefa-cms` | Consume `access` (`EnsureRole` con los correos de los admins, pantallas, recursos en `config.Modules()`); cierra P11 de su roadmap | ⚪ desbloqueada: M2 ejecutada en local, se despacha al publicarla |

## Bloqueos

Ninguno.

## Relación con otras olas

- `mjosefa-cms/docs/MVP_DEPLOYMENT_ROADMAP.md`: su P11 (permisos del segundo usuario) se resuelve en A3.
- SECURITY_BY_DEFAULT (cerrada): su diseño archivado vive en `veltylabs/iam/docs/SECURITY_BY_DEFAULT_DESIGN.md` (copiado a `access/_temp/iam/docs/`); `access` hereda sus reglas (operaciones con permiso explícito, MFA configurable).
- DEPLOY_WIZARD: `IAM_ADMIN_EMAILS` pasa a ser la lista que cada app le da a `access.EnsureRole`.
