← [10b input](STAGE_10B_INPUT.md) | Next → [10d orm](STAGE_10D_ORM.md)

# 10c — `storage`: reads return the columns asked for; `RewriteTextSearch`

Repo `webtyp/storage`. Decisions: RECORD_SEARCH D2 (per-row recompute, shared by `orm` and `ddl`). Skill **bug-fix-flow**.
Everything here was compiled and its tests run green on 2026-11-16.

## Setup

```bash
cd webtyp/storage
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt
GOFLAGS=-mod=mod go mod tidy
```

## 1. Bug first (bug-fix-flow): `mem` scans by schema order, not by the columns the query names

A read of `Columns: ["note","id"]` returns `id` into the first pointer and `name` into the second. Hole: no
conformance clause reads a subset of columns. Close it **before** the fix:

### 1a. New clause in `conformance/conformance.go` (runs on mem here, and on indexdb/sqlite/postgres in their repos)

In `Run`, after the `read_all_returns_every_row` line:

```go
	t.Run("read_of_chosen_columns_returns_them_in_order", func(t *testing.T) { readOfChosenColumns(t, f) })
```

At the end of the file:

```go
// readOfChosenColumns: a read that names its columns returns exactly those, in that order — never the model's schema
// order (storage.RewriteTextSearch and ddl's probes read a few columns of a table).
func readOfChosenColumns(t *testing.T, f Factory) {
	conn := setup(t, f, &Widget{Id: "w1", Name: "alpha", Qty: 3, Active: true, Note: "n1"})
	q := storage.Query{Action: storage.ActionReadAll, Table: WidgetModel.Name, Columns: []string{"note", "id"}, Conditions: []storage.Condition{storage.Eq("id", "w1")}}
	plan, err := conn.Compile(q, &Widget{})
	if err != nil {
		t.Fatalf("compile: %v", err)
	}
	rows, err := conn.Query(plan.Query, plan.Args...)
	if err != nil {
		t.Fatalf("query: %v", err)
	}
	defer rows.Close()
	if !rows.Next() {
		t.Fatal("no row")
	}
	var note, id string
	if err := rows.Scan(&note, &id); err != nil {
		t.Fatalf("scan: %v", err)
	}
	if note != "n1" || id != "w1" {
		t.Errorf("got note=%q id=%q, want note=n1 id=w1", note, id)
	}
}
```

Run `gotest`: the clause must FAIL on mem (red). Then:

### 1b. Fix `mem/mem.go` — apply exactly this diff

```diff
@@ -292,7 +292,7 @@
 	if len(rows) == 0 {
 		return &memScanner{err: storage.ErrNoRows}
 	}
-	return &memScanner{row: copyRow(rows[0]), schema: p.m.Schema()}
+	return &memScanner{row: copyRow(rows[0]), columns: readColumns(q, p.m)}
 }
 
 func (e *engine) Query(query string, args ...any) (storage.Rows, error) {
@@ -308,7 +308,7 @@
 	for i, r := range rows {
 		out[i] = copyRow(r) // the caller scans after the lock is released
 	}
-	return &memRows{rows: out, schema: p.m.Schema(), idx: -1}, nil
+	return &memRows{rows: out, columns: readColumns(q, p.m), idx: -1}, nil
 }
 
 // copyRow detaches a row from storage so a later Update cannot change what a caller is scanning.
@@ -446,42 +446,49 @@
 }
 
 type memScanner struct {
-	row    dbRow
-	schema []model.Field
-	err    error
+	row     dbRow
+	columns []string
+	err     error
 }
 
 func (s *memScanner) Scan(dest ...any) error {
 	if s.err != nil {
 		return s.err
 	}
-	return scanInto(s.row, s.schema, dest)
+	return scanInto(s.row, s.columns, dest)
 }
 
 type memRows struct {
-	rows   []dbRow
-	schema []model.Field
-	idx    int
-}
-
-func (r *memRows) Next() bool             { r.idx++; return r.idx < len(r.rows) }
-func (r *memRows) Scan(dest ...any) error { return scanInto(r.rows[r.idx], r.schema, dest) }
-func (r *memRows) Columns() ([]string, error) {
-	cols := make([]string, len(r.schema))
-	for i, f := range r.schema {
+	rows    []dbRow
+	columns []string
+	idx     int
+}
+
+func (r *memRows) Next() bool                 { r.idx++; return r.idx < len(r.rows) }
+func (r *memRows) Scan(dest ...any) error     { return scanInto(r.rows[r.idx], r.columns, dest) }
+func (r *memRows) Columns() ([]string, error) { return r.columns, nil }
+func (r *memRows) Close() error               { return nil }
+func (r *memRows) Err() error                 { return nil }
+
+// readColumns is what a read returns, in order: the columns the query names, or the model's schema when it names none.
+func readColumns(q storage.Query, m model.Model) []string {
+	if len(q.Columns) > 0 {
+		return q.Columns
+	}
+	schema := m.Schema()
+	cols := make([]string, len(schema))
+	for i, f := range schema {
 		cols[i] = f.Name
 	}
-	return cols, nil
+	return cols
 }
-func (r *memRows) Close() error { return nil }
-func (r *memRows) Err() error   { return nil }
 
-func scanInto(row dbRow, schema []model.Field, dest []any) error {
-	for i, f := range schema {
+func scanInto(row dbRow, columns []string, dest []any) error {
+	for i, col := range columns {
 		if i >= len(dest) {
 			break
 		}
-		v, _ := row.get(f.Name)
+		v, _ := row.get(col)
 		if err := storage.ScanAny(v, dest[i]); err != nil {
 			return err
 		}
```

