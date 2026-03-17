# Lead Pilot — Design Spec

**Date:** 2026-03-16
**Status:** Draft
**Author:** Eric Zdanowski + Claude

## Problem

Managing inbound rental leads across four platforms (Zillow, Apartments.com, Facebook Marketplace, Avail.co) is manual, scattered, and slow. Leads get missed, follow-ups are delayed, and responding individually on each platform is time-intensive. A Claude Skill already exists to draft responses in the landlord's voice, but there is no automation around ingestion, tracking, or review.

## Solution

**Lead Pilot** is a Claude Code plugin that automates lead ingestion from rental platforms, drafts responses using an existing Claude Skill, and provides a slash-command-based review workflow. Leads are stored in a Notion database for centralized tracking and mobile access.

## Architecture

```
Gmail MCP / Playwright MCP → Ingestion → Notion DB → Response Engine (Claude Skill) → Review Commands → Copy-paste to platform
```

### Components

1. **Lead Ingestion** — Gmail MCP polls for new lead notification emails; Playwright MCP handles click-throughs for platforms that don't include message content in emails
2. **Lead Database** — Notion database via Notion MCP; each lead is a page with structured properties and a message thread in the page body
3. **Response Engine** — Existing Claude Skill, fed with thread context + property config + screening data
4. **Review Workflow** — Slash commands for reviewing, iterating on, and approving drafted responses
5. **Property Config** — YAML files auto-generated from listing copy, one per property
6. **Automation** — `/loop` for background polling on a configurable interval

## Lead Ingestion

### Email-Based Ingestion (Zillow, Avail, Apartments.com)

The `/leads:check` command searches Gmail via MCP for lead notification emails. Read/unread status is ignored.

#### State Persistence

Ingestion state is stored in `.lead-pilot/state.json` within the plugin's working directory:

```json
{
  "last_checked_date": "2026/03/16",
  "processed_message_ids": ["msg_abc123", "msg_def456"]
}
```

- `last_checked_date`: Used as a coarse Gmail `after:` filter (Gmail only supports date granularity, not timestamps)
- `processed_message_ids`: The primary deduplication mechanism — every successfully processed email's Gmail message ID is recorded here. This handles multiple checks within the same day and prevents reprocessing.
- On first run, `last_checked_date` defaults to the current date and `processed_message_ids` is empty. The user is prompted to optionally backfill (e.g., "Check the last 7 days for existing leads?").

#### Platform-Specific Parsing

