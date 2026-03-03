# Epic 3: Properties (Frontmatter)

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/properties.ts`

---

## Overview

Three tools for reading and mutating YAML frontmatter (properties) on individual notes. Obsidian stores structured metadata in the frontmatter block at the top of each `.md` file. These tools allow the AI to inspect, set, and remove individual properties without touching the note body.

All tools target a specific file — there are no vault-wide property mutation tools (that would be too broad to be safe). All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 3.1: `properties_list`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all frontmatter properties of a note,
So that I can inspect its metadata before deciding how to interact with it.

**Acceptance Criteria:**
- [ ] Tool `properties_list` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `format` (enum, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns all frontmatter properties from the note
- [ ] When `format=yaml`, output is formatted as YAML key-value pairs
- [ ] When `format=tsv`, output is formatted as tab-separated values (key\tvalue per line)
- [ ] When `format` is omitted, CLI default format is returned
- [ ] Notes with no frontmatter return an empty result (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Array-valued properties are returned in full (not truncated)

**Tool Definition:**
- **Name:** `properties_list`
- **Description:** `"List all YAML frontmatter properties of a note. Returns all key-value pairs from the frontmatter block. Use format='yaml' for YAML output or format='tsv' for tab-separated key/value pairs. Returns empty if the note has no frontmatter. Use this before property_set or property_remove to inspect current state."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path (e.g. `"Projects/Sprint.md"`).
  - `format`: `string` (optional, enum: `"yaml"`, `"tsv"`) — Output format.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** All frontmatter properties in the requested format. Empty string if no frontmatter.

**CLI Mapping:**
```
obsidian properties file=<path> [format=<yaml|tsv>] [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "properties",
  `file=${params.file}`,
  ...buildGlobalArgs(params),
],
```

**Notes:**
- This is a read-only operation — it does not modify the note.
- The AI should use this tool before `property_remove` to confirm a property exists, since `property_remove` returns `PROPERTY_NOT_FOUND` for absent properties and the AI may need to communicate this gracefully to the user.
- Property values that are arrays (e.g. `tags: [a, b, c]`) should be returned in their full form, not flattened. The CLI handles serialization.

---

### Story 3.2: `property_set`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to set a frontmatter property on a note to a specific value,
So that I can update metadata without editing the note body.

**Acceptance Criteria:**
- [ ] Tool `property_set` is registered and visible in MCP tool list
- [ ] Input schema: `name` (string, required), `value` (string, required), `file` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] If the property already exists, its value is updated
- [ ] If the property does not exist, it is created
- [ ] The note body (content below frontmatter) is not modified
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Setting a property on a note with no existing frontmatter creates the frontmatter block

**Tool Definition:**
- **Name:** `property_set`
- **Description:** `"Set a frontmatter property on a note. If the property exists, its value is updated. If it does not exist, it is created. Setting a property on a note with no frontmatter creates the frontmatter block. Does not modify the note body. Example: set name='status' value='done' on file='Projects/Sprint.md' to mark a note complete."`
- **Parameters:**
  - `name`: `string` (required) — Property key to set (e.g. `"status"`, `"due-date"`, `"priority"`).
  - `value`: `string` (required) — Value to assign. Strings, numbers, and dates are accepted as strings and the CLI handles type coercion per Obsidian's property types.
  - `file`: `string` (required) — Note name or vault-relative path.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating the property was set.

**CLI Mapping:**
```
obsidian property:set name=<key> value=<value> file=<path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "property:set",
  `name=${params.name}`,
  `value=${params.value}`,
  `file=${params.file}`,
  ...buildGlobalArgs(params),
],
```

**Notes:**
- The `value` parameter is typed as `string` in the MCP schema for simplicity. The Obsidian CLI is responsible for coercing string representations into the correct YAML type based on the property's declared type in Obsidian's property configuration. This server does not attempt type coercion.
- Passing an empty string as `value` will set the property to an empty value, not remove it. Use `property_remove` to delete a property.
- The arg order matters: `name`, `value`, `file`, then global flags.

---

### Story 3.3: `property_remove`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to remove a specific frontmatter property from a note,
So that I can clean up stale metadata without affecting other properties or the note body.

**Acceptance Criteria:**
- [ ] Tool `property_remove` is registered and visible in MCP tool list
- [ ] Input schema: `name` (string, required), `file` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] Only the specified property is removed; all other properties and note body are unchanged
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When the property does not exist on the note, returns `PROPERTY_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Removing the last property from a frontmatter block does not break the file (the empty frontmatter block `---\n---` is removed or left, per CLI behavior — document whichever)

**Tool Definition:**
- **Name:** `property_remove`
- **Description:** `"Remove a specific frontmatter property from a note. Only the named property is removed; all other properties and the note body are unaffected. Returns an error if the note does not exist or the property is not present. Use properties_list first to confirm the property exists if you are unsure."`
- **Parameters:**
  - `name`: `string` (required) — Property key to remove (e.g. `"status"`, `"due-date"`).
  - `file`: `string` (required) — Note name or vault-relative path.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating the property was removed.

**CLI Mapping:**
```
obsidian property:remove name=<key> file=<path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Property not found → `{"error": "Property '{name}' was not found on '{file}'.", "code": "PROPERTY_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "property:remove",
  `name=${params.name}`,
  `file=${params.file}`,
  ...buildGlobalArgs(params),
],
```

**`PROPERTY_NOT_FOUND` Detection:**
Detect by checking stderr for patterns like "property not found", "no such property", or "does not have property" (case-insensitive) with a non-zero exit code. This detection should be implemented in the tool's post-execution handler (not the core executor) since it is domain-specific. The message should substitute the actual property name and file for clarity:

```typescript
// In the registry or tool-level handler, after execObsidian returns:
if (isError(result) && result.code === ErrorCode.COMMAND_FAILED) {
  if (/property not found|no such property|does not have property/i.test(result.detail ?? "")) {
    return formatMcpError({
      error: `Property '${params.name}' was not found on '${params.file}'.`,
      code: ErrorCode.PROPERTY_NOT_FOUND,
      detail: result.detail,
    });
  }
}
```

**Notes:**
- If the CLI returns a different error pattern for missing properties, adjust the regex accordingly. Test against actual CLI output during implementation.
- The `PROPERTY_NOT_FOUND` code is pre-defined in `src/types.ts` (added in Epic 0.3) so there is no need to add it.

---

## Implementation Checklist

- [ ] `src/tools/properties.ts` created and exports `propertyTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `propertyTools`
- [ ] All 3 tool names appear in the MCP tool list: `properties_list`, `property_set`, `property_remove`
- [ ] `file` parameter builds as `file=<value>` in all CLI args
- [ ] `name` parameter builds as `name=<value>` in CLI args
- [ ] `value` parameter builds as `value=<value>` in CLI args
- [ ] `PROPERTY_NOT_FOUND` detection implemented in `property_remove` handler
- [ ] Empty frontmatter returns empty string (not error) from `properties_list`
- [ ] `property_set` creates frontmatter if none exists (CLI handles; tool description documents this)
- [ ] TypeScript strict mode: no `any`
