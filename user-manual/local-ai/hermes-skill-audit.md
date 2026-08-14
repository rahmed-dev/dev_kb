# Hermes Agent skill audit — knowledge-base maintainer profile

Scanned `C:\Users\Rizwan Ahmed\AppData\Local\hermes\hermes-agent\skills` on 2026-08-14.
77 skills across 14 categories.

## Read this first: trimming skills will NOT fix the slow first prompt

The skills index that Hermes injects into the system prompt is only the `description:`
frontmatter of each SKILL.md — not the skill bodies. Measured across all 77 skills that is
**~1,075 tokens**, roughly **4% of the ~28,800-token opening request**.

Disabling every skill on the box would save under 1.1k tokens, about 1.5 seconds of prefill.
The other ~27,700 tokens are elsewhere. Hermes computes the split itself in
`agent/context_breakdown.py`, which buckets the prompt into: System prompt, Tool definitions,
Rules, Skills, MCP, Subagent definitions, Memory, Conversation.

**Get the real numbers before cutting anything** — run Hermes' own context breakdown (the
`/context` view) on a fresh session and read the per-bucket token counts. If "Tool definitions"
or "MCP" dominates, trimming *toolsets* is the lever, not skills. `config.yaml` has no
toolset list yet; the code paths use `enabled_toolsets` / `disabled_toolsets`.

So treat the list below as **noise reduction and safety**, not performance work.

## Per-category cost

| Category | Skills | Index tokens | Keep for a KB maintainer? |
|---|---|---|---|
| productivity | 16 | ~227 | partial |
| creative | 16 | ~215 | mostly drop |
| software-development | 11 | ~152 | mostly drop |
| research | 7 | ~100 | **keep** |
| github | 7 | ~98 | drop |
| autonomous-ai-agents | 6 | ~86 | mostly drop |
| apple | 4 | ~55 | drop |
| media | 3 | ~39 | drop |
| email | 2 | ~26 | drop |
| mlops | 4 | ~15 | drop |
| social-media | 1 | ~15 | drop |
| smart-home | 1 | ~15 | drop |
| note-taking | 1 | ~14 | **keep** |
| devops | 1 | ~12 | drop |

## Keep — core to the second brain

- `note-taking/obsidian` — the vault is the product.
- `research/llm-wiki` — this is literally the Karpathy wiki pattern the project chose.
- `research/grounded-citations` — sourcing claims in notes.
- `research/arxiv`, `research/blogwatcher`, `research/competitor-news-monitor` — ingest sources.
- `research/blocked-page-recovery` — needed when an ingest URL is paywalled or bot-blocked.
- `productivity/ocr-and-documents`, `pdf`, `nano-pdf`, `docx`, `xlsx` — turning source files
  into notes is the main ingest path.
- `productivity/document-to-action-items`, `meeting-action-items`, `weekly-review-planning` —
  the "todo / learning roadmap" workflow.
- `productivity/session-librarian` — keeps session history navigable.
- `autonomous-ai-agents/hermes-agent` — how Hermes answers questions about itself.
- `creative/architecture-diagram`, `excalidraw` — diagrams inside notes.

## Drop — no role in a KB maintainer

- **apple** (apple-notes, apple-reminders, findmy, imessage) — no Apple devices in play.
- **smart-home** (openhue), **social-media** (xurl), **media** (gif-search, songsee,
  youtube-content) — unrelated to the vault.
- **email** (email-inbox-triage, himalaya) — unless email becomes an ingest source.
- **github** (all 7) — code-review and PR workflows. The box was deliberately repurposed away
  from agentic coding.
- **devops** (sdlc-review), **mlops** (evaluation, huggingface-hub, inference, models) —
  keep `huggingface-hub` only if you plan to shop for quants from inside Hermes.
- **software-development** — drop dogfood, inspecting-hermes-desktop-dom,
  node-inspect-debugger, python-debugpy, requesting-code-review, simplify-code, spike,
  systematic-debugging, test-driven-development. Keep `plan` if you want structured planning
  for the KB build-out; keep `hermes-agent-skill-authoring` only if you intend to write skills.
- **autonomous-ai-agents** — drop claude-code, codex, opencode (pi and opencode were removed
  from this box), computer-use, merge-reconciler.
- **creative** — drop ascii-art, ascii-video, baoyu-infographic, claude-design, comfyui,
  design-md, manim-video, p5js, popular-web-designs, pretext, sketch,
  songwriting-and-ai-music, touchdesigner-mcp, humanizer.
- **productivity** — drop airtable, google-workspace, maps, notion, powerpoint,
  product-price-monitor, teams-meeting-pipeline. (Drop `notion` only if the vault stays
  Obsidian-only.)

That is roughly **50 of 77 disabled**, leaving ~27 focused on ingest, research, and the vault.

## How to disable

Skills are gated by a `skills.disabled` list in
`C:\Users\Rizwan Ahmed\AppData\Local\hermes\config.yaml` (resolved by
`agent/skill_utils.get_disabled_skill_names`). The `config.yaml` currently has a `skills:`
block with only `creation_nudge_interval`, so the list needs adding.

Inspect current state first:

```
hermes skills list
```

The CLI exposes enable/disable verbs (`hermes_cli/cli_commands_mixin.py`), so prefer those over
hand-editing YAML — they write the same `skills.disabled` key without risking a syntax error.

Note: disabling is an *index* gate. `skill_view` / `skills_list` can still reach a disabled
skill, and `hermes -s <skill>` deliberately bypasses the gate. Disabling hides a skill from the
model's default menu; it does not uninstall it.

## After changing the skill list

The system prompt changes, so the saved KV prefix goes stale:

```bash
~/llamacpp-setup/warm.sh save
```
