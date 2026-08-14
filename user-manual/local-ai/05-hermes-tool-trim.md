# Hermes tool trim — knowledge-base maintainer profile

Measured 2026-08-14 against a live session reporting **24,104 tokens**, of which
**Tool definitions = 15,038 (62%)**. Skills were not even a line item on the breakdown.

Per-tool schema sizes, extracted from the schema literals in
`C:\Users\Rizwan Ahmed\AppData\Local\hermes\hermes-agent\tools\*.py`:

| Tool | ~tokens | Verdict |
|---|---:|---|
| cronjob | 2,684 | **drop** |
| session_search | 2,151 | drop (see note) |
| delegate_task | 1,556 | **drop** |
| skill_manage | 1,526 | **drop** |
| memory | 919 | keep |
| clarify | 911 | keep |
| terminal | 678 | keep |
| patch | 637 | keep |
| text_to_speech | 608 | **drop** |
| todo | 553 | keep |
| search_files | 474 | keep |
| vision_analyze | 461 | keep |
| process | 436 | drop |
| read_file | 432 | keep |
| write_file | 346 | keep |
| browser_exec | 280 | keep |
| skill_view | 267 | keep |
| execute_code | 140 | keep |
| skills_list | 100 | keep |
| computer_use | not measured | **drop** |
| obsidian_context | not measured (small) | **keep — core** |

The measured sum is ~15,166 against Hermes' reported 15,038, so these numbers are sound to
within ~1%.

## Keep — the vault workflow

- **file** (`read_file`, `write_file`, `patch`, `search_files`) — the vault is markdown on disk.
  This is the whole job. ~1,889 tokens well spent.
- **plugin_obsidian_context_bridge** (`obsidian_context`) — live selection/cursor from Obsidian.
  The single most on-point tool for this setup.
- **memory** — a second brain that forgets between sessions is not one.
- **todo** — the "learning todo / roadmap" workflow that started this.
- **terminal** + **execute_code** — git on the vault, scripted bulk note edits. `execute_code`
  is only 140 tokens.
- **skill_view** + **skills_list** (367 total) — lets the model reach `llm-wiki`, `obsidian`,
  and the research skills on demand. Cheap, and it is what makes the ~1k skills index useful.
- **clarify** — cheaper than a wrong answer on an architecture decision.
- **vision_analyze** + **browser_exec** (741 total) — screenshots and web pages are ingest
  sources for notes.

## Drop — ~9,000 tokens

- `cronjob` (2,684) — the single most expensive tool in the list. Only worth it if you actually
  want scheduled digests. Re-enable the day you do.
- `delegate_task` (1,556) — subagents. Also drags in the **Subagent definitions** category
  (1,442 tokens on your breakdown), so dropping it is worth **~3,000 tokens**, not 1,556.
  Delegation is a coding-agent pattern; a KB maintainer works in one thread.
- `skill_manage` (1,526) — creating and deleting skills. You are consuming skills, not authoring
  them. `skill_view`/`skills_list` are what you keep.
- `text_to_speech` (608) — no role in maintaining a wiki.
- `process` (436) — managing background processes from `terminal(background=true)`. Nothing here
  runs servers.
- `computer_use` — driving the desktop via cua-driver. Unmeasured but schemas of this shape run
  large, and Obsidian is reachable through the file tools and the context bridge instead.

**session_search (2,151) is the judgement call.** It searches your past Hermes sessions, which
is genuinely second-brain-shaped. But it is the second most expensive tool on the box, and its
job overlaps `search_files` over the vault once conversations get written into notes. Drop it
first; if you find yourself hunting for "what did I say last week" and coming up empty, put it
back and accept the 2.1k.

## Expected result

24,104 → **~13,700 tokens**. At the measured 735 t/s that moves worst-case cold prefill from
**32.8 s to ~19 s**, cached or not. Trimming further hits diminishing returns — the remaining
6,432-token System prompt and 2,240-token Rules are not user-tunable.

## How

`/tools` takes per-tool arguments — no need to disable a whole toolset:

```
/tools list
/tools disable cronjob delegate_task skill_manage text_to_speech process computer_use session_search
```

Config also has a top-level `toolsets:` key (defaults to `["hermes-cli"]`) if you prefer
coarse-grained control.

## After trimming

The system prompt changes, so the saved KV prefix is stale:

```bash
~/llamacpp-setup/warm.sh save
```
