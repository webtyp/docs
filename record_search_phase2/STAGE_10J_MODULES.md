← [10i screen](STAGE_10I_SCREEN.md) | Next → step 11 in the master

# 10j — the four other modules with a `Search` rule drop `text`

Repos `veltylabs/modules/chat_room`, `editorial`, `room_layout`, `service_catalog` (Spanish comments: **dev-protocols**).
Decision: RECORD_SEARCH D6. `patient_directory` is NOT touched here (step 12 deletes its rule).

Why now: the cascade of gate 13 publishes `crud` and `definitionc` with the new `Search` signature; these modules
would stop compiling. Their text branches searched `name`/`title` with a case-sensitive `LIKE` — the field is an
`input.Text` in all three (`chat_room/definition.go:38`, `editorial/definition.go:24`, `service_catalog/definition.go:40`),
so crud's default search on `text_search` replaces it (and now ignores accents and case).

## Setup (each module)

```bash
cd veltylabs/modules/<module>
for d in model input storage fmt crud orm ddl definitioncheck; do go mod edit -replace webtyp.com/$d=../../../webtyp/$d; done
GOFLAGS=-mod=mod go mod tidy
```

(`go mod edit` adds a `replace` only for a module the `go.mod` requires after tidy; drop the line of any dep the module
does not use — `go mod tidy` reports it.) The local generator was installed in 10g (`go install ./cmd/definitionc`).

## Changes — exact

| File | Change |
|---|---|
| `chat_room/rule_chat_room_search.go` | signature → `func (p *Room) Search(a crud.CurrentUser, filter *Room, m *Module) ([]storage.Condition, error) {`; delete the 3 lines `if text != "" { conditions = append(conditions, storage.Like(Room_.Name, "%"+text+"%")) }` |
| `chat_room/rule_chat_member_search.go` | signature → `func (p *Member) Search(a crud.CurrentUser, filter *Member, m *Module) ([]storage.Condition, error) {` (body unchanged) |
| `editorial/rule_Post_search.go` | signature → `func (p *Post) Search(a crud.CurrentUser, filter *Post, m *Module) ([]storage.Condition, error) {`; delete the 3 lines `if text != "" { conds = append(conds, storage.Like("title", "%"+text+"%")) }` |
| `room_layout/rule_room_artifact_search.go` | signature → `func (p *RoomArtifact) Search(a crud.CurrentUser, filter *RoomArtifact, m *Module) ([]storage.Condition, error) {` |
| `room_layout/rule_room_category_search.go` | signature → `func (p *RoomCategory) Search(a crud.CurrentUser, filter *RoomCategory, m *Module) ([]storage.Condition, error) {` |
| `room_layout/rule_room_shift_search.go` | signature → `func (p *RoomShift) Search(a crud.CurrentUser, filter *RoomShift, m *Module) ([]storage.Condition, error) {` |
| `room_layout/rule_room_space_equipment_search.go` | signature → `func (p *RoomEquipment) Search(a crud.CurrentUser, filter *RoomEquipment, m *Module) ([]storage.Condition, error) {` |
| `service_catalog/rule_service_search.go` | signature → `func (p *Service) Search(a crud.CurrentUser, filter *Service, m *Module) ([]storage.Condition, error) {`; delete the 3 lines `if text != "" { conds = append(conds, orm.Like(Service_.Name, "%"+text+"%")) }` |

Verified 2026-11-16: no `room_layout` body and not `rule_chat_member_search.go` reads `text`; their comments speak
only of `room_id` filters and stay as they are.

Then, in each module: `definitionc` (regenerates `definition_gen.go` with the new `Search:` hook) and `gotest`.

Deletion check: `grep -rn "text string" --include=*.go veltylabs/modules/{chat_room,editorial,room_layout,service_catalog}` → empty.

## Done when

`gotest` green in the four modules. Their `go.mod` keep the `replace` lines until gate 13.
