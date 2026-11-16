← [10c storage](STAGE_10C_STORAGE.md) | Next → [10e indexdb](STAGE_10E_INDEXDB.md)

# 10d — `orm`: every write keeps `text_search`

Repo `webtyp/orm`. Decisions: RECORD_SEARCH D1 (the layer every write passes), D2 (recompute this row, same
transaction, merge on a partial edit). Compiled and tested green on 2026-11-16.

Why here: `crud` (`orm.Store`), imports (`Store.Atomic`), `dbsync` (`orm.DB.Create/Update`) and the browser modules
all write through `orm.DB`. No other layer computes `text_search`.

## Setup

```bash
cd webtyp/orm
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/storage=../storage -replace webtyp.com/fmt=../fmt
GOFLAGS=-mod=mod go mod tidy
```

## 1. New file `text_search.go`

```go
package orm

import (
	"webtyp.com/model"
	"webtyp.com/storage"
)

// withTextSearch adds the text_search column and its value to a write of the whole row (Create, Update) when m's
// table carries one (model.TextSearchFields): every write recomputes it from the row it writes.
func withTextSearch(m model.Model, columns []string, values []any) ([]string, []any) {
	schema := m.Schema()
	if len(model.TextSearchFields(schema)) == 0 {
		return columns, values
	}
	return append(columns, model.FieldTextSearch), append(values, model.TextSearchOf(schema, m.Pointers()))
}

// textSearchTouch is how an UpdateFields reaches text_search.
type textSearchTouch int

const (
	touchNone textSearchTouch = iota // no field it writes enters text_search
	touchAll                         // it writes every field that enters text_search: m has the whole value
	touchSome                        // it writes some of them: the others are read from the stored row
)

func textSearchTouchOf(schema []model.Field, fields []string) textSearchTouch {
	search := model.TextSearchFields(schema)
	written := 0
	for _, i := range search {
		for _, name := range fields {
			if name == schema[i].Name {
				written++
				break
			}
		}
	}
	switch {
	case written == 0:
		return touchNone
	case written == len(search):
		return touchAll
	default:
		return touchSome
	}
}

// rewriteThenUpdate runs an UpdateFields that writes some, not all, of the fields in text_search: in one transaction
// it first rewrites text_search of the rows conds match (new values from m, the rest from each row; first, because
// conds may name a column the update changes), then runs the update.
func (d *DB) rewriteThenUpdate(m model.Model, fields []string, q storage.Query) (storage.Result, error) {
	var result storage.Result
	err := d.Tx(func(tx *DB) error {
		if err := storage.RewriteTextSearch(tx.conn, m, fields, q.Conditions); err != nil {
			return err
		}
		plan, err := tx.conn.Compile(q, m)
		if err != nil {
			return err
		}
		result, err = tx.conn.Exec(plan.Query, plan.Args...)
		return err
	})
	return result, err
}
```

## 2. `db.go` — three insertions (exact diff)

```diff
--- a/db.go
+++ b/db.go
@@ -60,6 +60,7 @@ func (d *DB) Create(m model.Model) error {
 		columns = append(columns, f.Name)
 		values = append(values, allValues[i])
 	}
+	columns, values = withTextSearch(m, columns, values)
 	q := storage.Query{
 		Action:  storage.ActionCreate,
 		Table:   m.ModelName(),
@@ -164,6 +165,10 @@ func (d *DB) updateFields(m model.Model, fields []string, cond storage.Condition
 		}
 	}
 
+	touch := textSearchTouchOf(schema, fields)
+	if touch == touchAll {
+		columns, values = withTextSearch(m, columns, values)
+	}
 	conds := append([]storage.Condition{cond}, rest...)
 	q := storage.Query{
 		Action:     storage.ActionUpdate,
@@ -172,6 +177,9 @@ func (d *DB) updateFields(m model.Model, fields []string, cond storage.Condition
 		Values:     values,
 		Conditions: conds,
 	}
+	if touch == touchSome {
+		return d.rewriteThenUpdate(m, fields, q)
+	}
 	plan, err := d.conn.Compile(q, m)
 	if err != nil {
 		return nil, err
@@ -200,6 +208,7 @@ func (d *DB) Update(m model.Model, cond storage.Condition, rest ...storage.Condi
 			values[i] = nil
 		}
 	}
+	columns, values = withTextSearch(m, columns, values)
 	q := storage.Query{
 		Action:     storage.ActionUpdate,
 		Table:      m.ModelName(),
```

Behaviour table (each row is a test below):

| Write | `text_search` |
|---|---|
| `Create(rec)` | computed from `rec` |
| `Update(rec, cond)` (whole row) | computed from `rec` |
| `UpdateFields(rec, fields, …)` with no searchable field in `fields` | untouched |
| `UpdateFields` naming **every** searchable field | computed from `rec`, same statement |
| `UpdateFields` naming **some** | in one `Tx`: `storage.RewriteTextSearch` (new values from `rec`, rest from the row) **then** the update |
| `Store.UpdateFieldsIf` | same as `UpdateFields` (it calls `updateFields`; a revision mismatch matches 0 rows in both statements) |

