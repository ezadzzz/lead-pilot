---
name: leads:check
description: Poll Gmail for new rental lead notification emails. Parse each email, create or update leads in Notion, draft responses, and update state. Skips emails already processed.
---

# /leads:check

@lead-pilot-core
@lead-pilot-email-parser
@lead-pilot-response-drafter

## What This Command Does

Polls Gmail for new lead notification emails from Zillow, Avail, and Apartments.com. For each new email, parses the lead, creates/updates the lead in Notion, drafts a response, and records the email as processed.

## Steps to Follow

### 1. Load state

Read `.lead-pilot/state.json`. Extract:
- `last_checked_date` — Gmail date filter (use today's date if null)
- `processed_message_ids` — set of already-processed email IDs
- `notion_database_id` — required; if missing, print error and exit:
  ```
  ❌ Lead Pilot is not set up. Run /leads:setup first.
  ```

### 2. Check MCP connections

Before making any MCP calls, verify Gmail and Notion MCPs are accessible. If either fails, print the appropriate error from @lead-pilot-core and exit without updating state.

### 3. Search Gmail for lead emails

Run three Gmail searches using the Gmail MCP `gmail_search_messages` tool:

```
from:convo.zillow.com after:{last_checked_date}
from:support@avail.co subject:"new message on Avail" after:{last_checked_date}
from:lead@apartments.com after:{last_checked_date}
```

Combine results into a single list of emails. For each email, check if its message ID is in `processed_message_ids`. If yes, skip it.

Print:
```
📬 Found {N} new email(s) to process ({zillow_count} Zillow, {avail_count} Avail, {apartments_count} Apartments.com)
```

If no new emails:
```
✅ No new leads. ({total_already_processed} emails already processed since {last_checked_date})
```
Update `last_checked_date` to today. Write state. Exit.

### 4. For each new email: parse

Use @lead-pilot-email-parser to extract the lead object. Handle each platform's specific logic (Playwright for Apartments.com, truncation check for Avail).

If parsing fails entirely (cannot extract even a name or email), log the error and skip to the next email — do NOT add to processed_message_ids (will retry next run):
```
⚠ Could not parse email {message_id}. Will retry on next check.
```

### 5. For each parsed lead: match property

Use the property matching logic from @lead-pilot-core to match `raw_address` to a property config. Proceed whether matched or unmatched (unmatched leads still get created).

### 6. For each lead: dedup check

Use the dedup logic from @lead-pilot-core:
- Search Notion for an existing lead with matching email
- If found: append new message to existing lead's page body, update Platforms and Last Message. Mark email as processed. Skip to step 8.
- If not found by email, check name+property match. If found, print the merge suggestion but create a new page anyway.

### 7. Create new lead in Notion

Use Notion MCP `notion-create-pages` to create a new lead page with all extracted properties. Page body should contain the initial inbound message in the thread format defined in @lead-pilot-core.

If creation fails: print error for this lead, do NOT add to processed_message_ids, continue to next email.

### 8. Draft response

Use @lead-pilot-response-drafter to generate a draft response. Pass the lead object and matched property config.

If drafting fails: leave Status as "New", print warning from @lead-pilot-core error patterns. Mark email as processed anyway (lead was created successfully).

### 9. Mark email as processed

Add the Gmail message ID to `processed_message_ids`. Write state immediately after each successful lead creation (don't batch — ensures partial progress is saved if something fails mid-run).

### 10. Print summary

After processing all emails:
```
✅ Lead Pilot check complete

  New leads created: {n}
  Existing leads updated: {n}
  Responses drafted: {n}
  Needs manual review: {n}
  Errors: {n}

Run /leads:review to see leads awaiting your approval.
```

Update `last_checked_date` to today's date. Write final state.
