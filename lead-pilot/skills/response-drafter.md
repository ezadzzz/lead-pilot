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