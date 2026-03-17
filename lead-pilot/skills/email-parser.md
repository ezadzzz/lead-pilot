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
