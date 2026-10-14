# Master Plan — Selection history, source resolution, the minimal fmt, and translations as data
> **Status:** IN PROGRESS · 2026-10-07 · every phase shipped except **F** (fmt done, tag waits for H1) and **H** (mjosefa-cms + veltylabs modules, local) · open work: [SOURCE_SELECTION_ROADMAP.md](SOURCE_SELECTION_ROADMAP.md) · index: [MASTER_PLANS.md](MASTER_PLANS.md)

> Orchestrator only: the **why**, the **shared contracts**, the **decisions** and the **phase
> status**. Each library has its own `docs/PLAN.md` (or `docs/LAST_PLAN_EXECUTED.md` once run).
> Open items, one per section, live in the roadmap. Dispatch: skill **agents-workflow**.
>
> **Ecosystem rule adopted in this wave:** tests live in `tests/` (external package, public API
> only). A root-level test needs an unexported identifier and starts with
> `// Root-level test (justified): …`. **No symbol is ever exported just for a test.**

## 1. The problems

1. `browser_get_selected_element` kept **one** selection; pointing at several elements lost all but the last.
2. The source locator grepped for quoted class names, but webtyp builds classes from typed identity
   (`widget.Name("composebar").Class(PartInput)`), so it never found framework pieces.
3. Dependencies resolved to the read-only module cache, not the developer's checkout.
4. Paths printed as `/home/<user>/...`: more LLM context and a leaked user name.
5. `webtyp/fmt` (in every binary) carried paths, translation, message types and escaping, so each
   change to them forced an ecosystem-wide release.
6. Translations were Go code (`lang.RegisterWords`): recompile to add a word, every user downloads
   every language, forms showed raw column names, one global server language.

## 2. The new shape

- **Selection history.** Alt+click (or the DevTools pointer / Elements panel) marks an element with a
  numbered badge and stores a snapshot (report, source locations, screenshot). Last **10** kept;
  the tool takes `count` (1–10, newest first) and `clear`.
- **Locator.** `name__part` resolves through `widget.Name` → `widget.Part` → class var →
  **render** / **style** (`css.go`; the root class via `style.For(c).Root(`) / **declaration**.
  Then quoted text in the project and local checkouts (never the cache), then the nearest ancestor.
  Max 5 locations.
- **Local checkouts.** A dependency is read from the checkout under the outermost
  `*.code-workspace` (`modfind.Module.LocalDir`), else from the cache, labeled as such.
- **`~` paths** everywhere (`filepath.Tilde`).
- **fmt reduced** to the stdlib replacement layer plus its core. Four concerns left it:

  | Repo | Takes from fmt |
  |---|---|
  | `webtyp.com/filepath` | `PathJoin/Base/Ext/Short/RelativeTo` → `Join/Base/Ext/Short/RelativeTo`, new `Tilde`; `SetPathBase` deleted |
  | `webtyp.com/lang` | the `fmt/lang` package (+ `langc` generator, `Load`, `Current`, `Supported`) |
  | `webtyp.com/msgtype` | `MessageType`, `Msg.*`, `StringType()` → `Type`, constants (wire values 0–13 frozen), `Detect` |
  | `webtyp.com/escape` | `EscapeHTML`/`EscapeAttr` → `HTML`; `JSONEscape` → `JSON` |

- **Translations are data.**
  - `config/lang.json` per project, by key, values positional in the header's `languages` order
    (append-only): `{"default":"es","languages":["es","fr"],"keys":{"Delete":["Eliminar","Supprimer"]}}`.
  - Libraries ship `lang.json` at the module root (no `default`); the project wins.
  - `langc` keeps files in step with the code: adds empty slots, removes unused keys, never
    overwrites a translation; MCP `lang_missing` lists what is untranslated.
  - sitec inlines the merged dictionary in `index.html` as
    `<script type="application/json" id="webtyp-lang">` (no extra request, no rebuild).
  - Browser language if listed → `default` → English. Only the client translates.
  - One `Translate` argument = one key, exactly as written. `model.Field.Label` (English) is a key.
  - **Framework layer** (TUI/logs for the developer): each tool embeds its `lang.json`
    (`Dictionary() []byte`), `lang.Load` at startup, system language by default, choice persisted as
    `TUI_LANG`. Exactly the TUI's languages (en es zh hi ar pt fr de ru), from `lang.Supported()`.
  - `RegisterWords`/`DictEntry` deleted.

## 3. Shared contracts

