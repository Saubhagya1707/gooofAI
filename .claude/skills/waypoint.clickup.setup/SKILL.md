---
description: "Connect ClickUp MCP and configure workspace defaults"
---

# ClickUp: Setup

<!-- MCP tool prefix assumed: mcp__claude_ai_ClickUp__ (e.g. mcp__claude_ai_ClickUp__clickup_get_workspace_hierarchy).
     If the ClickUp connector changes its naming scheme, update all tool references in this file and clickup.create-task.md. -->

One-time setup for the ClickUp integration. Connects the ClickUp MCP server and configures workspace/project/epic defaults for `/waypoint.clickup.create-task`.

## Step 1: Check MCP Connection

### 1.1: Inspect tool list

Check if any `mcp__claude_ai_ClickUp__*` tools are available in the current session.

- **If tools are found**: call `mcp__claude_ai_ClickUp__clickup_get_workspace_hierarchy` to verify they work (auth may be expired).
  - **If the call succeeds**: store the hierarchy response. Proceed to Step 2.
  - **If the call fails** (auth error): display the error and suggest the user reconnect ClickUp at `https://claude.ai/customize/connectors`, then restart the session.
- **If NO tools are found**: proceed to Step 1.2.

### 1.2: Guide MCP connection

Display the following:

```
ClickUp MCP Setup
───────────────────────────────────
Step 1 of 2: Connect ClickUp to Claude

1. Open: https://claude.ai/customize/connectors
2. Find "ClickUp" and click "Connect"
3. Authorize ClickUp access in the browser popup

Then restart Claude Code to load the new connection:
4. Press Ctrl+D to exit this session
5. Run `claude` to start a new session
6. Run /waypoint.clickup.setup again to finish setup

Why? MCP integrations are loaded at session start.
/clear won't work — you need a full restart.
```

### 1.3: Write breadcrumb

Write `.waypoint/integrations/clickup-local.json`:

```json
{
  "status": "pending-mcp",
  "createdAt": "<today ISO date>"
}
```

This breadcrumb tells the next invocation that setup was started but MCP wasn't available yet.

**Exit the skill cleanly after writing the breadcrumb.** Do not proceed to Step 2.

### 1.4: Resume detection

On any invocation, read `.waypoint/integrations/clickup-local.json` first.

- **If the file has `"status": "pending-mcp"`**: this is a resume. Run Step 1.1 to check if MCP is now available.
  - If MCP is now available: display "ClickUp connected! Let's configure your workspace." and proceed to Step 2.
  - If MCP is still not available: display the connection instructions again (Step 1.2). Update the breadcrumb's `createdAt` and exit.
- **If the file has full state** (spaceId, listId, etc. — not `pending-mcp`): this is a re-setup. Ask the user: "You already have a ClickUp setup configured. Re-configure?" (default: **yes**). If they decline, exit.
- **If the file does not exist**: fresh setup. Run Step 1.1.

## Step 2: Configure Workspace

This step only runs when MCP tools are available and verified.

### 2.1: Hierarchy selection (Space > Folder > List)

Using the workspace hierarchy from Step 1.1, present the selection **adaptively**:

1. **Spaces**: List all spaces as a numbered list. Ask the user to pick one.
2. **Folders** (adaptive): Check if the selected space has folders.
   - If **yes**: list folders. Ask the user to pick one. Then list that folder's lists.
   - If **no folders**: skip directly to lists under the space.
3. **Lists**: List all lists in the selected folder (or space). Ask the user to pick one.

### 2.2: Create or reuse clickup.json

Read `.waypoint/integrations/clickup.json`.

- **If it exists** (a teammate already committed it): ask the user:
  > "StatusMapping was configured by a teammate. Re-detect from workspace?" (default: **no**)
  - If **no**: keep existing `clickup.json`. Skip to Step 2.3.
  - If **yes**: continue below to overwrite.
- **If it does not exist**: continue below.

Using the workspace hierarchy from Step 1.1 and the list selected in Step 2.1:

