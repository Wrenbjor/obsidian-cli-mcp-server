# Epic 7: Daily Notes

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/daily.ts`

---

## Overview

Four tools for working with Obsidian's daily notes feature. Daily notes are date-stamped notes created by Obsidian's built-in Daily Notes plugin (or the community Periodic Notes plugin). These tools interact with today's daily note — creating it if needed, reading it, and appending or prepending content.

All four tools share the same critical error state: `DAILY_NOTE_NOT_CONFIGURED`, returned when the Daily Notes plugin is not set up. The `daily_open` and `daily_append` tools implicitly create today's note if it doesn't exist. The `daily_read` and `daily_prepend` tools require the note to already exist.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Design Notes (Read Before Implementing)

The `DAILY_NOTE_NOT_CONFIGURED` error is distinct from `NO_DAILY_NOTE` (used in Epic 6 for task queries). The distinction:

- `DAILY_NOTE_NOT_CONFIGURED` → the plugin itself is not set up; cannot work at all
- `NO_DAILY_NOTE` → plugin is set up, but today's note doesn't exist yet (used in tasks context)

These are separate error codes with separate messages. All four tools in this epic use `DAILY_NOTE_NOT_CONFIGURED` because if the plugin isn't configured, none of them can function. The `daily_open` and `daily_append` tools can create the note if it's missing, so they do not error on a missing daily note — only on a missing plugin configuration.

Implement `DAILY_NOTE_NOT_CONFIGURED` detection in a shared helper used by all four `buildArgs` post-processors in this file, to avoid repetition.

---

## Stories

---

### Story 7.1: `daily_open`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to open or create today's daily note in Obsidian,
So that the user's Obsidian app navigates to today's note immediately.

**Acceptance Criteria:**
- [ ] Tool `daily_open` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution opens today's daily note in the Obsidian UI
- [ ] If today's daily note does not exist, it is created automatically, then opened
- [ ] Returns a confirmation string (not the note contents)
- [ ] When the Daily Notes plugin is not configured, returns `DAILY_NOTE_NOT_CONFIGURED` structured error
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] No parameters beyond global flags are required — "today" is determined by Obsidian using system time

**Tool Definition:**
- **Name:** `daily_open`
- **Description:** `"Open today's daily note in Obsidian, creating it if it doesn't exist yet. This brings the daily note into focus in the Obsidian UI. Does not return note contents — use daily_read for that. Returns an error if the Daily Notes plugin is not configured in Obsidian settings."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string that today's daily note was opened (or created and opened).

**CLI Mapping:**
```
obsidian daily [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "daily",
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin not configured → `{"error": "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.", "code": "DAILY_NOTE_NOT_CONFIGURED"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`DAILY_NOTE_NOT_CONFIGURED` Detection:**
Detect by checking stderr for patterns like "daily notes", "not configured", "plugin not enabled", "no daily notes plugin" (case-insensitive) with a non-zero exit code. Implement as a shared function in `src/tools/daily.ts`:

```typescript
function detectDailyNoteError(result: McpErrorResult): McpErrorResult {
  if (
    result.code === ErrorCode.COMMAND_FAILED &&
    /daily notes.*not configured|not.*daily notes|daily notes plugin|plugin not enabled/i.test(result.detail ?? "")
  ) {
    return {
      error: "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.",
      code: ErrorCode.DAILY_NOTE_NOT_CONFIGURED,
      detail: result.detail,
    };
  }
  return result;
}
```

Apply this function in the tool handler after `execObsidian` returns. Since the registry pattern (Story 0.4) handles all tool execution uniformly, the cleanest integration point is a `transformError` hook on the `ToolDefinition` interface — or handle it inline in a custom handler by bypassing the generic registry for daily tools. Choose whichever approach avoids the most duplication.

**Notes:**
- This is the only daily tool that is valid to call when today's daily note does not yet exist. It creates the note as a side effect.
- The AI should call this before `daily_read` if it is uncertain whether today's note exists, OR simply call `daily_read` and handle `DAILY_NOTE_NOT_CONFIGURED` vs. a missing note gracefully.

---

### Story 7.2: `daily_read`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to read the full contents of today's daily note,
So that I can summarize, analyze, or reference what the user has written today.

**Acceptance Criteria:**
- [ ] Tool `daily_read` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns the full text content of today's daily note
- [ ] Returned content includes frontmatter if present
- [ ] If today's daily note does not exist, returns `NO_DAILY_NOTE` structured error (not `DAILY_NOTE_NOT_CONFIGURED` — the plugin may be configured but today's note not yet created)
- [ ] If the Daily Notes plugin is not configured, returns `DAILY_NOTE_NOT_CONFIGURED` structured error
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] An existing but empty daily note returns empty string (not an error)

