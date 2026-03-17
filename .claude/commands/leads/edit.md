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
