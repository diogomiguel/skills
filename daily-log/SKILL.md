---
name: daily-log
description: Collect today's work activity across git, Jira, Confluence, Slack, Google Calendar, and Google Docs meeting notes into a standup-ready log. Invoke when the user asks "what did I do today", wants a daily log, standup notes, or a progress report. Pulls git commits from the configured repos, Jira/Confluence activity via the atlassian MCP, Slack messages via the slack MCP, and meetings/notes via Composio googlecalendar + googledocs.
---

# daily-log

Produce a chronological log of the user's work today across git, Jira, Confluence, Slack, Google Calendar, and Google Docs meeting notes. Output is a flat bullet list grouped by source, timestamped where possible, suitable for pasting into standup or a weekly report.

## Configuration

This skill reads per-user settings from `~/.config/daily-log/config.json`:

```json
{
  "email": "you@example.com",
  "repos": [
    "/absolute/path/to/repo-a",
    "/absolute/path/to/repo-b"
  ],
  "jira_domain": "yourorg.atlassian.net",
  "timezone": "Europe/London",
  "save_path": "~/Documents/daily-logs"
}
```

If the config is missing on first run, ask the user for these values and offer to write the file. Do not proceed with hardcoded defaults — the rest of the skill substitutes these values everywhere it refers to `<email>`, `<repo>`, `<jira_domain>`, `<timezone>`, `<save_path>`.

Discovery fallbacks if a field is absent from config:
- `email` → `git config --global user.email`
- `jira_domain` → first `url` returned by `mcp__atlassian__getAccessibleAtlassianResources` (strip `https://`)
- `timezone` → `date +%Z` on the host, or ask the user
- `repos` → no safe fallback; ask the user explicitly

## Arguments

Optional args via `$ARGUMENTS`:
- `yesterday` — shift window to yesterday instead of today
- `week` — shift window to the last 7 days
- `since:<ISO-date>` — custom start (e.g. `since:2026-04-20`)
- `standup` — output in Yesterday / Today / Blockers format instead of flat log
- `slack` — output a Slack-ready async standup post (see step 8b)

Default: today, flat chronological log.

## Steps

### 1. Resolve the time window

- Default: `since = today 00:00 local`, `until = now`
- `yesterday`: `since = yesterday 00:00`, `until = today 00:00`
- `week`: `since = 7 days ago 00:00`
- `since:<ISO>`: parse as start, `until = now`

Compute the ISO-8601 timestamps once and reuse. Use `<timezone>` from config for the local offset.

### 2. Git — commits across configured repos

For each `<repo>` in `repos`, run:

```bash
git -C <repo> log --author="<email>" --since="<ISO>" --until="<ISO>" --pretty=format:"%h|%aI|%s" --no-merges
```

Also check for uncommitted work in the current repo: `git -C <cwd-repo> status --short` and `git -C <cwd-repo> diff --stat` — useful signal of in-progress work.

Also list branches with recent commits: `git -C <repo> for-each-ref --sort=-committerdate refs/heads/ --format='%(refname:short)|%(committerdate:iso8601)'` and filter by window.

Run the `git log` calls in parallel Bash tool calls.

### 3. Jira — issue activity

