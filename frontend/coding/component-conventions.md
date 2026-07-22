# Component conventions

**Rule** — Pick one component authoring style and one state mechanism per app, and let the linter, not
review, enforce them.

**Context** — Most component frameworks ship more than one way to declare a component, and more than
one place to keep state. Both choices are near-arbitrary in isolation and expensive when mixed.

Decide once, at the start:

1. **Authoring style** — whichever the framework's current docs and ecosystem default to. Being on the
   documented path is worth more than any ergonomic argument between the styles.
2. **Shared state home** — one of: a state library, a set of composable/hook modules, or plain
   singleton modules. All three work. Two of them in the same app do not.
3. **Data fetching** — one layer that owns requests, caching, and loading state. Components call it;
   components do not call the transport directly.

Codify each in lint config the day you decide. A convention that lives only in review is a convention
that drifts, because review catches new files and never catches the file someone edited.

**Why** — Mixed styles impose a tax on every reader: before changing a component you must first work
out which world it lives in, and idioms do not transfer between them. Worse, shared utilities have to
either support both or be written twice, which is where the mixing becomes permanent.

Drift is not a decision, but it is indistinguishable from one after the fact. A reader cannot tell
whether the older style is deliberate — legacy kept on purpose — or simply the parts nobody has
touched. So nobody migrates, and both persist.

**Size** — watch the distribution, not the average. A healthy app has a low median and a short tail.
The tail is where the real problem lives: files that grew because every feature had to touch them.
Those are structural — extract the sub-view or the controller, not the styling.

**Evidence** — Two apps in one bench, measured 2026-07-23.

The mature one: 217 of 323 components in the current style, 2 in the legacy one — a real decision,
visible in the ratio. Seven state-library stores, twelve composables, one data layer used in 85 files.
Median component 75 lines. Its tail is bad, though: 1356 and 870-line components, both of which are
"the screen everything gets added to".

The smaller one: 28 current / 16 legacy, with zero state library and three ad-hoc singleton modules
totalling 163 lines. That ratio is drift, not a decision, and it appeared without anyone choosing it.
Its median is 64 lines and its worst file 559 — better discipline at the component level, worse at the
convention level. The two are independent, and the convention one is cheaper to fix early.
