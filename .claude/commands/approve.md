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
