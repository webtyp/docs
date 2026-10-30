# Self-hosted deployment runs on Docker — design

## What this is

Some webtyp apps do not run in the cloud: they run on a server the customer owns — a small server in an office, or a
developer's own PC while the app is being tried out. This document explains why **webtyp deploys every self-hosted app
in exactly one way: Docker Compose, with one container image per published version**, and why the earlier plan of
installing each piece by hand on the server was dropped.

**When you will run into it:** you publish a version of an app that runs on a server you own, you install that app on
a new server for the first time, or you wonder why the app repo has no `Dockerfile`, no `compose.yaml` and no `systemd`
unit.

**Scope:** apps that run on a server you own. Apps deployed to Cloudflare Workers keep their own destination
(`goflare`); nothing here changes them.

Vocabulary used below:

- **Container image** — a sealed package holding the app binary plus the few system files it needs (root certificates,
  time zones). It is identified by a **digest** (`sha256:…`), a fingerprint of its exact bytes; a **tag** (`:v1.4.0`)
  is a movable label that points to a digest.
- **Docker Compose** — a file (`compose.yaml`) that lists the containers of one app and how they connect; `docker compose
  up -d` starts all of them and Docker restarts them when they stop or the machine reboots.
- **Cloudflare Tunnel** — an outgoing connection from the server to Cloudflare, so visitors from the internet reach the
  app without opening any port on the office router. The program that holds it is `cloudflared`.

## Why: the manual path did not scale

The first design installed every piece directly on the server. Writing it down for one app and one server produced
this list, every item a command or a file someone types on **each** server:

| Piece | What the manual path needed |
|---|---|
| Tunnel | add Cloudflare's apt repository and its signing key, install `cloudflared`, store its token in a root-only file with `sudoedit`, write a `systemd` unit by hand (the vendor's one-liner leaves the token in the shell history and in a world-readable unit file) |
| Database | install PostgreSQL from the distribution, move it to another port when the developer PC already ran one, create the database and two roles, schedule `pg_dump` |
| App | a `systemd` unit per app, a capability so a non-root process can listen on 443, the working directory set to the version folder (the app resolves `web/public` relative to it) |
| Updates | download the new binary, swap it while it may be running (Windows refuses to replace a running `.exe`), a watchdog that puts the old one back if `/health` fails, one binary per operating system |
| Network | decide which address each door listens on, and a firewall so the internet-only port is never reachable from the LAN |

Each line is a place where the developer's trial machine and the production server can differ, and every difference
is a bug found only in production. The list also grows with every app and every server.

## The decision

1. **One `compose.yaml` per app, the same on every machine**, with four services: `db` (PostgreSQL, data in a named
   volume, no port on the host), `app` (the app image), `cloudflared` (the official image, pinned version) and the
   update supervisor. Docker restarts them; there are no `systemd` units.
2. **One image per published version.** The developer builds it on their machine and pushes it to the GitHub container
   registry, together with a **signed manifest** naming the version, the image and its digest. The server pulls
   `image@digest` only after the signature checks out against the public key compiled into the app.
3. **A supervisor is the only container allowed to talk to Docker.** The app decides *when* to update (no user
   connected, or the administrator presses the button) and asks the supervisor over the Compose internal network; the
   supervisor pulls, recreates the `app` service and returns to the previous image if `/health` fails. The app itself
   never touches Docker: access to the Docker socket equals root on the machine.
4. **The app repo never contains a `compose.yaml` or `Dockerfile`.** The app declares what it needs once
   (`config/deploy.go`: name, domain, database, secrets) and the tooling generates the rest. Secrets are created on the
   server or typed once there; none of them passes through the repo or a chat.
5. **The internet door is isolated by the network, not by configuration.** It exists only on the Compose internal
   network, where only `cloudflared` reaches it; the LAN door is published on the host's port 443.

## Prior art

- **The image is the version.** [Sigstore cosign](https://github.com/sigstore/cosign/blob/main/specs/SIGNATURE_SPEC.md)
  signs images by digest, never by tag, because a tag can be moved after signing. webtyp signs a small manifest that
  carries the digest, with the signing library it already has (`webtyp/crypto/asym`).
- **Self-updating binaries are turned off inside containers.** Syncthing updates itself when it runs as a plain binary,
  but its official Docker image disables that and expects a new image instead (unverified: the Dockerfile was not
  re-read for this document).
- **Generic auto-updaters update blindly.** Watchtower, the usual tool that pulls new images on a timer, was archived in
  December 2025 ([report](https://linuxiac.com/docker-update-tool-watchtower-reaches-end-of-maintenance/)), and it
  cannot know that someone is in the middle of their work. Home Assistant solves that with a supervisor the UI asks to
  update (unverified: no official page read for this document). webtyp follows the supervisor pattern.
- **Release metadata is verified before the artifact is fetched.** [The Update Framework](https://theupdateframework.github.io/specification/)
  verifies signed metadata naming the target by hash before downloading the target; webtyp does the same with one
  publisher and one manifest.

## What it costs

- **Docker must be installed on the server.** It is the only requirement: one package, the same on every Linux server.
- **One platform.** Images are built for `linux/amd64`. A Windows server is no longer a deployment target; a machine that
  must keep running Windows software runs it in a virtual machine, outside webtyp's deployment.
- **One extra component**, the supervisor, written once in webtyp and shared by every self-hosted app.
- **The registry is GitHub's.** Private images are free today; GitHub promises one month of notice before charging
  ([billing](https://docs.github.com/en/billing/concepts/product-billing/github-packages)).
- **Docker manages the host firewall rules for its containers.** A machine that also bridges virtual machines must
  allow that bridge in the `DOCKER-USER` chain, or Docker's `FORWARD` rule drops the VM's traffic.

## What it deleted

- Hand-written `systemd` units, apt repositories and token files per server.
- One binary per operating system and the in-place binary swap (with its Windows exception and working-directory rule).
- Firewall and listen-address configuration for the internet door.
- The temptation to keep a `compose.yaml` or `Dockerfile` in each app repo.

STATUS (remove this line when SELFHOST_DEPLOY_MASTER_PLAN closes): the signed manifest is being built in `webtyp/update`;
the image build in `gorelease`, the first install on a server and the supervisor (`webtyp/selfupdate`) are still being
designed.
