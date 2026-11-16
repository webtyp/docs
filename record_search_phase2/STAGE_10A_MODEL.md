← [Plan](PLAN.md) | Next → [10b input](STAGE_10B_INPUT.md)

# 10a — `model`: the `text_search` contract and `ListsOnlyAfterSearch`

Repo `webtyp/model`. Decisions: RECORD_SEARCH D3, D4, D5, D8. Skills: **api-harness**, **isomorphic-libraries**, **wasm**.

## Setup

```bash
cd webtyp/model
go mod edit -replace webtyp.com/fmt=../fmt   # ToSearchText exists only in the local fmt (step 8)
GOFLAGS=-mod=mod go mod tidy
```

## 1. New file `text_search.go` (verified: compiles, gives the D2/D5 examples)

```go
package model

import "webtyp.com/fmt"

// FieldTextSearch is the hidden column where the framework keeps each row's search text: every field that
// enters it (TextSearchFields), in its search form. No module declares it; a field with this name is refused
// at generation (definitioncheck).
const FieldTextSearch = "text_search"

// textSearchSeparator joins two fields so that their words never merge: «·» ends a field for fmt ToSearchText.
const textSearchSeparator = " · "

// SearchableText is implemented by a kind whose value is short free text a person types to find a record:
// a name, a phone, an email, a document number (webtyp.com/input: Text, Phone, Email, Rut). Options, numbers,
// dates, files, passwords and long texts (input.Textarea) do not implement it.
//
//	func (t *text) SearchableText() {}
type SearchableText interface {
	Kind
	SearchableText()
}

// TextSearchFields returns, in schema order, the indexes of the fields whose value enters the row's
// text_search: a SearchableText kind that is neither a primary key nor ServerOnly. Empty → the table has
// no text_search column.
//
//	idx := model.TextSearchFields(p.Schema()) // rut, name, phone, email of a patient
func TextSearchFields(schema []Field) []int {
	var idx []int
	for i, f := range schema {
		if f.IsPK() || f.ServerOnly {
			continue
		}
		if _, ok := f.Type.(SearchableText); ok {
			idx = append(idx, i)
		}
	}
	return idx
}

// TextSearchOf is the text_search value of one row: the values of its TextSearchFields, joined so that two
// fields never merge, in their search form. ptrs only needs a *string at those indexes.
//
//	model.TextSearchOf(p.Schema(), p.Pointers()) // "lucia perez 130479800 56912345678"
func TextSearchOf(schema []Field, ptrs []any) string {
	var parts []string
	for _, i := range TextSearchFields(schema) {
		if p, ok := ptrs[i].(*string); ok && p != nil && *p != "" {
			parts = append(parts, *p)
		}
	}
	if len(parts) == 0 {
		return ""
	}
	return fmt.Convert(fmt.JoinSlice(parts, textSearchSeparator)).ToSearchText().String()
}

// SearchFirst is implemented by the struct definitionc generates from a Definition with
// ListsOnlyAfterSearch: a read without text returns no rows (webtyp.com/crud) and the screen says
// «Type to search» (webtyp.com/layout/crudview).
//
//	if sf, ok := rec.(model.SearchFirst); ok && sf.ListsOnlyAfterSearch() { … }
type SearchFirst interface {
	ListsOnlyAfterSearch() bool
}
```

## 2. `definition.go` — new field, after `ExternalCalls` (line 40)

```go
	// ListsOnlyAfterSearch keeps the entity's list empty until the person types: a read without text returns no rows
	// (crud) and the screen says «Type to search» (crudview). For tables too large to show whole (a patient registry).
	// At least one field must enter text_search (crud refuses the mount otherwise).
	//
	//	var PatientModel = model.Definition{Name: "patient", ListsOnlyAfterSearch: true, …}
	ListsOnlyAfterSearch bool
```

## 3. Tests — new file `tests/text_search_test.go` (package `model_test`)

```go
package model_test

import (
	"testing"

	"webtyp.com/model"
)

// searchKind stands for input.Text/Phone/Email/Rut (model cannot import input).
type searchKind struct{ model.Kind }

func (searchKind) SearchableText() {}

func patientSchema() []model.Field {
	return []model.Field{
		{Name: "id", Type: searchKind{model.Text()}, DB: &model.FieldDB{PK: true}},
		{Name: "tenant_id", Type: searchKind{model.Text()}, ServerOnly: true},
		{Name: "name", Type: searchKind{model.Text()}},
		{Name: "rut", Type: searchKind{model.Text()}},
		{Name: "phone", Type: searchKind{model.Text()}},
		{Name: "notes", Type: model.Text()},
	}
}

func TestTextSearch_FieldsSkipKeyServerOnlyAndPlainKinds(t *testing.T) {
	got := model.TextSearchFields(patientSchema())
	if len(got) != 3 || got[0] != 2 || got[1] != 3 || got[2] != 4 {
		t.Errorf("TextSearchFields = %v, want [2 3 4] (name, rut, phone)", got)
	}
}

func TestTextSearch_ValueJoinsFieldsInSearchForm(t *testing.T) {
	id, tenant, name, rut, phone, notes := "p1", "t1", "Lucía Pérez", "12.345.678-K", "+56 9 1234 5678", "Ficha 12"
	got := model.TextSearchOf(patientSchema(), []any{&id, &tenant, &name, &rut, &phone, &notes})
	if got != "lucia perez 12345678k 56912345678" {
		t.Errorf("TextSearchOf = %q, want %q", got, "lucia perez 12345678k 56912345678")
	}
}
```

Table of the remaining cases (write each as its own test in the same file, same setup):

| Test | Values (name, rut, phone) | Expected `TextSearchOf` |
|---|---|---|
| `TestTextSearch_DigitsOfTwoFieldsNeverMerge` | `"Lucía Pérez"`, `"13047980-0"`, `"+56912345678"` | `"lucia perez 130479800 56912345678"` |
| `TestTextSearch_EmptyFieldsAreSkipped` | `"José Núñez"`, `""`, `""` | `"jose nunez"` |
| `TestTextSearch_NoValuesIsEmpty` | `""`, `""`, `""` | `""` |

Add the 5 rows to `docs/TESTS.md`.

## Done when

`gotest` green in `model`. `grep -rn "FieldTextSearch\|SearchableText\|ListsOnlyAfterSearch" model/*.go` shows the new names.
