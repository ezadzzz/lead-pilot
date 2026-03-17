---
name: leads:setup
description: First-run setup for Lead Pilot. Creates the Notion Leads database with the correct schema and initializes the state file. Run this once before using any other /leads: commands.
---

# /leads:setup

@lead-pilot-core

## What This Command Does

1. Checks if setup has already been run (state.json exists with a `notion_database_id`)
2. If already set up, confirms with the user before proceeding
3. Asks the user to confirm which Notion workspace/page to create the database in
4. Creates the "Leads" database with the full schema defined in @lead-pilot-core
5. Creates three default views in the database
6. Initializes `.lead-pilot/state.json`
7. Prints a success summary with next steps

## Steps to Follow

### 1. Check existing setup

Read `.lead-pilot/state.json`. If `notion_database_id` is set:
```
Lead Pilot is already set up. Notion database ID: {notion_database_id}

Run setup again to recreate the database? This will NOT delete existing leads. (yes / no)
```
If user says no, exit.

### 2. Confirm Notion workspace

Print:
```
Where should the Leads database be created in Notion?

Options:
  A) In my Notion workspace root
  B) Inside a specific page (I'll provide the page name or URL)
```

If B: ask the user to paste the page URL or name. Use the Notion MCP `notion-search` tool to find the page and get its ID.

### 3. Create the Notion database

Use the Notion MCP `notion-create-database` tool to create a database named "Leads" with the schema defined in the Notion Database Schema section of @lead-pilot-core.

Parent: the workspace root or the page ID from step 2.

If creation fails: print the error, suggest checking Notion MCP connection, and exit without writing state.

### 4. Create database views

Use the Notion MCP `notion-create-view` tool (or equivalent) to create these views on the new database:

1. **Needs Review** — Filter: Status = "Response Drafted", Sort: Last Message ascending
2. **By Property** — Group by: Property field
3. **All Active** — Filter: Status is NOT "Archived"

If view creation fails, print a warning but continue — the database is functional without views.
```
⚠ Could not create views automatically. Create them manually in Notion using the schema in the Lead Pilot docs.
```

### 5. Initialize state.json

Write the following to `.lead-pilot/state.json`:

```json
{
  "last_checked_date": null,
  "processed_message_ids": [],
  "notion_database_id": "{id from step 3}"
}
```

### 6. Ask about backfill

```
Lead Pilot is set up! Before your first /leads:check, do you want to backfill existing leads?

  A) Check only from today forward (recommended for fresh start)
  B) Check the last 7 days
  C) Check the last 30 days
  D) Check from a specific date (I'll provide it)
```

Based on the user's answer, set `last_checked_date` in state.json:
- A: today's date in YYYY/MM/DD format
- B: 7 days ago
- C: 30 days ago
- D: the date the user provides (validate it is a real date)

Write the updated state.json.

### 7. Print success summary

```
✅ Lead Pilot setup complete!

  Notion database: Leads ({notion_database_id})
  Checking emails from: {last_checked_date}

Next steps:
  1. Add your first property: /leads:add-property
  2. Check for new leads: /leads:check
  3. Set up automatic polling: /loop 15m /leads:check
```
