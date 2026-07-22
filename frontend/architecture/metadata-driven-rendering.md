# Metadata-driven rendering

**Rule** — Let the server describe read-only surfaces. Do not let it describe editable ones until you
are willing to own a form engine.

**Context** — "The client should not hardcode field names" is right, but it hides two very different
commitments. Pick deliberately.

**Level 1 — descriptor.** The server returns display shape, not field names:

```json
{ "title": "Subscription",
  "rows": [{ "label": "Plan", "value": "Fibre 500", "type": "text" }],
  "actions": [{ "id": "cancel", "label": "Cancel" }] }
```

The client is one dumb renderer over a small fixed vocabulary of value types. Read-only. Roughly a
hundred lines. Any team can add a panel without touching the client or rebuilding it.

**Level 2 — schema.** The server returns the model's schema and the client renders *inputs*: field
types, layout, conditional visibility, option sources, validation, dirty state, save. This is a form
engine. You are rebuilding what the underlying framework already has, in a second language.

Level 1 covers panels, summaries, detail views, anything a user reads. Level 2 is only worth it when
**editing arbitrary records is the product**. If your app edits three known forms, hand-write three
forms — they will be better and cost less.

**Why** — Both levels remove the hardcoded field names, so both look equivalent in a design
discussion. They are not: Level 1's cost is bounded by a fixed vocabulary you control, while Level 2's
cost tracks the schema system's full feature set forever. Every conditional-visibility rule, every
filtered picker, every validation mode has to be re-implemented and re-tested client-side, and each
one arrives as a bug report rather than a plan.

The trap is arriving at Level 2 by accident — shipping Level 1, then adding "just one editable field",
then a dropdown that must filter by another field, then a field that hides conditionally. Each step is
small and none is refusable on its own.

**Guard** — cap it explicitly and write the cap down: *surfaces that take no input render inline;
anything that takes input opens the screen that owns it.* A rule at the boundary is the only thing
that stops the slide, because no individual request will ever look like the one that crosses it.

**A fixed contract is not hardcoding.** Level 1 fixes a *shape* — sections, rows, actions — that both
sides agree on. Hardcoding is field *names* baked into a component. Confusing the two leads to chasing
a structureless API, which just moves the coupling somewhere harder to see.

**Evidence** — Measured on a mature open-source CRM, 2026-07-23. Its Level 2 stack: field renderer 617
lines, layout 111, section 41, column 20, transforms and mandatory-checking 175, conditional-visibility
expression evaluator 77, schema store 181, dialog shell 302, dialog API 62 — **1586 lines**, excluding
the input controls themselves. That is the real price, and it buys something genuine there, because
editing arbitrary records *is* that product.

Related: `backend/architecture/schema-driven-vs-purpose-built-endpoints.md`.