**Zillow:**
- From: `<hash>@convo.zillow.com` (lead's name in display name)
- Subject: `"{Name} is requesting information about {Address}"`
- Body: Message text inline, plus screening data (move-in date, credit score range, income, pets, lease length, bedrooms, occupants)
- Reply-to: convo.zillow.com address (supports direct email reply)

**Avail:**
- From: `support@avail.co`
- Subject: `"You have received a new message on Avail"`
- Body: Lead name, property address, message preview (may truncate longer messages)
- Fallback: Playwright MCP clicks "Reply Securely" link to retrieve full message if preview appears truncated (truncation detected by trailing "..." or message body under 280 characters, which suggests a cut-off)

**Apartments.com:**
- From: `lead@apartments.com`
- Reply-to: Lead's personal email (used for deduplication)
- Subject: `"You've received a new lead!"`
- Body: Property address only, no message content
- Required: Playwright MCP clicks "View Message" link to scrape the actual message
- Authentication: Playwright must have an active Apartments.com session. On first use (or session expiry), the user is prompted to log in manually via the Playwright browser window. Session cookies persist across checks.
- Fragility note: This is the most fragile ingestion path. If scraping fails (page structure change, CAPTCHA, timeout), the lead is still created in Notion with available data from the email (property address, lead email from reply-to header) and Status set to "Needs Manual Review" so the user knows to check the message on-platform.

#### Gmail Search Patterns

- Zillow: `from:convo.zillow.com after:{last_checked_date}`
- Avail: `from:support@avail.co subject:"new message on Avail" after:{last_checked_date}`
- Apartments.com: `from:lead@apartments.com after:{last_checked_date}`

These date-based filters serve as a coarse window. The `processed_message_ids` set is the authoritative dedup mechanism — any email whose Gmail message ID is already in the set is skipped regardless of the date filter.

Gmail account: `silkcityapartments@gmail.com`

### Facebook Messenger (Manual Entry)

Facebook does not send email notifications and actively resists automation. For the initial version, Facebook leads are added manually via `/leads:add`. The system still drafts responses for these leads using the same response engine.

Future consideration: Playwright-based Messenger scraping if manual entry becomes a bottleneck.

### Deduplication

**Within a platform:** Each ingested email is tracked by Gmail message ID to prevent duplicate processing on repeated checks.

**Across platforms:** Email address is the primary dedup key. When ingesting a new lead, the system checks the Notion database for an existing lead with the same email address. If found, the new message is appended to the existing lead's thread and the platform multi-select is updated.

If no email match is found but a name + property combination matches an existing lead, the system surfaces a suggestion to the user ("This may be the same person as [existing lead] — merge?") rather than auto-merging, since name matching is unreliable across platforms.

### Property Matching

Incoming leads are matched to a property by comparing the address in the email (subject line or body) against the `address`, `city`, `state`, and `zip` fields in property config files. Fuzzy matching handles minor formatting differences (e.g., "30 Cooper St" vs "30 Cooper Street").

If no property match is found, the lead is still created in Notion with Property set to "Unmatched" and the raw address string preserved in the page body. The user is notified during `/leads:check` output so they can assign it manually or add a missing property config.

## Lead Database (Notion)

### Database: "Leads"

Each lead is a Notion page with these properties:

| Property | Type | Description |
|---|---|---|
| Name | Title | Lead's name |
| Email | Email | Lead's email address |
| Phone | Phone | If provided |
| Property | Rich Text | Which unit they're asking about (free-text to avoid needing schema updates when properties are added) |
| Platform(s) | Multi-select | Zillow, Avail, Apartments.com, Facebook |
| Status | Select | New → Response Drafted → Awaiting Approval → Sent → Screening → Archived |
| Priority | Select | High / Medium / Low |
| Last Message | Date | Timestamp of most recent message |
| Auto-Send Eligible | Checkbox | For future auto-send rules |
| Move-In Date | Date | From Zillow screening data |
| Credit Score | Text | From Zillow (e.g., "620-659") |
| Income | Number | From Zillow |
| Pets | Checkbox | From Zillow |
| Lease Length | Text | From Zillow (e.g., "18 months") |
| Occupants | Number | From Zillow |

### Page Body

The page body contains the full message thread as a chronological list of blocks:

```
[2026-03-13 5:24 PM] [Zillow] [Inbound]
"I would like to schedule a tour."

[2026-03-13 6:00 PM] [Draft Response]
"Hi Ashlie! Thanks for your interest in 30 Cooper St..."

[2026-03-13 6:05 PM] [Approved → Sent]
"Hi Ashlie! Thanks for your interest in 30 Cooper St..."
```

### Notion Views

- **Needs Review** — Filtered to "Response Drafted" status, sorted oldest first
- **By Property** — Grouped by property
- **All Active** — Everything not archived

## Response Engine

### Input Context

For each response, the engine receives:

1. **Lead's message** (and full thread history for follow-ups)
2. **Property config** — All structured fields plus raw listing copy
3. **Screening data** — Zillow-provided data if available
4. **Pre-leasing status** — If `pre_leasing: true`, the engine offers virtual tours/photos/video instead of in-person showings

### Integration

The response engine invokes the existing Claude Skill that already handles:
- Voice and tone matching
- Property-specific Q&A
- Screening criteria awareness

The plugin passes the above context to the skill and receives a drafted response.

### Iteration

Users can refine drafted responses conversationally:
- "Make it shorter"
- "Add the pet deposit info"
- "Be more formal"
- "Ask about their move-in timeline"

Each iteration updates the draft in the lead's Notion page.

## Review Workflow

### Slash Commands

| Command | Description |
|---|---|
| `/leads:setup` | First-run setup — creates Notion database, initializes state file |
| `/leads:check` | Poll Gmail for new leads, ingest them, draft responses |
| `/leads:review` | Show next lead needing review — displays full thread + draft response |
| `/leads:edit` | Conversational iteration on a draft |
| `/leads:approve` | Mark response as approved, copy to clipboard for pasting to platform |
| `/leads:add` | Manually add a lead (Facebook, walk-ins) |
| `/leads:add-property` | Add a property from listing copy or URL; Claude parses into YAML config |
| `/leads:skip` | Dismiss a lead (spam, unqualified, already handled) — moves to Archived |
| `/leads:status` | Summary of leads by status count |

### Approval Workflow

**Phase 1 (launch):** All responses require manual approval via `/leads:approve`.

**Phase 2 (future):** Configurable auto-send rules. Routine responses (availability confirmations, basic property info) can be auto-approved based on message classification. The `Auto-Send Eligible` field in Notion tracks which leads qualify.

### Automation

`/loop 15m /leads:check` runs ingestion automatically every 15 minutes. New leads appear in Notion with drafted responses ready for review. The interval is configurable.

## Error Handling

The system favors creating partial records over dropping leads. Errors are surfaced to the user, never silently swallowed.

| Failure | Behavior |
|---|---|
| Gmail MCP unavailable / auth expired | `/leads:check` reports the error and exits. No leads are processed. User is prompted to reconnect Gmail MCP. |
| Notion MCP unavailable | `/leads:check` reports the error and exits. No state is updated — the same emails will be reprocessed on the next successful run. |
| Playwright scrape fails (Apartments.com or Avail) | Lead is created in Notion with available email data (name, email, property). Status set to "Needs Manual Review." User notified. |
| Property match fails | Lead created with Property = "Unmatched." User notified to assign manually or add a property config. |
| Response engine fails | Lead remains at "New" status. Error logged. User can retry via `/leads:review`. |
| Notion page creation fails for a single lead | Error reported for that lead. Remaining leads in the batch continue processing. |

Note on config changes: Drafted responses are generated from the property config at the time of drafting. If the config changes (e.g., rent update), already-drafted responses are not automatically regenerated. The user can re-draft via `/leads:review` if needed.

## Property Configuration

### Schema

```yaml
# properties/30-cooper-st-unit-30a.yaml
address: "30 Cooper St, Unit 30A"
city: "Manchester"
state: "CT"
zip: "06040"
bedrooms: 2
bathrooms: 1
rent: 1895
available_date: "2026-05-01"
pre_leasing: true  # Unit occupied — no in-person showings

security_deposit: "1 month"
application_fee: 50  # per leaseholder

renters_insurance:
  required: true
  min_liability: 100000

pet_policy:
  allowed: true
  fee: 350              # one-time, non-refundable
  monthly_rent: "25-50" # based on FIDO score
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
  - "Dishwasher"
  - "Gas range"
  - "Central AC"
  - "Programmable thermostats"
  - "Keyless entry"
  - "Private first-floor entrance"

screening:
  min_credit_score: 650
  min_income_multiplier: 3
  max_occupants: 2

lease_terms:
  - 12

listing_copy: |
  (Full listing text stored here for response engine context)
```

### Adding Properties

The `/leads:add-property` command accepts:
- Pasted listing copy (Claude parses into YAML structure)
- A URL to a live listing (Playwright MCP scrapes the listing page)

The user reviews and confirms the parsed config before it's saved. Conversational editing supported ("rent is actually $1,250", "add that utilities are not included").

### Removing Properties

Delete the YAML file or move it to a `properties/inactive/` directory.

## Distribution Model

The plugin is designed to be shareable with other landlords who use Claude Code:

1. Install the `lead-pilot` plugin
2. Connect Gmail MCP, Notion MCP, and Playwright MCP
3. Run `/leads:setup` (creates the Notion database with the correct schema and initializes `.lead-pilot/state.json`)
4. Create property configs via `/leads:add-property`
5. Run `/leads:check` or set up `/loop` for automatic polling

Each landlord's data is isolated — their own Gmail, their own Notion workspace, their own property configs.

**V1 scope note:** The distribution packaging (npm registry, plugin marketplace listing, onboarding docs) is out of scope for V1. The plugin will be developed and tested for the author's own use first. Sharing mechanics will be addressed once the core workflow is validated.

## Future Enhancements (Out of Scope for V1)

- Auto-send rules for routine responses
- Facebook Messenger scraping via Playwright
- Web dashboard for non-Claude-Code users
- Lead scoring and prioritization logic
- Analytics (response time tracking, conversion rates)
- Multi-user support (property management teams)
