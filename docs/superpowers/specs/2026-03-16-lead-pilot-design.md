# Lead Pilot — Design Spec

**Date:** 2026-03-16
**Status:** Approved
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

The `/leads:check` command searches Gmail via MCP for lead notification emails received after a stored "last checked" timestamp. Read/unread status is ignored — timestamp-based tracking ensures no leads are missed regardless of manual email reading.

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
- Fallback: Playwright MCP clicks "Reply Securely" link to retrieve full message if preview appears truncated

**Apartments.com:**
- From: `lead@apartments.com`
- Reply-to: Lead's personal email (used for deduplication)
- Subject: `"You've received a new lead!"`
- Body: Property address only, no message content
- Required: Playwright MCP clicks "View Message" link to scrape the actual message

#### Gmail Search Patterns

- Zillow: `from:convo.zillow.com after:{last_checked}`
- Avail: `from:support@avail.co subject:"new message on Avail" after:{last_checked}`
- Apartments.com: `from:lead@apartments.com after:{last_checked}`

Gmail account: `silkcityapartments@gmail.com`

### Facebook Messenger (Manual Entry)

Facebook does not send email notifications and actively resists automation. For the initial version, Facebook leads are added manually via `/leads:add`. The system still drafts responses for these leads using the same response engine.

Future consideration: Playwright-based Messenger scraping if manual entry becomes a bottleneck.

### Deduplication

**Within a platform:** Each ingested email is tracked by Gmail message ID to prevent duplicate processing on repeated checks.

**Across platforms:** When ingesting a new lead, the system checks the Notion database for existing leads with the same email address or name + property combination. If a match is found, the new message is appended to the existing lead's thread rather than creating a duplicate. The lead's platform multi-select field is updated to reflect all platforms they've reached out on.

### Property Matching

Incoming leads are matched to a property by comparing the address in the email (subject line or body) against the `address`, `city`, `state`, and `zip` fields in property config files. Fuzzy matching handles minor formatting differences (e.g., "30 Cooper St" vs "30 Cooper Street").

## Lead Database (Notion)

### Database: "Leads"

Each lead is a Notion page with these properties:

| Property | Type | Description |
|---|---|---|
| Name | Title | Lead's name |
| Email | Email | Lead's email address |
| Phone | Phone | If provided |
| Property | Select | Which unit they're asking about |
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
| `/leads:check` | Poll Gmail for new leads, ingest them, draft responses |
| `/leads:review` | Show next lead needing review — displays full thread + draft response |
| `/leads:edit` | Conversational iteration on a draft |
| `/leads:approve` | Mark response as approved, copy to clipboard for pasting to platform |
| `/leads:add` | Manually add a lead (Facebook, walk-ins) |
| `/leads:add-property` | Add a property from listing copy or URL; Claude parses into YAML config |
| `/leads:status` | Summary of leads by status count |

### Approval Workflow

**Phase 1 (launch):** All responses require manual approval via `/leads:approve`.

**Phase 2 (future):** Configurable auto-send rules. Routine responses (availability confirmations, basic property info) can be auto-approved based on message classification. The `Auto-Send Eligible` field in Notion tracks which leads qualify.

### Automation

`/loop 15m /leads:check` runs ingestion automatically every 15 minutes. New leads appear in Notion with drafted responses ready for review. The interval is configurable.

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
2. Connect Gmail MCP and Notion MCP
3. Create property configs via `/leads:add-property`
4. Run `/leads:check` or set up `/loop` for automatic polling

Each landlord's data is isolated — their own Gmail, their own Notion workspace, their own property configs.

## Future Enhancements (Out of Scope for V1)

- Auto-send rules for routine responses
- Facebook Messenger scraping via Playwright
- Web dashboard for non-Claude-Code users
- Lead scoring and prioritization logic
- Analytics (response time tracking, conversion rates)
- Multi-user support (property management teams)
