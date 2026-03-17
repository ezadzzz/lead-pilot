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

Same process as /leads:check step 7. Set `received_at` to now.

### 5. Draft response

Same process as /leads:check step 8. Use @lead-pilot-response-drafter.

### 6. Confirm

```
✅ Lead created: {name} ({platform})

Draft response generated. Run /leads:review to see it.
```