**Tool Definition:**
- **Name:** `daily_read`
- **Description:** `"Read the full contents of today's daily note, including frontmatter. Returns the complete note text. Returns an error if the Daily Notes plugin is not configured, or if today's note has not been created yet (use daily_open to create it first). Returns empty string if the note exists but is empty."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Full text content of today's daily note. Empty string if the note exists but is empty.

**CLI Mapping:**
```
obsidian daily:read [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "daily:read",
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin not configured → `{"error": "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.", "code": "DAILY_NOTE_NOT_CONFIGURED"}`
- Today's note not found → `{"error": "Today's daily note does not exist. Use daily_open to create it first.", "code": "NO_DAILY_NOTE"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Error Disambiguation:**
The CLI stderr must be inspected to distinguish between "plugin not configured" and "note not found today". Implement pattern matching as follows:

```typescript
// Apply after execObsidian returns an error
function handleDailyReadError(result: McpErrorResult): McpErrorResult {
  const detail = result.detail ?? "";
  if (/not configured|plugin not enabled/i.test(detail)) {
    return {
      error: "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.",
      code: ErrorCode.DAILY_NOTE_NOT_CONFIGURED,
      detail,
    };
  }
  if (/not found|does not exist|no daily note/i.test(detail)) {
    return {
      error: "Today's daily note does not exist. Use daily_open to create it first.",
      code: ErrorCode.NO_DAILY_NOTE,
      detail,
    };
  }
  return result;
}
```

**Notes:**
- If the CLI does not distinguish between these two failure modes in its stderr output, default to `DAILY_NOTE_NOT_CONFIGURED` (the more actionable error) and document the limitation.
- The AI should handle `NO_DAILY_NOTE` by suggesting `daily_open`, not by treating it as an unrecoverable error.

---

### Story 7.3: `daily_append`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to append content to today's daily note,
So that I can add new entries, notes, or logs without reading and rewriting the full note.

**Acceptance Criteria:**
- [ ] Tool `daily_append` is registered and visible in MCP tool list
- [ ] Input schema: `content` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] Content is added to the very end of today's daily note
- [ ] If today's daily note does not exist, it is created first, then content is appended
- [ ] If the Daily Notes plugin is not configured, returns `DAILY_NOTE_NOT_CONFIGURED` structured error
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Multi-line content is preserved as-is

**Tool Definition:**
- **Name:** `daily_append`
- **Description:** `"Append text to the end of today's daily note. Creates today's note first if it doesn't exist. Use this to log entries, add tasks, or capture thoughts to today's note without reading or overwriting existing content. Returns an error only if the Daily Notes plugin is not configured."`
- **Parameters:**
  - `content`: `string` (required) — Text to append to today's daily note. May be multi-line.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating content was appended.

