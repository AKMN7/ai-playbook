---
name: frontend-creator
description: Bootstrap a new frontend project from the scaffold at https://github.com/nexerium-code/scaffold-frontend.git. Use this skill whenever a frontend, web client, or user-facing UI needs to be created — explicitly ("scaffold a new frontend", "spin up a React app", "create the UI") or implicitly ("I need a dashboard for the API I just built", "let me start the web client"). Every frontend produced is ALWAYS a Vite + React 19 + TypeScript strict single-page app with the locked stack (TanStack Router/Query/Table, Clerk auth, Tailwind 4 + shadcn/ui, React Hook Form + Zod, i18next with English + Arabic) — Next.js, Remix, Astro, SSR/SSG, and mobile apps are out of scope. The project is ALWAYS its own separate repo (never a subfolder or monorepo package) and ALWAYS organised as feature folders mirrored across components/, hooks/, and services/. If the user implies any default should change, push back before proceeding. Non-technical users may also trigger this skill, so keep explanations plain-language.
---

# Frontend Creator

A skill for starting a new frontend project from the frontend scaffold: clone the scaffold, wire in the new name, hand the user a clean repo that they can continue from.

This skill may be invoked by users who aren't themselves engineers — someone describing a product idea that clearly needs a user-facing web app, for example. When talking to the user, keep jargon minimal and explain what you're doing in everyday language; the bash commands in this skill are for you to run, not for the user to read.

## What this skill produces

A freshly cloned copy of `https://github.com/nexerium-code/scaffold-frontend.git`, renamed, with its git history reset so it's a brand-new repo. **It is a starting point, not a finished project.** Tell the user that explicitly when you hand it off — the scaffold ships with example modules (`scopes` and `resources`), a couple of placeholder values, and up-to-date agent rules and conventions for different providers (rule files tailored for Claude, Codex, Cursor, and similar tools). The user's job from there is to replace the example modules with real features, and any agent that continues the work should consult whichever rule files match its own provider.

## Non-negotiable defaults (and why)

These are the things this skill won't quietly deviate from. If the user asks for any of them, raise the conflict before scaffolding — don't just silently comply.

- **Vite + React 19 + TypeScript strict, always — and always a single-page app.** Not Next.js, not Remix, not Astro, not vanilla CRA, not Vue/Svelte/Solid/Angular, not server-rendered, not a mobile app. Every frontend produced from this skill is a client-rendered SPA built with Vite. The locked stack also includes TanStack Router (file-system routing) + TanStack Query (all server state) + TanStack Table, Clerk for authentication, Tailwind CSS 4 with shadcn/ui primitives, React Hook Form + Zod for forms, i18next with English and Arabic (with RTL support), Axios via `src/services/API.ts`, sonner for mutation toasts, and lucide-react for icons. Teams using this scaffold standardize on this stack so engineers stay fluent across codebases, tooling and patterns transfer cleanly, and there's a single way to do auth, routing, server state, forms, and styling. If the user asks for a different framework, asks for SSR/SSG, or asks to swap out one of these libraries (Redux instead of Query, another router, another UI kit, dropping i18n, dropping Clerk, etc.), ask them to confirm they want to leave the standard this scaffold enforces before proceeding.
- **Separate repo, never a monorepo.** This frontend lives in its own git repository, with its own CI, its own deploys, its own lifecycle. It is never added as a subfolder of an existing backend repo, never turned into a workspace in an npm/yarn/pnpm monorepo, never glued into a Turborepo/Nx setup. If the user says "add a frontend to my existing backend repo" or "put the web app in the same repo as the API", name the conflict: what they're describing is a monorepo, which this scaffold deliberately avoids. Confirm they actually want the separate repo before you scaffold.
- **Modular feature structure.** Features live as parallel folders across `src/components/<feature>/`, `src/hooks/<feature>/`, and `src/services/<feature>/`, with the same `<feature>` name in all three places. There is no flat top-level `components/` collecting everything horizontally, no `pages/` directory holding business logic, no per-feature folder that mixes UI and API calls into one big bag. shadcn/ui primitives live in `src/components/ui/` and are CLI-managed — don't edit them by hand. Shared app chrome (providers, layout) lives in `components/_app/`, generic reusable UI in `components/general/`, empty/error states in `components/empty-states/`, and skeletons in `components/skeleton/`. The architectural rationale is documented in the scaffold's project rules — which come in both a general form (`AGENTS.md`) and agent-specific forms (for example, `.claude/CLAUDE.md` and `.claude/rules/` for Claude, `.cursor/rules/` for Cursor). Point the user (or the continuing agent) at whichever file matches their tooling if they want to understand the reasoning in depth.
  If the user insists on a deviation after the conflict is named, that's their call. Note the deviation clearly in the handoff so they know they're off-standard.

