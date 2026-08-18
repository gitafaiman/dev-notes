# dev-notes

Personal reference notes on commands, tools, processes, and concepts — organized by topic, searchable, and always up to date.

This is a living notebook, not a polished wiki. Entries are meant to be quick to add and quick to grep through later.

## Structure

Notes are organized by topic, one folder per broad area, one markdown file per subtopic or tool.

```
dev-notes/
├── README.md
├── git/
│   ├── rebasing.md
│   └── worktrees.md
├── docker/
│   └── compose-basics.md
├── python/
│   ├── venvs.md
│   └── packaging.md
├── shell/
│   └── kill-process-on-port.md
├── supabase/
│   ├── migration-workflow.md
|   └── pgrst303-jwt-issued-at-future.md
└── concepts/
    └── idempotency.md
```

Rough guidance for splitting things up:

- A tool or technology gets its own folder (`git/`, `docker/`, `postgres/`).
- A workflow, task, or command reference within that tool gets its own file (`git/rebasing.md`).
- General concepts that aren't tied to one tool go in `concepts/` (`idempotency.md`, `cap-theorem.md`).

## Note template

Each note follows a light template so entries are easy to scan later:

```markdown
# Title

One-line summary of what this covers.

## Commands / Steps

    actual commands or step-by-step process here

## Notes

- Anything non-obvious, edge cases, or "why" context.

## Gotchas

- Things that bit you before, so you don't relearn them the hard way.

## Related

- Links to other notes in this repo, or external docs.
```

Not every note needs every section — skip what doesn't apply.

## Index

Update this table as notes are added. It's the fastest way to browse without digging through folders.

| Topic | File | Summary |
|---|---|---|
| Git | [rebasing](git/rebasing.md) | Interactive rebase, squashing, fixing history |
| Git | [worktrees](git/worktrees.md) | Working on multiple branches at once |
| Docker | [compose basics](docker/compose-basics.md) | Common docker-compose commands and patterns |
| Python | [venvs](python/venvs.md) | Creating and managing virtual environments |
| Python | [packaging](python/packaging.md) | pyproject.toml, building, publishing |
| Shell | [kill-process-on-port](shell/kill-process-on-port.md) | Handy grep/awk/find/sed snippets |
| Supabase | [migration-workflow](migration-workflow.md) | Schema migration workflow: branch → dev → PR → prod, essgourmet setup |
| Supabase | [pgrst303-jwt-issued-at-future](pgrst303-jwt-issued-at-future.md) | PGRST303 JWT clock-skew error — cause and fix |
| Concepts | [idempotency](concepts/idempotency.md) | What it means and why it matters |

## Conventions

- Write notes for future-you, who has forgotten everything and is in a hurry.
- Prefer copy-pasteable commands over prose descriptions of commands.
- Tag anything you got burned by once under **Gotchas** — that's usually the highest-value line in the whole repo.
- Keep entries short. If a note is getting long, it's probably two notes.

## Searching

Locally:

```bash
grep -ril "keyword" .
```

Or open the repo in [Obsidian](https://obsidian.md) for backlinking and a nicer browsing experience — it works directly on this folder of markdown files, no migration needed.