## 3. Tests — new file `tests/text_search_test.go` (package `tests`)

```go
package tests

import (
	"testing"

	"webtyp.com/model"
	"webtyp.com/orm"
	"webtyp.com/storage"
	"webtyp.com/storage/mem"
)

// searchKind is a text kind that enters text_search (webtyp.com/input does this for Text, Phone, Email, Rut).
type searchKind struct{ model.Kind }

func (searchKind) SearchableText() {}

type person struct {
	ID, Name, Phone, Notes, Status string
}

func (p *person) ModelName() string { return "person" }
func (p *person) Schema() []model.Field {
	return []model.Field{
		{Name: "id", Type: model.Text(), DB: &model.FieldDB{PK: true}},
		{Name: "name", Type: searchKind{model.Text()}},
		{Name: "phone", Type: searchKind{model.Text()}},
		{Name: "notes", Type: model.Text()},
		{Name: "status", Type: model.Text()},
	}
}
func (p *person) Pointers() []any                { return []any{&p.ID, &p.Name, &p.Phone, &p.Notes, &p.Status} }
func (p *person) IsNil() bool                    { return p == nil }
func (p *person) EncodeFields(model.FieldWriter) {}
func (p *person) DecodeFields(model.FieldReader) {}

// storedTextSearch reads the text_search column of row id straight from the connection.
func storedTextSearch(t *testing.T, db *orm.DB, id string) string {
	t.Helper()
	conn := db.RawConn()
	plan, err := conn.Compile(storage.Query{Action: storage.ActionReadOne, Table: "person", Columns: []string{model.FieldTextSearch}, Conditions: []storage.Condition{storage.Eq("id", id)}, Limit: 1}, &person{})
	if err != nil {
		t.Fatal(err)
	}
	var got string
	if err := conn.QueryRow(plan.Query, plan.Args...).Scan(&got); err != nil {
		t.Fatal(err)
	}
	return got
}

func TestTextSearch_CreateWritesIt(t *testing.T) {
	db := orm.New(mem.New())
	if err := db.Create(&person{ID: "p1", Name: "Lucía Pérez", Phone: "+56 9 1234 5678", Notes: "Ficha 12"}); err != nil {
		t.Fatal(err)
	}
	if got := storedTextSearch(t, db, "p1"); got != "lucia perez 56912345678" {
		t.Errorf("text_search = %q, want %q", got, "lucia perez 56912345678")
	}
}

func TestTextSearch_PartialUpdateMixesStoredRow(t *testing.T) {
	db := orm.New(mem.New())
	if err := db.Create(&person{ID: "p1", Name: "Ana", Phone: "+56911111111"}); err != nil {
		t.Fatal(err)
	}
	if err := db.UpdateFields(&person{Phone: "+56922222222"}, []string{"phone"}, orm.Eq("id", "p1")); err != nil {
		t.Fatal(err)
	}
	if got := storedTextSearch(t, db, "p1"); got != "ana 56922222222" {
		t.Errorf("text_search = %q, want %q", got, "ana 56922222222")
	}
}

func TestTextSearch_UpdateOfOtherFieldsKeepsIt(t *testing.T) {
	db := orm.New(mem.New())
	if err := db.Create(&person{ID: "p1", Name: "Ana"}); err != nil {
		t.Fatal(err)
	}
	if err := db.UpdateFields(&person{Status: "retired"}, []string{"status"}, orm.Eq("id", "p1")); err != nil {
		t.Fatal(err)
	}
	if got := storedTextSearch(t, db, "p1"); got != "ana" {
		t.Errorf("text_search = %q, want ana", got)
	}
}

func TestTextSearch_PartialUpdateWhoseConditionChanges(t *testing.T) {
	db := orm.New(mem.New())
	if err := db.Create(&person{ID: "p1", Name: "Ana", Phone: "+56911111111", Status: "new"}); err != nil {
		t.Fatal(err)
	}
	// the condition names status, which the same update changes: text_search is still rewritten
	if err := db.UpdateFields(&person{Name: "Bea", Status: "done"}, []string{"name", "status"}, orm.Eq("status", "new")); err != nil {
		t.Fatal(err)
	}
	if got := storedTextSearch(t, db, "p1"); got != "bea 56911111111" {
		t.Errorf("text_search = %q, want %q", got, "bea 56911111111")
	}
}

func TestTextSearch_FullUpdateWritesIt(t *testing.T) {
	db := orm.New(mem.New())
	if err := db.Create(&person{ID: "p1", Name: "Ana"}); err != nil {
		t.Fatal(err)
	}
	if err := db.Update(&person{ID: "p1", Name: "José Núñez"}, orm.Eq("id", "p1")); err != nil {
		t.Fatal(err)
	}
	if got := storedTextSearch(t, db, "p1"); got != "jose nunez" {
		t.Errorf("text_search = %q, want %q", got, "jose nunez")
	}
}
```

Add the 5 rows to `docs/TESTS.md`.

## Done when

`gotest` green in `orm`.
