---
name: backend-creator
description: Bootstrap a new backend API project from the scaffold at https://github.com/nexerium-code/scaffold-backend.git. Use this skill whenever a server-side backend needs to be created — whether the user asks for it explicitly ("scaffold a new backend", "start a new API") or only implies it ("I'm starting a new project with a database and some endpoints", "I need somewhere to store users and serve them to the app"). The backend is ALWAYS NestJS, ALWAYS in its own separate repo (never added to an existing repo as a subfolder or monorepo package), and ALWAYS a modular monolith. If the user's request implies any of those defaults should change, push back before proceeding. Non-technical users may also trigger this skill, so explanations to the user should stay plain-language.
---

# Backend Creator

A skill for starting a new backend API project from the backend scaffold: clone the scaffold, wire in the new name, hand the user a clean repo that they can continue from.

This skill may be invoked by users who aren't themselves engineers — someone describing a product idea that clearly needs a server-side component, for example. When talking to the user, keep jargon minimal and explain what you're doing in everyday language; the bash commands in this skill are for you to run, not for the user to read.

## What this skill produces

A freshly cloned copy of `https://github.com/nexerium-code/scaffold-backend.git`, renamed, with its git history reset so it's a brand-new repo. **It is a starting point, not a finished project.** Tell the user that explicitly when you hand it off — the scaffold ships with example modules, placeholder values, and up-to-date agent rules and conventions for different providers (rule files tailored for Claude, Codex, Cursor, and similar tools). The user's job from there is to replace the example modules with real features, and any agent that continues the work should consult whichever rule files match its own provider.

## Non-negotiable defaults (and why)

These are the things this skill won't quietly deviate from. If the user asks for any of them, raise the conflict before scaffolding — don't just silently comply.

- **NestJS, always.** Not Express, not Fastify standalone, not Hono, not Bun-native, not Go. Teams using this scaffold standardize on NestJS so engineers stay fluent across codebases, tooling (Nest CLI, DI, testing harness) is shared, and architectural patterns transfer cleanly. If the user asks for a different framework, ask them to confirm they want to leave the standard this scaffold enforces before proceeding.
- **Separate repo, never a monorepo.** This backend lives in its own git repository, with its own CI, its own deploys, its own lifecycle. It is never added as a subfolder of an existing frontend repo, never turned into a workspace in an npm/yarn/pnpm monorepo, never glued into a Turborepo/Nx setup. If the user says "add a backend to my existing app repo," name the conflict: what they're describing is a monorepo, which this scaffold deliberately avoids. Confirm they actually want the separate repo before you scaffold.
- **Modular monolith.** Features live under `src/modules/<feature>/` with their own controller, service, DTOs, schemas, and module wiring colocated. There are no top-level `controllers/`, `services/`, or `models/` directories collecting everything horizontally. Child resources nest under their parent feature. The architectural rationale is documented in the scaffold's project rules — which come in both a general form and agent-specific forms (for example, `AGENTS.md` for Codex, `.claude/` for Claude, `.cursor/` for Cursor). Point the user (or the continuing agent) at whichever file matches their tooling if they want to understand the reasoning in depth.
  If the user insists on a deviation after the conflict is named, that's their call. Note the deviation clearly in the handoff so they know they're off-standard.

## Procedure

### 1. Confirm the target location

Before cloning, agree with the user on:

- **Project name** — kebab-case, typically something like `events-be`, `booking-be`, `<domain>-be`. This becomes the folder name and the `name` field in `package.json`.
- **Target directory** — where to clone on disk. The one thing to verify: the target path is **not inside an existing git repository**. Run `git rev-parse --show-toplevel` from the intended parent directory; if it returns a path, the user is about to nest this inside another repo, which violates the separate-repo rule. Surface it and confirm before continuing.

### 2. Clone and reset git

```bash
git clone https://github.com/nexerium-code/scaffold-backend.git <project-name>
cd <project-name>
rm -rf .git
git init
git add .
git commit -m "chore: initial commit from backend scaffold"
```

The `rm -rf .git` + `git init` is deliberate: the new project is independent of the scaffold's commit history. The scaffold is a template, not a parent.

### 3. Replace placeholders

