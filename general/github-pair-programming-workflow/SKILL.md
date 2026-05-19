---
name: github-pair-programming-workflow
description: Use this skill whenever two people are collaborating on the same Git repository and need a lightweight workflow that keeps changes small and reviewable — especially when AI coding tools (Claude Code, Cursor, Copilot, etc.) are accelerating their work. Trigger when the user mentions collaborating with a teammate on a shared repo, opening pull requests, naming branches, merging into a shared `dev` branch, conflicts piling up, splitting a big change, GitHub remote setup with `origin`, pushing for the first time, GitHub Personal Access Token (PAT) auth, or asks things like "how should we both work on this repo", "what should I name this branch", or "can I merge my own PR". Also trigger when someone is about to merge their own PR and wants a sanity check, or needs a PR template. Defines `main` / `dev` / short-lived branches, branch and PR naming, the one-PR-one-change rule, merge guardrails, and GitHub remote + PAT setup. Written to be readable by non-technical collaborators too.
---

# GitHub Pair Programming Workflow

A lightweight Git workflow for two people working on the same codebase, especially when AI coding tools are involved. The whole point: keep changes small, integrate often, and never let one branch drift into a giant unreviewable mess.

In the examples below the two collaborators are called **anas** and **muath** — substitute your own names.

## The setup

Three layers of branches:

- `main` — stable. Production-ready code only. Never edited directly.
- `dev` — the shared integration branch. Both people merge their work here.
- `<creator>/<type>-<short-description>` — short-lived working branches. Created from the latest `dev`, deleted after merge.
  Why this shape: `main` stays clean for releases. `dev` absorbs daily work so the two of you see each other's changes quickly. Working branches stay small and disposable so conflicts surface in days, not weeks.

## Core rules

- `main` is stable and never touched directly.
- `dev` is where everyone's work meets.
- Every piece of work starts as a fresh branch off the latest `dev`.
- PRs go into `dev`, not `main`.
- Draft PRs are fine — opening early is encouraged.
- One PR equals one small visible change or fix.
- Conflicts should be found early, not at the end.
- The creator may merge their own PR into `dev` — but only if all the merge guardrails pass.

## The workflow

> First time on this machine? Jump down to **Git remote setup** and get the repo cloned before running step 1.

1. Pull latest `dev`.
2. Create a small branch from `dev`.
3. Work on one small visible change or fix.
4. Confirm it works manually.
5. Push and open a PR into `dev` (draft is fine).
6. Resolve conflicts early — don't let them pile up.
7. Review the PR.
8. Merge into `dev` if it passes the guardrails.
9. Pull latest `dev` before starting the next change.

## Always pull before you start

Before creating a new working branch — and again before merging your own PR — pull the latest `dev`. The other person may have pushed work since you last synced, and branching from (or merging onto) a stale `dev` is the single most common cause of conflicts in this workflow.

Every new piece of work starts with:

```bash
git checkout dev
git pull origin dev
```

If `git pull` brings in new commits, take a quick look at what changed before you branch. You may find the other person already started something close to what you were about to do — better to find out now than after writing 200 lines.

Right before merging your own PR, do the same thing — pull `dev`, then bring it into your working branch:

```bash
git checkout dev
git pull origin dev
git checkout <your-branch>
git merge dev
```

If `git merge dev` reports conflicts, resolve them locally, confirm the app still runs, push the resolution, then merge the PR. The remote moves while you work — assume it has, and verify.

For agents: never skip this step, even if you "just pulled a few minutes ago". Treat it as the first action of every new task, and the last check before any merge.

## Branch naming

Format:

```
<creator>/<type>-<short-description>
```

Allowed `<type>` values: `feature`, `fix`, `ui`, `api`, `refactor`, `chore`.

Examples:

