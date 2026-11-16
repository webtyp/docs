# RECORD_SEARCH phase 2 — `text_search` column, default search, `ListsOnlyAfterSearch`

> STATUS (remove this folder when RECORD_SEARCH_MASTER_PLAN closes): executed **locally** by Sonnet, step by step,
> from [EXECUTION_ORDER_MASTER_PLAN.md](../EXECUTION_ORDER_MASTER_PLAN.md) steps 10a–10j. Nothing is published
> until gate 🔵 13. The owner reviews this plan (🔵 on step 10) before 10a starts.

Spec: [RECORD_SEARCH_MASTER_PLAN.md](../RECORD_SEARCH_MASTER_PLAN.md) D1–D9. `fmt.Conv.ToSearchText()` (D9) is done
locally in `webtyp/fmt/search_text.go` (step 8). Every stage below names its repo, files, exact code and tests.
The code of stages 10a, 10c, 10d, 10f and 10h was **compiled and its tests run** in a scratch copy on 2026-11-16
(`gotest`-equivalent green); copy it as written.

## How every stage is run (read once)

1. Skills: **testing**, **zero-debt**, **bug-fix-flow**, **isomorphic-libraries**, **wasm**, plus the ones the stage names.
2. Each repo points at the local copies with `replace` (never publish): `go mod edit -replace webtyp.com/<dep>=../<dep>`
   for every dep the stage lists, then `GOFLAGS=-mod=mod go mod tidy`. Never hand-write `go.sum`/`go.mod` lines.
3. New test files go in `tests/`; every new test gets its row in that repo's `docs/TESTS.md`
   (`| [Test<Area>_<Rule>](../tests/<file>_test.go) | rule |`) — `gotest` checks the index.
4. Run `gotest` in every repo the stage touches. Green = done. A test you cannot make green → stop, `⚠` in the master.
5. Do not reformat lines you did not change (a `gofmt` pass on a whole file is fine only if `gotest` demands it).
6. Never edit a generated file (`*_gen.go`) by hand: regenerate with `definitionc`.

## What was verified in the code (the answers step 9 owed)

- **Where every write passes: `orm.DB`.** `crud` writes through `crud.Tx` = `orm.Store` (`orm/store.go:231-295`), which
  calls `orm.DB.Create/UpdateFields/updateFields`. The Pa100T import (`patient_directory/import.go:48-60`) uses
  `m.store.Atomic` → the same `orm.Store`. `dbsync` applies server rows with `orm.DB.Create/Update`
  (`dbsync/sync.go:357-363`) and the browser modules write through `orm.New(localLog)` (`dbsync/client.go:169`).
  So `text_search` is computed in **`orm.DB`** (Create, Update, UpdateFields) and nowhere else — server and browser alike.
- **D7 needs no code in `dbsync` or `json`:** `text_search` is not a model field (not in `Schema()`), so no encoder
  sees it and the server's reads (`SELECT` of the schema columns, `orm/qb.go:138-160`) never return it; the browser
  recomputes it when `dbsync` applies the row through `orm.DB`.
- **The column lives outside the model.** No struct field, no `Schema()` entry: `orm` adds it to every write, `ddl`
  adds it to the table, `crud` filters on it. SQL engines and `storage/mem` accept a column outside the schema in DML
  (`sqlt/translate.go:270-300`, `postgres/translate.go:120-140`, `storage/mem/mem.go:160-240`).
- **Two engine defects found (bug-fix-flow):** `storage/mem` (`mem.go:295,311,479`) and `indexdb`
  (`adapter.go:270-280`) scan a read by the **model's schema order**, ignoring the columns the query names — a read
  of `["note","id"]` returns `id,name`. `ddl`'s safe-drop probe (`ddl/sync.go:174-196`) already reads the wrong column
  on `mem`. And `indexdb`'s update rebuilds the stored object from schema fields only (`execute.go:131-160, 213-240`),
  so it would **drop `text_search`** on any edit. Closed in 10c and 10e with a conformance clause.
