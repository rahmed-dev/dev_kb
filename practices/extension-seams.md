# Extension seams

**Rule** — Build your own features on the extension point you publish. A seam you do not use yourself
is already broken.

**Context** — Applies whenever something outside your codebase must add behaviour to it: plugins,
another team's module, a second app in the same deployment, customer scripting.

Three properties make a seam survive:

1. **Same API, several sources.** Behaviour loaded from your own source tree and behaviour supplied by
   a third party run through one pipeline. Not a parallel path that happens to look similar.
2. **You are a consumer.** At least one shipped feature is implemented through the seam, so breaking
   it breaks your product and your tests, not just someone else's.
3. **Registration is not free-form.** Callers name a registered capability; they never hand you a path,
   a symbol, or code to resolve.

**Why** — A seam used only by outsiders has no feedback loop. It has no tests you care about, its
breakages surface as someone else's bug report weeks later, and its awkward parts are never felt by
anyone able to fix them. It rots from the day it ships, quietly, while continuing to exist.

Dogfooding converts all of that into ordinary maintenance: the seam is on your critical path, so it is
exercised by your own suite and its ergonomics are your problem.

**Security — resolving what the caller sends**

Never turn a caller-supplied string into code. Taking a path, module, symbol, or class name from a
request and resolving it is remote code execution, whatever the surrounding framework is called.

The safe shape: the caller sends an **identifier**, the server looks it up in a registry populated at
startup by installed code, and anything absent from the registry is rejected. The registry — not the
request — decides what can run.

Two more that get skipped:

- **Extensions run with the caller's permissions.** A blanket permission bypass inside a seam silently
  grants every extension's data to every user who can reach it. A user lacking access must see the
  panel absent, not an error and not the data.
- **Stored code is code.** If extensions are strings evaluated at runtime, whoever can write that
  storage executes arbitrary code in every user's session. That is an administrative privilege, never
  an end-user one, and it should be reviewed like a deployment.

**Evidence** — A mature open-source CRM, 2026-07-23. Extensions are stored records containing a class,
evaluated at runtime; the published contract lets them add header actions, inject rendered content,
and open dialogs. Its *own* built-in behaviours are written as the same class shape, loaded from a
directory by convention and merged into the same pipeline as the stored ones. Both sources, one path.
The seam cannot silently rot, because the vendor's own features ride on it.

Its cost is the one above: evaluated stored code is arbitrary execution in every user's browser, which
is only acceptable because authoring is administrator-restricted.
