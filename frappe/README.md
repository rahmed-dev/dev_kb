# frappe

Framework-specific material for Frappe / ERPNext: **runnable snippets and
framework notes, not knowledge-base notes.**

This is the one directory exempt from the Rule / Context / Why / Evidence format
in [AGENTS.md](../AGENTS.md). A Server Script body is code that gets pasted into
a running site — rewriting it as prose would destroy the only thing it is for.
Everything else in this repo stays framework-agnostic; if a lesson here
generalises, write the generic note in `backend/` or `practices/` and cite this
directory as its evidence.

| Directory | What it is | How it is used |
|---|---|---|
| `server_scripts/` | Server Script bodies | pasted into a site's **Server Script** doc |
| `console_scripts/` | one-off System Console snippets | pasted into a console, **never run unattended** |
| `notes/` | framework behaviour worth writing down | read |

## Before pasting anything

- **Every script has placeholders at the top.** Set them. A script that runs on
  the wrong doctype or reports on the wrong employee is worse than one that
  errors.
- **`console_scripts/delete_all_doctype_records` is destructive and permanent.**
  It deletes with `delete_permanently=True`, so there is no Deleted Document row
  to restore from. Read the count it prints before enabling Commit.
- A Server Script cannot `print()` anywhere you will see. Both scripts here
  either `log()` or write an Error Log row on purpose.

## Public repo

No credentials, hostnames, customer names, employee names or record IDs from a
real site. The auto-checkout diagnostic arrived here carrying an employee's
first name and their `HR-EMP-…` ID hardcoded in two queries; both are now
parameters at the top of the file. That is the failure mode to watch for — a
diagnostic is written against one real record and keeps it.
