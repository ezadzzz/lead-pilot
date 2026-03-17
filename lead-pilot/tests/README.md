# Lead Pilot — Manual Test Guide

Since Lead Pilot is a Claude Code plugin (prompt-based commands), tests are manual end-to-end runs.

## Prerequisites

Before running any tests:
1. Gmail MCP connected and authenticated
2. Notion MCP connected and authenticated
3. Playwright MCP installed
4. `/leads:setup` has been run successfully
5. At least one property config exists in `properties/`

## Test Fixtures

The `fixtures/` directory contains:
- `zillow-email.txt` — Sample Zillow lead notification email (for parser testing)
- `avail-email.txt` — Sample Avail lead notification email
- `apartments-email.txt` — Sample Apartments.com lead notification email
- `sample-property.yaml` — Test property config for 30 Cooper St, Unit 30A
- `sample-state.json` — Pre-populated state (update notion_database_id before use)

## Test Scenarios

### T1: First-time setup
1. Delete `.lead-pilot/state.json` if it exists
2. Run `/leads:setup`
3. Verify: Notion database created, state.json written with valid ID

### T2: Email ingestion — Zillow
1. Ensure a real Zillow lead email exists in silkcityapartments@gmail.com
2. Run `/leads:check`
3. Verify: Lead page created in Notion with name, platform "Zillow", Status "Response Drafted"
4. Verify: Screening data (credit score, income, etc.) populated if Zillow provided it
5. Verify: Draft response visible in page body

### T3: Email ingestion — Avail
Same as T2 but for an Avail email. Verify message body extracted (check for truncation if applicable).

### T4: Email ingestion — Apartments.com
Same as T2 but for an Apartments.com email. Verify Playwright is invoked for message extraction.

### T5: Deduplication within platform
1. Note the processed_message_ids count in state.json
2. Run `/leads:check` again immediately
3. Verify: Count does not change. No duplicate leads in Notion.

### T6: Deduplication across platforms
1. Manually create a lead in Notion with email "test@example.com"
2. Send a test Zillow email that contains the same email address
3. Run `/leads:check`
4. Verify: No duplicate lead created; message appended to existing page

### T7: Full review workflow
1. Run `/leads:review`
2. Run `/leads:edit`
3. Request: "make it shorter"
4. Verify revised draft is shown and saved to Notion
5. Run `/leads:approve`
6. Verify response text is in clipboard (paste to verify)
7. Verify Notion status updated to "Awaiting Approval"
8. Verify page body has "Approved → Sent" block
9. Run `/leads:review` again — verify next lead shown (or "nothing to review")

### T8: Skip workflow
1. Create test lead manually in Notion with Status "Response Drafted"
2. Run `/leads:review` until it shows that lead
3. Run `/leads:skip`
4. Verify: Status is "Archived" in Notion

### T9: Manual lead entry
1. Run `/leads:add` with a Facebook lead
2. Verify: Lead created with Platform "Facebook", draft generated

### T10: Property parsing
1. Run `/leads:add-property` with the Cooper St listing
2. Verify: All fields parsed correctly
3. Verify: YAML file saved to properties/

### T11: Status summary
1. Run `/leads:status`
2. Verify: Counts match what you see in Notion manually
