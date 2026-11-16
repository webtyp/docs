← [10f ddl](STAGE_10F_DDL.md) | Next → [10h crud](STAGE_10H_CRUD.md)

# 10g — `definitioncheck` + `definitionc`: reserved name, `ListsOnlyAfterSearch`, `Search` without `text`

Repos `webtyp/definitioncheck`, `webtyp/definitionc`. Decisions: RECORD_SEARCH D3 (reserved name), D6 (the rule only
adds conditions → its signature loses `text`), D8 (generated `ListsOnlyAfterSearch()`). Skill **form-codegen**.

## Setup

```bash
cd webtyp/definitioncheck
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt && GOFLAGS=-mod=mod go mod tidy
cd ../definitionc
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt -replace webtyp.com/definitioncheck=../definitioncheck && GOFLAGS=-mod=mod go mod tidy
```

## 1. `definitioncheck/rules.go`

Two constants in the `const (…)` block (verbatim):

```go
	errTextSearchReserved = "text_search is the column the framework keeps for search; rename the field: "
	errSearchFirstNoRead  = "ListsOnlyAfterSearch needs Read in Allow: "
```

In `CheckRules`, right before `return checkRevision(def)`:

```go
	if def.ListsOnlyAfterSearch && !def.Allow.Has(model.Read) {
		return fmt.Err(errSearchFirstNoRead + def.Name)
	}
	for _, f := range def.Fields {
		if f.Name == model.FieldTextSearch {
			return fmt.Err(errTextSearchReserved + def.Name + "." + f.Name)
		}
	}
```

Tests — append to `tests/rules_test.go` (same package and style as the file):

| Test | Definition | Expected error text |
|---|---|---|
| `TestRules_TextSearchNameIsReserved` | `{Name: "patient", Allow: model.Read, Rules: model.Rules{Read: model.HaveNoBusinessRule}, Fields: model.Fields{{Name: "text_search", Type: model.Text()}}}` | `"text_search is the column the framework keeps for search; rename the field: patient.text_search"` |
| `TestRules_ListsOnlyAfterSearchNeedsRead` | `{Name: "patient", Allow: model.Create, Rules: model.Rules{Create: model.HaveNoBusinessRule}, ListsOnlyAfterSearch: true}` | `"ListsOnlyAfterSearch needs Read in Allow: patient"` |

Add both rows to `definitioncheck/docs/TESTS.md` if the repo has one.

## 2. `definitionc` — parse and generate `ListsOnlyAfterSearch()`

`generator.go`, `StructInfo` (after `Wire bool`, line 79):

```go
	ListsOnlyAfterSearch bool // model.Definition{ListsOnlyAfterSearch: true}: definitionc generates ListsOnlyAfterSearch()
```

`parse_definition.go`, in the `switch key` (after `case "Wire":`, line 96):

```go
		case "ListsOnlyAfterSearch":
			if id, ok := kv.Value.(*ast.Ident); ok && id.Name == "true" {
				info.ListsOnlyAfterSearch = true
			}
```

Both places that build a `model.Definition` for `definitioncheck.CheckRules` must carry it, so the new rule runs at
generation: `parse_definition.go:155` (`def := model.Definition{…}`) and `generator.go:404-407` (`checkResolved`):
add `ListsOnlyAfterSearch: info.ListsOnlyAfterSearch,` to each literal.

`generate.go`, right after the two `Allow()` lines (393-394), inside the same `if`:

```go
			if info.ListsOnlyAfterSearch {
				buf.Write(fmt.Sprintf("func (*%s) ListsOnlyAfterSearch() bool { return %sModel.ListsOnlyAfterSearch }\n\n", info.Name, info.Name))
			}
```

## 3. `definitionc` — the `Search` rule loses `text` (D6)

`module_gen.go:193` becomes:

```go
		b.WriteString(fmt.Sprintf("\t\tSearch: func(a crud.CurrentUser, filter model.Model) ([]storage.Condition, error) {\n\t\t\treturn (&%s{}).Search(a, filter.(*%s), m)\n\t\t},\n", n, n))
```

`rule_skeleton.go`, `skeletonSearch` (line 210) becomes exactly:

```go
const skeletonSearch = `package {{pkg}}

import (
	"webtyp.com/crud"
	"webtyp.com/storage"
)

// Search agrega condiciones a cada lectura de {{entity}} (un estado, una sede…), a partir de los campos que envió quien llama (filter). Solo condiciones unidas con AND (nunca OR).
// La búsqueda por texto ya la hace crud en la columna text_search: aquí nunca se busca texto.
func (p *{{type}}) Search(a crud.CurrentUser, filter *{{type}}, m *Module) ([]storage.Condition, error) {
	// COMPLETA TU REGLA DE NEGOCIO AQUÍ (condiciones de {{entity}}) y termina con: return conditions, rejected
	var conditions []storage.Condition // las condiciones que agrega tu regla
	var rejected error                 // el rechazo de tu regla; nil si todo está bien
}
`
```

Tests to update (exact new strings):

| File:line | Old | New |
|---|---|---|
| `tests/module_gen_test.go:44` | `` `Search: func(a crud.CurrentUser, filter model.Model, text string) ([]storage.Condition, error) {` `` | `` `Search: func(a crud.CurrentUser, filter model.Model) ([]storage.Condition, error) {` `` |
| `tests/module_gen_test.go:45` | `` `return (&Patient{}).Search(a, filter.(*Patient), text, m)` `` | `` `return (&Patient{}).Search(a, filter.(*Patient), m)` `` |
| `tests/rule_skeleton_test.go:65` | `"func (p *Patient) Search(a crud.CurrentUser, filter *Patient, text string, m *Module) ([]storage.Condition, error) {"` | `"func (p *Patient) Search(a crud.CurrentUser, filter *Patient, m *Module) ([]storage.Condition, error) {"` |

`tests/testdata/fixturecrud/definition_gen.go` is generated: if `gotest` reports it stale, regenerate it with the
command the failing test names — never edit it by hand.

New test in `tests/module_gen_test.go` — `TestModule_ListsOnlyAfterSearchIsGenerated`: take `patientModel`
(`tests/helpers_test.go:79` writes the name with TWO spaces: `Name:  "patient",`) and insert the flag
(`strings.Replace(patientModel, "Name:  \"patient\",", "Name:  \"patient\", ListsOnlyAfterSearch: true,", 1)`),
generate as `TestModule_StandardActionsHaveNoHook` does, and `wantContains(t, got, "func (*Patient) ListsOnlyAfterSearch() bool { return PatientModel.ListsOnlyAfterSearch }")`
on the squashed `definition_gen.go`. A second assertion on the plain `patientDef` output: it must NOT contain
`ListsOnlyAfterSearch()`. Add the row to `definitionc/docs/TESTS.md` if the repo has one.

## Done when

`gotest` green in `definitioncheck` and `definitionc`. `grep -rn "text string" definitionc --include=*.go` → empty.
Install the local generator for 10j and step 12: `cd webtyp/definitionc && go install ./cmd/definitionc`.
