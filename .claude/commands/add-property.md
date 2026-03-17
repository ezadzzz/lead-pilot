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
