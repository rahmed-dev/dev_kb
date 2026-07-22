# Where business logic lives

**Rule** — Put behaviour on the entity when the app has few entities and one dominant process. Put it
in service modules when the app has few entities but many external protocols.

**Context** — This is the fat-model vs thin-model question, and neither answer is universal. The
deciding variable is not the size of the codebase — it is **how many different outside worlds the code
talks to**.

Choose the entity (fat model) when:
- The app is *about* its records. The record is the product.
- One process dominates: create it, move it through states, close it.
- Logic reads and writes mostly the entity's own fields.

Choose service modules (thin model) when:
- Several external systems each speak a different dialect, and each needs its own adapter.
- A single operation spans multiple entities, so no one entity is its natural owner.
- The same operation must run from several triggers — a request, a webhook, a scheduled job.

**Why** — Fat models fail when adapters accumulate: each new protocol adds methods to a class that has
nothing to do with that protocol, and the class becomes a junk drawer that cannot be tested without
the whole persistence layer.

Thin models fail the opposite way: with one dominant process and no external systems, service modules
add a hop with no boundary behind it. You navigate two files to read one behaviour, and the entity
becomes an anaemic bag of fields that no longer enforces its own invariants.

The failure mode of guessing wrong is not a crash — it is a slow rise in the cost of every change,
which is exactly the kind of cost nobody attributes to the original decision.

**Evidence** — Two apps measured side by side, 2026-07-23.

A CRM (~15k backend LOC): its two core entities carry the behaviour. The lead entity is 545 lines with
`create_contact`, `create_organization`, `convert_to_deal` as methods. Two entities, one funnel, no
protocol adapters. Fat model, and correct — the record *is* the product.

An omnichannel inbox (~5k backend LOC): entity classes hold 3–4 methods each and do almost nothing.
Behaviour lives in two package trees — one per-protocol adapter set (four adapters, 300–640 lines
each) and one protocol-agnostic core (ingest, routing, delivery, notification). Thin model, and
correct — four outside worlds means four adapters, and none of them belongs on a record class.

Same framework, same team, opposite structures. Both defensible, because the deciding variable differed.

See `case-studies/frappe-crm.md` for the full teardown.
