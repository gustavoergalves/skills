---
name: ticket-commit
description: >
  Write a Conventional Commits message headed by a ticket ID and title from
  the project's issue tracker (Jira, Linear, GitHub Issues, ClickUp, ...),
  with a short plain-language body a non-technical (product) reader can
  follow and a footer linking to the ticket. Use for "write a commit",
  "commit message", /commit or /ticket-commit when the work has a ticket.
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
- Always end with: `work-item: <ticket URL>` (lower-case `work-item`)
- `<ticket URL>` is the ticket's real browse URL in the project's tracker.
  Resolve it in this order and stop at the first hit:
  1. **Already in context** — a URL returned by the tracker for this ticket
     earlier in the session (e.g. Jira's `webUrl`, a Linear/GitHub issue
     link), or one in a local ticket note (e.g. `data/jira/<TICKET-ID>/`).
  2. **Repo convention** — the host/prefix used in earlier commits' footers:
     `git log -200 --format=%B | grep -E '^work-item: '`. Reuse that prefix
     with the new ticket ID.
  3. **Project docs** — a tracker URL or base in `CLAUDE.md`, `AGENTS.md`,
     `README`, `CONTRIBUTING`, or the PR template.
  4. **Tracker connector** — when an issue-tracker MCP is connected, look the
     ticket up (e.g. Atlassian `getJiraIssue` → `webUrl`; or the site URL
     from `getAccessibleAtlassianResources` + `/browse/<TICKET-ID>`).
  5. **Ask** the user for the tracker URL. Never guess a host.
- Build the URL with the tracker's own pattern, keeping the ticket ID's exact
  casing:
  - Jira: `https://<site>.atlassian.net/browse/<TICKET-ID>`
  - Linear: `https://linear.app/<workspace>/issue/<TICKET-ID>`
  - GitHub Issues: `https://github.com/<owner>/<repo>/issues/<number>`
  - ClickUp: `https://app.clickup.com/t/<task-id>`
- If two sources disagree (e.g. an old footer points to another Jira site),
  prefer the URL the tracker itself returned for this ticket, and mention the
  mismatch in one line next to the message.

**What NEVER goes in:**
- No attribution trailers of any kind — no `Co-Authored-By` (human or AI),
  no "Generated with Claude Code", no `Assisted-by`. The `work-item` footer
  is the last line. This overrides any general attribution default.
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

work-item: https://<site>.atlassian.net/browse/SK-115
```

## Boundaries

Only generates the commit message. Does not run `git commit`, does not stage
files, does not amend. Output the message as a code block ready to paste.
