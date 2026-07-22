# Schema-driven vs purpose-built endpoints

**Rule** — Make an endpoint schema-driven when the client renders whatever the server describes. Keep
it purpose-built when the client's screen is a composition the server does not model.

**Context** — A schema-driven endpoint takes the *thing* as a parameter and returns both data and its
shape: `get_records(entity, filters)`, `get_fields(entity)`. One endpoint serves every screen of that
kind, and adding a field to the model makes it appear in the UI with no client change.

That only works when **the screen maps 1:1 onto a stored entity** — a list of one thing, a form of one
thing, a board of one thing.

It does not work when the screen is a *composition*. A view that merges four sources and derives
display state has no single entity to parameterise on. Forcing it produces an endpoint with a growing
pile of flags, which is a purpose-built endpoint wearing a costume.

Verbs are never schema-driven. `send_reply`, `mark_read`, `refund` take an action's arguments and have
no schema to describe. Do not try.

So the split inside one app is usually:
- **Panels, lists, forms, detail views over one entity** → schema-driven
- **Composed screens and all verbs** → purpose-built

**Why** — Purpose-built read endpoints hardcode field names, and the client hardcodes them again to
render. One added field means a server change, a client change, and a redeployed client bundle. The
second hardcoding is the expensive one: it puts UI changes behind a build, and it makes it impossible
for another team to contribute a panel without editing your client.

Schema-driven endpoints cost you a render layer and the discipline to keep it thin. That price is
worth paying exactly where the 1:1 mapping holds, and is a bad trade everywhere else.

**Evidence** — Measured 2026-07-23.

A CRM ships one 837-line module whose read endpoints all take the entity name as a parameter. Its
client renders any entity's list, board, and form without knowing a single field name — which is why a
user-defined custom field shows up in the UI with no code change at all.

An inbox app in the same bench ships 22 purpose-built endpoints. Most are verbs and correctly stay
that way. But its contact panel returns four fixed keys and the client renders four fixed rows — the
same four names hardcoded twice. Adding a field there requires editing both sides plus a bundle
rebuild, and no second app can contribute a panel at all.

Related: `frontend/architecture/metadata-driven-rendering.md` — the client half of this decision, and
the one that carries the real cost.
