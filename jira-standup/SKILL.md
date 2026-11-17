---
name: "jira-standup"
description: "Write my short daily standup post (Yesterday / Today) from my Jira activity on the Backend/Ops Board. Use when asked for a standup, a daily update, or what I did yesterday and will do today."
argument-hint: "Optional: output language (default English)"
user-invocable: true
disable-model-invocation: false
---

# Jira standup

Build a short standup post from my Jira activity on the Backend/Ops Board (project `BE`). Each Yesterday line tells
the ticket's journey in the window — every status move in order, who made it when it wasn't me — not just its last
event.

## Output

Reply with only the post, in English unless `$ARGUMENTS` names another language, in this shape:

```
**Yesterday**
- BE-851: implemented and opened the PR, merged and ran QA on staging
- BE-800: QA sent it back, fixed it, merged, self-tested and sent back to QA; QA approved, now in Review
- BE-854: opened the PR, merged, self-tested and sent to QA; QA approved, now in Review
- BE-848: created and fixed the bug, opened the PR
- BE-855: started

**Today**
- BE-851: finish self-test and send to QA
- BE-800 / BE-854: follow up on review
- BE-855: keep going
```

- One line per ticket: `KEY: journey`. The journey is the steps in chronological order, joined with commas and a final
  "and"; use `;` before steps made by someone else (QA, reviewer). At most ~20 words: when a ticket has more steps,
  squeeze the middle ones ("fixed it, merged") but keep the first and last, and keep every send-back.
- No ticket titles, links, explanations or blockers section.
- Tickets with the same journey share one line: `BE-854 / BE-800`.
- An empty section gets a single line: `- nothing on Jira`.
- No text before or after the post. Do not post it to Slack or transition any ticket.

## Jira context