`gotest` green (the new clause passes, every old one still passes).

## 2. New file `text_search.go`

```go
package storage

import (
	"webtyp.com/fmt"
	"webtyp.com/model"
)

// ErrTextSearchNoKey: a table whose rows must carry text_search has no primary key, so a row cannot be rewritten alone.
var ErrTextSearchNoKey = fmt.Err("storage: text_search needs a primary key to rewrite a row")

// RewriteTextSearch recomputes model.FieldTextSearch of every row of m's table that matches conds and writes it,
// one UPDATE per row. The value of a field named in fields comes from m (the new value of an UpdateFields that is
// about to run); every other value comes from the row as stored. fields nil reads every value from the row (the
// rebuild of a whole table). m is never changed. A table with no TextSearchFields does nothing. Call it inside the
// transaction of the write, BEFORE the UPDATE of fields: conds may name a column that UPDATE changes.
//
//	err := storage.RewriteTextSearch(tx, rec, []string{"phone"}, []storage.Condition{storage.Eq("id", "p1")})
func RewriteTextSearch(conn Conn, m model.Model, fields []string, conds []Condition) error {
	schema := m.Schema()
	search := model.TextSearchFields(schema)
	if len(search) == 0 {
		return nil
	}
	var keys []int
	for i, f := range schema {
		if f.IsPK() {
			keys = append(keys, i)
		}
	}
	if len(keys) == 0 {
		return ErrTextSearchNoKey
	}
	read := append(make([]int, 0, len(keys)+len(search)), keys...)
	for _, i := range search {
		if !named(fields, schema[i].Name) {
			read = append(read, i)
		}
	}
	given := m.Pointers()
	columns := make([]string, len(read))
	for j, i := range read {
		columns[j] = schema[i].Name
	}
	plan, err := conn.Compile(Query{Action: ActionReadAll, Table: m.ModelName(), Columns: columns, Conditions: conds}, m)
	if err != nil {
		return err
	}
	rows, err := conn.Query(plan.Query, plan.Args...)
	if err != nil {
		return err
	}
	type rewrite struct {
		key  []any
		text string
	}
	var todo []rewrite
	for rows.Next() {
		ptrs := make([]any, len(schema))
		for _, i := range search {
			if named(fields, schema[i].Name) {
				ptrs[i] = given[i]
			}
		}
		dest := make([]any, len(read))
		for j, i := range read {
			if schema[i].Type.Storage() == model.FieldInt {
				dest[j] = new(int64)
			} else {
				dest[j] = new(string)
			}
			ptrs[i] = dest[j]
		}
		if err := rows.Scan(dest...); err != nil {
			rows.Close()
			return err
		}
		key := make([]any, len(keys))
		for j := range keys {
			switch v := dest[j].(type) {
			case *int64:
				key[j] = *v
			case *string:
				key[j] = *v
			}
		}
		todo = append(todo, rewrite{key: key, text: model.TextSearchOf(schema, ptrs)})
	}
	if err := rows.Err(); err != nil {
		rows.Close()
		return err
	}
	rows.Close()
	for _, r := range todo {
		where := make([]Condition, len(keys))
		for j, i := range keys {
			where[j] = Eq(schema[i].Name, r.key[j])
		}
		plan, err := conn.Compile(Query{Action: ActionUpdate, Table: m.ModelName(), Columns: []string{model.FieldTextSearch}, Values: []any{r.text}, Conditions: where}, m)
		if err != nil {
			return err
		}
		if _, err := conn.Exec(plan.Query, plan.Args...); err != nil {
			return err
		}
	}
	return nil
}

// named reports whether name is one of fields.
func named(fields []string, name string) bool {
	for _, f := range fields {
		if f == name {
			return true
		}
	}
	return false
}
```