**CLI Mapping:**
```
obsidian daily:append content=<text> [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "daily:append",
  `content=${params.content}`,
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin not configured → `{"error": "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.", "code": "DAILY_NOTE_NOT_CONFIGURED"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- Unlike `note_append`, this tool does NOT return `FILE_NOT_FOUND` when today's note is missing. It creates the note automatically. This is the expected behavior for daily workflows.
- `content` is passed as a single key=value argument in the `execFile` args array. No shell escaping is required. Multi-line strings with newlines are passed as-is.
- This is the safest write tool for daily workflows: it can only add content, never overwrite.

---

### Story 7.4: `daily_prepend`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to prepend content to the top of today's daily note (after any frontmatter),
So that I can insert priority items at the top of the day's note without disturbing existing content.

**Acceptance Criteria:**
- [ ] Tool `daily_prepend` is registered and visible in MCP tool list
- [ ] Input schema: `content` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] Content is inserted at the beginning of today's daily note body, after any frontmatter
- [ ] When the note has no frontmatter, content is inserted at the very beginning
- [ ] If today's daily note does not exist, returns `NO_DAILY_NOTE` structured error (unlike `daily_append`, this tool does not implicitly create the note)
- [ ] If the Daily Notes plugin is not configured, returns `DAILY_NOTE_NOT_CONFIGURED` structured error
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error

**Tool Definition:**
- **Name:** `daily_prepend`
- **Description:** `"Prepend text to the beginning of today's daily note body, after any YAML frontmatter. Use this to insert high-priority items at the top of the day's note. Unlike daily_append, this does not create today's note if it is missing — use daily_open first if needed. Returns an error if the Daily Notes plugin is not configured or today's note doesn't exist."`
- **Parameters:**
  - `content`: `string` (required) — Text to prepend. May be multi-line.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating content was prepended.

**CLI Mapping:**
```
obsidian daily:prepend content=<text> [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "daily:prepend",
  `content=${params.content}`,
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin not configured → `{"error": "Daily Notes plugin is not configured. Enable it in Obsidian: Settings → Core Plugins → Daily Notes.", "code": "DAILY_NOTE_NOT_CONFIGURED"}`
- Today's note not found → `{"error": "Today's daily note does not exist. Use daily_open to create it first.", "code": "NO_DAILY_NOTE"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- This tool does NOT implicitly create today's note, unlike `daily_append`. This is intentional: prepending to a note that doesn't exist yet would create the note with only the prepended content, which may not match the configured template. Requiring the note to exist first ensures the template was applied.
- If the CLI's actual behavior for `daily:prepend` on a missing note differs (e.g. it always creates), document the discrepancy and adjust the error state accordingly. The tool description should match actual behavior.
- The frontmatter-awareness (inserting after `--- ... ---`) is handled by the Obsidian CLI, not this server.

---

## Implementation Checklist

- [ ] `src/tools/daily.ts` created and exports `dailyTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `dailyTools`
- [ ] All 4 tool names appear in the MCP tool list: `daily_open`, `daily_read`, `daily_append`, `daily_prepend`
- [ ] `DAILY_NOTE_NOT_CONFIGURED` detection shared across all 4 tools (via helper function in `daily.ts`)
- [ ] `daily_read` additionally detects `NO_DAILY_NOTE` (separate from `DAILY_NOTE_NOT_CONFIGURED`)
- [ ] `daily_prepend` returns `NO_DAILY_NOTE` when today's note is absent
- [ ] `daily_append` does NOT return `NO_DAILY_NOTE` — it creates the note implicitly
- [ ] `content` parameter builds as `content=<value>` in CLI args for append/prepend tools
- [ ] `daily_open` and `daily_read` have no required parameters
- [ ] TypeScript strict mode: no `any`

---

## Error Matrix

| Tool | `OBSIDIAN_NOT_RUNNING` | `DAILY_NOTE_NOT_CONFIGURED` | `NO_DAILY_NOTE` | `VAULT_NOT_FOUND` |
|---|---|---|---|---|
| `daily_open` | Yes | Yes | No (creates note) | Yes |
| `daily_read` | Yes | Yes | Yes | Yes |
| `daily_append` | Yes | Yes | No (creates note) | Yes |
| `daily_prepend` | Yes | Yes | Yes | Yes |