## Procedure

### 1. Confirm the target location

Before cloning, agree with the user on:

- **Project name** — kebab-case, typically something like `events-fe`, `booking-fe`, `<domain>-fe`. The `-fe` suffix is the convention paired with the backend's `-be`. This becomes the folder name and the `name` field in `package.json`.
- **Target directory** — where to clone on disk. The one thing to verify: the target path is **not inside an existing git repository**. Run `git rev-parse --show-toplevel` from the intended parent directory; if it returns a path, the user is about to nest this inside another repo, which violates the separate-repo rule. Surface it and confirm before continuing.

### 2. Clone and reset git

```bash
git clone https://github.com/nexerium-code/scaffold-frontend.git <project-name>
cd <project-name>
rm -rf .git
git init
git add .
git commit -m "chore: initial commit from frontend scaffold"
```

The `rm -rf .git` + `git init` is deliberate: the new project is independent of the scaffold's commit history. The scaffold is a template, not a parent.

### 3. Replace placeholders

The scaffold has explicit placeholder values that must be swapped. Do these edits before handing back to the user.

**In `package.json`:**

- `name`: currently `"scaffold-frontend"` → set to the project name (kebab-case, matches folder)
  The scaffold's `package.json` doesn't ship with `description` or `author` fields. Don't add them unless the user asks; the scaffold's house style is to keep `package.json` lean.

**In `index.html`:**

- `<title>Scaffold Frontend</title>` → human-readable project name (e.g., `<title>Events</title>`).
- The scaffold ships with `<link rel="icon" type="image/png" href="/S.png" />` but **no `S.png` is included in `public/`**. Leave the link as-is and flag it as a TODO in the handoff: the user needs to drop a real favicon into `public/` and update the path. Don't invent a favicon.
  **In `README.md`:**

- `# Scaffold Frontend` heading → `# {Name}` (the human-readable project name).
- The first paragraph (`Reusable Vite + React frontend scaffold for product-style applications.`) → a one-line description of what this app is.
  **In `.claude/CLAUDE.md`:**

The scaffold uses `{Curly_Brace}` placeholders at the top — find and replace:

| Placeholder     | Replace with                                            |
| --------------- | ------------------------------------------------------- |
| `{Name}`        | Human-readable project name (e.g., "Events", "Booking") |
| `{Description}` | One-line description of what this app is                |

If the user hasn't given you a description yet, leave it as a clearly-marked `TODO:` value rather than making something up. Flag the TODOs in the handoff.

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

The variables the user will need to fill in:

- `VITE_BE_ENDPOINT` — the URL of the backend API this frontend talks to (e.g., `http://localhost:3000` for local dev, or the deployed API URL).
- `VITE_CLERK_PUBLISHABLE_KEY` — the publishable key from the user's Clerk project. If they haven't set up a Clerk account yet, point them to clerk.com — they'll need to create an application there and grab the publishable key from the dashboard. The app **will not start** without a valid key.
- `VITE_ENV` — environment label, defaults to `DEV`. Usually fine to leave as is for local development.
- `NPM_FLAGS` — leave as `--legacy-peer-deps`. This is **not** used by your local `npm install`; it's read by deploy platforms like Netlify to apply that flag during their server-side install step. It exists in `.env.example` so the deploy build mirrors what the scaffold expects.
  **Tell the user clearly** that `.env` now contains placeholder values and they must fill in the real ones — the backend URL and the Clerk publishable key in particular. Without correct values, the dev server will crash on first load. Point them at the file by name and let them know the variables are listed in the same order in both files.

