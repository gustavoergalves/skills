---
name: ticket-commit
description: >
  Write a commit message headed by a ticket ID and title from the project's
  issue tracker (Jira, Linear, GitHub Issues, ClickUp, ...). Two styles:
  `bracket` (`[TICKET-ID] Title`, optionally with a work-item link) and
  `conventional` (Conventional Commits subject, plain-language body for a
  product reader, work-item footer). Style is picked from the repo's own
  history unless passed explicitly. Use for "write a commit", "commit
  message", /commit or /ticket-commit when the work has a ticket.
argument-hint: "[detailed] [style=bracket|conventional] [TICKET-ID]"
---

Generate one commit message whose subject is the ticket itself. Which shape
it takes depends on the style the repo uses.

## Arguments

All optional, any order:

| Argument | Short | Effect |
|---|---|---|
| `style=bracket` | `s=b` | Force the bracket style |
| `style=conventional` | `s=c` | Force the conventional style |
| `detailed` | `d` | Bracket style only: add the `work-item` footer |
| a ticket ID (e.g. `ABC-123`) | | Use this ticket instead of inferring it |

## 1. Pick the style

Stop at the first hit:

1. **Explicit argument** — `style=` / `s=`.
2. **Repo convention** — read recent subjects:
   `git log -50 --no-merges --format=%s`
   - `[ABC-123] ...` subjects → `bracket`
   - `type(ABC-123): ...` subjects → `conventional`
   - Mixed: go with the majority of the most recent 20.
3. **Ask** the user which style, in one short question. Never guess.

State the chosen style and why in one line above the message
(e.g. `Style: bracket (repo history)`).

## 2. Ticket ID and title

- Ticket ID: from the argument, otherwise from context (branch name such as
  `feature/ABC-123-...`, a ticket fetched earlier, a local ticket note).
- Title: the ticket's title **verbatim** — not a paraphrase or an
  imperative summary. Fetch it from the tracker connector if one is
  connected and the title isn't already in context.
- If either is still unknown, ask for it before writing anything — never
  invent a ticket ID or title.

## 3a. Bracket style

**Default (short)** — the subject line only, nothing else:

```
[ABC-123] Ticket title verbatim
```

**`detailed`** — subject, blank line, footer:

```
[ABC-123] Ticket title verbatim

work-item: https://<site>.atlassian.net/browse/ABC-123
```

- No body, no type prefix, in either variant.
- The blank line before `work-item` is required: git treats everything up to
  the first blank line as the subject, and trailers are only recognised after
  one.

## 3b. Conventional style

`detailed` has no effect here — this style always has body and footer.

**Subject:** `<type>(<TICKET-ID>): <ticket title>`
- Types: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `chore`,
  `build`, `ci`, `style`, `revert`

**Body** — written for a product person:
- 2–5 short lines, plain prose (no bullet list of every file/technique)
- What changed and why in terms a product manager understands: symptoms
  fixed, user/ops impact — not implementation mechanics (no "index", "lock",
  "query", class/file names, SQL, etc.)
- Skip only if the title alone already says it all
- Wrap at 72 chars

**Footer:** `work-item: <ticket URL>` (lower-case `work-item`), last line.

```
perf(SK-115): [BB05] Object Listing & Search Page

Staging was running at ~165% CPU: slow lot-detail queries plus a
crawler made it worse, and the catalogue sync kept rewriting
unchanged data. Added a faster query/index, a sync lock, a
skip-if-unchanged check, and an env-aware robots.txt.

work-item: https://<site>.atlassian.net/browse/SK-115
```

## Resolving the ticket URL

Only needed when the message has a `work-item` footer. Stop at the first hit:

1. **Already in context** — a URL the tracker returned for this ticket
   earlier in the session (e.g. Jira's `webUrl`, a Linear/GitHub issue
   link), or one in a local ticket note (e.g. `data/jira/<TICKET-ID>/`).
2. **Repo convention** — the host/prefix in earlier footers:
   `git log -200 --format=%B | grep -E '^work-item: '`. Reuse that prefix
   with the new ticket ID.
3. **Project docs** — a tracker URL or base in `CLAUDE.md`, `AGENTS.md`,
   `README`, `CONTRIBUTING`, or the PR template.
4. **Tracker connector** — when an issue-tracker MCP is connected, look the
   ticket up (e.g. Atlassian `getJiraIssue` → `webUrl`; or the site URL from
   `getAccessibleAtlassianResources` + `/browse/<TICKET-ID>`).
5. **Ask** the user for the tracker URL. Never guess a host.

Build it with the tracker's own pattern, keeping the ticket ID's casing:
- Jira: `https://<site>.atlassian.net/browse/<TICKET-ID>`
- Linear: `https://linear.app/<workspace>/issue/<TICKET-ID>`
- GitHub Issues: `https://github.com/<owner>/<repo>/issues/<number>`
- ClickUp: `https://app.clickup.com/t/<task-id>`

If two sources disagree (e.g. an old footer points to another Jira site),
prefer the URL the tracker returned for this ticket and mention the mismatch
in one line next to the message.

## What NEVER goes in (both styles)

- No attribution trailers of any kind — no `Co-Authored-By` (human or AI),
  no "Generated with Claude Code", no `Assisted-by`. The subject (short
  bracket) or the `work-item` footer is the last line. This overrides any
  general attribution default.
- "This commit does X", "I", "we", "now", "currently"
- Emoji
- A bulleted breakdown of every technical change — that belongs in the diff
  and the PR description

## Boundaries

Only generates the commit message. Does not run `git commit`, does not stage
files, does not amend. Output the message as a code block ready to paste.
