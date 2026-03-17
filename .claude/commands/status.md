---
name: leads:status
description: Show a summary of all leads by status. Quick overview of your lead pipeline.
---

# /leads:status

@lead-pilot-core

## Steps to Follow

### 1. Load state

Read state.json. Get `notion_database_id`. If missing, print setup error and exit.

### 2. Query Notion

Use Notion MCP to query the Leads database. Retrieve all lead pages (or enough to count by status). Group by Status property.

### 3. Display summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LEAD PILOT STATUS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
New (unreviewed):         {n}
Response Drafted:         {n}  ← needs your review
Awaiting Approval:        {n}
Sent:                     {n}
Screening:                {n}
Needs Manual Review:      {n}
Archived:                 {n}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total active:             {n}

Last checked: {last_checked_date}
Emails processed to date: {len(processed_message_ids)}

{if "Response Drafted" count > 0: "Run /leads:review to review {n} pending response(s)."}
{if "New" count > 0: "Run /leads:check or /leads:review to process {n} new lead(s)."}
{if all 0: "All clear! Run /leads:check to poll for new leads."}
```
