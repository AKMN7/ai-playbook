---
name: handoff
description: Compact the current conversation into a handoff document for another agent to pick up.
argument-hint: "What will the next session be used for?"
---

Write a handoff document summarising the current conversation so a fresh agent can continue the work. Save it to the root directory, named `handoff.md` (read the file before you write to it if it exists).

Suggest components to be used, if any, by the next session.

Do not duplicate content already captured in other artifacts (plans, issues, commits, diffs). Reference them by path or URL instead.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the doc accordingly.

Give the user a summarized first prompt to use in the next session, don't write it to the handoff document.
