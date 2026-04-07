# AI Playbook

A curated collection of project-specific rule sets for AI coding agents. Each project type ships a complete set of standards for both **Cursor** (`.cursor/rules/`) and **Claude** (`.claude/`), so agents stay consistent from the first prompt.

## Project Types

| Type          | Description                               |
| ------------- | ----------------------------------------- |
| `frontend`    | Standard web front-end applications       |
| `interactive` | Touch-first kiosk and exhibit interfaces  |
| `backend`     | Server-side APIs and services             |
| `scripts`     | Standalone automation and utility scripts |

## Structure

```
<project-type>/
├── .claude/
│   ├── CLAUDE.md       # Main project brief
│   └── rules/
│       └── ...         # Topic-specific rule files (.md)
└── .cursor/
    └── rules/
        └── ...         # Topic-specific rule files (.mdc)
```

## Usage

Copy the folder for your project type into your repo root. The agent tools will pick up the rules automatically.
