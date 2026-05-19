# AI Playbook

A curated collection of project-specific rule sets for AI coding agents. Each project type keeps a short agent entrypoint plus detailed `.claude/rules/` files so Codex and Claude follow the same project standards.

## Project Types

| Type          | Status   | Description                               |
| ------------- | -------- | ----------------------------------------- |
| `frontend`    | Ready    | Standard web front-end applications       |
| `backend`     | Ready    | Server-side APIs and services             |
| `interactive` | Ready    | Touch-first kiosk and exhibit interfaces  |
| `scripts`     | Reserved | Standalone automation and utility scripts |

`general/` contains reusable skills and is intentionally separate from the project-type rule sets. `scripts/` is currently empty by design.

## Structure

```text
<project-type>/
+-- AGENTS.md           # Codex project brief and rule index
`-- .claude/
    +-- CLAUDE.md       # Claude project brief and rule index
    `-- rules/
        `-- ...         # Topic-specific source-of-truth rule files
```

## Usage

Copy the folder for your project type into your repo root. Keep `AGENTS.md`, `.claude/CLAUDE.md`, and `.claude/rules/` together so both agents receive the same standards.

When a rule changes, update the relevant file under `.claude/rules/` first, then adjust the agent entrypoints only if the rule index or high-level brief changes.
