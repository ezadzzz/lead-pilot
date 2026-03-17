---
name: lead-pilot-core
description: Shared logic for Lead Pilot — state management, property loading and matching, Notion helpers, and error handling patterns. Include this skill in every Lead Pilot command.
---

# Lead Pilot Core

## State Management

State is stored in `.lead-pilot/state.json` relative to the plugin's working directory (the same directory as `plugin.json`).

### State Schema

{
  "last_checked_date": "YYYY/MM/DD",
  "processed_message_ids": ["gmail_message_id_1", "gmail_message_id_2"],
  "notion_database_id": "notion-db-uuid-here"
}

### Reading State

1. Use the Read tool to read `.lead-pilot/state.json`
2. If the file does not exist, treat state as: { "last_checked_date": null, "processed_message_ids": [], "notion_database_id": null }
3. Parse as JSON

### Writing State

1. Make your changes to the parsed state object
2. Use the Write tool to write the updated object back to `.lead-pilot/state.json`
3. Always write the full object — never partial updates
4. Write immediately after each successful operation so state is not lost if a later step fails

### Updating last_checked_date

After a successful /leads:check run, update `last_checked_date` to today's date in `YYYY/MM/DD` format (e.g., "2026/03/16"). This is used as the Gmail `after:` filter on the next run.

### Tracking Processed Message IDs

Whenever an email is successfully processed (lead created or updated in Notion), add its Gmail message ID to `processed_message_ids`. Always check this list before processing any email — if the ID is already present, skip the email silently.

Note: This list grows indefinitely in V1. That is acceptable for the expected volume of a small landlord's leads.

---

## Property Config Management

Property configs are YAML files in the `properties/` directory. Each file describes one rental unit.

### Loading All Properties

1. Use the Glob tool to find all `.yaml` files in `properties/` that are NOT `_template.yaml`
2. For each file, use the Read tool to read its contents
3. Parse as YAML (the YAML structure follows the template in `properties/_template.yaml`)
4. Return a list of parsed property objects, each tagged with its source filename

Exclude `properties/inactive/` directory — properties there are archived.

### Matching a Lead to a Property

Given an address string from an email (e.g., "30 Cooper St, Manchester, CT 06040"):

1. Load all properties
2. For each property, construct a canonical address string: "{address}, {city}, {state} {zip}"
3. Compare the email address string against each canonical address using these rules (in order):
   - Exact match (case-insensitive): highest confidence
   - Street number + street name match: e.g., "30 Cooper" appears in both strings — high confidence
   - Street name + city match: lower confidence, surface as a suggestion if other fields are ambiguous
4. Return the best match, or null if no confident match found

If null, create the lead in Notion with Property = "Unmatched" and log the raw address string in the page body. Print a warning to the user:

  ⚠ Could not match address "{raw_address}" to a property config. Lead created as "Unmatched". Add a property config or assign the property manually in Notion.

---

## Notion Database Schema

When creating the Notion database (during /leads:setup), use these properties:

| Property Name | Notion Type | Notes |
|---|---|---|
| Name | title | Lead's full name |
| Email | email | Lead's email address — primary dedup key |
| Phone | phone_number | Optional |
| Property | rich_text | Free-text — which unit they're asking about |
| Platforms | multi_select | Options: Zillow, Avail, Apartments.com, Facebook |
| Status | select | Options: New, Response Drafted, Awaiting Approval, Sent, Screening, Archived, Needs Manual Review |
| Priority | select | Options: High, Medium, Low |
| Last Message | date | Timestamp of most recent message |
| Auto-Send Eligible | checkbox | For future auto-send rules |
| Move-In Date | date | Zillow screening data |
| Credit Score | rich_text | Zillow screening data (e.g., "620-659") |
| Income | number | Zillow screening data (annual, in dollars) |
| Pets | checkbox | Zillow screening data |
| Lease Length | rich_text | Zillow screening data (e.g., "18 months") |
| Occupants | number | Zillow screening data |

The `notion_database_id` returned from creating this database must be saved to state.json.

### Notion Page Body Format

When appending messages to a lead's page body, use this format for each entry:

  [YYYY-MM-DD H:MM AM/PM] [Platform] [Direction]
  "Message text here"

Where:
- Direction is: Inbound, Draft Response, or Approved → Sent
- Platform is the source platform name or Manual for /leads:add entries

Use Notion paragraph blocks for each message entry.

---

## Deduplication (Cross-Platform)

When ingesting a new lead:

1. Read `notion_database_id` from state
2. Search the Notion database for a page where `Email` property equals the lead's email address
3. If found: this is an existing lead — append the new message to the page body and update `Platforms` and `Last Message`. Do NOT create a new page.
4. If not found by email: search for pages where `Name` matches the lead's name AND `Property` matches the lead's property
5. If a name+property match is found: surface a warning to the user:
     ⚠ Possible duplicate: "{name}" is already in the database for {property} (via {existing platform}). This may be the same person. Check Notion to merge manually if needed.
   Then create the new lead as a separate page — do not auto-merge.
6. If no match: create a new lead page

---

## Error Handling Patterns

Apply these patterns in every command:

**Gmail MCP unavailable:**
  ❌ Gmail MCP is not responding. Check your Gmail MCP connection and try again.
  (No state was updated. Running /leads:check again will retry safely.)
Do not update state.json. Exit.

**Notion MCP unavailable:**
  ❌ Notion MCP is not responding. Check your Notion MCP connection and try again.
  (No state was updated. Running /leads:check again will retry safely.)
Do not update state.json. Exit.

**Playwright scrape fails:**
Create the lead in Notion with the data available from the email. Set Status to "Needs Manual Review". Print:
  ⚠ Could not scrape message from {platform} for lead {name}. Lead created with partial data. Check {platform} directly for the full message.

**Response engine fails:**
Leave lead Status as "New". Print:
  ⚠ Could not draft response for {name}. Lead saved. Run /leads:review to retry.

**Notion page creation fails for a single lead:**
Print the error for that lead and continue processing remaining leads. Do not add the failed email's message ID to `processed_message_ids` — it will be retried on the next /leads:check run.
