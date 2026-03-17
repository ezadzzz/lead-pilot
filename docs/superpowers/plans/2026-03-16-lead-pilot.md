# Lead Pilot Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that automatically ingests rental leads from Zillow, Avail, and Apartments.com via Gmail, stores them in Notion, drafts responses using an existing Claude Skill, and provides slash commands for reviewing and approving responses.

**Architecture:** Plugin-based slash commands backed by shared skills. All platform I/O (Gmail, Notion, Playwright) happens through MCP tool calls Claude makes when executing commands. State is persisted in `.lead-pilot/state.json`. Property configs are YAML files in the `properties/` directory.

**Tech Stack:** Claude Code plugin (markdown command/skill files), Gmail MCP, Notion MCP, Playwright MCP, YAML (js-yaml or PyYAML if needed), JSON for state

---

## Chunk 1: Plugin Scaffold + State Management + Setup Command

### Task 1: Initialize plugin directory structure

**Files:**
- Create: `lead-pilot/plugin.json`
- Create: `lead-pilot/.lead-pilot/.gitignore`
- Create: `lead-pilot/properties/_template.yaml`

- [ ] **Step 1: Create the plugin directory**

```bash
mkdir -p lead-pilot/skills lead-pilot/commands lead-pilot/properties lead-pilot/.lead-pilot lead-pilot/tests/fixtures
```

- [ ] **Step 2: Verify the structure was created**

```bash
find lead-pilot -type d
```

Expected output:
```
lead-pilot
lead-pilot/skills
lead-pilot/commands
lead-pilot/properties
lead-pilot/.lead-pilot
lead-pilot/tests
lead-pilot/tests/fixtures
```

- [ ] **Step 3: Create plugin.json**

```json
{
  "name": "lead-pilot",
  "version": "0.1.0",
  "description": "Automates rental lead ingestion, response drafting, and review workflow for property managers",
  "commands": [
    {
      "name": "leads:setup",
      "description": "First-run setup — creates Notion database and initializes state file",
      "path": "commands/setup.md"
    },
    {
      "name": "leads:check",
      "description": "Poll Gmail for new leads, ingest them, and draft responses",
      "path": "commands/check.md"
    },
    {
      "name": "leads:review",
      "description": "Show next lead needing review — full thread and drafted response",
      "path": "commands/review.md"
    },
    {
      "name": "leads:edit",
      "description": "Iterate conversationally on a drafted response",
      "path": "commands/edit.md"
    },
    {
      "name": "leads:approve",
      "description": "Approve a drafted response and copy to clipboard",
      "path": "commands/approve.md"
    },
    {
      "name": "leads:skip",
      "description": "Dismiss a lead without responding — moves to Archived",
      "path": "commands/skip.md"
    },
    {
      "name": "leads:add",
      "description": "Manually add a lead (for Facebook or walk-ins)",
      "path": "commands/add.md"
    },
    {
      "name": "leads:add-property",
      "description": "Add a property from listing copy or URL — Claude parses into YAML config",
      "path": "commands/add-property.md"
    },
    {
      "name": "leads:status",
      "description": "Show lead counts by status",
      "path": "commands/status.md"
    }
  ],
  "skills": [
    {
      "name": "lead-pilot-core",
      "path": "skills/core.md"
    },
    {
      "name": "lead-pilot-email-parser",
      "path": "skills/email-parser.md"
    },
    {
      "name": "lead-pilot-response-drafter",
      "path": "skills/response-drafter.md"
    }
  ]
}
```

Save to `lead-pilot/plugin.json`.

- [ ] **Step 4: Create .gitignore for state directory**

```
state.json
```

Save to `lead-pilot/.lead-pilot/.gitignore`.

- [ ] **Step 5: Create property template**

```yaml
# properties/_template.yaml
# Copy this file and rename it to match your property address
# e.g.: properties/30-cooper-st-unit-30a.yaml

address: ""           # Street address including unit, e.g. "30 Cooper St, Unit 30A"
city: ""
state: ""
zip: ""
bedrooms: 0
bathrooms: 0
rent: 0               # Monthly rent in dollars
available_date: ""    # YYYY-MM-DD format
pre_leasing: false    # Set to true if unit is occupied — disables in-person showing offers

security_deposit: "1 month"
application_fee: 0    # Per leaseholder, in dollars

renters_insurance:
  required: false
  min_liability: 100000

pet_policy:
  allowed: false
  fee: 0              # One-time, non-refundable
  monthly_rent: ""    # e.g. "25-50"
  screening: ""       # e.g. "PetScreening FIDO"

parking: ""           # e.g. "1 off-street space included per leaseholder"
optional_extras: []   # e.g. ["Private garage: $125/mo"]

laundry: ""           # e.g. "Shared on-site, no cost" or "In-unit"

utilities_included: []   # e.g. ["Water"]
utilities_tenant: []     # e.g. ["Electric", "Heat", "Hot water"]

amenities: []

screening:
  min_credit_score: 0
  min_income_multiplier: 3
  max_occupants: 0

lease_terms: []       # e.g. [12] or [12, 18]

listing_copy: |
  Paste your full listing text here.
  This gives the response engine full context for answering detailed questions.
```