1. Extract the `workspaceId` from the hierarchy response
2. Call `mcp__claude_ai_ClickUp__clickup_get_list` on the selected list to fetch its statuses
3. Auto-generate a `statusMapping` from the list's statuses:
   - Statuses containing "progress" → `"in-progress"`
   - Statuses containing "done", "closed", "complete" → `"done"`
   - Statuses containing "block" → `"blocked"`
   - Statuses containing "review" → `"in-review"`
   - All others → `"todo"`
4. Show the mapping to the user and ask them to confirm or tweak
5. Write `.waypoint/integrations/clickup.json`:

```json
{
  "type": "clickup",
  "workspaceId": "<from hierarchy>",
  "statusMapping": {
    "<status name>": "<normalized value>",
    ...
  },
  "enabled": true
}
```

### 2.3: Epic selection

1. Call `mcp__claude_ai_ClickUp__clickup_filter_tasks` on the chosen list to fetch top-level tasks (potential epics). Limit to the 10 most recent.
2. Present the list to the user with options:
   - Numbered list of existing epics
   - **"Create new epic"** option
   - **"No epic (standalone task)"** option
3. **If user picks an existing epic**: record the epicTaskId and epicName.
4. **If user picks "Create new epic"**: ask for the epic name, then call `mcp__claude_ai_ClickUp__clickup_create_task` with `tags: ["epic"]` in the chosen list. Record the returned task ID and name.
5. **If user picks "No epic"**: set currentEpicTaskId to null.
6. **If no epics exist in the list**: skip the list and only offer "Create new epic" or "No epic (standalone task)".

### 2.4: Save state

If re-configuring an existing setup, **preserve the existing `epicMemory`** — only update the space/folder/list/epic fields.

Write `.waypoint/integrations/clickup-local.json`:

```json
{
  "spaceId": "<selected>",
  "spaceName": "<selected>",
  "folderId": "<selected or null>",
  "folderName": "<selected or null>",
  "listId": "<selected>",
  "listName": "<selected>",
  "currentEpicTaskId": "<selected or null>",
  "currentEpicName": "<selected or null>",
  "epicMemory": { ... },
  "lastUpdated": "<today ISO date>"
}
```

### 2.5: Ensure clickup-local.json is gitignored

Read `.gitignore` and check if it contains `.waypoint/integrations/clickup-local.json`.

- **If already present**: do nothing.
- **If missing**: append the following to `.gitignore`:

```
# WayPoint integration local state (personal, not team-shared)
.waypoint/integrations/clickup-local.json
```

This prevents personal ClickUp state from being accidentally committed.

## Step 3: Complete

Display:

```
ClickUp Setup Complete
───────────────────────────────────
  Workspace:  <workspace name>
  Space:      <space name>
  List:       <list name>
  Epic:       <epic name or "None (standalone)">

Run /waypoint.clickup.create-task to create your first task!
```

## Config File Schemas

### .waypoint/integrations/clickup.json (team config — committed)

```json
{
  "type": "clickup",
  "workspaceId": "string — ClickUp workspace ID",
  "statusMapping": {
    "status name": "normalized value (todo|in-progress|done|blocked|in-review|backlog)"
  },
  "enabled": true
}
```

### .waypoint/integrations/clickup-local.json (personal state — gitignored)

Full state (after successful setup):
```json
{
  "spaceId": "string",
  "spaceName": "string",
  "folderId": "string or null",
  "folderName": "string or null",
  "listId": "string",
  "listName": "string",
  "currentEpicTaskId": "string or null",
  "currentEpicName": "string or null",
  "epicMemory": {
    "<epicTaskId>": {
      "epicName": "string",
      "lastPriority": "urgent | high | medium | low",
      "lastAssignee": "string — name or 'unassigned'",
      "lastUsed": "ISO date string"
    }
  },
  "lastUpdated": "ISO date string"
}
```

Breadcrumb state (MCP not yet connected):
```json
{
  "status": "pending-mcp",
  "createdAt": "ISO date string"
}
```

## Arguments

$ARGUMENTS