Do not invent secret values. Do not commit the `.env` file — `.gitignore` should already exclude it, but double-check if you have any doubt.

### 6. Hand it off

When you report back to the user, be explicit about four things:

1. **This is a starting point.** The cloned project has example modules (`scopes` and `resources` — present in `components/`, `hooks/`, and `services/`), placeholder documentation, and boilerplate — it is not a finished app. These examples exist as pattern references; the user replaces them with real features as work begins.
2. **Project rules live in the repo.** The scaffold includes rule files describing the architectural standard — directory layout, feature workflow with TanStack Query, forms with React Hook Form + Zod + the shadcn `Field` primitive, routing and Clerk integration, Tailwind/shadcn styling and RTL, naming conventions, error/loading/empty states, i18n, and so on. There are multiple files, each tailored to a different coding agent: `AGENTS.md` for Codex and generic use, `.claude/CLAUDE.md` + `.claude/rules/` for Claude, `.cursor/rules/` for Cursor, and shared skills under `.agents/skills/` (e.g., the bundled shadcn skill). Whichever agent continues the work should read the file(s) matching its own provider before making architectural decisions.
3. **The Clerk + backend dependency.** Unlike a pure boilerplate, this scaffold expects two external things to be in place: a Clerk application (for the publishable key) and a backend at `VITE_BE_ENDPOINT`. If the user hasn't set those up yet, name it clearly — the app won't run end-to-end without them.
4. **Remaining TODOs.** Any placeholders you couldn't fill (description if not provided, etc.), the `.env` values the user still needs to provide (backend URL, Clerk key), the missing `public/S.png` favicon, and — if the dependency install had any warnings — surface those too.
   To start the dev server once `.env` is filled in:

```bash
npm run dev
```

You can mention this command in the handoff but you don't need to run it yourself — the user will run it when they're ready.

## When working on the scaffold afterwards

If the conversation continues after scaffolding — the user wants to add their first real feature, remove the example `scopes`/`resources` modules, configure something — don't re-derive architectural guidance from this skill. Instead, the agent that continues the work should read the rule files in the repo that are relevant to its own provider or environment (`.claude/CLAUDE.md` + `.claude/rules/` if it's Claude, `AGENTS.md` if it's Codex, `.cursor/rules/` if it's Cursor, and so on) and follow what it finds there. Those rule files are the maintained source of truth; this skill is only the scaffolding entry point.

The scaffold also bundles a shadcn skill under `.agents/skills/shadcn/` (and mirrored into `.claude/skills/shadcn/`) and configures the shadcn MCP server in `.mcp.json` and `.cursor/mcp.json`. If the user asks for new UI primitives, the continuing agent should use that skill rather than hand-rolling components or installing a different UI library.

## What not to do

- Don't treat this skill as permission to write a fresh Vite + React project from scratch with `npm create vite@latest`. That produces a bare Vite app without this scaffold's TanStack wiring, Clerk integration, shadcn primitives, i18n setup, example modules, or agent rule files. Always clone the scaffold.
- Don't skip the git reset. Leaving the scaffold's `.git` in place means the user's first `git push` will try to push to the scaffold repo or carry irrelevant history.
- Don't edit anything inside `src/components/ui/` by hand. Those files are managed by the shadcn CLI; manual edits get overwritten when components are updated and break the convention the scaffold enforces.
- Don't commit the `.env` file or ever print its real values into the chat once the user has filled them in.
- Don't try to "simplify" the scaffold by removing Arabic from `src/locales/`, removing Clerk, or stripping the example modules before handing back. The user can do that themselves once they decide what they actually need; pre-stripping makes assumptions about a project that hasn't started yet.
