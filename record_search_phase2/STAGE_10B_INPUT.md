← [10a model](STAGE_10A_MODEL.md) | Next → [10c storage](STAGE_10C_STORAGE.md)

# 10b — `input`: which kinds enter `text_search`

Repo `webtyp/input`. Decision: RECORD_SEARCH D4 (the kind decides, nothing is declared).

## Setup

```bash
cd webtyp/input
go mod edit -replace webtyp.com/model=../model -replace webtyp.com/fmt=../fmt
GOFLAGS=-mod=mod go mod tidy
```

## 1. One method per kind that enters — exactly these four, nothing else

| File | Add after the type's `Clone` method |
|---|---|
| `text.go` | `// SearchableText: a name or any short free text enters the row's text_search (model.SearchableText).`<br>`func (t *text) SearchableText() {}` |
| `phone.go` | `// SearchableText: a phone enters the row's text_search, so «9 1234 5678» finds it (model.SearchableText).`<br>`func (p *phone) SearchableText() {}` |
| `email.go` | `// SearchableText: an email enters the row's text_search (model.SearchableText).`<br>`func (e *email) SearchableText() {}` |
| `rut.go` | `// SearchableText: a document number enters the row's text_search, so «12345678» finds «12.345.678-5» (model.SearchableText).`<br>`func (r *rut) SearchableText() {}` |

Anti-footgun: `address.go`, `ip.go`, `mac.go`, `filepath.go` also use the HTML type `"text"` — they do **not** enter
(D4 lists only free short text, phone, email, document number). `textarea.go`, `select.go`, `radio.go`, `checkbox.go`,
`gender.go`, `number.go`, `decimal.go`, `date.go`, `hour.go`, `password.go`, `file.go`, `search.go`, `datalist.go` do
not enter either. Do not add the method to any of them.

## 2. Test — new file `tests/searchable_text_test.go` (package `input_test`)

```go
package input_test

import (
	"testing"

	"webtyp.com/input"
	"webtyp.com/model"
)

func TestSearchableText_ShortFreeTextKindsEnter(t *testing.T) {
	for name, k := range map[string]input.Input{"text": input.Text(), "phone": input.Phone(), "email": input.Email(), "rut": input.Rut()} {
		if _, ok := k.(model.SearchableText); !ok {
			t.Errorf("%s must implement model.SearchableText", name)
		}
	}
}

func TestSearchableText_OtherKindsStayOut(t *testing.T) {
	for name, k := range map[string]input.Input{"textarea": input.Textarea(), "address": input.Address(), "select": input.Select(), "checkbox": input.Checkbox(), "gender": input.Gender(), "password": input.Password()} {
		if _, ok := k.(model.SearchableText); ok {
			t.Errorf("%s must not implement model.SearchableText", name)
		}
	}
}
```

All constructors above take no arguments (`Select(opts ...fmt.KeyValue)` is variadic; verified 2026-11-16).
Add the 2 rows to `docs/TESTS.md` (this repo's header is
`| Test | setup → call → expected |`).

## Done when

`gotest` green in `input`.