| Contract | Owner | Consumers |
|---|---|---|
| `filepath.Join/Base/Ext/Short/RelativeTo/Tilde` | filepath | devbrowser, app, devtui, fetch, ddlc |
| `lang.*` (path `webtyp.com/lang`), `lang.Load/Current/Supported`, `lang.ScriptID` | lang | 12+ repos |
| `langc.Translations` (`SyncTranslations`, `BundleTranslations`, `MissingTranslations`) | lang | sitec (via interface), app |
| `msgtype.Type`, constants, `Detect` | msgtype | 10 repos |
| `escape.HTML`, `escape.JSON(b, s)` | escape | dom, json, goflare-demo (sitetheme pending) |
| `modfind.Discoverer`, `Module.LocalDir/SourceDir()`, `WorkspaceRoot` (`Seed`/`Dirs`/`Refresh` deleted in v0.0.11) | modfind | devflow, sitec, ormc, image, app |
| `sitec.Translations` + `SetTranslations` | sitec | app |
| `model.Field.Label` / `Help` (English = translation key) | model | form, langc, every module |
| `devbrowser.SourceLocation{File, Line, Kind, Token, Match, Origin}` | devbrowser | app |
| `Dictionary() []byte` (embedded tool `lang.json`) | devtui, devbrowser, server, sitec, app | app startup |
| `fmt.SetTranslator` + the Conv primitives the new repos use | fmt | lang, msgtype, escape, filepath |

## 4. Decisions

- `fmt` phase F is tagged **v1.1.0**, no `/v2`: every consumer is in the ecosystem and migrates first.
- No backward compatibility, no deprecation shims.
- `Capitalize`, `CamelUp`, `SnakeLow`, `Truncate` and `Quote` (as `%q`) stay in `fmt`.
- Escaping has its own repo: not `dom` (WASM init loads the DOM engine), not `html` (import cycle).
- Translation format and rules as in §2; no plurals yet (the format allows ICU later).
- Rule 2 (`fmt.Err`) in project mode covers only the project's own module; a library translates its
  own errors in its `lang.json`.
- `modfind.Seed`/`Dirs`/`Refresh` removed by rule: test-only, duplicate and dead API respectively.
  A future cache-invalidation need is a bug to prove with a failing test, not a reason to keep it.
- sitetheme and client-monjitaschillan-platform are left as they are (maintainer).

## 5. Phase status

| Phase | Repo | State | Notes |
|---|---|---|---|
| T0 | dom · widget | ✅ v0.13.20 · v0.6.36 | `DescribedBy`, `PartHelp` |
| A1 | filepath | ✅ v0.1.0 | |
| A2 | modfind | ✅ v0.0.10 | e2e: 57/68 mjosefa-cms modules resolve to local checkouts |
| A3 · T6 | lang | ✅ v0.1.3 | Jules left `langc` as stubs → planner implemented; page dictionary retried until the element exists; named-constant keys |
| A4 · A5 | msgtype · escape | ✅ v0.1.0 · v0.1.0 | |
| T1 · T2 | model · form | ✅ v0.2.2 · v0.4.24 | |
| T3 | sitec | ✅ v0.2.50 | Jules PR had no implementation → planner |
| T4 · T5 | components · layout | ✅ v0.8.12 · v0.3.29 | both finished by planner on the branch |
| B | devflow | ✅ v0.4.122 | |
| C | devbrowser | ✅ v0.6.2 | run locally (Jules session wrote Python) |
| E · E2 | consumers, tool dictionaries | ✅ | fetch, ddlc, dom, json, goflare-demo, binary, chat_room, ormc, image, view, devtui, input, pdf, server, date, business_calendar; crudp and appointment_booking committed but unpublished → roadmap |
| D | app | ✅ v0.6.114 | run locally (private repo; Jules blocked); e2e on mjosefa-cms found the root-class style gap, fixed with a red test |
| G | modfind | ✅ v0.0.11 | `Seed`, `Dirs`, `Refresh` deleted |
| F | fmt v1.1.0 | ✅ run locally, tag pending | Jules session vanished; commit on `main` untagged until H1 |
| H | mjosefa-cms + veltylabs modules | ⏳ local | roadmap H1–H3 |

## 6. Lessons for the next wave

- Jules often returns a PR with stubs or no implementation (lang, sitec, layout): always check the
  diff against the plan's stages, not only the tests.
- Private repos (`app`, veltylabs) go local: Jules either stalls or sets `GOPRIVATE` and fails to
  fetch private deps.
- An end-to-end check against a real project finds what unit fixtures miss (the root-class style
  rule, `langc` keys behind named constants).
