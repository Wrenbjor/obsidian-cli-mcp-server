# Epic 6: Tasks

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/tasks.ts`

---

## Overview

Three tools for querying tasks (checkboxes) across the vault. Tasks are Obsidian's markdown checkbox items: `- [ ] unchecked` and `- [x] checked`. These tools return task content, source file, and line context so the AI can report on, summarize, or reference specific tasks.

All three tools are read-only — they do not create, update, or complete tasks. Zero results is never an error. The `tasks_daily` tool has a unique error state: `NO_DAILY_NOTE`, returned when today's daily note doesn't exist or the daily notes plugin isn't configured.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 6.1: `tasks_all`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all tasks across the entire vault,
So that I can give the user a comprehensive view of every open and closed task regardless of where they live.

**Acceptance Criteria:**
- [ ] Tool `tasks_all` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns all tasks across the vault (both checked and unchecked)
- [ ] Each task entry includes: the task text, the source file path, and line context (line number or surrounding text)
- [ ] When no tasks exist in the vault, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Tasks from all notes are included, not just the active or recent notes

**Tool Definition:**
- **Name:** `tasks_all`
- **Description:** `"List all tasks (checkbox items) across the entire vault. Returns both completed and incomplete tasks. Each result includes the task text, source file path, and line context. Returns empty if the vault has no tasks. Use tasks_todo to filter to only incomplete tasks, or tasks_daily to see only today's tasks."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** All tasks across the vault with file path and line context. Format depends on CLI default. Empty string if no tasks.

**CLI Mapping:**
```
obsidian tasks [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "tasks",
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- This returns both checked (`- [x]`) and unchecked (`- [ ]`) tasks. Use `tasks_todo` to filter to only incomplete tasks.
- On large vaults this may produce substantial output. The AI should prefer `tasks_todo` or `tasks_daily` when the user only cares about open items.
- The output format (how the CLI represents "task text + source file + line") should be documented in a comment in the implementation once observed. No `transformOutput` is needed — raw output is returned.

---

### Story 6.2: `tasks_daily`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all tasks from today's daily note,
So that I can give the user a focused view of what they planned to do today.

**Acceptance Criteria:**
- [ ] Tool `tasks_daily` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns all tasks (checked and unchecked) from today's daily note
- [ ] Each task entry includes the task text and line context
- [ ] When today's daily note exists but has no tasks, returns an empty string (not an error)
- [ ] When today's daily note does not exist, returns `NO_DAILY_NOTE` structured error
- [ ] When the daily notes plugin is not configured, returns `NO_DAILY_NOTE` structured error
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] "Today" is determined by Obsidian/the CLI using system time — the MCP server does not inject a date

**Tool Definition:**
- **Name:** `tasks_daily`
- **Description:** `"List all tasks from today's daily note. Returns both completed and incomplete tasks from the daily note for the current date. Returns an error if today's daily note does not exist or the Daily Notes plugin is not configured. Returns empty if the daily note exists but has no tasks."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** All tasks from today's daily note with line context. Empty string if the note has no tasks.

**CLI Mapping:**
```
obsidian tasks daily [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "tasks",
  "daily",
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- No daily note → `{"error": "Today's daily note was not found. Create it with daily_open or check that the Daily Notes plugin is configured.", "code": "NO_DAILY_NOTE"}`
- Plugin not configured → `{"error": "Today's daily note was not found. Create it with daily_open or check that the Daily Notes plugin is configured.", "code": "NO_DAILY_NOTE"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`NO_DAILY_NOTE` Detection:**
Detect by checking stderr for patterns like "daily note not found", "no daily note", "daily notes plugin", or "not configured" (case-insensitive) with a non-zero exit code. The "not found" and "not configured" states produce the same error code and same user message — the user action is the same either way (open/create the daily note or configure the plugin).

```typescript
// Post-execution detection in tool handler or registry
if (isError(result) && result.code === ErrorCode.COMMAND_FAILED) {
  if (/daily note not found|no daily note|daily notes plugin|not configured/i.test(result.detail ?? "")) {
    return formatMcpError({
      error: "Today's daily note was not found. Create it with daily_open or check that the Daily Notes plugin is configured.",
      code: ErrorCode.NO_DAILY_NOTE,
      detail: result.detail,
    });
  }
}
```

**Notes:**
- The `daily` subcommand is a positional argument, not a key=value pair. It is placed immediately after `tasks` in the args array.
- This tool is designed to be called at the start of a morning workflow. The AI should suggest `daily_open` or `daily_append` as follow-up actions if the user has tasks to add.

---

### Story 6.3: `tasks_todo`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list only incomplete (unchecked) tasks across the vault,
So that I can give the user an actionable list of outstanding work without the noise of already-completed items.

**Acceptance Criteria:**
- [ ] Tool `tasks_todo` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns only unchecked tasks (`- [ ]`) from across the vault
- [ ] Completed tasks (`- [x]`) are excluded
- [ ] Each task entry includes the task text, source file path, and line context
- [ ] When no incomplete tasks exist, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error

**Tool Definition:**
- **Name:** `tasks_todo`
- **Description:** `"List all incomplete (unchecked) tasks across the vault. Returns only tasks marked '- [ ]', not completed '- [x]' tasks. Each result includes the task text, source file path, and line context. Returns empty if there are no incomplete tasks. Use this to get an actionable list of outstanding work."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Incomplete tasks across the vault with file path and line context. Empty string if all tasks are complete or no tasks exist.

**CLI Mapping:**
```
obsidian tasks todo [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "tasks",
  "todo",
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- The `todo` subcommand is a positional argument placed immediately after `tasks`.
- This is the most focused of the three task tools and the most appropriate default when the user asks "what are my tasks?" or "what do I need to do?"
- The AI should note that this returns vault-wide todos, which can be large. It may be appropriate to follow up with `tasks_daily` if the user wants to focus on today's work specifically.

---

## Implementation Checklist

- [ ] `src/tools/tasks.ts` created and exports `taskTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `taskTools`
- [ ] All 3 tool names appear in the MCP tool list: `tasks_all`, `tasks_daily`, `tasks_todo`
- [ ] `tasks_all` builds args as `["tasks", ...globalArgs]`
- [ ] `tasks_daily` builds args as `["tasks", "daily", ...globalArgs]`
- [ ] `tasks_todo` builds args as `["tasks", "todo", ...globalArgs]`
- [ ] `NO_DAILY_NOTE` detection implemented for `tasks_daily`
- [ ] Empty results return empty string (not error) for all 3 tools
- [ ] TypeScript strict mode: no `any`
