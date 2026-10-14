# Roadmap — Source selection wave: open items
> **Updated:** 2026-10-07 · master: [SOURCE_SELECTION_MASTER_PLAN.md](SOURCE_SELECTION_MASTER_PLAN.md)

One section per item. Work on one at a time; when it changes, update its **State** line and append
a line to its **Log**. A finished item moves to the master's phase table and is deleted here.

Order: **H1 → F publish** (F's cascade would break mjosefa-cms while `config/lang.go` imports
`webtyp.com/fmt/lang`). H2 and H3 can go in any order after H1. The rest is independent.

| # | Item | Repo(s) | Mode | State |
|---|---|---|---|---|
| F | fmt v1.1.0 (minimal layer) | webtyp/fmt | local | ✅ done, tag pending H1 |
| H1 | Project translations as data | mjosefa-cms | local | ⏳ next |
| H2 | `Label`/`Help` and English keys in modules | veltylabs/modules/* | local | ⏳ after H1 |
| H3 | appointment_booking seed test | veltylabs/modules/appointment_booking | local | ⏳ |
| X1 | crudp: broken tests, unpublished commit | webtyp/crudp | plan needed | ⛔ blocked |
| X2 | sitetheme · client-monjitaschillan-platform | veltylabs | maintainer | ⏸ parked |
| X3 | components tests → `tests/` (~75 files) | webtyp/components | plan needed | ⏸ |
| X4 | Browser check of the locator | app + mjosefa-cms | manual | ⏳ |
| L1 | Login UI + selection fixes (theme, form grid, reload numbering, SVG target, shared-identity locator) | css, dom, widget, form, components, layout, devbrowser, app, mjosefa-cms | local | ✅ published; mjosefa-cms uncommitted |
| L2 | Login wasm size, phase 0b (measure only) | app-releases docs | local | ✅ measured — `CLIENT_PAYLOAD_STAGING_MASTER_PLAN.md` §9; owner decides the split |
| L3 | «Restore pages?» bubble on every dev-browser start | devbrowser | codejob PR #16 + review fixes | ✅ v0.6.5 |

---

## F — fmt v1.1.0

**State:** ✅ executed locally, commit `a940153` on `main` **without a tag**. Publish with
`gopush` (tag `v1.1.0`) right after H1, or the cascade breaks mjosefa-cms.

- [x] Stages 1–5; `gotest` green (vet, race, WASM); acceptance greps empty; one justified root test.
- [ ] After H1: workspace grep shows only X2 repos → `gopush '…' v1.1.0`; watch the cascade.

**Log**
- 2026-10-07 — workspace precheck: remaining users are mjosefa-cms `config/lang.go` (H1),
  sitetheme `jsonld.go` and platform `help.go` (X2); `webtyp/model/permitted.go` mentions
  `EscapeHTML` in a comment only.
- 2026-10-07 — session finished without a PR asking two things (`Sprintf("%q", 123)` returned
  ""; Thousands/HasUpperPrefix in `benchmark/reporter.go`). Answered with `codejob --reply`
  (devflow v0.4.124): fix `%q` for non-strings to quote the string form (old `.Quote()` behavior,
  documented as differing from Go); `quote_test.go` stays in `tests/` via `Sprintf("%q")`; drop the
  two calls from the reporter; finish and open the PR. Session resumed (working).
- 2026-10-07 — the session then vanished (API 404, no PR, no branch). Plan run locally instead,
  with those same answers; notes under "Executor notes" in `fmt/docs/LAST_PLAN_EXECUTED.md`.

## H1 — mjosefa-cms: translations as data

**State:** ⏳ next. Local (private). No plan file needed: the steps are below.

- [ ] Delete `config/lang.go` (~76 entries), including `OutLang(lang.ES)`; `"default": "es"` replaces it.
- [ ] Start the daemon so `config/lang.json` is generated; move the existing ES values into it,
      each key exactly as written in the code (`Pending confirmation` is one key; crudview words are
      separate keys).
- [ ] Fill the rest with `lang_missing`.
- [ ] Spanish hard-coded in mjosefa-cms → English keys + translation: `SetTitle("Clave")`,
      `SetTitle("RUT")`, any presenter `Title`/`SearchPlaceholder` in Spanish ("Equipos" → "Devices").
- [ ] Known effect, by rule: a browser in English now sees English (`default` applies only when
      the browser's language is not translated).
- [ ] `grep -rn "webtyp.com/fmt/lang\|RegisterWords" --include='*.go' .` → empty; gopush.

**Log**
- 2026-10-07 — repo clean, nothing started.

## H2 — veltylabs modules: `Label`, `Help`, English keys

**State:** ⏳ after H1. Local, one repo at a time + gopush.

Modules: device_manager (first), appointment_booking, business_calendar, chat_room,
clinical_encounter, editorial, item_catalog, patient_directory, provider_payouts, room_layout,
staff_manager, work_schedule.

Per module:
- [ ] `Label` (and `Help` where a field needs instructions) on every `model.Definition` field,
      English (`ip` → "IP address", `is_active` → "Active", …). Never another language there.
- [ ] Spanish literals → English keys, translated in the project's `lang.json`:
      room_layout (`stepindicator` steps "Planta"…, `segmentedcontrol` "Planificar"/"Ahora"…),
      clinical_encounter (`Placeholder: "Seleccione un paciente…"`).
- [ ] Presenters implementing `SearchPlaceholder() lang.Text` (view v0.6.22) compile.
- [ ] `langc sync` in the module → its `lang.json`.

**Log**
- 2026-10-07 — no module has `Label` yet (device_manager checked).

## H3 — appointment_booking seed test

**State:** ⏳ one local commit (the `webtyp.com/lang` migration) not pushed.
`TestSeed_LoadPopulatesServicesAndReservations` fails before and after that commit: the 2nd demo
reservation at 09:30 hits "establishment is closed on that date". The red test exists.

- [ ] Fix the seed data (or the calendar it seeds), test green, gopush.

## X1 — crudp

**State:** ⛔ one local commit (`Html` → `Sprintf`, lang/msgtype imports) not published. Its tests
have not compiled since the json typed codec (pre-existing, about a month), and
`example/modules/user/front.go` (wasm) uses removed `dom.Reference` methods. No consumer in the
workspace, so it does not block F. Needs its own `docs/PLAN.md`.

## X2 — sitetheme · client-monjitaschillan-platform

**State:** ⏸ parked by the maintainer. Both have uncommitted work that is not ours. They pin the
old fmt; when they move to fmt v1.1.0: sitetheme `jsonld.go` `fmt.JSONEscape(s, b)` →
`escape.JSON(b, s)`; platform `modules/help/help.go` `fmt.Html(` → `fmt.Sprintf(`.

## X3 — components tests to `tests/`

**State:** ⏸ ~75 tests inside package directories. Move to `tests/` or justify at the root
(skill **testing**). Separate plan.

## X4 — browser check of the locator

**State:** ⏳ the programmatic e2e passed (2026-10-07, mjosefa-cms, `composebar__input` →
render/style/declaration in `~/Dev/Project/webtyp/components`, origin "local checkout; project pins
v0.8.12"). Pending: Alt+click a composebar input in the running app and read
`browser_get_selected_element`.
