← [10g generator](STAGE_10G_GENERATOR.md) | Next → [10i screen](STAGE_10I_SCREEN.md)

# 10h — `crud`: default search on `text_search`, rule only adds (D6), `ListsOnlyAfterSearch` (D8)

Repo `webtyp/crud`. Decisions: RECORD_SEARCH D1 (default search), D2 (same function at search time), D6, D8.
Compiled and tested green on 2026-11-16 (all existing `crud` tests still pass).

`crud/docs/PLAN.md` (file upload) is another local plan of the same repo, not running anywhere: ignore it here and
do not touch its files; it runs in its own turn (**agents-workflow** «No remote executor»).

## Setup

```bash
cd webtyp/crud
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/storage=../storage -replace webtyp.com/fmt=../fmt
GOFLAGS=-mod=mod go mod tidy
```

## 1. New file `text_search.go`

```go
package crud

import (
	"webtyp.com/fmt"
	"webtyp.com/model"
	"webtyp.com/storage"
)

// textSearchConditions turns what the person typed into one condition per word on the row's text_search, passed
// through the same function that wrote it (fmt ToSearchText), joined with AND. "" → none.
//
//	textSearchConditions("Lucía 13.047") // text_search LIKE '%lucia%' AND text_search LIKE '%13047%'
func textSearchConditions(text string) []storage.Condition {
	folded := fmt.Convert(text).ToSearchText().String()
	if folded == "" {
		return nil
	}
	words := fmt.Convert(folded).Split()
	conds := make([]storage.Condition, 0, len(words))
	for _, w := range words {
		conds = append(conds, storage.Like(model.FieldTextSearch, "%"+w+"%"))
	}
	return conds
}

// listsOnlyAfterSearch reports whether the entity's model declares ListsOnlyAfterSearch (model.SearchFirst).
func listsOnlyAfterSearch(m model.Model) bool {
	sf, ok := m.(model.SearchFirst)
	return ok && sf.ListsOnlyAfterSearch()
}
```

## 2. `stored_entity_read.go` — the search block

Delete the import `"webtyp.com/fmt"` (and its blank line). In `readRecordsHandler`, replace everything from
`currentUser := currentUserOf(ctx)` down to (not including) `limit := Cap(args.Limit)` — today's lines 46-76, which
hold the `Search(currentUser, args.Filter, args.Search)` call and the `ItemLabel` `LIKE` — with:

```go
		textConds := textSearchConditions(args.Search)
		if len(textConds) == 0 && listsOnlyAfterSearch(entity.Model) {
			if enc, ok := entity.NewList().(model.Encodable); ok {
				if err := ctx.Encode(enc); err != nil {
					router.WriteError(ctx, err)
				}
			}
			return
		}
		if len(textConds) > 0 && len(model.TextSearchFields(schemaFields)) == 0 {
			router.WriteError(ctx, ErrNoSearchField)
			return
		}

		var conds []storage.Condition
		if entity.Search != nil {
			hooks, err := entity.Search(currentUserOf(ctx), args.Filter)
			if err != nil {
				router.WriteError(ctx, err)
				return
			}
			for _, c := range hooks {
				if c.Logic() == "OR" {
					router.WriteError(ctx, ErrOrCondition)
					return
				}
			}
			conds = append(conds, hooks...)
		}
		conds = append(conds, textConds...)
```

Deletion check: `grep -n "ItemLabel\|ToLower\|args.Search)" stored_entity_read.go` → empty.

## 3. `contract.go`

`StoredEntity.Search` (comment + type) becomes:

```go
	// Search adds AND-only conditions to every read of the table (a state, a site…). Declared with Rules.Read == model.AddMyBusinessRule.
	// It never searches the text: crud always adds one condition per typed word on the row's text_search (model.FieldTextSearch).
	// filter is a record of the entity itself carrying only the fields the caller sent (the entity is its own filter: `filter.sku` on the wire).
	Search func(a CurrentUser, filter model.Model) ([]storage.Condition, error)
```

In `store.go`, the comment of `type Store interface` gets one more line (the contract a hand-written Store must keep):

