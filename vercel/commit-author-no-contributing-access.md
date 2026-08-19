# Vercel: "Commit author does not have contributing access" — Cheatsheet

Full error:

> The deployment was blocked; The Deployment was blocked because the commit author does not have contributing access to the project on Vercel.

## Why this happens

Vercel checks the email address on the commit (not the GitHub username shown in the UI) against accounts that have access to the Vercel project/team. If the commit's author email doesn't map to a Vercel account with access, the deploy is blocked — even if the commit landed on `main` via a PR from an account that does have access, because the person who clicked "Merge" (and thus authored the merge commit) may be a different identity than the branch/PR author.

Common root causes:

- Local `git config user.email` doesn't match the email tied to your GitHub/Vercel account.
- You have two GitHub identities (e.g. a personal account and a work/org account) and committed or merged under the wrong one.
- GitHub's private "noreply" email is being used and Vercel can't map it to the right account.
- On a Hobby team: the commit author must literally be the Hobby team's owner.
- On a Pro team: the commit author must be a team member (auto- or manual-approved, depending on team collaboration settings).

## Step 1 — Check which identity you're actually signed in as, everywhere

Signing into a tool's UI (e.g. VS Code's Accounts panel) changes that tool's auth/push credentials — it does not automatically change your local git commit identity. Check each layer separately:

```bash
# What identity will the NEXT commit use? (local repo overrides global)
git config user.name
git config user.email
git config --global user.name
git config --global user.email

# What identity did a SPECIFIC past commit use?
git log -1 <commit-sha> --format="%an <%ae>"

# GitHub CLI — which account is authenticated?
gh auth status

# VS Code — check the Accounts icon (bottom-left) to see which GitHub
# account is signed in for Source Control / PR features. This affects
# push auth, NOT commit author identity.
```

Also check on the Vercel side:

- vercel.com/account/settings/authentication → Login Connections → confirm the right GitHub account is linked to your Vercel account.
- Team Settings → Members → confirm the intended GitHub account is actually a member of the team that owns the project (Pro), or is the team owner (Hobby).
- Project Settings → Git → confirm the correct repo is linked and the Production Branch is set to `main`.

## Step 2 — Fix the identity going forward

```bash
# Set the correct identity for ALL future commits in this repo
git config user.name "essgourmet"
git config user.email "the-correct-email@example.com"

# Or globally, for every repo on this machine
git config --global user.name "essgourmet"
git config --global user.email "the-correct-email@example.com"
```

Find the "correct" email in one of these ways:

- GitHub → Settings → Emails (while logged into the account that has Vercel access) — grab either the real verified email or the private noreply address (`ID+username@users.noreply.github.com`).
- Or pull it straight from a commit already authored correctly by that account:

```bash
git log --author=essgourmet -1 --format="%an <%ae>"
```

## Step 3 — Fix the author on an already-pushed commit

If the bad commit is the current tip (`HEAD`) of the branch — this is the common case for a just-merged PR:

```bash
# Confirm it's HEAD
git log -1 --format="%H %an <%ae>"

# Amend the author in place
git commit --amend --author="essgourmet <the-correct-email@example.com>" --no-edit

# Double check
git log -1 --format="%an <%ae>"

# Push the rewritten commit
git push origin main --force-with-lease
```

If it's a merge commit, the PR branch's own commits usually already have the right author — you can lift the exact string from the branch tip (2nd parent of the merge commit) instead of typing it by hand:

```bash
git log <merge-commit-sha>^2 -1 --format="%an <%ae>"
```

If the bad commit is buried further back in history (not `HEAD`):

```bash
git rebase -i <bad-commit-sha>^
# In the editor: change "pick" to "edit" on the target commit's line, save & close

git commit --amend --author="essgourmet <the-correct-email@example.com>" --no-edit
git rebase --continue

git push origin main --force-with-lease
```

⚠️ Note: default `git rebase -i` drops/flattens merge commits. If the range you're rebasing includes a merge commit you need to preserve, add `--rebase-merges` to the `git rebase -i` command.

Branch protection: if GitHub rejects the force-push, temporarily disable "Do not allow force pushes" under repo Settings → Branches → main, push, then re-enable it. Force-pushing also rewrites history for anyone else who has the repo cloned — they'll need to run `git fetch && git reset --hard origin/main` afterward.

## Step 4 — Trigger a fresh Vercel deploy

A force-push should fire a new webhook automatically. If it doesn't (e.g. right after reconnecting the GitHub integration), trigger one manually:

```bash
# Option A: empty commit under the correct identity (lowest risk, no history rewrite)
git commit --allow-empty -m "trigger deploy" --author="essgourmet <the-correct-email@example.com>"
git push origin main
```

Or from the Vercel dashboard: Deployments tab → Deploy → select `main` / latest commit.

## Lowest-risk shortcut

If you don't actually need a past commit's author fixed — just need Vercel to accept the next deploy — skip the rebase entirely:

1. Set the correct `git config user.email` (Step 2).
2. Push a new empty commit under that identity (Step 4, Option A).

No force-push, no rewritten history, no branch-protection headaches.

## Prevention checklist (going forward)

- [ ] `git config user.email` (local, per-repo) matches the account with Vercel access
- [ ] VS Code is signed in with the same GitHub account you intend to author commits as
- [ ] `gh auth status` shows the correct account (if using GitHub CLI)
- [ ] That GitHub account is a member of the Vercel team (Pro) or is the team owner (Hobby)
- [ ] Vercel account's Login Connections → GitHub is linked correctly
- [ ] Project Settings → Git → Production Branch is `main`, correct repo linked
- [ ] Project Settings → Git → no unexpected "Ignored Build Step" silently skipping builds

## References

- [Troubleshoot project collaboration — Vercel Docs](https://vercel.com/docs/deployments/troubleshoot-project-collaboration)
- [Vercel Community: deployment blocked — commit author lacks contributing access](https://community.vercel.com/t/vercel-deployment-blocked-because-commit-author-does-not-have-contributing-access/36381)
