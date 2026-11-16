← [10d orm](STAGE_10D_ORM.md) | Next → [10f ddl](STAGE_10F_DDL.md)

# 10e — `indexdb`: reads return the columns asked for; an edit keeps `text_search`

Repo `webtyp/indexdb` (wasm only: every file has `//go:build wasm`; `model` is dot-imported, so `Field`, `Model`,
`Text()`, `FieldTextSearch` are unqualified). Decisions: RECORD_SEARCH D7 (the browser keeps its own `text_search`).
Skills **bug-fix-flow**, **wasm** (no maps, no stdlib `strings`).

## Setup

```bash
cd webtyp/indexdb
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/storage=../storage -replace webtyp.com/fmt=../fmt
GOFLAGS=-mod=mod go mod tidy
```

The conformance clause added in 10c (`read_of_chosen_columns_returns_them_in_order`) runs here through
`tests/conformance_test.go`. Run `gotest` first: it must FAIL (red) — that is the first defect.

## 1. Defect 1 — `Query` scans by schema (`adapter.go`, `Query`, the `return &simpleRows{…}` near line 270)

Replace `fields: m.Schema(),` with `fields: readFields(q, m),` and add below `Query`:

```go
// readFields is what a read returns, in order: the fields of the columns the query names, or the model's schema
// when it names none. A named column outside the schema is FieldTextSearch, the one text column the framework
// keeps outside every model (orm writes it, crud filters on it).
func readFields(q storage.Query, m Model) []Field {
	schema := m.Schema()
	if len(q.Columns) == 0 {
		return schema
	}
	out := make([]Field, len(q.Columns))
	for i, col := range q.Columns {
		out[i] = Field{Name: col, Type: Text()}
		for _, f := range schema {
			if f.Name == col {
				out[i] = f
				break
			}
		}
	}
	return out
}
```

`simpleRows.Columns()` already reads `r.fields`: no change. The `models` (factory) path is unchanged: `orm` reads whole
models through it. Do **not** touch `QueryRow`/`simpleScanner` (no framework code reads a subset through it).

## 2. Defect 2 — an update rebuilds the object from schema fields only (`execute.go`, `update`)

The same 30-line loop appears twice: the by-primary-key path (after `// Overwrite fields`, ~line 131) and the
cursor path (`for _, item := range matched {`, ~line 213). Both start from an empty object, so `text_search` is
dropped on every edit. Extract ONE helper and call it from both places:

```go
// copyStored copies a stored object into a new one, each schema field with its Go type, plus FieldTextSearch:
// an update then overlays the columns it names, so a column it does not name keeps its value.
func copyStored(val js.Value, fields []Field) (js.Value, error) {
	data := js.Global().Get("Object").New()
	for _, f := range fields {
		jsVal := val.Get(f.Name)
		if jsVal.IsUndefined() {
			continue
		}
		switch f.Type.Storage() {
		case FieldText, FieldRaw:
			data.Set(f.Name, jsVal.String())
		case FieldInt:
			data.Set(f.Name, int64(jsVal.Int()))
		case FieldFloat:
			data.Set(f.Name, jsVal.Float())
		case FieldBool:
			data.Set(f.Name, jsVal.Bool())
		case FieldBlob:
			n := jsVal.Get("length").Int()
			b := make([]byte, n)
			if n > 0 {
				js.CopyBytesToGo(b, jsVal)
			}
			arr, err := toJSValue(b)
			if err != nil {
				return js.Value{}, err
			}
			data.Set(f.Name, arr)
		default:
			return js.Value{}, fmt.Err("indexdb: unsupported field type", f.Type.Storage().String(), "in column", f.Name)
		}
	}
	if ts := val.Get(FieldTextSearch); !ts.IsUndefined() && !ts.IsNull() {
		data.Set(FieldTextSearch, ts.String())
	}
	return data, nil
}
```

At each of the two sites, the whole `data := js.Global().Get("Object").New()` + `for _, f := range fields { … }` block
becomes:

```go
		data, err := copyStored(val, fields)      // by-primary-key path: val is the stored object
		if err != nil {
			return 0, err
		}
```