```go
// A Store of a table with text-search fields (model.TextSearchFields) keeps model.FieldTextSearch on every write; orm.Store does.
```

## 4. `stored_entity.go`

Add one constant to the `const (…)` block, after `ErrNoSearchField` (keep the block's existing alignment; do not
reformat the other lines):

```go
	ErrSearchFirstWithoutText mountError = "crud: the model lists only after a search but no field enters text_search; give it an input.Text, Phone, Email or Rut field"
```

In `AddStoredEntity`, right before `r.mountedEntities = append(r.mountedEntities, def.Name)`:

```go
	if listsOnlyAfterSearch(entity.Model) && len(model.TextSearchFields(entity.Model.Schema())) == 0 {
		r.failMount(ErrSearchFirstWithoutText)
		return
	}
```

## 5. Existing tests — exact changes

```diff
diff --git a/tests/memstore_test.go b/tests/memstore_test.go
index 107413f..40d8f59 100644
--- a/tests/memstore_test.go
+++ b/tests/memstore_test.go
@@ -78,6 +78,7 @@ func (s *memStore) Create(rec model.Model) error {
 			row[f.Name] = *p
 		}
 	}
+	keepTextSearch(row, schema)
 	id := model.JoinKey(parts...)
 	if _, exists := s.rows[id]; exists {
 		return storage.Duplicate(fmt.Err("memstore: duplicate key", id))
@@ -120,6 +121,7 @@ func (s *memStore) UpdateFields(rec model.Model, id string, fields []string) err
 			}
 		}
 	}
+	keepTextSearch(row, schema)
 	return nil
 }
 
@@ -151,6 +153,7 @@ func (s *memStore) UpdateFieldsIf(rec model.Model, id string, fields []string, f
 			}
 		}
 	}
+	keepTextSearch(row, schema)
 	return 1, nil
 }
 
@@ -261,3 +264,18 @@ func (s *memStore) Atomic(fn func(crud.Tx) error) error {
 	}
 	return err
 }
+
+// keepTextSearch writes text_search of row from the values the row holds now, as every crud.Store of a table with
+// text-search fields must (orm.Store does it in orm.DB).
+func keepTextSearch(row map[string]any, schema []model.Field) {
+	idx := model.TextSearchFields(schema)
+	if len(idx) == 0 {
+		return
+	}
+	ptrs := make([]any, len(schema))
+	for _, i := range idx {
+		v, _ := row[schema[i].Name].(string)
+		ptrs[i] = &v
+	}
+	row[model.FieldTextSearch] = model.TextSearchOf(schema, ptrs)
+}
diff --git a/tests/mount_check_test.go b/tests/mount_check_test.go
index 1279481..4d7fb53 100644
--- a/tests/mount_check_test.go
+++ b/tests/mount_check_test.go
@@ -52,7 +52,7 @@ func TestMount_SearchHookFollowsReadRule(t *testing.T) {
 		NewList: func() model.ModelSlice { return &tRecList{} },
 		Store:   newMemStore(),
 		Rules:   declareRules(model.AllActions, 0),
-		Search: func(a crud.CurrentUser, _ model.Model, text string) ([]storage.Condition, error) {
+		Search: func(a crud.CurrentUser, _ model.Model) ([]storage.Condition, error) {
 			return nil, nil
 		},
 	}}
diff --git a/tests/stored_entity_test.go b/tests/stored_entity_test.go
index f6aef30..56ef2ec 100644
--- a/tests/stored_entity_test.go
+++ b/tests/stored_entity_test.go
@@ -51,7 +51,7 @@ func (m *tRec) Pointers() []any {
 func (m *tRec) Schema() []model.Field {
 	return []model.Field{
 		{Name: "id", Type: model.Text(), DB: &model.FieldDB{PK: true, AutoInc: true}},
-		{Name: "name", Type: model.Text(), Item: model.ItemLabel},
+		{Name: "name", Type: searchText{model.Text()}, Item: model.ItemLabel},
 		{Name: "note", Type: model.Text()},
 		{Name: "created_at", Type: model.Int(), ServerOnly: true},
 		{Name: "updated_at", Type: model.Int(), ServerOnly: true},
@@ -298,7 +298,7 @@ func TestStoredEntity_UpdateHookSeesOldAndNext(t *testing.T) {
 	}
 }
 
-func TestStoredEntity_SearchDefaultUsesItemLabel(t *testing.T) {
+func TestStoredEntity_SearchDefaultUsesTextSearch(t *testing.T) {
 	store := newMemStore()
 	store.Create(&tRec{ID: "1", Name: "Ana"})
 	store.Create(&tRec{ID: "2", Name: "Bruno"})
@@ -326,7 +326,7 @@ func TestStoredEntity_SearchHookOrIsRejected(t *testing.T) {
 		NewList: func() model.ModelSlice { return &tRecList{} },
 		Store:   newMemStore(),
 		Rules:   declareRules(model.AllActions, model.Read),
-		Search: func(c crud.CurrentUser, _ model.Model, text string) ([]storage.Condition, error) {
+		Search: func(c crud.CurrentUser, _ model.Model) ([]storage.Condition, error) {
 			return []storage.Condition{storage.Or(storage.Eq("name", "1"))}, nil
 		},
 	}}
@@ -348,7 +348,7 @@ func TestStoredEntity_SearchReceivesTheFilterAsAnEntityRecord(t *testing.T) {
 		NewList: func() model.ModelSlice { return &tRecList{} },
 		Store:   store,
 		Rules:   declareRules(model.AllActions, model.Read),
-		Search: func(c crud.CurrentUser, filter model.Model, text string) ([]storage.Condition, error) {
+		Search: func(c crud.CurrentUser, filter model.Model) ([]storage.Condition, error) {
 			got = filter.(*tRec)
 			return []storage.Condition{storage.Eq("name", got.Name)}, nil
 		},
@@ -372,8 +372,8 @@ func TestStoredEntity_SearchRunsWithoutText(t *testing.T) {
 		NewList: func() model.ModelSlice { return &tRecList{} },
 		Store:   newMemStore(),
 		Rules:   declareRules(model.AllActions, model.Read),
-		Search: func(c crud.CurrentUser, filter model.Model, text string) ([]storage.Condition, error) {
-			called = text == ""
+		Search: func(c crud.CurrentUser, filter model.Model) ([]storage.Condition, error) {
+			called = true
 			return nil, nil
 		},
 	}}
@@ -423,7 +423,7 @@ func TestStoredEntity_FilterNeedsNoRequiredField(t *testing.T) {
 		NewList: func() model.ModelSlice { return &fRecList{} },
 		Store:   newMemStore(),
 		Rules:   declareRules(model.Read, model.Read),
-		Search: func(c crud.CurrentUser, filter model.Model, text string) ([]storage.Condition, error) {
+		Search: func(c crud.CurrentUser, filter model.Model) ([]storage.Condition, error) {
 			got = filter.(*fRec)
 			return nil, nil
 		},
```