Save to `lead-pilot/properties/_template.yaml`.

- [ ] **Step 6: Commit initial scaffold**

```bash
cd lead-pilot
git add plugin.json .lead-pilot/.gitignore properties/_template.yaml
git commit -m "feat: scaffold lead-pilot plugin structure"
```

---

### Task 2: Core shared skill

**Files:**
- Create: `lead-pilot/skills/core.md`

This skill is included by all commands. It defines how to read/write state, load property configs, match properties, and handle errors consistently.

- [ ] **Step 1: Create core.md**

```markdown
---
name: lead-pilot-core
description: Shared logic for Lead Pilot — state management, property loading and matching, Notion helpers, and error handling patterns. Include this skill in every Lead Pilot command.
---

# Lead Pilot Core

## State Management

State is stored in `.lead-pilot/state.json` relative to the plugin's working directory (the same directory as `plugin.json`).

### State Schema

```json
{
  "last_checked_date": "YYYY/MM/DD",
  "processed_message_ids": ["gmail_message_id_1", "gmail_message_id_2"],
  "notion_database_id": "notion-db-uuid-here"
}
```

### Reading State

1. Use the Read tool to read `.lead-pilot/state.json`
2. If the file does not exist, treat state as: `{ "last_checked_date": null, "processed_message_ids": [], "notion_database_id": null }`
3. Parse as JSON

### Writing State

1. Make your changes to the parsed state object
2. Use the Write tool to write the updated object back to `.lead-pilot/state.json`
3. Always write the full object — never partial updates
4. Write immediately after each successful operation so state is not lost if a later step fails

### Updating last_checked_date

After a successful `/leads:check` run, update `last_checked_date` to today's date in `YYYY/MM/DD` format (e.g., "2026/03/16"). This is used as the Gmail `after:` filter on the next run.

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
2. For each property, construct a canonical address string: `"{address}, {city}, {state} {zip}"`
3. Compare the email address string against each canonical address using these rules (in order):
   - **Exact match** (case-insensitive): highest confidence
   - **Street number + street name match**: e.g., "30 Cooper" appears in both strings — high confidence
   - **Street name + city match**: lower confidence, surface as a suggestion if other fields are ambiguous
4. Return the best match, or `null` if no confident match found

If `null`, create the lead in Notion with `Property = "Unmatched"` and log the raw address string in the page body. Print a warning to the user:
```
⚠ Could not match address "{raw_address}" to a property config. Lead created as "Unmatched". Add a property config or assign the property manually in Notion.
```

---

## Notion Database Schema

When creating the Notion database (during `/leads:setup`), use these properties:

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

```
[YYYY-MM-DD H:MM AM/PM] [Platform] [Direction]
"Message text here"
```

Where:
- Direction is: `Inbound`, `Draft Response`, or `Approved → Sent`
- Platform is the source platform name or `Manual` for `/leads:add` entries

Use Notion paragraph blocks for each message entry.

---

## Deduplication (Cross-Platform)

When ingesting a new lead:

1. Read `notion_database_id` from state
2. Search the Notion database for a page where `Email` property equals the lead's email address
3. If found: this is an existing lead — append the new message to the page body and update `Platforms` and `Last Message`. Do NOT create a new page.
4. If not found by email: search for pages where `Name` matches the lead's name AND `Property` matches the lead's property
5. If a name+property match is found: surface a warning to the user:
   ```
   ⚠ Possible duplicate: "{name}" is already in the database for {property} (via {existing platform}). This may be the same person. Check Notion to merge manually if needed.
   ```
   Then create the new lead as a separate page — do not auto-merge.
6. If no match: create a new lead page

---

## Error Handling Patterns

Apply these patterns in every command:

**Gmail MCP unavailable:**
```
❌ Gmail MCP is not responding. Check your Gmail MCP connection and try again.
(No state was updated. Running /leads:check again will retry safely.)
```
Do not update state.json. Exit.

**Notion MCP unavailable:**
```
❌ Notion MCP is not responding. Check your Notion MCP connection and try again.
(No state was updated. Running /leads:check again will retry safely.)
```
Do not update state.json. Exit.

**Playwright scrape fails:**
Create the lead in Notion with the data available from the email. Set Status to "Needs Manual Review". Print:
```
⚠ Could not scrape message from {platform} for lead {name}. Lead created with partial data. Check {platform} directly for the full message.
```

**Response engine fails:**
Leave lead Status as "New". Print:
```
⚠ Could not draft response for {name}. Lead saved. Run /leads:review to retry.
```

**Notion page creation fails for a single lead:**
Print the error for that lead and continue processing remaining leads. Do not add the failed email's message ID to `processed_message_ids` — it will be retried on the next `/leads:check` run.
```

Save to `lead-pilot/skills/core.md`.

- [ ] **Step 2: Verify the file was created**

```bash
wc -l lead-pilot/skills/core.md
```

