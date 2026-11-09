---
DRAFT: A6.11 — external effects. NOT a dispatchable plan (no codejob frontmatter): it exists so the owner can judge the code before it is a plan.
---

# Draft — external effects orchestrated by `crud`

Decision D36. A rule only DECIDES which effects it wants; `crud` stores the intent with the record, runs it, retries it and recovers it.

## What I found in `outbox` (changes the design)

`outbox.Enqueue(e Entry)` writes through **its own** `*orm.DB`, outside the transaction `crud` opens for the record. So "save the record and the intent in the same
transaction" (the property that closes the loss window) is **impossible with `outbox` as it is**: a crash between the two writes loses the call.
Also: `Deliver(Sender)` blocks, is one-tab by design, and keeps only `Op` + `Payload` text. It is a browser queue, not a server executor (**not verified on a server**).

Recommendation (owner decides, question E2): `crud` owns a small model `EffectIntent` and writes it with the same `crud.Tx` it already has (`tx.Create(&EffectIntent{…})`).
`outbox` is not touched and does not become a server library. Cost: retry/backoff logic lives in `crud` (≈ 80 lines); it is not copied from `outbox`, it is written once here.

## The pieces, as code

**1. Declaration (`definition.go`)** — an effect belongs to the entity that has it, so it is declared **inside that entity's definition**, next to its `Fields`; the name is written once, there, and limits are written out (no hidden defaults):

```go
var PaymentModel = model.Definition{
	Name:  "payment",
	Allow: model.Create | model.Read,
	Rules: model.Rules{Create: model.AddMyBusinessRule, Read: model.HaveNoBusinessRule},
	Effects: []model.EffectDefinition{
		{Name: "charge", Mode: model.WaitForReply, MaxWait: 10 * time.Second},
		{Name: "receipt", Mode: model.RetryInBackground, MaxAttempts: 8},
	},
	Fields: …,
}
```
Same precedent as `Fields`: a field name is written once in `Fields` and the code uses the generated `Payment_.PayURL`, never a string. Here `definitionc` generates one type per effect that answers `ModelName()`
(`type PaymentCharge struct{}`, `ModelName()` = `"payment.charge"`), and `crud.Effect` is the interface `interface{ ModelName() string }`.
The rule, the table in the database and the executor use the type or `ModelName()`; the module's `Definition` does not change (no `Effects` list there).

**2. The rule decides** (`rule_payment_create.go`, written once by `definitionc`; the body is yours):

```go
func (p *Payment) Create(a crud.CurrentUser, tx crud.Tx, m *Module) (crud.RuleAnswer, error) {
	p.Status = StatusPending
	return crud.ApplyAndPublish(PaymentCharge{}, PaymentReceipt{}), nil   // store, publish, and run these two effects
}
```
`crud.ApplyAndPublish(effects ...crud.Effect)` and `crud.ApplyWithoutPublishing(effects ...crud.Effect)` become **functions** (no effects: `crud.ApplyAndPublish()`). One form for "no effect" and "some effects". **Decision E1** (it touches every plan that writes `return crud.ApplyAndPublish, nil`).

**3. The effect says WHAT to call** (`effect_payment_charge.go`, a skeleton that does not compile until completed, like `rule_*.go`):

```go
func (p *Payment) Charge(a crud.CurrentUser, m *Module, key crud.IdempotencyKey) (crud.FieldsToSave, error) {
	// COMPLETA TU EFECTO AQUÍ (cobrar payment) y termina con: return fields, failure
	var fields crud.FieldsToSave // los campos que crud guarda después de la respuesta: crud.Fields(Payment_.PayURL)
	var failure error            // el fallo del proveedor; crud lo reintenta según el modo
}
```
`key` is the record id: the provider receives the same key on every retry, so a retried charge is never charged twice.

**4. Webhook / return of the customer** — the provider adapter (mounted by the app, D27) authenticates the provider and calls a Go entry point, never a public route:

```go
func (a *Adapter) Return(w http.ResponseWriter, r *http.Request) {
	err := crud.ConfirmEffect(r.Context(), a.payments, id, PaymentSettle{}, crud.Evidence{Token: token})
}
```
and the module's rule asks the provider again (never trusts the request): `func (p *Payment) ConfirmSettle(a crud.CurrentUser, ev crud.Evidence, m *Module) (crud.RuleAnswer, error)`.

**5. What `crud` stores** (`effect_intent.go`, in `crud`): `EffectIntent{Id, Module, Entity, RecordId, Effect, State (pending|done|failed), Attempts, NextAttemptAt, LastError}`.
The record gets its state from the effect result (`done`/`failed`); a failed effect never deletes the record.

**6. The executor** (`appkit` starts it, like the other background pieces): `crud.RunEffects(ctx, store, mods…)` polls due intents, calls the module's effect with the id as key, applies the saved fields, schedules the next attempt with backoff or marks `failed` after `MaxAttempts`.
`WaitForReply` runs the effect inside the request and waits up to `MaxWait`; if it times out the intent stays `pending` and the executor takes over (no lost charge).

## Flow (diagram to write in `docs/diagrams/EFFECTS_FLOW.md` when this becomes a plan)

rule answers `ApplyAndPublish(effects…)` → crud: record + `EffectIntent` rows in ONE transaction → commit → publish event →
`WaitForReply`: run now, wait `MaxWait` · `RetryInBackground`: answer now, executor retries · `WaitForConfirmation`: stays `pending` until `crud.ConfirmEffect`.

## Questions for the owner

- **E1** — `crud.ApplyAndPublish(effects…)` as a function (one form, effects optional) instead of a constant. Recommended: yes. It changes `RuleAnswer` from `uint8` to a small struct and every skeleton comment/return in the plans already written.
- **E2** — `EffectIntent` owned by `crud` and written with `crud.Tx` (recommended), or extend `outbox` to accept the caller's transaction (couples `outbox` to `crud`'s port and keeps its browser-only one-tab rule).
- **E3** — names: `crud.Effect`, `crud.IdempotencyKey`, `crud.FieldsToSave`, `crud.Evidence`, `crud.ConfirmEffect`, `crud.RunEffects`, `model.EffectDefinition`, `model.EffectMode`. Do they pass "no need to read docs"?

## Not decided here

Exact stages, tests and the `appkit` wiring: written only after E1–E3. Verify first: that `crud.Tx` can create a row of a model `crud` itself owns (it needs a table created by `migrate`), and the WASM size of `model.EffectDefinition`.
