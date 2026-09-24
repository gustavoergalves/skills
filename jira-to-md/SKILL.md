---
name: "jira-to-md"
description: "Fetch one or more Jira tickets and create data/jira/<TICKET-ID>/index.md per ticket, downloading any description images into the same folder."
argument-hint: "One or more Jira ticket IDs (e.g. SK-91 SK-92 SK-104)"
metadata:
  author: "gustavo.reis@99x.io"
user-invocable: true
disable-model-invocation: false
---

# Jira to Markdown

Fetch Jira ticket(s) by ID and create a `data/jira/<TICKET-ID>/index.md` file for each one in the current project. Each ticket gets its own folder so related files (downloaded images, notes, sub-docs) live alongside `index.md`.

## Output root

All files this skill writes go under `data/` at the root of the project the agent is running in (the current working directory) — never the project root or any other folder. Create `data/` if it does not exist.

## Arguments

`$ARGUMENTS` — one or more Jira ticket IDs separated by spaces (e.g. `SK-91 SK-92 SK-104`).
If no arguments are provided, ask the user for the ticket ID(s) before proceeding.

## Steps

1. **Resolve the Atlassian cloud ID**
   Call `mcp__claude_ai_Atlassian_Rovo__getAccessibleAtlassianResources` and use the first Jira-scoped result. Skip this step if the cloud ID is already known from earlier in the session (the seeds.atlassian.net cloudId is `e60384d0-0f35-4290-a14f-62299923d1c2`).

2. **Fetch each ticket**
   For every ticket ID in `$ARGUMENTS`, call `mcp__claude_ai_Atlassian_Rovo__getJiraIssue` with:
   - `cloudId`: the ID resolved above
   - `issueIdOrKey`: the ticket ID
   - `responseContentFormat`: `"markdown"`
   - `fields`: `["summary", "issuetype", "status", "assignee", "description", "attachment"]`

   Fetch all tickets in parallel when there are multiple.

3. **Download description images** (only when the description contains image references)
   In the markdown description, embedded images come back as `![](blob:https://media...atl-paas.net/?...&width=W&height=H...)`. The blob `id` is a Media Services UUID and does **not** map to a Jira attachment id, so images must be matched to the issue's `attachment[]` array by **pixel dimensions**.

   **Auth:** the Atlassian MCP token canNOT download attachments (returns HTTP 403). Use a personal Jira API token via `curl` basic auth. Read credentials in this order: env vars `JIRA_EMAIL` + `JIRA_API_TOKEN`, otherwise `~/.netrc` (entry `machine api.atlassian.com`). If neither is available and the description has images, skip downloads, write the file with each image ref replaced by `> _Image not downloaded (no Jira API token configured)._`, and warn in the summary.

   For each `attachment[]` entry whose `mimeType` starts with `image/`:
   - Download to a temp file and confirm success:
     ```bash
     curl -sS -L -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -o "<tmp>" \
       -w "%{http_code}" "<attachment.content>"     # expect 200
     # (or, when using ~/.netrc:  curl -sS -L --netrc ... )
     ```
   - Read its real dimensions with `file "<tmp>"` (e.g. `PNG image data, 1691 x 927`).

   Then walk the description's image refs in document order and match each ref's `width`x`height` to the attachment with the same dimensions. Save each matched file into `data/jira/<TICKET-ID>/` as `image-01.<ext>`, `image-02.<ext>`, ... (N = order of appearance in the description; `<ext>` from the mimeType, e.g. `png`/`jpg`). If two images share identical dimensions, fall back to attachment document order for that group and note the ambiguity in the summary.

4. **Write the markdown file**
   Write `data/jira/<TICKET-ID>/index.md` (create the `data/jira/<TICKET-ID>/` folder if it does not exist) with this structure:

   ```markdown
   # <TICKET-ID>: <summary>

   **Type:** <issuetype.name>
   **Status:** <status.name>
   **Assignee:** <assignee.displayName> (or "Unassigned" if null)

   ## Description

   <description - use the markdown content from the API response, with every blob image ref rewritten to its local file, e.g. ![<original filename>](image-01.png). If empty, write "No description provided.">
   ```

5. **Report results**
   After all files are written, output a short summary listing each `index.md` created, each image downloaded (local name <- original Jira filename + dimensions), and anything that failed (ticket not found, permission error, HTTP 403, missing token, or an image whose dimensions matched no attachment).

## Rules

- Never overwrite an existing file without warning. If `data/jira/<TICKET-ID>/index.md` already exists, ask the user whether to overwrite it. Do not delete or touch other files in the ticket folder.
- Match images by pixel dimensions, not by attachment array order (the blob Media UUID is not the attachment id). Never guess a mapping; if no attachment dimension matches a ref, leave the ref in place and flag it in the summary.
- Keep images in their original format (PNG stays PNG); do not convert or resize.
- Do not add extra sections (acceptance criteria, subtasks, attachment tables) unless the user explicitly asks for them.
- Match the ticket ID casing exactly as returned by the API (e.g. `SK-91`, not `sk-91`).
- Never write the API token into the command, the repo, or `index.md`. Read it only from env vars or `~/.netrc`.