Expected: a non-zero line count (file exists and has content).

- [ ] **Step 3: Commit**

```bash
git add skills/core.md
git commit -m "feat: add core shared skill with state, property matching, Notion schema, error patterns"
```

---

### Task 3: Setup command

**Files:**
- Create: `lead-pilot/commands/setup.md`

- [ ] **Step 1: Create setup.md**

```markdown
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
```

Save to `lead-pilot/commands/setup.md`.

- [ ] **Step 2: Manually test setup command**

Run `/leads:setup` in Claude Code. Verify:
- Notion database is created with all 15 properties listed in @lead-pilot-core
- `.lead-pilot/state.json` exists and contains a valid `notion_database_id`
- Three views are visible in Notion
- State shows the correct `last_checked_date` based on your backfill choice

- [ ] **Step 3: Verify state.json was written**

```bash
cat lead-pilot/.lead-pilot/state.json
```

Expected: valid JSON with `notion_database_id`, `last_checked_date`, and empty `processed_message_ids` array.

- [ ] **Step 4: Commit**

```bash
git add commands/setup.md
git commit -m "feat: add /leads:setup command with Notion DB creation and state init"
```

---

## Chunk 2: Email Parser + Lead Ingestion (/leads:check)

### Task 4: Email parser skill

**Files:**
- Create: `lead-pilot/skills/email-parser.md`
- Create: `lead-pilot/tests/fixtures/zillow-email.txt`
- Create: `lead-pilot/tests/fixtures/avail-email.txt`
- Create: `lead-pilot/tests/fixtures/apartments-email.txt`

- [ ] **Step 1: Create test fixture — Zillow email**

Create a representative Zillow email text file that mirrors the format observed in screenshots:

```
FROM: Ashlie DeMarco <18duq4s6jgtwhi5gmp0jqamkzn4@convo.zillow.com>
REPLY-TO: Ashlie DeMarco <18duq4s6jgtwhi5gmp0jqamkzn4@convo.zillow.com>
TO: silkcityapartments@gmail.com
SUBJECT: Ashlie is requesting information about 30 Cooper St, Manchester, CT, 06040

Ashlie DeMarco says:
I would like to schedule a tour.

About Ashlie DeMarco
Move in: Jun 06, 2026
Credit score: 620 to 659
Income: $65004
Pets: Yes
Lease Length: 18 months
Number of Bedrooms: 1
Number of Occupants: 1
```

Save to `lead-pilot/tests/fixtures/zillow-email.txt`.

- [ ] **Step 2: Create test fixture — Avail email**

```
FROM: Avail Team <support@avail.co>
TO: silkcityapartments@gmail.com
SUBJECT: You have received a new message on Avail

Hi Silk,

We wanted to let you know you've received a new message from Rebecca Meyer about your rental at 28 30 Cooper St.

Here's a preview:
"I am interested in 28 30 Cooper St Unit 30A."

To see the full message and reply, use the secure link below. It will expire after 48 hours.
[Reply Securely] https://avail.co/secure-reply/...
```

Save to `lead-pilot/tests/fixtures/avail-email.txt`.

- [ ] **Step 3: Create test fixture — Apartments.com email**

```
FROM: lead@apartments.com
REPLY-TO: barbarahuarcaya@icloud.com
TO: silkcityapartments@gmail.com
SUBJECT: You've received a new lead!

You've received a new lead for the following property:

30 Cooper St, Manchester, CT 06040

Sign in to your account to view message details and continue the conversation.
[View Message] https://www.apartments.com/...
```

Save to `lead-pilot/tests/fixtures/apartments-email.txt`.

- [ ] **Step 4: Create email-parser.md**