## 6. New tests — `tests/text_search_test.go`

```go
package tests

import (
	"errors"
	"testing"

	"webtyp.com/crud"
	"webtyp.com/model"
	"webtyp.com/storage"
)

// searchText is a text kind that enters text_search (webtyp.com/input does this for Text, Phone, Email, Rut).
type searchText struct{ model.Kind }

func (searchText) SearchableText() {}

// searchFirstRec is tRec declared with ListsOnlyAfterSearch (definitionc generates this method).
type searchFirstRec struct{ tRec }

func (*searchFirstRec) ListsOnlyAfterSearch() bool { return true }

// noTextRec is fRec (no field enters text_search) declared with ListsOnlyAfterSearch.
type noTextRec struct{ fRec }

func (*noTextRec) ListsOnlyAfterSearch() bool { return true }

func searchModule(m model.Model, store crud.Store, search func(crud.CurrentUser, model.Model) ([]storage.Condition, error)) *testModule {
	domain := model.Action(0)
	if search != nil {
		domain = model.Read
	}
	return &testModule{table: crud.StoredEntity{
		Model:   m,
		NewList: func() model.ModelSlice { return &tRecList{} },
		Store:   store,
		Rules:   declareRules(model.AllActions, domain),
		Search:  search,
	}}
}

func readNames(t *testing.T, mod *testModule, args *crud.ReadArgs) []string {
	t.Helper()
	var list tRecList
	var err error
	crud.LoopbackActingAs(nil, "u1", mod).Call(operationOf(model.Read), args, &list, func(e error) { err = e })
	if err != nil {
		t.Fatal(err)
	}
	names := make([]string, len(list))
	for i, r := range list {
		names[i] = r.Name
	}
	return names
}

func TestTextSearch_IgnoresAccentsAndCase(t *testing.T) {
	store := newMemStore()
	store.Create(&tRec{ID: "1", Name: "Lucía Pérez"})
	store.Create(&tRec{ID: "2", Name: "Bruno Díaz"})
	got := readNames(t, searchModule(&tRec{}, store, nil), &crud.ReadArgs{Search: "LUCIA"})
	if len(got) != 1 || got[0] != "Lucía Pérez" {
		t.Errorf("got %v, want [Lucía Pérez]", got)
	}
}

func TestTextSearch_EveryWordMustAppear(t *testing.T) {
	store := newMemStore()
	store.Create(&tRec{ID: "1", Name: "Lucía Pérez"})
	store.Create(&tRec{ID: "2", Name: "Lucía Soto"})
	got := readNames(t, searchModule(&tRec{}, store, nil), &crud.ReadArgs{Search: "lucia soto"})
	if len(got) != 1 || got[0] != "Lucía Soto" {
		t.Errorf("got %v, want [Lucía Soto]", got)
	}
}

func TestTextSearch_RuleAddsConditionsNeverReplacesText(t *testing.T) {
	store := newMemStore()
	store.Create(&tRec{ID: "1", Name: "Ana", Note: "active"})
	store.Create(&tRec{ID: "2", Name: "Ana", Note: "retired"})
	store.Create(&tRec{ID: "3", Name: "Bea", Note: "active"})
	rule := func(crud.CurrentUser, model.Model) ([]storage.Condition, error) {
		return []storage.Condition{storage.Eq("note", "active")}, nil
	}
	got := readNames(t, searchModule(&tRec{}, store, rule), &crud.ReadArgs{Search: "ana"})
	if len(got) != 1 || got[0] != "Ana" {
		t.Errorf("got %v, want the one active Ana", got)
	}
}

func TestTextSearch_ListsOnlyAfterSearchReturnsNothingWithoutText(t *testing.T) {
	store := newMemStore()
	store.Create(&tRec{ID: "1", Name: "Ana"})
	if got := readNames(t, searchModule(&searchFirstRec{}, store, nil), &crud.ReadArgs{}); len(got) != 0 {
		t.Errorf("got %v, want no rows without text", got)
	}
	if got := readNames(t, searchModule(&searchFirstRec{}, store, nil), &crud.ReadArgs{Search: "ana"}); len(got) != 1 {
		t.Errorf("got %v, want Ana with text", got)
	}
}

func TestTextSearch_ListsOnlyAfterSearchNeedsATextField(t *testing.T) {
	mod := &testModule{table: crud.StoredEntity{
		Model:   &noTextRec{},
		NewList: func() model.ModelSlice { return &fRecList{} },
		Store:   newMemStore(),
		Rules:   declareRules(model.Read, 0),
	}}
	if err := crud.Mount(nil, nil, nil, mod); !errors.Is(err, crud.ErrSearchFirstWithoutText) {
		t.Errorf("err = %v, want ErrSearchFirstWithoutText", err)
	}
}
```

Rename the row of `TestStoredEntity_SearchDefaultUsesItemLabel` in `docs/TESTS.md` to
`TestStoredEntity_SearchDefaultUsesTextSearch` and add the 5 new rows.

## 7. Translations

`ErrSearchFirstWithoutText` is a new `mountError` text: run `langc sync .` (skill **langc**) and fill every empty value
it reports, zero untranslated keys.

## Done when

`gotest` green in `crud`. `grep -rn "text string" --include=*.go .` → empty. Then re-run `gotest` in `orm` (it
implements `crud.Tx` and its `go.mod` must now `replace webtyp.com/crud=../crud`: add it).