- **SQLite cannot strip accents**, so the rebuild runs in Go inside `ddl.Sync` (10f). `ddl.Capture`'s recorder must not
  rebuild (it touches no database) — explicit marker, not a fallback.

## Design gate (api-design; the owner reviews these names before 10a)

| New name | Package | What a novice reads | Replaces / deletes |
|---|---|---|---|
| `model.FieldTextSearch = "text_search"` | model | the hidden column with the row's search text | — (D3) |
| `model.SearchableText` (interface, method `SearchableText()`) | model | "this kind is text people search by" — implemented by `input.Text/Phone/Email/Rut` | the hand list of fields in each `Search` rule (D4) |
| `model.TextSearchFields(schema) []int`, `model.TextSearchOf(schema, ptrs) string` | model | which fields enter; the value for one row | `rule_patient_search.go` RUT/phone/name branches |
| `model.Definition.ListsOnlyAfterSearch bool` + `model.SearchFirst` (method `ListsOnlyAfterSearch() bool`, generated like `Allow()`) | model, definitionc | "this list stays empty until you type" | the impossible-id trick `rule_patient_search.go:22-24` (D8) |
| `storage.RewriteTextSearch(conn, m, fields, conds) error` | storage | recompute `text_search` of matching rows (new values from `m` for `fields`, the rest from the row) | — (shared by `orm` partial updates and `ddl` rebuild: one copy, zero-debt) |
| `crud.StoredEntity.Search func(a, filter) ([]Condition, error)` — **`text` parameter removed** | crud, definitionc | a rule only adds conditions; it cannot replace the text search | the `text` branches of 9 `rule_*_search.go` in 5 modules (D6) |
| `crud.ErrSearchFirstWithoutText` (mount), `definitioncheck` refuses a field named `text_search` | crud, definitioncheck | fails at boot/generation, never at runtime | — |
| ddl table `text_search_state(table_name, fields)` | ddl (internal) | where ddl remembers which fields built each table's `text_search` | — (D2: rebuild only when the column is new or the list changed) |

Prior art: `tenant_id` (a reserved column the framework stamps: `orm/store.go:12-23`), `Allow()` generated per model
(`definitionc/generate.go:393`). Complexity ledger: +1 interface marker, +4 funcs in `model`, +1 func in `storage`,
+1 internal table; deletes 9 hand search rules' text branches, `rule_patient_search.go` (step 12) and
`crud/stored_entity_read.go:62-74`. Known limit (stated, not hidden): a migration through a DDL-only transport
(an `Execer` without `Query`) on a table with text-search fields fails with a clear error (10f) instead of leaving
`text_search` empty.

## Stages

| Step | Repo | File | Depends on |
|---|---|---|---|
| 10a | `model` | [STAGE_10A_MODEL.md](STAGE_10A_MODEL.md) | 8 |
| 10b | `input` | [STAGE_10B_INPUT.md](STAGE_10B_INPUT.md) | 10a |
| 10c | `storage` | [STAGE_10C_STORAGE.md](STAGE_10C_STORAGE.md) | 10a |
| 10d | `orm` | [STAGE_10D_ORM.md](STAGE_10D_ORM.md) | 10c |
| 10e | `indexdb` | [STAGE_10E_INDEXDB.md](STAGE_10E_INDEXDB.md) | 10c |
| 10f | `ddl` | [STAGE_10F_DDL.md](STAGE_10F_DDL.md) | 10c |
| 10g | `definitioncheck`, `definitionc` | [STAGE_10G_GENERATOR.md](STAGE_10G_GENERATOR.md) | 10a |
| 10h | `crud` | [STAGE_10H_CRUD.md](STAGE_10H_CRUD.md) | 10a, 10c |
| 10i | `view`, `layout/crudview` | [STAGE_10I_SCREEN.md](STAGE_10I_SCREEN.md) | 10a |
| 10j | `chat_room`, `editorial`, `room_layout`, `service_catalog` | [STAGE_10J_MODULES.md](STAGE_10J_MODULES.md) | 10g, 10h |

`patient_directory` adopts it in step 12 (not here). Step 11 runs the conformance in `sqlite`, `postgres`, `indexdb`.
