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