Use the atlassian MCP. First get the account id once via `mcp__atlassian__atlassianUserInfo` (cache it mentally — it's stable).

Then run `mcp__atlassian__searchJiraIssuesUsingJql` with:

```
(assignee = currentUser() OR reporter = currentUser() OR comment ~ currentUser()) AND updated >= "<YYYY-MM-DD HH:mm>"
```

The JQL `updated` clause accepts `YYYY-MM-DD HH:mm` format. Use the window start.

For each returned issue, report: key, summary, status, and whether the user is assignee/reporter. If the window is small, fetch the issue via `mcp__atlassian__getJiraIssue` to get the latest comment/transition the user made — but only do this for ≤5 issues to avoid noise.

Cloud id: resolve via `mcp__atlassian__getAccessibleAtlassianResources` on first call.

**Response size guard**: the default `fields` list may return >30k tokens for large result sets. Request only `["summary", "status", "issuetype", "priority", "updated"]` unless you need more, and cap `maxResults` at 30.

### 4. Confluence — pages created/edited/commented

Use `mcp__atlassian__searchConfluenceUsingCql` with:

```
(contributor = currentUser() OR creator = currentUser()) AND lastmodified >= "<YYYY-MM-DD>"
```

Report: page title, space, last-modified timestamp, and role (creator vs contributor). For pages edited, optionally fetch via `mcp__atlassian__getConfluencePage` to grab the title/excerpt — but skip content bodies by default (too noisy for a daily log).

Also check for comments: `searchConfluenceUsingCql` with `type = comment AND creator = currentUser() AND created >= "<YYYY-MM-DD>"` — report as "commented on <page>".

Use the same cloud id resolved in step 3.

### 5. Google Calendar — meetings attended

Use `mcp__composio__GOOGLECALENDAR_EVENTS_LIST_ALL_CALENDARS` with:
- `time_min` = window start as RFC3339 with local offset (e.g. `2026-04-22T00:00:00+02:00`)
- `time_max` = window end as RFC3339 with local offset
- `single_events` = true
- `response_detail` = `minimal` (use `summary_view` from response)

**Important**: Use explicit timezone offset, not `Z` (UTC), or the day boundary will be wrong. Use `<timezone>` from config. Compute the current offset via `mcp__composio__GOOGLECALENDAR_GET_CURRENT_DATE_TIME` with `timezone: "<timezone>"` (accounts for DST).

Report: time, title, attendee count (if available). Filter out all-day events unless they look meaningful (e.g. contain "OOO", "PTO" — still worth surfacing).

Skip declined events if the response indicates them.

### 6. Google Docs — meeting notes

Use `mcp__composio__GOOGLEDOCS_SEARCH_DOCUMENTS` with:
- `modified_after` = window start (RFC3339 with offset)
- `query` = empty (get all modified docs — filter client-side)
- `max_results` = 20

Filter results to docs whose `name` matches meeting-note patterns: `"Notes by Gemini"`, `"Notes: "`, `"Notes -"`, or names matching calendar event titles from step 5 (fuzzy match on ≥3 shared words, or substring match on a distinctive title word).

For matched notes, fetch content via `mcp__composio__GOOGLEDOCS_GET_DOCUMENT_PLAINTEXT` and extract:
- The "Summary" section (Gemini notes have a heading like "Summary" or "Meeting summary")
- The "Action items" section if present, filtering to items assigned to the user (match on first name or "me")

Report: doc title, link (`https://docs.google.com/document/d/<id>`), 1-2 sentence summary, and any action items assigned to the user.

Skip docs ≥10k chars unless the user explicitly asks for full meeting content — keep it to summaries.

### 7. Slack — messages sent today

Use `mcp__plugin_slack_slack__slack_search_public_and_private` with query:

```
from:me after:<YYYY-MM-DD>
```

(Slack's `after:` is inclusive of the date; use the window start date. For `yesterday`, also add `before:` to upper-bound.)

Report: channel, timestamp, message excerpt (first ~100 chars). Group by channel.

Slack's search is best-effort — if it returns nothing surprising, that's fine, don't retry aggressively.

### 8. Assemble the log

**Default (flat chronological):**

```
# Daily log — <date>

## Meetings
- <HH:mm> <title> (<N> attendees) — <1-2 sentence summary from notes, if available>
  - Action items: <items assigned to you>
  - Notes: <doc link>

## Git
- <HH:mm> [<repo>] <hash> <subject>
...

## Jira
- <KEY> <status> — <summary> (<role: assignee/reporter/commenter>)
...

## Confluence
- <page title> (<space>) — <created|edited|commented>
...

## Slack
### #<channel>
- <HH:mm> <excerpt>
...

## In-progress (uncommitted)
- <file> — <lines +/->
```

**Slack async standup mode** (`slack` arg):

Collect **two** windows:
- Yesterday: previous working day (if today is Monday, use last Friday)
- Today: the current day (in-progress work + today's Jira assignments + today's calendar)

Ask the user how they plan to deliver the message, because the link syntax differs:

- **Send via the Slack MCP** (`slack_send_message` or `slack_send_message_draft`) — use Slack mrkdwn link syntax: `<https://<jira_domain>/browse/<KEY>|<KEY>>`. This is the only place `<url|label>` works.
- **Paste into the Slack composer textbox** (most common) — the composer does NOT parse `<url|label>`; it treats it as literal text. Use one of:
  - Markdown-style links `[KEY](https://<jira_domain>/browse/<KEY>)` — renders as a link if the user has "Format messages with markup" enabled, or when using a plugin that supports markdown.
  - Plain URLs inline — Slack auto-linkifies them: `• KEY https://<jira_domain>/browse/KEY — …`

Default to asking which format the user wants before generating. If they don't specify, produce the MCP/mrkdwn version and note the alternatives.

Example (MCP/mrkdwn format):

```
*What have I done yesterday?*
• <https://<jira_domain>/browse/PROJ-123|PROJ-123> — shipped <feature>
• Reviewed <https://github.com/<org>/<repo>/pull/<N>|#<N>>
• Paired with @x on <topic>

*What will I do today?*
• <https://<jira_domain>/browse/PROJ-124|PROJ-124> — continue review cycle
• Follow up on <https://<jira_domain>/browse/PROJ-125|PROJ-125> QA feedback

*Impediments*
• None
(or: • Blocked on <TICKET> waiting for <X>)

*Notes for the team*
• <only populate if signal exists>
```

Rules for the Slack mode:
- Use `*bold*` (single asterisks) not `**bold**` — Slack mrkdwn is not Markdown
- Use `•` for bullets, not `-` (renders more cleanly in Slack)
- Every Jira key in the message should be linkified — format depends on delivery channel (see above)
- Keep each bullet to ≤1 line when possible — async standup should skim in 10 seconds
- Derive "Yesterday" from: git commits merged + Jira tickets you transitioned/commented on + notable Slack messages + meeting outcomes
- Derive "Today" from: in-progress uncommitted work + Jira tickets assigned to you with status In Progress/To Do + today's calendar meetings (only mention meetings if they're load-bearing, e.g. a review session — skip recurring 1:1s)
- For "Impediments": only include if evident (Jira status = Blocked, Slack messages mentioning "stuck"/"blocked", or user tells you). Default to "• None"
- For "Notes for the team": leave empty by default. Only populate if there's a signal in calendar (OOO event) or Slack (announced time off). If no signal, omit the section
- After generating, offer to copy to clipboard via `pbcopy` or send directly via the slack MCP

**Standup mode** (`standup` arg):

```
## Yesterday
- <bullet per meaningful item from git + jira + slack>

## Today
- <derived from in-progress work + open Jira assignments>

## Blockers
- <none, unless evident from Jira status=Blocked or Slack messages mentioning "blocked"/"stuck">
```

Sort git commits by authoredDate ascending. Keep Slack grouped by channel (chronology within channel is fine). Omit sections that have zero items.

### 9. Offer to save

After printing, ask the user if they want it saved. If yes, write to `<save_path>/YYYY-MM-DD.md` (expand `~`). Do not save proactively.

## Notes

- If a source errors (e.g. Jira MCP unavailable), continue with the others and note the gap at the top.
- Do not call Slack search with no query — always scope with `from:me`.
- Keep excerpts short. The user wants signal, not transcripts.
- Run the source calls in parallel (one message, multiple tool calls) — git, Jira, Confluence, Slack, Calendar, and initial Docs search are all independent. Only fan out `GOOGLEDOCS_GET_DOCUMENT_PLAINTEXT` after the search + calendar results are back (matching needs both).
- Composio tools (googlecalendar, googledocs) require active connections. If a call returns "no active connection", tell the user to run the skill again after reconnecting via Composio — don't try to re-auth mid-flow.
- Atlassian Jira JQL responses can exceed the MCP token limit. Always pass an explicit minimal `fields` array and cap `maxResults` ≤ 30.
