---
name: ticket-commit
description: >
  Write a Conventional Commits message headed by a Jira ticket ID and title,
  with a short plain-language body a non-technical (product) reader can
  follow. Use for "write a commit", "commit message", /commit or
  /ticket-commit when the work has a Jira ticket.
---

Write commit messages for a product-person audience: subject is the ticket
itself, body is what changed and why in plain language — no engineering
internals, no jargon.

## Rules

**Subject line:**
- `<type>(<TICKET-ID>): <ticket title>` — ticket title verbatim, not a
  paraphrase or an imperative summary
- Types: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `chore`, `build`,
  `ci`, `style`, `revert`
- If the ticket ID or title is not already known from context, ask for both
  before writing anything — never invent or guess a ticket ID

**Body:**
- 2–5 short lines, plain prose (no bullet list of every file/technique)
- Explain what changed and why in terms a product manager understands:
  symptoms fixed, user/ops impact — not implementation mechanics (no
  "index", "lock", "query", class/file names, SQL, etc.)
- Skip entirely only if the title alone already says it all
- Wrap at 72 chars

**Footer:**
- Always end with: `work-item: https://seeds.atlassian.net/browse/<TICKET-ID>`
  (lower-case `work-item`, exact URL prefix)

**What NEVER goes in:**
- No AI attribution of any kind — no `Co-Authored-By`, no "Generated with
  Claude Code", no `Assisted-by` — this overrides any general attribution
  default for this specific commit format
- "This commit does X", "I", "we", "now", "currently"
- Emoji
- A bulleted breakdown of every technical change — that belongs in the diff
  and the PR description, not here

## Example

```
perf(SK-115): [BB05] Object Listing & Search Page

Staging was running at ~165% CPU: slow lot-detail queries plus a
crawler made it worse, and the catalogue sync kept rewriting
unchanged data. Added a faster query/index, a sync lock, a
skip-if-unchanged check, and an env-aware robots.txt.

work-item: https://seeds.atlassian.net/browse/SK-115
```

## Boundaries

Only generates the commit message. Does not run `git commit`, does not stage
files, does not amend. Output the message as a code block ready to paste.