- `anas/feature-login-form`
- `anas/fix-mobile-menu`
- `muath/ui-product-card`
- `muath/api-create-order`
- `muath/chore-update-copy`
  The `<creator>` prefix makes it obvious who owns the branch. The `<type>` makes the intent readable in a list. Keep the description short — 2 to 4 words, lowercase, hyphens.

## PR naming

A PR title should describe the visible change in plain language.

Good:

- `Add login form`
- `Fix mobile menu`
- `Update product card layout`
- `Add create order endpoint`
  Bad — vague, doesn't explain anything:

- `Update app`
- `Fix stuff`
- `Improve dashboard`
- `Changes`
- `Final updates`
  If a PR title would be too vague to be useful, the PR is probably too big. Split it.

## PR size rules

- One PR = one visible change.
- If more than 8–10 files changed, stop and ask before merging.
- Don't sneak in unrelated refactors.
- Don't include global formatting / lint-the-whole-repo changes.
- Don't mix multiple features in one PR.
  A PR that's hard to describe in one sentence is too big.

## Merge guardrails

The creator can merge their own PR into `dev` only if **all** of these are true:

1. The PR is one visible change.
2. It was manually tested.
3. It changes fewer than 10 files.
4. There are no conflicts.
5. The app still runs after the change.
6. The diff does not include unrelated refactors.
   If any rule fails, do not merge. Ask the technical owner first.

## PR template

Use this in every PR description. Keep it short:

```
Changed:

Tested:

Notes:
```

- **Changed:** what visibly changed (one or two lines).
- **Tested:** how you confirmed it works.
- **Notes:** anything reviewers should know — risks, follow-ups, screenshots, links.

## Git remote setup

The remote repository on GitHub already has `main` and `dev` set up. You don't create either of them — both are assumed to exist. Two ways to get a working local copy:

### Cloning the repo (most common)

```bash
git clone https://github.com/<owner>/<repo>.git
cd <repo>
git checkout dev
git pull origin dev
```

`git clone` brings down all branches; `git checkout dev` switches into the shared integration branch and starts tracking it.

### Connecting an existing local project to GitHub

If the project is already on your machine and just needs to be wired up to the existing GitHub remote:

```bash
git remote -v                                                       # check current remotes
git remote add origin https://github.com/<owner>/<repo>.git
git fetch origin
git checkout dev
git pull origin dev
```

`git fetch origin` pulls down references to the existing `main` and `dev` so you can switch into them locally.

### Starting a new working branch from `dev`

Always do this from an up-to-date `dev`:

```bash
git checkout dev
git pull origin dev
git checkout -b muath/feature-example
git push -u origin muath/feature-example
```

## GitHub sign-in with a Personal Access Token (PAT)

Git over HTTPS asks for a username and password the first time you push. GitHub no longer accepts your account password here — use a **Personal Access Token (PAT)** instead.

When Git prompts:

- **Username:** your GitHub username
- **Password:** your GitHub Personal Access Token (NOT your GitHub account password)
  Keep in mind:

- Never paste a PAT into code, commits, docs, chat messages, or screenshots. If it leaks, revoke it on GitHub immediately and create a new one.
- On Windows, Git Credential Manager usually saves the token after the first successful login, so you won't be asked every time.
- For private repos, the PAT must have access to that specific repo.
- A fine-grained PAT should include the target repository and **Contents: read/write** permission.

## Working with AI coding tools

When an AI coding tool is generating changes for you, the workflow above is what keeps things sane:

- Stay on a small branch — never let the AI loose directly on `dev` or `main`.
- After each chunk of AI-generated work, stop and check it manually before pushing.
- If the AI starts touching files that aren't part of the visible change, push back and keep it on-task.
- Open the PR as a draft early. It makes the diff easy to review as it grows and surfaces conflicts sooner.
- If the diff sprawls past 10 files, split it. Open a second small PR rather than one giant one.
  The AI is fast. The guardrails exist precisely because speed makes it easy to land a 50-file PR that no one can actually review.
