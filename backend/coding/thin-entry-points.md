# Thin entry points

**Rule** — An entry point lists what happens. It does not implement any of it.

**Context** — Applies to every function the framework calls rather than you: lifecycle hooks,
request handlers, job bodies, event listeners, CLI commands, message consumers.

The body is a sequence of named calls and nothing else. Each call names a step in the vocabulary of
the domain. Conditionals that decide *whether* a step runs may stay; the step's own logic may not.

```
def on_save(self):
    self.set_defaults()
    self.validate_status()
    if self.owner_changed():
        self.reassign()
```

Read it once, know everything that happens on save. Read one method, know how one step works.

**Context where it does not apply** — a genuinely single-step entry point. Wrapping a one-line handler
in a one-line private method buys nothing and costs a jump.

**Why** — Entry points are the only place a reader can discover *what the framework does on their
behalf*, and there is usually no other index of it. When logic is inlined, that index is destroyed:
finding out what happens on save means reading 200 lines and holding the whole thing in your head.

The second cost is testing. Framework entry points drag in the framework — persistence, request
context, session. A named step is a plain method you can call directly. Inlined logic can only be
reached by triggering the whole lifecycle, so it tends to end up untested.

The third is diffs. When each step is a method, a behaviour change touches one method and the diff
states which behaviour changed. When it is inlined, every change is a hunk in the middle of one long
function and the diff says nothing.

**Evidence** — A CRM's deal entity, 2026-07-23: `before_validate` is one line, `before_save` is one
line, `validate` is nine calls with two conditionals. Twenty methods on the class, none over ~25
lines. The class is 472 lines and stays readable because the entry points are an index into it.

The same pattern holds in that codebase's inbox counterpart, where entry points delegate to private
methods with a leading underscore — a naming convention worth copying, since it marks at the call site
that the method is a step and not an API.