- Site 12thbean.atlassian.net, cloudId `3b1f92fc-6b5e-4513-8ef2-855a9e075212`.
- Board "Backend/Ops Board" (id 187), scrum, project `BE`.
- Workflow: To Do → In Progress → Code Review → SELF-TESTING (merged, on staging) → Testing (QA) → Review (product
  review) → Done (or WON'T DO). Tickets can go back: QA moves Testing → In Progress, a reviewer moves Code Review →
  In Progress.
- Moving a ticket to Testing unassigns me (automation) and QA assigns themselves; moving to Review reassigns it to
  the reviewer. So `assignee = currentUser()` misses tickets that are past Testing — the queries below cover them.
- A comment of mine starting with `/qa` triggers an automated QA run on staging. The bot (author `Test Pilot`)
  posts its evidence as attachments (`run-condensed.mp4`, `*-verification.md`, `*evidence*`) within about an hour,
  so the changelog shows each run without reading comment text. Don't read comment bodies: QA reports are long HTML.
- The Atlassian tools are deferred. Load `mcp__claude_ai_Atlassian_MCP__searchJiraIssuesUsingJql`,
  `mcp__claude_ai_Atlassian_MCP__getWorkActivity` and `mcp__claude_ai_Atlassian_MCP__executeRead` with ToolSearch
  first.
- Do not call `getJiraBoardSprintData` (about 75k characters) or `getJiraIssue` with `expand: "changelog"` (it
  returns every field's full history, description bodies included). Read history only as in step 3.

## Steps

1. **Window.** `N = 3` on Monday, otherwise `N = 1`. The window runs from `startOfDay(-N)` to `now()`, so moves made
   this morning count as Yesterday. `FROM` is that start date as `YYYY-MM-DD`.

2. **Find the tickets, in one parallel batch.** JQL calls use
   `fields: ["summary", "status", "issuetype", "reporter", "created"]` and `maxResults: 50`.
   - Touched: `project = BE AND updated >= startOfDay(-N) AND (assignee = currentUser() OR reporter = currentUser() OR status CHANGED BY currentUser() DURING (startOfDay(-N), now()) OR (assignee WAS currentUser() AND status CHANGED DURING (startOfDay(-N), now())))`
     — the last clause catches tickets QA or a reviewer moved after I handed them over.
   - Open sprint: `project = BE AND sprint in openSprints() AND assignee = currentUser() AND statusCategory != Done`
   - My comments: `getWorkActivity` with `activity: "all"`, `types: "comments"`, `from: FROM`,
     `itemsPerSection: "20"`. The ticket key is in each comment's `webUrl`, its time in `created` (UTC).

3. **Read each ticket's journey.** For every ticket from Touched or My comments, read the end of its changelog with
   `executeRead` and `name: "listJiraIssueChangelogs"` (oldest-first, so read the last page). All calls of a round go
   in one parallel batch:
   - Round 1, size: `inputs: {issueIdOrKey, maxResults: 1}`, `responseFields: ["total"]`.
   - Round 2, last page: `inputs: {issueIdOrKey, startAt: max(0, total - 50), maxResults: 50}`,
     `responseFields: ["values.created", "values.author.displayName", "values.items.field", "values.items.fromString", "values.items.toString"]`.

   From the entries whose `created` is inside the window, build the ticket's timeline:
   - **Moves:** items with `field = "status"` (`fromString` → `toString`, author, time). Moves by the same person
     within 5 minutes collapse into their net move (QA going Testing → Review → In Progress in one minute is
     Testing → In Progress).
   - **QA runs:** a burst of `Attachment` items from the QA bot after one of my comments is one QA run, placed at my
     comment's time.
   - **Other comments of mine:** comments from My comments not followed by a bot run.

   If the oldest entry on the page is still inside the window, also read the page before it.

4. **Yesterday.** One line per ticket, steps in time order, each phrased from the table. A step's actor is me unless
   the table says otherwise; when a move by someone else isn't covered, write "<role> moved it to <status>" (QA for
   Testing moves, reviewer for Code Review/Review moves).

   | Event in the window                               | Phrase                                          |
   |---------------------------------------------------|-------------------------------------------------|
   | I created it                                      | "created", or "created and fixed the bug" for a Bug I also took to Code Review |
   | To Do → In Progress                               | started                                         |
   | In Progress → Code Review (first time)            | opened the PR; "implemented and opened the PR" when it also started in the window |
   | In Progress → Code Review (after a send-back)     | fixed it                                        |
   | Code Review → In Progress                         | review asked for changes                        |
   | Code Review → SELF-TESTING                        | merged                                          |
   | QA run on staging                                 | ran QA on staging ("re-ran" if not the first)   |
   | SELF-TESTING → Testing                            | self-tested and sent to QA ("sent back to QA" after a send-back); absorbs QA runs just before it |
   | Testing or Review → In Progress (someone else)    | QA sent it back (from Testing) / review sent it back (from Review) |
   | Testing → Review (someone else)                   | QA approved, now in Review                      |
   | → Done                                            | done                                            |
   | → WON'T DO                                        | closed as won't do                              |
   | other comment of mine                             | updated the ticket                              |
   | no move or comment, only updated                  | its current status: in progress, in code review, self-testing, in QA, in review |

   Merge back-to-back steps that read as one ("started" + "opened the PR" → "implemented and opened the PR").
   Order lines by most recent activity first.

   Check: BE-800 on 2026-11-16 had QA Testing → Review → In Progress (1 min apart), my In Progress → Code Review →
   SELF-TESTING, a `/qa` comment plus two bot runs, my SELF-TESTING → Testing, then QA Testing → Review. Its line is
   "QA sent it back, fixed it, merged, self-tested and sent back to QA; QA approved, now in Review".

5. **Today.** Open sprint tickets plus Yesterday tickets not yet Done. Drop To Do tickets and Story/Epic tickets
   (their sub-tasks carry the work). Phrase each from its current status and the last move:

   | Current status | Phrase                                                                           |
   |----------------|----------------------------------------------------------------------------------|
   | In Progress    | "fix QA feedback" if QA sent it back, "address review" if review did, else "keep going" |
   | Code Review    | address review and merge                                                         |
   | SELF-TESTING   | self-test; "finish self-test and send to QA" if QA already ran on staging        |
   | Testing        | follow up on QA                                                                  |
   | Review         | follow up on review                                                              |

   Keep the Yesterday order; tickets that only appear in Today go last.