```markdown
---
name: lead-pilot-email-parser
description: Platform-specific email parsing logic for Lead Pilot. Extracts lead name, email, message, property address, and screening data from Zillow, Avail, and Apartments.com notification emails.
---

# Lead Pilot Email Parser

@lead-pilot-core

Given the raw content of a Gmail message (headers + body), extract a lead record. The output of parsing is always a lead object with these fields (null if not found):

```json
{
  "name": "Ashlie DeMarco",
  "email": "ashlie@example.com",
  "phone": null,
  "platform": "Zillow",
  "raw_address": "30 Cooper St, Manchester, CT, 06040",
  "message": "I would like to schedule a tour.",
  "screening": {
    "move_in_date": "2026-06-06",
    "credit_score": "620-659",
    "income": 65004,
    "pets": true,
    "lease_length": "18 months",
    "bedrooms": 1,
    "occupants": 1
  },
  "source_email_id": "gmail-message-id-here",
  "received_at": "2026-03-13T17:24:00"
}
```

`screening` fields are only populated for Zillow emails. Set to null for other platforms.

---

## Zillow

**Detect:** `From` header contains `@convo.zillow.com`

**Extract:**
- `name`: Display name from the `From` header (e.g., "Ashlie DeMarco")
- `email`: The full `From` email address (e.g., `18duq4s6@convo.zillow.com`) — this is the Zillow relay address; it can receive replies but is not the lead's personal email
- `raw_address`: From the subject line — everything after "requesting information about "
- `message`: The line after "{name} says:" in the email body (the lead's actual message text)
- `screening.move_in_date`: The value on the "Move in:" line, converted to ISO date format (YYYY-MM-DD)
- `screening.credit_score`: The value on the "Credit score:" line, formatted as "NNN-NNN"
- `screening.income`: The value on the "Income:" line, parsed as a number (strip "$" and commas)
- `screening.pets`: "Yes" → true, "No" → false
- `screening.lease_length`: The value on the "Lease Length:" line (e.g., "18 months")
- `screening.bedrooms`: The value on the "Number of Bedrooms:" line, parsed as integer
- `screening.occupants`: The value on the "Number of Occupants:" line, parsed as integer
- `received_at`: From the email Date header

**Note:** Zillow emails show the lead's name and message inline. No Playwright needed.

---

## Avail

**Detect:** `From` header contains `support@avail.co`

**Extract:**
- `name`: From the sentence "you've received a new message from {name} about" in the body
- `email`: Avail does not include the lead's email in the notification. Set to null initially — look for it in the Avail platform if needed.
- `raw_address`: From the sentence "about your rental at {address}" in the body
- `message`: The text between the quotation marks in the "Here's a preview:" section

**Truncation detection:**
After extracting the preview message, check if either of these conditions is true:
1. The message text ends with `...`
2. The message text is fewer than 280 characters AND there is a "Reply Securely" link in the email

If truncated:
1. Extract the "Reply Securely" URL from the email body
2. Use Playwright MCP to navigate to the URL
3. Wait for the page to load (look for a message text container)
4. Extract the full message text from the page
5. If Playwright fails, keep the truncated preview and set lead Status to "Needs Manual Review"

---

## Apartments.com

**Detect:** `From` header contains `lead@apartments.com`

**Extract:**
- `name`: Apartments.com does not include the lead's name in the email. Set to null — it will be extracted from the Apartments.com page via Playwright.
- `email`: From the `Reply-To` header (this is the lead's actual personal email)
- `raw_address`: The bolded property address in the email body (appears before "Sign in to your account...")
- `message`: NOT in the email. Must be retrieved via Playwright (see below).

**Playwright extraction:**
1. Extract the "View Message" URL from the email body
2. Check for an active Apartments.com session. Use Playwright MCP to navigate to `https://www.apartments.com`. If the page shows a login screen, prompt the user:
   ```
   ⚠ Apartments.com requires you to log in. Opening browser... Please log in to continue.
   ```
   Wait for the user to confirm login is complete before proceeding.
3. Navigate to the "View Message" URL
4. Extract: the lead's name (usually displayed as a heading), and the message text
5. If extraction fails for any reason: create the lead with available data (email from reply-to, address from email body) and set Status to "Needs Manual Review". Print the error to the user.
```

Save to `lead-pilot/skills/email-parser.md`.

- [ ] **Step 5: Manually test email parser with each fixture**

For each fixture file, ask Claude: "Using the lead-pilot-email-parser skill, parse this email and show me the extracted lead object:" followed by the contents of the fixture file.

Verify:
- **Zillow fixture**: All fields populated including screening data, address parsed from subject
- **Avail fixture**: Name, address, and message extracted correctly; no screening data
- **Apartments.com fixture**: Email extracted from reply-to, address extracted from body, message is null (awaiting Playwright)

- [ ] **Step 6: Commit**

```bash
git add skills/email-parser.md tests/fixtures/
git commit -m "feat: add email parser skill with Zillow, Avail, and Apartments.com extraction"
```

---

### Task 5: Response drafter skill

**Files:**
- Create: `lead-pilot/skills/response-drafter.md`

- [ ] **Step 1: Create response-drafter.md**

```markdown
---
name: lead-pilot-response-drafter
description: Builds response drafts for rental leads by assembling context from the lead record and property config, then invoking the lead-response Claude Skill.
---

# Lead Pilot Response Drafter

@lead-pilot-core

## What This Skill Does

Given a lead object and a matched property config, assembles the full context payload and generates a drafted response. The draft is returned as a string and saved to the lead's Notion page.

## Assembling Context

Build a context block containing:

**Lead context:**
- Lead's name
- Lead's message (and full thread history if this is a follow-up)
- Screening data if available (credit score, income, pets, move-in date, etc.)
- Platform the message came from
- Timestamp

**Property context (from YAML config):**
- All structured fields: rent, bedrooms, bathrooms, available_date, security_deposit, application_fee, pet_policy, parking, laundry, utilities, amenities, screening criteria, lease terms
- `pre_leasing` status: if true, include this instruction: "This unit is currently occupied and is being pre-leased. Do NOT offer in-person showings. Instead, offer to share photos, a video walkthrough, or schedule a showing closer to the move-in date."
- `listing_copy`: the full listing text for rich context on unit features and neighborhood

