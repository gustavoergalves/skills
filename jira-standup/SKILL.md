---
name: "jira-standup"
description: "Write my short daily standup post (Yesterday / Today) from my Jira activity on the Backend/Ops Board. Use when asked for a standup, a daily update, or what I did yesterday and will do today."
argument-hint: "Optional: output language (default English)"
user-invocable: true
disable-model-invocation: false
---

# Jira standup

Build a short standup post from my Jira activity on the Backend/Ops Board (project `BE`): ticket keys and a few words
each, nothing more.

## Output

Reply with only the post, in English unless `$ARGUMENTS` names another language, in this shape:

```
**Yesterday**
- BE-848: created and fixed the bug (on staging)
- BE-840: updated the ticket and re-ran QA
- BE-844 / BE-845: sent to QA
- BE-833: started
- BE-819: in code review

**Today**
- BE-844 / BE-845: follow up on QA
- BE-848: self-test
- BE-833: keep going
- BE-819: address review and merge
```

- One line per ticket: `KEY: phrase`, the phrase 2–6 words. No ticket titles, links, explanations or blockers section.
- Tickets with the same phrase share one line: `BE-844 / BE-845`.
- An empty section gets a single line: `- nothing on Jira`.
- No text before or after the post. Do not post it to Slack or transition any ticket.

## Jira context

- Site 12thbean.atlassian.net, cloudId `3b1f92fc-6b5e-4513-8ef2-855a9e075212`.
- Board "Backend/Ops Board" (id 187), scrum, project `BE`.
- Workflow: To Do → In Progress → Code Review → SELF-TESTING (merged, on staging) → Testing (QA) → Done.
- The Atlassian tools are deferred. Load `mcp__claude_ai_Atlassian_MCP__searchJiraIssuesUsingJql`,
  `mcp__claude_ai_Atlassian_MCP__getWorkActivity` and `mcp__claude_ai_Atlassian_MCP__listJiraIssueComments` with
  ToolSearch first.
- Do not call `getJiraBoardSprintData` (about 75k characters of output) or fetch changelogs: the JQL below answers both.

## Steps

1. **Window.** `N = 3` on Monday, otherwise `N = 1`. The window runs from `startOfDay(-N)` to `now()`, so moves made
   this morning count as Yesterday. `FROM` is that start date as `YYYY-MM-DD`.

2. **Fetch everything in one parallel batch.** JQL calls use
   `fields: ["summary", "status", "issuetype", "reporter", "created"]` and `maxResults: 50`.
   - Touched: `project = BE AND updated >= startOfDay(-N) AND (assignee = currentUser() OR reporter = currentUser() OR status CHANGED BY currentUser() DURING (startOfDay(-N), now()))`
   - Moves, one query per status `S` in In Progress, Code Review, SELF-TESTING, Testing, Done:
     `project = BE AND status CHANGED TO "S" BY currentUser() DURING (startOfDay(-N), now())`
   - Open sprint: `project = BE AND sprint in openSprints() AND assignee = currentUser() AND statusCategory != Done`
   - My comments: `getWorkActivity` with `activity: "all"`, `types: "comments"`, `from: FROM`,
     `itemsPerSection: "20"`. The ticket key is in each comment's `webUrl`.

3. **Yesterday.** Every ticket from Touched, Moves or My comments. Phrase it from its strongest event, adding a second
   one with "and" only when it says something new ("created and fixed the bug", "updated the ticket and re-ran QA"):

   | Event in the window                     | Phrase                                                    |
   |-----------------------------------------|-----------------------------------------------------------|
   | I created it                            | "created", or "created and fixed the bug" for a merged Bug |
   | moved to Done                           | done                                                      |
   | moved to Testing                        | sent to QA                                                |
   | moved to SELF-TESTING                   | merged (on staging)                                       |
   | moved to Code Review                    | opened the PR                                             |
   | moved to In Progress                    | started                                                   |
   | my comment starting with `/qa`          | re-ran QA                                                 |
   | any other comment of mine               | updated the ticket                                        |
   | only updated                            | its current status: in progress, in code review, self-testing, in QA |

   Read comment text with `listJiraIssueComments` (`orderBy: "-created"`, `maxResults: 5`) only for tickets whose
   phrase depends on it. Order lines by most recent activity first.

4. **Today.** Open sprint tickets plus Yesterday tickets not yet Done. Drop To Do tickets and Story/Epic tickets
   (their sub-tasks carry the work). Phrase each from its current status:

   | Status       | Phrase                   |
   |--------------|--------------------------|
   | In Progress  | keep going               |
   | Code Review  | address review and merge |
   | SELF-TESTING | self-test                |
   | Testing      | follow up on QA          |

   Keep the Yesterday order; tickets that only appear in Today go last.
