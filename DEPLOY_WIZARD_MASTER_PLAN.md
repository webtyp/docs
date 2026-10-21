# Despliegue guiado — master plan
> **Status:** CLOSED · 2026-10-21 · todas las librerías y `app` v0.7.0 publicados, comprobados con una ejecución real de `deployment()` sobre un proyecto de prueba. Solo falta la **aceptación con cuenta real de Cloudflare**, que prueba el usuario en local.

Diseño, decisiones y evidencia (histórico): [deploy/docs/DEPLOY_WIZARD_DESIGN.md](https://github.com/webtyp/deploy/blob/main/docs/DEPLOY_WIZARD_DESIGN.md).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

Ninguna: todas publicadas (ver el diseño archivado).

## Fases

Ninguna pendiente de despacho. `misitio` (re-layout y declaración de despliegue) sigue en
[BUILD_DESTINATIONS_MASTER_PLAN.md](BUILD_DESTINATIONS_MASTER_PLAN.md), no en esta ola.

## Aceptación — pendiente, la hace el usuario en local

Dos máquinas, dos personas. En cada una, solo `gh auth login` hecho y su propio token de arranque
de Cloudflare en su keyring:

1. Un agente llama `deploy_status` sobre misitio y recibe la lista de lo que falta, sin valores.
2. Con `deploy_guide` guía al desarrollador, que escribe cada valor solo en el wizard de la TUI.
3. `deploy_provision` y `deploy_release` dejan el Worker respondiendo en `misitio.velty.cl`.
4. La segunda persona repite 1–3 sin recibir nada de la primera; la primera vuelve a desplegar
   después sin errores.
5. Ningún secreto en `.env`, en el repo, en el keyring de nadie (salvo tokens de arranque y
   secretos de desarrollo) ni en el chat.

**Si falla:** anotar el paso y el mensaje exacto aquí y abrir un plan nuevo en `app` con la firma
exacta de cada API que toque.

## Bloqueos

Ninguno (depende solo de que el usuario tenga una cuenta de Cloudflare a mano para probar).

## Relación con otras olas

- BUILD_DESTINATIONS: `misitio` termina su re-layout (M1–M8) y su declaración de despliegue ahí
  antes de poder correr esta aceptación.
- SECURITY_BY_DEFAULT (cerrada, archivada): este wizard entregó `IAM_CLIENT_PRIVATE_KEY` al Worker.
