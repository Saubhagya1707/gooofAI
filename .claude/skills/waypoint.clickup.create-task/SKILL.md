---
description: "Create a task in ClickUp with smart epic matching and per-epic memory"
---

# ClickUp: Create Task

<!-- MCP tool prefix assumed: mcp__claude_ai_ClickUp__ (e.g. mcp__claude_ai_ClickUp__clickup_get_workspace_hierarchy).
     If the ClickUp connector changes its naming scheme, update all tool references in this file and clickup.setup.md. -->

Create a task in ClickUp with smart epic matching, per-epic memory for priority/assignee defaults, and persistent project/list/epic selection.

**Prerequisite**: Run `/waypoint.clickup.setup` first to connect ClickUp and configure your workspace.

## Preconditions

Before starting, check these three conditions. If any fail, **stop immediately** with the indicated message.

### Check 1: clickup-local.json exists

Read `.waypoint/integrations/clickup-local.json`.

- **If the file does not exist**: display and stop:
  > "ClickUp is not configured. Run `/waypoint.clickup.setup` first to configure ClickUp."

### Check 2: Setup is complete (not pending)

- **If the file has `"status": "pending-mcp"`**: display and stop:
  > "ClickUp setup is incomplete. Start a new session (`Ctrl+D`, then `claude`) and run `/waypoint.clickup.setup` to finish connecting."

### Check 3: MCP tools are available

Check if any `mcp__claude_ai_ClickUp__*` tools are in the tool list.

- **If NO tools are found**: display and stop:
  > "ClickUp MCP not connected. Run `/waypoint.clickup.setup` to get started."

If all three checks pass, proceed to Epic Selection.

## Epic Selection

### Load state and fetch epics

1. Read `.waypoint/integrations/clickup-local.json` for saved state.
2. Call `mcp__claude_ai_ClickUp__clickup_filter_tasks` on the saved `listId` to fetch the 10 most recent top-level tasks (epics). Always fetch fresh — do not use cached data.

### Smart epic match

- **If `$ARGUMENTS` is non-empty**: use it to analyze against the fetched epic names and descriptions.
  - **If there is a strong semantic match**: suggest that epic to the user:
    > "This looks related to **[Epic Name]** in **[List Name]**. Add to this epic?"
  - **If no strong match**: fall through to the default suggestion below.
- **If `$ARGUMENTS` is empty**: skip smart matching entirely — the task name hasn't been collected yet.

**Default suggestion** (when smart match is skipped or finds no match):
- If `currentEpicTaskId` is set in `clickup-local.json`: suggest it:
  > "Add to epic **[Current Epic Name]** in **[List Name]**?"
- If `currentEpicTaskId` is null (last task was standalone): skip suggestion and show the epic picker directly (same as "User declines" → "Same project" flow below).

- **If no epics exist in the list**: skip matching entirely. Offer "Create new epic" or "Create standalone task".

### User confirms or overrides

- **User confirms**: proceed to Task Details with the selected epic.
- **User declines**:
  1. Ask: "Same project **[List Name]**?" (default: **yes**)
  2. **If yes (same project)**:
     - Show the 10 most recent epics as a numbered list, plus:
       - **"Create new epic"** option
       - **"No epic (standalone task)"** option
     - If the user can't find their epic, they can type a name to search.
     - User picks an epic (or creates new / standalone).
  3. **If no (different project)**:
     - Call `mcp__claude_ai_ClickUp__clickup_get_workspace_hierarchy` and run hierarchy selection (Space > Folder > List, adaptive depth).
     - Then fetch epics from the new list and let the user select.
     - Ask: "Switch your default project to **[List Name]** for future tasks?" (default: **no**)
       - If **yes**: update `clickup-local.json` with the new list and epic immediately.
       - If **no**: use the selected list/epic for this task only. Do not update `clickup-local.json`.

**Creating a new epic on-the-fly**: If the user picks "Create new epic" at any point, ask for the epic name and call `mcp__claude_ai_ClickUp__clickup_create_task` with `tags: ["epic"]` in the current list. Use the returned task ID as the parent for the new task.

## Task Details

### Task name

- **If `$ARGUMENTS` is provided and non-empty**: use it as the task name. Do not prompt.
- **If `$ARGUMENTS` is empty or not provided**: ask the user: "What is the task name?"

### Priority

Look up the current epic in `epicMemory` from `clickup-local.json`.

Ask the user:
> Priority? (a) urgent, (b) high, (c) medium, (d) low — default: **[lastPriority or "medium"]**

Accept a single letter (a/b/c/d) or the full word. If the user presses Enter with no input, use the default.

### Assignee

Ask the user:
> Assignee? — default: **[lastAssignee or "unassigned"]**

Accept a name or "unassigned". If the user presses Enter with no input, use the default.

### Optional fields

Ask the user:
> Description? (press Enter to skip)

> Tags? (comma-separated, press Enter to skip)

## Create Task

### Confirmation preview

Display a summary before creating:

```
Task Preview
─────────────────────────────────
  Name:      <task name>
  Epic:      <epic name or "Standalone">
  List:      <list name>
  Priority:  <priority>
  Assignee:  <assignee>
  Desc:      <description or "(none)">
  Tags:      <tags or "(none)">
─────────────────────────────────
Create this task?
```

If user declines, return to Task Details.

### Resolve assignee

If the assignee is not "unassigned", call `mcp__claude_ai_ClickUp__clickup_resolve_assignees` to map the name to a ClickUp user ID. If the name cannot be resolved, inform the user and ask for a different name or "unassigned".

### Create the task

Call `mcp__claude_ai_ClickUp__clickup_create_task` with:

| Parameter | Value |
|-----------|-------|
| `list_id` | from `clickup-local.json` listId |
| `name` | task name |
| `description` | description (if provided) |
| `priority` | mapped value: urgent=1, high=2, medium=3, low=4 |
| `parent` | epicTaskId (or omit if standalone) |
| `assignees` | [resolved user ID] (or omit if unassigned) |
| `tags` | tags array (or omit if none) |

### Handle result

- **Success**: Display the task URL:
  ```
  Task created: <task name>
  URL: <task url>
  ```
  Proceed to Update State.

- **Failure**: Display the error clearly, then ask:
  > "Retry with the same parameters?"
  - **Yes**: retry the `clickup_create_task` call.
  - **No**: exit gracefully. The user can re-invoke the skill.

## Update State

After successful task creation, update `clickup-local.json`:

1. Set `currentEpicTaskId` and `currentEpicName` to the epic used (or null if standalone)
2. Update (or create) the entry in `epicMemory` for the epic used:
   ```json
   {
     "epicName": "<epic name>",
     "lastPriority": "<priority chosen>",
     "lastAssignee": "<assignee chosen>",
     "lastUsed": "<today ISO date>"
   }
   ```
3. Set `lastUpdated` to today's ISO date
4. Write the updated JSON back to `.waypoint/integrations/clickup-local.json`

If the task was standalone (no epic), skip updating epicMemory but still update currentEpicTaskId to null.

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

## Arguments

$ARGUMENTS
