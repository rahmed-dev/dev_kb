# dev_kb — Rules

Personal engineering knowledge base. Durable lessons, not project state.

Read this file before adding or editing anything.

## The one rule that matters

**A note earns its place by being reusable on a project that does not exist yet.**

If a note only makes sense while you remember a particular sprint, ticket, or client, it is not
knowledge — it is status. Status belongs in the project repo, not here.

## Generic by default

Notes are written **framework-agnostic**. The rule is the knowledge; the framework is an accident of
where you happened to learn it.

- Write "keep entry points thin", not "keep `validate()` thin".
- Write "schema-driven endpoint", not "the `get_data(doctype)` endpoint".
- Name a framework only inside **Evidence**, where it is a citation rather than the subject.

The exception is `case-studies/` — those are teardowns of real, named codebases, and being specific is
the entire point. Specifics live there and are referenced from generic notes.

If a lesson genuinely cannot be stated without naming one framework, it is a framework quirk. Those go
in `case-studies/` too, under the codebase or vendor that owns the quirk.

## What does not belong

- Project status, roadmaps, TODOs, feature specs
- Anything a customer or employer would not want published — this repo is on a public host
- Credentials, tokens, hostnames, connection strings, DB dumps, customer names, real data samples
- Copy-pasted vendor documentation. Link it, then write what the docs failed to tell you

## Layout

```
AGENTS.md              rules of the KB (this file)
README.md              index — one line per note
backend/
  architecture/        server-side system shape: boundaries, layers, contracts, data flow
  coding/              server-side craft: structure, naming, idiom, style
frontend/
  architecture/        client-side system shape: rendering strategy, state boundaries, data layer
  coding/              client-side craft: component conventions, style, structure
practices/             cross-cutting: testing, extensibility, docs, workflow
case-studies/          teardowns of real codebases — the evidence generic notes cite
```

### Where does it go?

| The knowledge is about | Put it in |
|---|---|
| How to *shape* a system — what talks to what, where a boundary sits | `*/architecture/` |
| How to *write* it — structure, naming, idiom, size, style | `*/coding/` |
| Runs on a server, in a job, against a database | `backend/` |
| Runs in a browser or a build step | `frontend/` |
| Applies to both sides, or to neither — process, testing, docs | `practices/` |
| Observed while reading someone else's codebase | `case-studies/` |

Two plausible homes means the note is two notes. Split it.

## Note format

Every note follows this shape. No exceptions, because the shape is what makes the KB searchable.

```markdown
# <the claim, stated as a title>

**Rule** — one imperative sentence. What to do.

**Context** — when this applies, and explicitly when it does not.

**Why** — the mechanism. Not "it is cleaner" — what actually goes wrong without it.

**Evidence** — measurement, `file:line`, or the incident that taught it.
```

### Evidence is mandatory

A note without evidence is an opinion, and opinions rot silently.

Evidence is a measured number, a real citation, or a described failure. "Best practice" and "everyone
knows" are not evidence. If you have no evidence yet, the note is not ready — keep it in your head.

### Context must include the negative case

A rule that always applies is either trivial or wrong. Every rule has a boundary; write it down. The
boundary is what makes the note safe to follow later, when you have forgotten why you wrote it.

## Naming

- Filenames are kebab-case and state the topic, not the verdict: `where-business-logic-lives.md`.
- One note, one idea. A note that needs two `#` headings is two notes.
- Aim under 80 lines. A note you will not re-read is a note that does not work.

## Maintenance

- **Edit in place.** No changelogs, no "UPDATE 2026-…" sections, no dated appendices. Git holds history.
- **Contradiction is a bug.** When a new lesson conflicts with an existing note, rewrite the existing
  note. Never leave both standing for a future reader to arbitrate.
- **Delete what proved wrong.** A wrong note is worse than a missing one — it will be trusted.
- **Update `README.md`** whenever a note is added, renamed, or removed. One line per note.

## Commits

`add:` / `update:` / `delete:` / `docs:` followed by the note topic.

One commit per note. A commit touching five notes cannot be reverted usefully.