**Response instructions:**
```
Draft a warm, professional response to this rental lead in the landlord's personal voice.
- Answer any specific questions asked
- If the lead asks for a showing and pre_leasing is true, explain that the unit is pre-leasing and offer alternatives
- If the lead appears to meet screening criteria (based on Zillow data), be welcoming
- If the lead clearly does not meet criteria (e.g., credit score far below minimum), be politely informative about requirements
- Keep the response concise — 3-5 sentences for simple inquiries, longer for complex ones
- Do not include the lead's name in the greeting — use a natural opener
```

## Generating the Draft

Pass the assembled context to the existing Claude Skill for lead response drafting (the user's pre-built skill that handles voice/tone matching, property Q&A, and screening awareness). If no such skill is found, generate the response directly using the context above.

## Saving to Notion

After generating the draft:
1. Look up the lead's Notion page ID (stored with the lead object or searchable by email)
2. Append a new block to the page body:
   ```
   [{timestamp}] [Draft Response]
   "{draft text}"
   ```
3. Update the lead's Notion `Status` property to "Response Drafted"
4. Update `Last Message` to the current timestamp

## Iteration

When the user requests changes to a draft (via /leads:edit):
1. Show the current draft
2. Apply the user's requested changes
3. Show the revised draft
4. Ask: "Update the draft in Notion? (yes / no)"
5. If yes: append the revised draft to the page body with a note: `[{timestamp}] [Draft Revised]` and replace the previous draft block if possible, or append a new one labeled "Revised Draft"

## Output

Return the final draft text as a string. Also print it clearly to the terminal:
```
--- DRAFT RESPONSE ---
{draft text}
----------------------
```
```

Save to `lead-pilot/skills/response-drafter.md`.

- [ ] **Step 2: Test response drafter manually**

Using the Zillow fixture lead object from Task 4 and the sample property config from `properties/_template.yaml` (filled in with test data), ask Claude to run the response drafter skill.

Verify:
- Output is a coherent, property-specific response
- Draft matches the tone described in the skill
- If pre_leasing is true in the test config, showing is not offered

- [ ] **Step 3: Commit**

```bash
git add skills/response-drafter.md
git commit -m "feat: add response drafter skill with context assembly and Notion integration"
```

---

### Task 6: /leads:check command

**Files:**
- Create: `lead-pilot/commands/check.md`

- [ ] **Step 1: Create check.md**

```markdown
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
```

Save to `lead-pilot/commands/check.md`.

- [ ] **Step 2: Create a sample property config for testing**

```bash
cp lead-pilot/properties/_template.yaml lead-pilot/tests/fixtures/sample-property.yaml
```

Open the file and fill in test values (use the 30 Cooper St listing from the spec):

```yaml
address: "30 Cooper St, Unit 30A"
city: "Manchester"
state: "CT"
zip: "06040"
bedrooms: 2
bathrooms: 1
rent: 1895
available_date: "2026-05-01"
pre_leasing: true
security_deposit: "1 month"
application_fee: 50
renters_insurance:
  required: true
  min_liability: 100000
pet_policy:
  allowed: true
  fee: 350
  monthly_rent: "25-50"
  screening: "PetScreening FIDO"
parking: "1 off-street space included per leaseholder"
optional_extras:
  - "Private garage: $125/mo"
laundry: "Shared on-site, no cost"
utilities_included:
  - "Water"
utilities_tenant:
  - "Electric"
  - "Heat"
  - "Hot water"
amenities:
  - "Original hardwood floors"
  - "Exposed brick"
  - "GE stainless steel appliances"
screening:
  min_credit_score: 650
  min_income_multiplier: 3
  max_occupants: 2
lease_terms:
  - 12
listing_copy: |
  Beautifully-Restored 2BD/1BA in Manchester's Historic West Side. $1,895/mo. Available May 1st.
```

- [ ] **Step 3: Create a sample state.json for testing dedup**

```json
{
  "last_checked_date": "2026/03/01",
  "processed_message_ids": [],
  "notion_database_id": "REPLACE_WITH_REAL_DATABASE_ID_FROM_SETUP"
}
```

Save to `lead-pilot/tests/fixtures/sample-state.json`. Update `notion_database_id` with the ID from your `/leads:setup` run.

- [ ] **Step 4: End-to-end test of /leads:check**

Prerequisite: Run `/leads:setup` successfully (Task 3). Have at least one real lead email in `silkcityapartments@gmail.com`.

Run `/leads:check`. Verify:
- Gmail searches execute without error
- At least one email is parsed and a lead is created in Notion
- The Notion page has correct properties populated (name, platform, property, status = "Response Drafted")
- The page body contains the inbound message and a draft response
- `.lead-pilot/state.json` has been updated with the processed message ID and today's date

- [ ] **Step 5: Test dedup by running /leads:check a second time immediately**

Run `/leads:check` again. Verify:
- No duplicate leads are created
- Output shows "0 new emails to process" (all IDs in processed_message_ids)

- [ ] **Step 6: Commit**

```bash
git add commands/check.md tests/fixtures/sample-property.yaml tests/fixtures/sample-state.json
git commit -m "feat: add /leads:check ingestion pipeline command"
```

---

## Chunk 3: Review Workflow Commands

### Task 7: /leads:review command

**Files:**
- Create: `lead-pilot/commands/review.md`

- [ ] **Step 1: Create review.md**

```markdown
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
```

Save to `lead-pilot/commands/review.md`.

- [ ] **Step 2: Test /leads:review**

Prerequisite: At least one lead with Status = "Response Drafted" exists in Notion (from `/leads:check` test).

Run `/leads:review`. Verify:
- Lead details are displayed clearly and correctly
- Full message thread is visible
- Draft response is shown at the bottom

- [ ] **Step 3: Commit**

```bash
git add commands/review.md
git commit -m "feat: add /leads:review command with thread display and draft preview"
```

---

### Task 8: Approve, Edit, and Skip commands

**Files:**
- Create: `lead-pilot/commands/approve.md`
- Create: `lead-pilot/commands/edit.md`
- Create: `lead-pilot/commands/skip.md`

- [ ] **Step 1: Create approve.md**

```markdown
---
name: leads:approve
description: Approve the drafted response for the current lead. Copies the response to clipboard and updates the lead status in Notion.
---

# /leads:approve

@lead-pilot-core

## Steps to Follow

### 1. Identify current lead

This command should be run immediately after /leads:review. The current lead's Notion page ID should be available from the previous command's context. If not, ask: "Which lead would you like to approve? (Provide a name or email)"

Use Notion MCP to search for the lead if needed.

### 2. Get the draft response

Fetch the lead's Notion page. Find the most recent block labeled "Draft Response" or "Draft Revised" in the page body. Extract the response text.

### 3. Copy to clipboard

Run the appropriate system clipboard command for the platform:
- Windows: `echo "{response text}" | clip`
- macOS: `echo "{response text}" | pbcopy`
- Linux: `echo "{response text}" | xclip -selection clipboard`

Print:
```
📋 Response copied to clipboard!
```

### 4. Update Notion

Set Status to "Awaiting Approval" — the response is approved and ready to paste, but we cannot confirm the user has actually sent it on the platform. The user can update to "Sent" manually in Notion once they've pasted and sent it.

- Append to page body:
  ```
  [{timestamp}] [Approved → Sent]
  "{draft text}"
  ```
- Update `Last Message` to current timestamp

Print:
```
✅ Approved! Paste this response into {platform} to send it.

Status updated to: Awaiting Approval

Tip: Once you've sent the message, update the status to "Sent" in Notion.

Next: /leads:review to see the next lead.
```
```

Save to `lead-pilot/commands/approve.md`.

- [ ] **Step 2: Create edit.md**

```markdown
---
name: leads:edit
description: Iterate on a drafted response conversationally. Describe the changes you want and Claude will revise the draft.
---

# /leads:edit

@lead-pilot-core
@lead-pilot-response-drafter

## Steps to Follow

### 1. Identify current lead and draft

Same as /leads:approve step 1 — use current session context or ask for lead identifier.

Fetch and display the current draft:
```
Current draft:
{draft text}

What changes would you like? (describe naturally, e.g., "make it shorter", "add the pet deposit amount", "ask about their move-in timeline")
```

### 2. Accept edit instructions

Wait for the user's natural language edit request.

### 3. Revise the draft

Using the edit instructions plus the original lead context and property config, revise the response.

If property config is needed: reload it from `properties/` using the lead's Property field to find the matching YAML file.

Show the revised draft:
```
Revised draft:
{new draft text}

Save this as the new draft? (yes / try again / cancel)
```

### 4. Save or iterate

- **yes**: Save to Notion (append "[Draft Revised]" block), update Status to "Response Drafted" if it changed. Print: `✅ Draft updated. Run /leads:approve to approve it.`
- **try again**: Go back to step 3 with additional context
- **cancel**: Discard changes. Print: `Draft unchanged.`
```

Save to `lead-pilot/commands/edit.md`.

- [ ] **Step 3: Create skip.md**

```markdown
---
name: leads:skip
description: Dismiss a lead without responding. Use for spam, unqualified leads, or leads already handled outside the system. Moves the lead to Archived status.
---

# /leads:skip

@lead-pilot-core

## Steps to Follow

### 1. Identify current lead

Use session context or ask: "Which lead would you like to skip? (Provide name or email)"

### 2. Confirm

```
Skip {name} ({platform}) and archive without responding?

Reason (optional — press Enter to skip):
```

Accept an optional reason string from the user.

### 3. Update Notion

- Set Status to "Archived"
- Append to page body:
  ```
  [{timestamp}] [Skipped — No Response Sent]
  {reason if provided, otherwise "Dismissed by landlord"}
  ```

Print:
```
✅ {name} archived.

Next: /leads:review to see the next lead.
```
```

Save to `lead-pilot/commands/skip.md`.

- [ ] **Step 4: Test approve, edit, skip workflow**

Using the lead created during the `/leads:check` test:

**Test edit:**
- Run `/leads:review`
- Run `/leads:edit`
- Request: "make it shorter"
- Verify revised draft is shown and saved to Notion

**Test approve:**
- Run `/leads:approve`
- Verify response text is in clipboard (paste to verify)
- Verify Notion status updated to "Awaiting Approval"
- Verify page body has "Approved → Sent" block

**Test skip (use a different test lead or reset the first one):**
- Create a test lead manually in Notion with Status = "Response Drafted"
- Run `/leads:review` until it shows that lead
- Run `/leads:skip`
- Verify status is "Archived" in Notion

- [ ] **Step 5: Commit**

```bash
git add commands/approve.md commands/edit.md commands/skip.md
git commit -m "feat: add /leads:approve, /leads:edit, /leads:skip review workflow commands"
```

---

### Task 9: Manual add and status commands

**Files:**
- Create: `lead-pilot/commands/add.md`
- Create: `lead-pilot/commands/status.md`

- [ ] **Step 1: Create add.md**

```markdown
---
name: leads:add
description: Manually add a rental lead. Use for Facebook Marketplace leads, walk-ins, or any lead not captured by email ingestion.
---

# /leads:add

@lead-pilot-core
@lead-pilot-response-drafter

## Steps to Follow

### 1. Load state

Read state.json. Get `notion_database_id`. If missing, print setup error and exit.

### 2. Collect lead information

Ask for each field, one at a time:

```
Adding a new lead manually.

1. Lead's name:
```
(wait for response)

```
2. Lead's email (or press Enter to skip):
```
(wait for response)

```
3. Lead's message (paste their full message):
```
(wait for response)

```
4. Platform: (Facebook / Walk-in / Other)
```
(wait for response)

```
5. Which property are they asking about?
```
List available property names from the `properties/` directory. User picks one.

Ask optional fields only if they seem relevant:
- Phone number
- Any notes to add

### 3. Dedup check

Run the dedup check from @lead-pilot-core using the provided email (if given).

### 4. Create lead in Notion

Same as /leads:check step 7. Set `received_at` to now.

### 5. Draft response

Same as /leads:check step 8. Use @lead-pilot-response-drafter.

### 6. Confirm

```
✅ Lead created: {name} ({platform})

Draft response generated. Run /leads:review to see it.
```
```

Save to `lead-pilot/commands/add.md`.

- [ ] **Step 2: Create status.md**

```markdown
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
```

Save to `lead-pilot/commands/status.md`.

- [ ] **Step 3: Test /leads:add**

Run `/leads:add`. Simulate a Facebook lead:
- Name: "Test Facebook Lead"
- Email: (skip)
- Message: "Hey, is the 2 bedroom still available? I'm looking to move in around May."
- Platform: Facebook

Verify:
- Lead is created in Notion with Platform = "Facebook"
- Status = "Response Drafted"
- Page body has the inbound message and a draft response
- Response appropriately doesn't offer in-person showings (since test property has pre_leasing: true)

- [ ] **Step 4: Test /leads:status**

Run `/leads:status`. Verify counts match what you see in Notion.

- [ ] **Step 5: Commit**

```bash
git add commands/add.md commands/status.md
git commit -m "feat: add /leads:add manual entry and /leads:status summary commands"
```

---

### Task 10: /leads:add-property command

**Files:**
- Create: `lead-pilot/commands/add-property.md`

- [ ] **Step 1: Create add-property.md**

```markdown
---
name: leads:add-property
description: Add a rental property by pasting listing copy or providing a URL. Claude parses the content into a YAML config file and saves it to the properties/ directory.
---

# /leads:add-property

@lead-pilot-core

## Steps to Follow

### 1. Ask for listing source

```
Add a new property to Lead Pilot.

How would you like to provide the listing?
  A) Paste listing copy directly
  B) Provide a URL to the live listing (Claude will scrape it)
```

### 2A. If paste: receive listing copy

```
Paste your listing copy below (press Enter twice when done):
```

Accept multi-line input. Use the pasted text as the source.

### 2B. If URL: scrape the listing

Use Playwright MCP to navigate to the URL. Extract the listing text (heading, description, details, requirements sections). If scraping fails:
```
⚠ Could not load the listing URL. Please paste the listing copy manually instead.
```
Fall through to paste mode.

### 3. Parse listing into YAML

Analyze the listing text and extract every field from the property YAML schema. For each field:
- `address`: Look for a street address in the heading or text
- `city`, `state`, `zip`: From the address
- `bedrooms`, `bathrooms`: From "2BD/1BA" or "2 Bedroom" patterns
- `rent`: Dollar amount in the heading or price line
- `available_date`: From "Available [date]" pattern, convert to YYYY-MM-DD
- `pre_leasing`: Set to false by default (ask at end if ambiguous)
- `security_deposit`: From "security deposit" mention
- `application_fee`: From "application fee" mention
- `renters_insurance`: From "renter's insurance" mention
- `pet_policy.allowed`: From "pet-friendly" or "no pets"
- `pet_policy.fee`: Dollar amount near "pet fee"
- `pet_policy.monthly_rent`: Range near "pet rent"
- `pet_policy.screening`: Tool name if mentioned (e.g., "PetScreening")
- `parking`: From parking section
- `optional_extras`: Any add-on amenities with costs
- `laundry`: From laundry mention
- `utilities_included`: From "included" utilities
- `utilities_tenant`: From separately-metered or tenant-paid utilities
- `amenities`: Feature list from the listing
- `screening.min_credit_score`: From "minimum credit score" mention
- `screening.min_income_multiplier`: From income ratio mention
- `screening.max_occupants`: If mentioned
- `lease_terms`: From lease length mention
- `listing_copy`: The full raw listing text

### 4. Show parsed result

Display the parsed YAML to the user:

```
Parsed property config:

address: "30 Cooper St, Unit 30A"
city: "Manchester"
...

Does this look right? (yes / make changes)
```

If user says "make changes", accept natural language corrections:
- "rent is actually $1,250" → update rent field
- "it's a 1 bedroom, not 2" → update bedrooms field
- "add that heat is included" → add Heat to utilities_included

Show the updated config after each change and ask again.

### 5. Ask about pre_leasing

If not clear from listing:
```
Is this unit currently occupied (pre-leasing)?
  yes — Unit is occupied, no in-person showings until closer to move-in date
  no  — Unit is available to show now
```

### 6. Generate filename and save

Generate a filename from the address: lowercase, spaces and special chars replaced with hyphens.
Example: "30 Cooper St, Unit 30A" → `30-cooper-st-unit-30a.yaml`

```
Save property config as properties/30-cooper-st-unit-30a.yaml? (yes / different name)
```

Write the YAML file to `lead-pilot/properties/{filename}`.

### 7. Confirm

```
✅ Property saved: properties/{filename}

  Address:   {address}
  Rent:      ${rent}/mo
  Available: {available_date}

This property will now be matched to incoming leads. Run /leads:check to process any existing leads for this property.
```
```

Save to `lead-pilot/commands/add-property.md`.

- [ ] **Step 2: Test /leads:add-property with paste**

Run `/leads:add-property`. Paste the full listing copy from the spec (the Cooper St listing).

Verify:
- All fields are parsed correctly (check against the expected YAML in the spec)
- Pre_leasing is set correctly
- listing_copy contains the full text
- File is saved to `properties/30-cooper-st-unit-30a.yaml`

- [ ] **Step 3: Verify saved YAML is valid**

```bash
cat lead-pilot/properties/30-cooper-st-unit-30a.yaml
```

Spot check: `rent: 1895`, `bedrooms: 2`, `pre_leasing: true`, `pet_policy.fee: 350`

- [ ] **Step 4: Commit**

```bash
git add commands/add-property.md properties/30-cooper-st-unit-30a.yaml
git commit -m "feat: add /leads:add-property command + save first property config"
```

---

### Task 11: Tests README and final integration test

**Files:**
- Create: `lead-pilot/tests/README.md`

- [ ] **Step 1: Create tests README**

```markdown
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
1. Run `/leads:review` — verify lead displayed correctly
2. Run `/leads:edit "make it shorter"` — verify revised draft shown and saved
3. Run `/leads:approve` — verify clipboard contains response text, status updated
4. Run `/leads:review` again — verify next lead shown (or "nothing to review")

### T8: Skip workflow
1. Create test lead with Status "Response Drafted"
2. Run `/leads:review` to surface it
3. Run `/leads:skip`
4. Verify: Status = "Archived" in Notion

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
```

Save to `lead-pilot/tests/README.md`.

- [ ] **Step 2: Run full integration test T7 (review → edit → approve flow)**

Follow steps in T7 above. This is the primary daily workflow.

- [ ] **Step 3: Commit everything and tag v0.1.0**

```bash
git add tests/README.md
git commit -m "docs: add manual test guide for all commands"

git tag -a v0.1.0 -m "Lead Pilot v0.1.0 — initial plugin with full ingestion and review workflow"
```

- [ ] **Step 4: Final sanity check — all commands exist**

```bash
ls lead-pilot/commands/
```

Expected:
```
add-property.md  add.md  approve.md  check.md  edit.md  review.md  setup.md  skip.md  status.md
```

```bash
ls lead-pilot/skills/
```

Expected:
```
core.md  email-parser.md  response-drafter.md
```

---

## End State

After completing all tasks, Lead Pilot V1 is fully functional:

| Command | What it does |
|---|---|
| `/leads:setup` | One-time Notion database creation and state initialization |
| `/leads:check` | Polls Gmail, ingests leads, drafts responses |
| `/leads:review` | Shows next unreviewed lead with thread and draft |
| `/leads:edit` | Revises a draft conversationally |
| `/leads:approve` | Copies approved response to clipboard |
| `/leads:skip` | Archives lead without responding |
| `/leads:add` | Manually adds a Facebook or walk-in lead |
| `/leads:add-property` | Parses listing copy into property YAML config |
| `/leads:status` | Shows lead pipeline at a glance |

**Automation:** `/loop 15m /leads:check` polls for new leads every 15 minutes.
