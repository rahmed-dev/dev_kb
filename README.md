# dev_kb

Personal engineering knowledge base. Durable lessons, not project state.

Rules for writing and maintaining notes: [AGENTS.md](./AGENTS.md). Read it before adding anything.

Notes are framework-agnostic. Framework specifics live in `case-studies/`, cited as evidence.

## Backend

### Architecture
- [Where business logic lives](./backend/architecture/where-business-logic-lives.md) — fat entity vs
  service modules; the deciding variable is how many outside worlds you talk to.
- [Schema-driven vs purpose-built endpoints](./backend/architecture/schema-driven-vs-purpose-built-endpoints.md)
  — schema-driven only where a screen maps 1:1 onto one entity; verbs never.

### Coding
- [Thin entry points](./backend/coding/thin-entry-points.md) — anything the framework calls lists
  steps and implements none of them.

## Frontend

### Architecture
- [Metadata-driven rendering](./frontend/architecture/metadata-driven-rendering.md) — descriptor
  (~100 lines) vs schema (~1600); the second is a form engine, and you arrive at it by accident.

### Coding
- [Component conventions](./frontend/coding/component-conventions.md) — one authoring style, one state
  home, one data layer; enforced by lint, not review.

## Practices
- [Extension seams](./practices/extension-seams.md) — build your own features on the seam you publish;
  never resolve a caller-supplied path.

## Case studies
- [Frappe CRM](./case-studies/frappe-crm.md) — v1.79.1. Fat records, schema-driven reads, a dogfooded
  scripting seam, a 1586-line form engine.
