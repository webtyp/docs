← [10h crud](STAGE_10H_CRUD.md) | Next → [10j modules](STAGE_10J_MODULES.md)

# 10i — `view` + `layout/crudview`: «Type to search» (D8 on the screen)

Repos `webtyp/view`, `webtyp/layout`. Decision: RECORD_SEARCH D8. Skills **dom-elements**, **translations**,
**langc**, **ui-consistency**. **Pantalla** (protocol rule 5).

Today a screen already opens empty with `view.Page{First: 0}` (`view/presenter.go:96-101`); what is missing is (a) a
guard so a search-first model cannot be given a page that lists on open, and (b) the text that tells the person to
type, instead of the generic empty message.

## Setup

```bash
cd webtyp/view   && go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt && GOFLAGS=-mod=mod go mod tidy
cd ../layout     && go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt -replace webtyp.com/view=../view && GOFLAGS=-mod=mod go mod tidy
```

(`layout/go.mod` may already carry other `replace` lines from block A — keep them.)

## 1. `view/view.go` — `New` refuses a contradiction (loud, like its other panics)

After the `if model.IsNil(record) { panic(…) }` block:

```go
	if sf, ok := record.(model.SearchFirst); ok && sf.ListsOnlyAfterSearch() && l.Page().declared().First != 0 {
		panic("view: New: the record lists only after a search; its Page.First must be 0")
	}
```

Note: `Page{}` (zero) declares `DefaultPage` (`First: 50`), so a search-first module must write
`view.Page{First: 0, Size: <n>}` — `patient_directory/view.go` already does (`Size: 20`).

Test in `view/tests/page_test.go` — `TestNew_SearchFirstNeedsAnEmptyOpening`:

```go
// searchFirstRecord is conformance.MockRecord declared with ListsOnlyAfterSearch.
type searchFirstRecord struct{ conformance.MockRecord }

func (*searchFirstRecord) ListsOnlyAfterSearch() bool { return true }

func TestNew_SearchFirstNeedsAnEmptyOpening(t *testing.T) {
	defer func() {
		if r := recover(); r != "view: New: the record lists only after a search; its Page.First must be 0" {
			t.Errorf("panic = %v", r)
		}
	}()
	view.New(&conformance.FakeLister{PageSize: view.Page{First: view.All}}, &searchFirstRecord{})
}
```

and a second one, `TestNew_SearchFirstWithEmptyOpeningIsAccepted`: `view.New(&conformance.FakeLister{PageSize: view.Page{First: 0, Size: 20}}, &searchFirstRecord{})`
must not panic. `MockRecord`'s methods have pointer receivers (`view/conformance/conformance.go:246-260`); embedding
it by value gives `*searchFirstRecord` the whole method set (verified). `view` has no `docs/TESTS.md`.

## 2. `layout/crudview/crudview.go` — the paragraph

Constant, in the `const` block next to `textNoMore` (line 53):

```go
	textTypeToSearch lang.Text = "Type to search"
```

In the render code (around line 1028), replace the `if emptyMsg := v.Presenter.EmptyMessage(); emptyMsg != "" { … }`
block with:

```go
		// A search-first record (model.SearchFirst) opens with nothing listed: until the person types, say so
		// instead of the empty message, which would claim there are no records.
		searchFirst := false
		if sf, ok := v.Presenter.Record().(model.SearchFirst); ok {
			searchFirst = sf.ListsOnlyAfterSearch()
		}
		waitingForText := func() bool { return searchFirst && v.search.Get() == "" }
		if searchFirst {
			asideContent.Child(
				P().Text(lang.Translate(textTypeToSearch).String()).
					BindStateFunc(widget.Open, func() bool { return waitingForText() && !v.hasRows.Get() }),
			)
		}
		if emptyMsg := v.Presenter.EmptyMessage(); emptyMsg != "" {
			asideContent.Child(
				P().Text(lang.Translate(emptyMsg).String()).
					BindStateFunc(widget.Open, func() bool { return !waitingForText() && !v.hasRows.Get() }),
			)
		}
```

(`model` is already imported in `crudview.go`, line 17.)

Test — new file `layout/crudview/crudview_search_first_wasm_test.go`, following `crudview_empty_wasm_test.go`
(same package, `Device` from `consumer_test.go`, `mockCtxWasm`):

```go
//go:build wasm

package crudview

import (
	"strings"
	"testing"

	"webtyp.com/view"
	"webtyp.com/view/conformance"
)

// searchFirstDevice is Device declared with ListsOnlyAfterSearch.
type searchFirstDevice struct{ Device }

func (*searchFirstDevice) ListsOnlyAfterSearch() bool { return true }

func TestCrudView_Wasm_SearchFirstSaysTypeToSearch(t *testing.T) {
	fb := &conformance.FakeLister{PageSize: view.Page{First: 0, Size: 20}}
	p := view.New(fb, &searchFirstDevice{}, view.WithTitle("Wasm Test"), view.WithEmpty("No items msg"))
	v := &CrudView{Presenter: p}
	v.Init(&mockCtxWasm{})
	v.Reload(nil)

	rendered := v.Render().String()
	if !strings.Contains(rendered, `data-open='true'>Type to search</p>`) {
		t.Errorf("a search-first list with no text must say «Type to search». Got:\n%s", rendered)
	}
	if strings.Contains(rendered, `data-open='true'>No items msg</p>`) {
		t.Errorf("the empty message must stay closed while waiting for text")
	}

	v.search.Set("does-not-exist")
	v.filter()
	rendered = v.Render().String()
	if !strings.Contains(rendered, `data-open='true'>No items msg</p>`) || strings.Contains(rendered, `data-open='true'>Type to search</p>`) {
		t.Errorf("with text and no rows, the empty message shows and «Type to search» does not. Got:\n%s", rendered)
	}
}
```

`Device` (`consumer_test.go:42-49`, no build tag) has pointer-receiver methods; `*searchFirstDevice` gets them all.

## 3. Translations

`cd webtyp/layout && langc sync .` — fill «Type to search» in every language it lists (es: «Escribe para buscar»),
zero untranslated keys, zero reuse warnings.

## 4. Screen

After step 12 adopts `ListsOnlyAfterSearch` in `patient_directory`, the screenshot of that step shows «Escribe para
buscar» on open (972 px and iPhone SE). Here: `gotest` in `view` and `layout` (wasm tests included).

## Done when

`gotest` green in `view` and `layout`.