## 3. Tests — new file `tests/text_search_test.go` (package `tests`, like the other files there)

```go
package tests

import (
	"testing"

	"webtyp.com/model"
	"webtyp.com/storage"
	"webtyp.com/storage/mem"
)

// searchKind is a text kind that enters text_search (webtyp.com/input does this for Text, Phone, Email, Rut).
type searchKind struct{ model.Kind }

func (searchKind) SearchableText() {}

type person struct {
	ID, Name, Phone, Notes string
}

func (p *person) ModelName() string { return "person" }
func (p *person) Schema() []model.Field {
	return []model.Field{
		{Name: "id", Type: model.Text(), DB: &model.FieldDB{PK: true}},
		{Name: "name", Type: searchKind{model.Text()}},
		{Name: "phone", Type: searchKind{model.Text()}},
		{Name: "notes", Type: model.Text()},
	}
}
func (p *person) Pointers() []any                { return []any{&p.ID, &p.Name, &p.Phone, &p.Notes} }
func (p *person) IsNil() bool                    { return p == nil }
func (p *person) EncodeFields(model.FieldWriter) {}
func (p *person) DecodeFields(model.FieldReader) {}

// insert writes a row with the columns given, as orm does (text_search included when given).
func insert(t *testing.T, conn storage.Conn, cols []string, vals []any) {
	t.Helper()
	plan, err := conn.Compile(storage.Query{Action: storage.ActionCreate, Table: "person", Columns: cols, Values: vals}, &person{})
	if err != nil {
		t.Fatal(err)
	}
	if _, err := conn.Exec(plan.Query, plan.Args...); err != nil {
		t.Fatal(err)
	}
}

// textSearchOf reads the text_search column of row id.
func textSearchOf(t *testing.T, conn storage.Conn, id string) string {
	t.Helper()
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

func TestTextSearch_RewriteFromStoredRow(t *testing.T) {
	conn := mem.New()
	insert(t, conn, []string{"id", "name", "phone", "notes", model.FieldTextSearch}, []any{"p1", "Lucía Pérez", "+56912345678", "Ficha 12", "stale"})
	if err := storage.RewriteTextSearch(conn, &person{}, nil, []storage.Condition{storage.Eq("id", "p1")}); err != nil {
		t.Fatal(err)
	}
	if got := textSearchOf(t, conn, "p1"); got != "lucia perez 56912345678" {
		t.Errorf("text_search = %q, want %q", got, "lucia perez 56912345678")
	}
}

func TestTextSearch_RewriteOnlyMatchingRows(t *testing.T) {
	conn := mem.New()
	insert(t, conn, []string{"id", "name", model.FieldTextSearch}, []any{"p1", "Ana", "old1"})
	insert(t, conn, []string{"id", "name", model.FieldTextSearch}, []any{"p2", "Bruno", "old2"})
	if err := storage.RewriteTextSearch(conn, &person{}, nil, []storage.Condition{storage.Eq("id", "p1")}); err != nil {
		t.Fatal(err)
	}
	if got := textSearchOf(t, conn, "p1"); got != "ana" {
		t.Errorf("p1 = %q, want ana", got)
	}
	if got := textSearchOf(t, conn, "p2"); got != "old2" {
		t.Errorf("p2 = %q, want old2 (not matched, not touched)", got)
	}
}

func TestTextSearch_RewriteMixesNewValuesWithStoredOnes(t *testing.T) {
	conn := mem.New()
	insert(t, conn, []string{"id", "name", "phone", model.FieldTextSearch}, []any{"p1", "Ana", "+56911111111", "ana 56911111111"})
	next := &person{ID: "p1", Phone: "+56 9 2222 2222"} // an UpdateFields of phone only: Name is empty in the record
	if err := storage.RewriteTextSearch(conn, next, []string{"phone"}, []storage.Condition{storage.Eq("id", "p1")}); err != nil {
		t.Fatal(err)
	}
	if got := textSearchOf(t, conn, "p1"); got != "ana 56922222222" {
		t.Errorf("text_search = %q, want %q", got, "ana 56922222222")
	}
}
```

Add to `docs/TESTS.md` the 3 tests and the conformance clause (follow the existing rows' format).

## Done when

`gotest` green in `storage`. Note for step 11: the new clause will run in `sqlite`, `postgres` and `indexdb` through
their conformance tests; `indexdb` is fixed in 10e.