The scaffold has explicit placeholder values that must be swapped. Do these edits before handing back to the user.

**In `package.json`:**

- `name`: currently `"name-be"` → set to the project name (kebab-case, matches folder)
- `description`: currently `""` → set to the one-line description
- `author`: currently `""` → ask the user what to put here; if they don't care, set it to `"scaffold-backend"` as the default
  **In `README.md`** (the scaffold uses `{Curly_Brace}` placeholders — find and replace):

| Placeholder        | Replace with                                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------------------------------- |
| `{Name}`           | Human-readable project name (e.g., "Events", "Booking")                                                         |
| `{Description}`    | One-line description                                                                                            |
| `{Aspects}`        | A short bulleted list of what this service will handle — or leave as a TODO marker if the user doesn't know yet |
| `{Project_Link}`   | The new git remote URL once the user pushes it up, or a `TODO` if not yet created                               |
| `{Directory_Path}` | The project folder name                                                                                         |
| `{Docker_Name}`    | Typically the project name, lowercase with hyphens                                                              |

If the user hasn't provided some of these yet (e.g., no remote URL, no aspect list), leave them as clearly-marked `TODO:` values rather than making something up. Flag the TODOs in the handoff.

### 4. Install dependencies

First, verify that the user's machine has the tools this project needs:

```bash
node --version
npm --version
```

Both commands must succeed. Node.js should be a current LTS release (if the version looks unusually old, mention it and suggest upgrading). If either command is missing, stop here and tell the user in plain language: "You'll need Node.js installed first — download the LTS version from nodejs.org, and npm will come with it. Once it's installed, I can pick this back up." Don't try to install Node yourself.

Once both are confirmed, install the project's dependencies:

```bash
npm install
```

If the install fails, report the error to the user verbatim rather than guessing — scaffold issues are rare and usually worth looking at together.

### 5. Set up environment variables

The scaffold ships with a `.env.example` file that lists the environment variables the application expects. Copy it to create the actual `.env` file the app will read:

```bash
cp .env.example .env
```

Then **tell the user clearly** that `.env` now contains placeholder or blank values and they must fill in the real ones — things like database connection strings, API keys, and secrets. Without correct values, the service won't start. Point them at the file by name and let them know the variables are listed in the same order in both files, so they can work through them top to bottom.

Do not invent secret values. Do not commit the `.env` file — `.gitignore` should already exclude it, but double-check if you have any doubt.

### 6. Hand it off

When you report back to the user, be explicit about three things:

1. **This is a starting point.** The cloned project has example modules (`example-nested`, `example-standalone`), placeholder documentation, and boilerplate — it is not a finished app. These examples exist as pattern references; the user replaces them with real features as work begins.
2. **Project rules live in the repo.** The scaffold includes rule files describing the architectural standard — directory layout, module design, DTO/validation conventions, error handling shape, and so on. There may be multiple files, each tailored to a different coding agent: `AGENTS.md` for Codex and generic use, `.claude/` for Claude, `.cursor/` for Cursor, and so on. Whichever agent continues the work should read the file(s) matching its own provider before making architectural decisions.
3. **Remaining TODOs.** Any placeholders you couldn't fill (remote URL, aspects, etc.), the `.env` values the user still needs to provide, and — if the dependency install had any warnings — surface those too.

## When working on the scaffold afterwards

If the conversation continues after scaffolding — the user wants to add their first real module, remove the example modules, configure something — don't re-derive architectural guidance from this skill. Instead, the agent that continues the work should analyze the rule files in the repo that are relevant to its own provider or environment (`.claude/` if it's Claude, `AGENTS.md` if it's Codex, `.cursor/` if it's Cursor, and so on) and follow what it finds there. Those rule files are the maintained source of truth; this skill is only the scaffolding entry point.

## What not to do

- Don't treat this skill as permission to write a fresh NestJS project from scratch with `nest new`. That produces a bare Nest app without this scaffold's conventions, filters, common utilities, example modules, or agent rule files. Always clone the scaffold.
- Don't skip the git reset. Leaving the scaffold's `.git` in place means the user's first `git push` will try to push to the scaffold repo or carry irrelevant history.
- Don't commit the `.env` file or ever print its real values into the chat once the user has filled them in.
