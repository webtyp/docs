# Pestaña DEPLOY para cualquier destino — master plan
> **Status:** IN PROGRESS · 2026-10-31 · P1–P4 consolidadas; preparando despacho de fases W1–W4

**Disparador:** ejecutar `webtyp dev` en cualquier proyecto debe mostrar en la pestaña DEPLOY qué falta para desplegar, sea un Worker o un servidor propio (`mjosefa-cms` como caso de referencia).

**Lo que ya existe y se reutiliza:**
- Contrato `deploy.Plan` (`Check`, `Provision`, `Release`) de DEPLOY_WIZARD.
- Regla de carpetas de `web/` de BUILD_DESTINATIONS (`web/edge/` es Worker; `web/server/` es Servidor propio).
- `selfupdate install` y `gorelease` (`ReleaseApp`, `EnsureSigningKey`, `PublicKeyText`).

**Objetivo:** cada proyecto ve la misma checklist guiada. Sin `config/deploy.go`, el wizard pide el dominio y lo genera automáticamente. Con `Target: selfupdate.Server`, el wizard guía con enlaces directos para crear los tokens y `gorelease` publica la versión al hacer click en Desplegar.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **`app` elige destino y CI según la declaración** (`goflare.Worker` o `selfupdate.Server`; `github.Actions` o `gorelease.Local`). El compilador de Worker se registra solo si existe `web/edge/main.go` (corrige `app/section-deploy.go:41`) | W4 |
| D2 | **`selfupdate` implementa `deploy.Target`** (`selfupdate.NewTarget`): valida nombre y dominio con `ComposeFile`, verifica salud con `/health` (D6), expone `Runtime` para secretos y no exige CI remoto | W2 |
| D3 | **`gorelease` implementa `deploy.CI`** (`gorelease.NewCI`): valida Docker, asegura la llave de firma en el keyring y empaqueta la versión actual con `ReleaseApp` | W3 |
| D4 | **La ola cierra en `mjosefa-cms`**: `webtyp dev` → pestaña DEPLOY → genera `config/deploy.go`, guía con enlaces para tokens y verifica checklist en captura | W5 |
| D5 (P1) | **El wizard genera `config/deploy.go` si falta**: deduce el destino según `web/` (`web/edge` → Worker, `web/server` → Server), pregunta el dominio y escribe el archivo con `deploy.Write(rootDir, Plan)`. Si no hay binario, avisa que el repo no se despliega | W1, W4 |
| D6 (P2) | **Detección de instalación por salud**: `selfupdate.Target.Check` sondea `GET https://<Domain>/health`. Si responde `200 OK`, el servidor ya está instalado; si no, muestra el ítem con los pasos | W2 |
| D7 (P3) | **Guía con enlaces directos y entrada en wizard**: para cada token (`GITHUB_TOKEN`, `TUNNEL_TOKEN`, `DNS_TOKEN`), la checklist muestra la URL exacta para crearlo. El valor se tipea en el wizard de la TUI (`FixInput`) y se entrega a `Runtime` | W2, W4 |
| D8 (P4) | **Despliegue del tag de HEAD**: `gorelease.CI` toma el tag existente en HEAD (creado por `gopush`). Si el árbol está sucio o no tiene tag, la checklist exige correr `gopush` primero | W3 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| W0 | docs | Preguntas P1–P4 consolidadas | cerrada 2026-10-31 |
| W1 | `deploy` | `deploy.Write(rootDir, Plan)` (inverso de `Scan`, con tests de ida y vuelta) | cerrada v0.2.115 |
| W2 | `selfupdate` | `selfupdate.NewTarget(cfg)` implementando `deploy.Target` (D2, D6, D7) | cerrada v0.0.11 |
| W3 | `gorelease` | `gorelease.NewCI(cfg)` implementando `deploy.CI` (D3, D8) | cerrada v0.1.13 |
| W4 | `app` | Tabla de targets/CI en `deploy_tools.go`, deducción por `web/` y wizard de `config/deploy.go` (D1, D5, D7) | cerrada v0.7.46 |
| W5 | `mjosefa-cms` | Prueba real en `webtyp dev`, generación de `config/deploy.go` y captura de DEPLOY | en curso |
| W6 | `devskills` | Actualizar skill **secrets** y **project-layout** | cerrada v0.0.33 |

## Bloqueos

- Ninguno.

## Relación con otras olas

- SELFHOST_DEPLOY: reabre S3 con código en `selfupdate` y `gorelease`.
- DEPLOY_WIZARD (cerrada): amplía la tabla de targets/CI en `app` sin alterar el contrato.
- BUILD_DESTINATIONS: `web/edge/` vs `web/server/` decide el destino predeterminado.
- MVP_DEPLOYMENT_ROADMAP (`mjosefa-cms`): consume esta ola para el despliegue del MVP.
