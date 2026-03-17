---
name: leads:review
description: Show the next rental lead awaiting review. Displays the full message thread, lead details, and drafted response. After reviewing, use /leads:approve, /leads:edit, or /leads:skip.
---

# /leads:review

@lead-pilot-core
@lead-pilot-response-drafter

## Steps to Follow

### 1. Load state

Read state.json. Get `notion_database_id`. If missing, print setup error and exit.

### 2. Find next lead to review

Use Notion MCP `notion-fetch` or `notion-search` to query the Leads database. Filter:
- Status = "Response Drafted"

Sort by: Last Message ascending (oldest first — FIFO review queue).

If no leads match:
```
✅ Nothing to review. All caught up!

  Run /leads:check to poll for new leads.
  Run /leads:status for a full summary.
```
Exit.

### 3. Fetch lead details

Use Notion MCP to fetch the full lead page (properties + page body blocks).

### 4. Display lead details

Print the following to the terminal:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LEAD: {name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Platform:   {platforms}
Property:   {property}
Email:      {email}
Status:     {status}

SCREENING (from Zillow):
  Move-in:      {move_in_date or "—"}
  Credit score: {credit_score or "—"}
  Income:       {income formatted as "$XX,XXX" or "—"}
  Pets:         {yes/no or "—"}
  Lease:        {lease_length or "—"}
  Occupants:    {occupants or "—"}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MESSAGE THREAD:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{full thread from page body, in chronological order}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DRAFTED RESPONSE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{most recent draft from page body}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Actions: /leads:approve  |  /leads:edit  |  /leads:skip
```

### 5. Stay in context

After printing, remain in context so the user can immediately run `/leads:approve`, `/leads:edit`, or `/leads:skip` without re-fetching. The current lead's Notion page ID should be retained in the session for the next command.