(in the cursor path the stored object is `item.val`: `data, err := copyStored(item.val, fields)`). The overlay loop
`for i, col := range q.Columns { … data.Set(col, jsVal) }` that follows stays as it is. Deletion check:
`grep -c 'Get("Object").New()' execute.go` drops by 2 compared to before.

## 3. Test — new file `tests/text_search_test.go`

```go
//go:build wasm

package tests_test

import (
	"testing"

	"webtyp.com/model"
	"webtyp.com/storage"
)

func execQuery(t *testing.T, db storage.Conn, q storage.Query) {
	t.Helper()
	if _, err := db.Exec("", q, &User{}); err != nil {
		t.Fatalf("%v: %v", q.Action, err)
	}
}

func storedTextSearch(t *testing.T, db storage.Conn, id string) string {
	t.Helper()
	rows, err := db.Query("", storage.Query{Action: storage.ActionReadAll, Table: "user", Columns: []string{model.FieldTextSearch}, Conditions: []storage.Condition{storage.Eq("ID", id)}}, &User{})
	if err != nil {
		t.Fatal(err)
	}
	defer rows.Close()
	if !rows.Next() {
		t.Fatal("no row")
	}
	var got string
	if err := rows.Scan(&got); err != nil {
		t.Fatal(err)
	}
	return got
}

func TestTextSearch_UpdateByKeyKeepsIt(t *testing.T) {
	db := SetupDB(t, "test_text_search_update_by_key", &User{})
	defer db.Close()
	execQuery(t, db, storage.Query{Action: storage.ActionCreate, Table: "user", Columns: []string{"ID", "Name", model.FieldTextSearch}, Values: []any{"1", "Ana", "ana"}})
	execQuery(t, db, storage.Query{Action: storage.ActionUpdate, Table: "user", Columns: []string{"Email"}, Values: []any{"a@b.cl"}, Conditions: []storage.Condition{storage.Eq("ID", "1")}})
	if got := storedTextSearch(t, db, "1"); got != "ana" {
		t.Errorf("text_search = %q, want ana", got)
	}
}

func TestTextSearch_UpdateByCursorKeepsIt(t *testing.T) {
	db := SetupDB(t, "test_text_search_update_by_cursor", &User{})
	defer db.Close()
	execQuery(t, db, storage.Query{Action: storage.ActionCreate, Table: "user", Columns: []string{"ID", "Name", model.FieldTextSearch}, Values: []any{"1", "Ana", "ana"}})
	execQuery(t, db, storage.Query{Action: storage.ActionUpdate, Table: "user", Columns: []string{"Email"}, Values: []any{"a@b.cl"}, Conditions: []storage.Condition{storage.Eq("Name", "Ana")}})
	if got := storedTextSearch(t, db, "1"); got != "ana" {
		t.Errorf("text_search = %q, want ana", got)
	}
}

func TestTextSearch_LikeFindsIt(t *testing.T) {
	db := SetupDB(t, "test_text_search_like", &User{})
	defer db.Close()
	execQuery(t, db, storage.Query{Action: storage.ActionCreate, Table: "user", Columns: []string{"ID", "Name", model.FieldTextSearch}, Values: []any{"1", "Lucía", "lucia"}})
	execQuery(t, db, storage.Query{Action: storage.ActionCreate, Table: "user", Columns: []string{"ID", "Name", model.FieldTextSearch}, Values: []any{"2", "Bruno", "bruno"}})
	rows, err := db.Query("", storage.Query{Action: storage.ActionReadAll, Table: "user", Columns: []string{"ID"}, Conditions: []storage.Condition{storage.Like(model.FieldTextSearch, "%luc%")}}, &User{})
	if err != nil {
		t.Fatal(err)
	}
	defer rows.Close()
	var ids []string
	for rows.Next() {
		var id string
		if err := rows.Scan(&id); err != nil {
			t.Fatal(err)
		}
		ids = append(ids, id)
	}
	if len(ids) != 1 || ids[0] != "1" {
		t.Errorf("ids = %v, want [1]", ids)
	}
}
```

This repo has no `docs/TESTS.md`; do not create one unless `gotest` asks for it.

## Done when

`gotest` green in `indexdb` (wasm tests included: the conformance clause and the 3 tests above).
