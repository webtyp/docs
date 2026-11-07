# docs

- [ORIGIN.md](ORIGIN.md) — where webtyp comes from: a clinic, a gastroscopy screen that JavaScript made painful, and the first Go-in-WebAssembly experiment.
- [UI_CONSISTENCY_DESIGN.md](UI_CONSISTENCY_DESIGN.md) — the form, title and colour rules every module screen follows, with the reasoning and prior art.
- [SELFHOST_DOCKER_DESIGN.md](SELFHOST_DOCKER_DESIGN.md) — why every self-hosted webtyp app is deployed with Docker Compose and one signed image per version, and what the manual path cost.
- [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md) — end-to-end transport, validation, and RBAC flow (`form`/`view` → `rpc`/`mcp`/`loopback` → `httpd`/`security` → `crud.Mount*` vs raw `reg.Operation` → `module` → `orm`).
