# UI consistency — the rules every module screen follows, and why

## What this is about

A webtyp application is a shell (`platformd`: header, rail, drawer) around **modules**
(`veltylabs/modules/*`), and almost every module is, in practice, a **form** plus a
**list**: you pick a record from the list on the right and edit its fields on the left
(`crudview`). This document records how those two pieces must look so that moving from one
module to another feels like the same product, and the reasoning behind each rule.

## When you will run into it

You are styling a `css.go`, adding a field to a model, writing a list row, or reviewing a
screenshot and something "feels off" — a field that looks different from the one in another
module, a focus ring in a different colour, a tag that looks like a button. The short
version of every rule is in the skill **ui-consistency**; this page holds the background.

## The boundary: who owns what

- **Fields → `form`.** A module declares a model; `form.New(model)` draws it. A module never
  writes its own `<input>` or its own field stylesheet.
- **Module title → `platformd`.** The shell shows the active module's name; the module only
  supplies its content.
- **Colours → the theme.** A widget says *which role* a colour plays ("selected", "primary
  action"); the theme decides the value. Nothing in markup writes a colour.
- A module's own pieces that nobody else reuses (a floor plan, a schedule editor) are styled
  inside the module. There is no build-time block on that: the consistency boundary is only
  the three lines above.

## Form fields

```
 Full name *                 ← label: text, translated, never truncated
┌──────────────────────────┐
│ Dr. Ana Rojas            │ ← control: full width of its column, ≥ 44 px tall
└──────────────────────────┘
 Format: 12.345.678-9        ← help (only if the model declares Help)
 ⚠ Invalid RUT               ← error; replaces the help while it exists
```

- **Label above the control**, same on phone and desktop. Floating or on-the-border labels
  were removed: the "chip on the border" truncated names to 14 bytes and hid the field's
  own text. Label-above is the format with the best completion speed and the fewest
  ambiguities in the research from Baymard, NN/g, GOV.UK and Polaris.
- **One column on a phone, at most two on a desktop**, decided by the form's own width
  (`style.Grid(2, style.ColumnMedium, …)`: two columns once two 20 rem tracks fit). A single
  capped column (it was tried) leaves most of a desktop stage empty and forces scrolling;
  a fixed-width one overflows a phone. Textarea and submit span both columns.
- **Validate when the person leaves the field and when they submit**, never on every
  keystroke.
- **Every field with a widget declares `Label`** (an English key, translated). `form` raises
  a development diagnostic when it is missing instead of showing the technical field name.
- **A lone checkbox carries its text once**, inside the option, with no label above (it would
  repeat it), and takes the whole row so it never leaves a gap beside a neighbour that has a label.
- **A select with nothing chosen shows a prompt** ("Select an option") instead of presenting its
  first option as if it had been picked; a required field cannot go back to the prompt.
- **Saving a loaded record keeps what the form does not show.** A field with no widget (a tenant, a
  timestamp, a position) is remembered when the record is loaded and written back on sync; without it the
  scratch record the form syncs into carried those fields blank and the save was rejected.
- **Accessible names are the visible label.** `<label for>` ties each control to its label; a radio
  group (a `div`) names itself with `role="radiogroup"` + `aria-labelledby`; a title equal to the
  generated field name is never rendered as a tooltip.
- **`form` writes no colour.** The checkbox is a CSS box, not drawn colours, so the theme
  reaches it. A test in `form/tests` fails if a renderer writes a `--color-` token.
- A `Radio` is drawn as chips (touch-sized boxes in a wrapping row), the chosen one filled.

## The module's title belongs to the shell

`platformd`'s header carries the page's single `<h1>`: the module's name (inside a group, the active
child's name) in the large size a module title always had, beside the app's **mark**; on a phone it is
centred and smaller. The app's name and version are not in the header — they compete with the module's
name — but one tap away, in a small panel the mark opens (the user menu's `<details>` pattern, no script);
the same rule on every device: where I am, large, always; which app and version, on demand. The header is the primary
surface at every width, with on-primary text; the user menu trigger reads on it. `rightpanel` draws
no title band, so the form starts at the top of the frame, and the controls that used to sit beside the
title moved: a control that picks **which records the list shows** is `crudview`'s `Scope`, in the list
header above the search box (never wired to the filter); a control that is a **value of the record** is
a form field. Inside content only `<h2>` sections exist, through `style.Heading()`; no component fixes
a title's size or weight. Four competing title hierarchies (a gradient `<h1>` band, `<h3>` in the
accordion, loose `<h3>`/`<h1>` in modules) were reduced to this one.

## Colour roles

The primary colour is a **pair**: `--color-primary` fading into `--color-primary-gradient`
(a gradient by default; a solid if the app calls `css.ClearGradient`). The accent is a single
amber, `--color-accent` (`#e8a33d`) with `--color-on-accent`, with **one value in the light
and the dark theme**.

| Role | Colour | Examples |
|---|---|---|
| Brand and **actions** | primary (gradient) | action buttons, header |
| **Selected** and **focused** | amber accent | chosen row, chosen chip, checked box, current nav item, focus ring |
| **Data** (codes, counts, tags) | neutral | the `MD001` code on a list row |

- **Focus and selection are one family, amber** — and never a second colour inside the same
  indicator. The ring is the global 2 px inset outline in the accent; the label of a focused
  field does not change colour (amber text on the light card measures about 1.7:1, below
  WCAG 1.4.3).
- **Outline for a passing state, fill for a confirmed one.** Selected and pressed are an amber
  *fill*; hover is a 1 px amber *outline* and focus a 2 px one, so a hover never paints the
  row's content and the three states differ by weight.
- **Recipes.** The outline is `style.AccentOutline`, a surface that draws an inset 1 px ring
  in the accent and repaints nothing (no background, no text colour), so a hovered row keeps its
  own fill and label. The select's option list is `style.Picker()` on a part that carries
  `widget.PartSelect`. Selected controls (`chipgrid`, `cellgrid`, the form's chips and checkbox,
  a schedule's selected day) are `As(Accent)`.
- **Data is neutral.** A tag drawn with the primary fill is indistinguishable from an action
  button. Tags are outlined or tonal. In a list row the code rides the row's bottom-right edge,
  out of the flow: the label always has the row's whole width and the tag never moves (inline in
  the flow it wrapped under the label whenever the two did not fit side by side).
- **The native `<select>` list** is drawn by the browser and highlights in the system blue; CSS
  cannot recolour it. Where `appearance: base-select` exists (Chrome/Edge ≥ 135) the options
  are styled in amber; elsewhere the native list stays.

### Prior art for the colour rules

- **GOV.UK** — focus is its own reserved colour (yellow `#ffdd00` with a black border),
  separate from the brand blue, chosen for contrast on any background.
- **IBM Carbon** — focus `$focus` is the interactive blue, paired with a white inset
  (`$focus-inset`) for contrast; *selected* layers are neutral greys, not the focus colour.
- **Material 3 (Angular / Flutter)** — focus is a state layer that changes with selection.
- **WCAG 2.2** — 1.4.11 asks 3:1 against adjacent colours; 2.4.13 asks an indicator of at
  least 2 px.
- Common to all: focus and selection never share a role of colour *by accident*, and the
  indicator carries its own contrast.

### Tried and dropped

A gradient focus ring (`style.Ring`: a gradient border, padding taken back so the box never
resizes) and gradient text (`Glyph(Primary)` clipped to the text) were built and verified,
then dropped when focus became amber. Two facts worth keeping: a CSS `outline` cannot carry
a gradient, and an indicator the same colour as a filled primary control disappears on it —
the reason focus and *primary* must not be the same colour.

## Messages (toasts) and the phone's list/detail strip

- **A message is the inverse surface** (`style.Inverse`: body ink as the fill, the page colour as the
  text), the same on every device and severity (severity travels in `role="status"`/`"alert"`, never in
  colour). A primary-filled toast vanished on the primary header; a text-only one in the desktop header
  went unnoticed. The inverse reads over a primary header, white content and the dark theme alike.
- **On a phone it sits below the header**, not over it (`Docked` + `DockedClear`): over the header it
  covered the menu button.
- **It stays long enough to be read**: `platformd.Auto()` is 2.5 s + 350 ms per word, floored at 4 s and
  capped at 10 s (it was 1.2 s and a 2 s floor).
- **Errors stay until dismissed** (WCAG 2.2.1): an error whose length was left to `Auto()` is
  persistent, an explicit `For(ms)` is respected, and an identical message already on screen is not
  stacked again (a failing auto-save fires once per field commit).
- **An error names the action, then the cause**: "Could not save: tenant_id required". The bare cause
  names a field and no action.
- **On a phone the form takes the strip's whole width.** The list/detail strip used to leave a sliver of the
  list beside the form "to show where you came from"; it only confused and cost 10 % of a narrow screen.
  The way back is the footer's undo button or a swipe (`MasterDetail(Full)`).

## Related

- Skill **ui-consistency** — the rules in checklist form.
- Skill **widget-styling** — how a `css.go` is written.
- `form/README.md` — the form's API.
