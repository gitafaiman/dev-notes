# Git Worktrees (with Claude Code)

Working on multiple branches at once, in parallel, without stashing or switching.

## The core idea

Normally one folder = one branch at a time. Worktrees let you check out multiple branches into separate folders simultaneously, all sharing the same `.git` history — so you're not duplicating the repo, and each folder has its own independent files/branch.

## Easiest path: let Claude Code do it for you

Claude Code has native worktree support built in. From your repo:

```bash
claude --worktree feature/wholesale-customers
```

This creates an isolated working directory (under `.claude/worktrees/`), checks out that branch there, and starts a Claude session in it — in one command.

Then open a second terminal tab, and start a different task on a different branch:

```bash
claude --worktree fix/some-other-bug
```

You now have two Claude Code sessions running fully in parallel, each with its own files on disk, each unaware of the other. No stashing, no conflicts, no waiting.

## Manual version (more control)

If you want to place the worktree somewhere specific, or check out a branch that already exists:

```bash
git worktree add ../es-gourmet-wholesale feature/wholesale-customers
cd ../es-gourmet-wholesale
claude
```

```bash
# in a second terminal
git worktree add ../es-gourmet-hotfix fix/some-bug
cd ../es-gourmet-hotfix
claude
```

Useful commands:

```bash
git worktree list                              # see all active worktrees
git worktree remove ../es-gourmet-wholesale    # clean up when done
```

## Notes

- `node_modules` and `.env.local`: each worktree is a fresh checkout, so you'll need to run `npm install` again in each new worktree folder, and copy over `.env.local` (dev Supabase/Clerk keys) since it's gitignored and won't come along automatically.
- Windows/Git Bash: worktrees work fine here, just use forward-slash paths as usual (`../es-gourmet-wholesale`, not `..\`).
- Open each worktree as its own VS Code window (`code .` inside the folder) so your editor state doesn't get confused between the two.
- Cleanup discipline: since these are extra folders on disk, remove them (`git worktree remove`) once a branch is merged, or they pile up — especially relevant given how many feature branches you tend to churn through.

## Gotchas

- Two worktrees can't have the same branch checked out at once — git will block that. This is for genuinely separate tasks/branches, not two sessions on the same branch.

## Related

- `supabase/migration-workflow.md`
