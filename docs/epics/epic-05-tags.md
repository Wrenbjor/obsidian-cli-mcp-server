# Epic 5: Tags

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/tags.ts`

---

## Overview

Two tools for working with Obsidian tags. The first lists all tags in the vault, optionally with usage counts. The second filters notes by a specific tag, returning every note that contains it. Both tools are read-only. Zero results is never an error.

Tags in Obsidian appear as `#tagname` in note content and/or as entries in the `tags` frontmatter property. The CLI handles the union of both sources — these tools do not need to distinguish between inline tags and frontmatter tags.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 5.1: `tags_list`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all tags used in the vault,
So that I can understand how the vault is categorized and help the user navigate or apply tags correctly.

**Acceptance Criteria:**
- [ ] Tool `tags_list` is registered and visible in MCP tool list
- [ ] Input schema: `sort` (enum, optional), `counts` (boolean, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a list of all tag names used across the vault
- [ ] When `sort=count`, results are sorted by usage count (most used first)
- [ ] When `counts` is `true`, each tag is returned with its usage count (e.g. `"#project (14)"` or structured per CLI format)
- [ ] When no tags exist in the vault, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Tag names are returned with the `#` prefix (e.g. `#project`, not `project`)
- [ ] Nested tags (e.g. `#project/active`) are returned as full paths

**Tool Definition:**
- **Name:** `tags_list`
- **Description:** `"List all tags used in the Obsidian vault. Returns tag names with the '#' prefix (e.g. '#project', '#project/active'). Use sort='count' to order by usage frequency (most-used first). Use counts=true to include the number of notes each tag appears in. Returns empty if no tags are used in the vault."`
- **Parameters:**
  - `sort`: `string` (optional, enum: `"count"`) — Sort order. `count` = most-used first.
  - `counts`: `boolean` (optional) — Include usage count for each tag.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** List of tag names, optionally with usage counts. Format depends on `counts` flag. Empty string if no tags exist.

**CLI Mapping:**
```
obsidian tags [sort=count] [counts] [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => {
  const args: string[] = ["tags"];
  if (params.sort === "count") args.push("sort=count");
  if (params.counts === true) args.push("counts");
  return [...args, ...buildGlobalArgs(params)];
},
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- The `sort` enum currently has only one value (`"count"`). Define it as an enum in the schema anyway to allow future additions without breaking the API contract.
- The `counts` flag is a boolean CLI flag (not a key=value pair). Pass it as a standalone string: `args.push("counts")`, not `args.push("counts=true")`.
- When `counts=true` and `sort=count` are combined, the output is sorted by count with counts shown — this is the most useful combination for the AI to present to a user.

---

### Story 5.2: `tag_filter`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all notes that contain a specific tag,
So that I can retrieve, summarize, or operate on a set of notes grouped by that tag.

**Acceptance Criteria:**
- [ ] Tool `tag_filter` is registered and visible in MCP tool list
- [ ] Input schema: `name` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a newline-separated list of vault-relative file paths
- [ ] All notes containing the specified tag (inline or frontmatter) are returned
- [ ] When no notes contain the tag, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] The `name` parameter accepts both `#tagname` and `tagname` formats (with or without `#` prefix)
- [ ] Nested tags match exactly — `#project` does not match `#project/active` unless the CLI behavior differs (document whichever is true)

**Tool Definition:**
- **Name:** `tag_filter`
- **Description:** `"List all notes that contain a specific tag. Returns a newline-separated list of vault-relative file paths. Accepts the tag with or without the '#' prefix (e.g. '#project' or 'project'). Returns empty if no notes contain the tag. Use this to gather a set of notes for summarizing, processing, or reporting on a category."`
- **Parameters:**
  - `name`: `string` (required) — Tag name to filter by. Accepts with or without `#` prefix.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Newline-separated list of vault-relative paths for notes containing the specified tag. Empty string if none.

**CLI Mapping:**
```
obsidian tag name=<tagname> [vault=<name>] [copy]
```

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "tag",
  `name=${params.name}`,
  ...buildGlobalArgs(params),
],
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:**
- Note the CLI command is `tag` (singular) not `tags`. The MCP tool name is `tag_filter` to make its purpose clear: it is a filter, not a listing tool.
- The difference between `tags_list` (all tags in vault) and `tag_filter` (all notes with a given tag) must be unambiguous in the tool descriptions. These serve opposite directions: one enumerates tags, the other enumerates notes.
- If the CLI is case-sensitive for tag names, document this. Obsidian tags are typically case-insensitive in the app but behavior may vary in CLI.

---

## Implementation Checklist

- [ ] `src/tools/tags.ts` created and exports `tagTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `tagTools`
- [ ] Both tool names appear in the MCP tool list: `tags_list`, `tag_filter`
- [ ] `tags_list` passes `sort=count` (not just `sort`) when sort param is provided
- [ ] `tags_list` passes `counts` as a bare flag (not `counts=true`) when counts is true
- [ ] `tag_filter` passes `name=<value>` in CLI args
- [ ] Empty results return empty string (not error) for both tools
- [ ] TypeScript strict mode: no `any`
