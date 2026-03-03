# Epic 4: Links

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/links.ts`

---

## Overview

Three tools for querying the link graph between notes. Obsidian tracks which notes link to which — these tools expose that graph to the AI. Link queries are read-only and do not modify vault content.

Two tools target a specific file (`links_backlinks`, `links_outgoing`). One is vault-wide (`links_unresolved`). Zero results from any link query is never an error — it simply means no links exist. `FILE_NOT_FOUND` is returned when a target file does not exist.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 4.1: `links_backlinks`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all notes that link to a specific note,
So that I can understand what context a note exists in and what other notes reference it.

**Acceptance Criteria:**
- [ ] Tool `links_backlinks` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a list of vault-relative file paths for all notes containing links to the target
- [ ] Output is newline-separated file paths
- [ ] When no notes link to the target, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Both wiki-links (`[[target]]`) and markdown links (`[text](target.md)`) are included in backlink detection (Obsidian handles this)

**Tool Definition:**
- **Name:** `links_backlinks`
- **Description:** `"List all notes that link to a specified note (backlinks). Returns a newline-separated list of vault-relative paths for notes containing wiki-links or markdown links to the target. Returns empty if no notes link to it. Use this to understand which notes reference a given note before moving or deleting it."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path to find backlinks for.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Newline-separated list of vault-relative paths of notes that link to the target. Empty string if none.

**CLI Mapping:**
```
obsidian backlinks file=<path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "backlinks",
  `file=${params.file}`,
  ...buildGlobalArgs(params),
],
```

**Notes:**
- The distinction between `links_backlinks` (what links TO this note) and `links_outgoing` (what this note links TO) must be unambiguous in the tool descriptions. The AI must never confuse these two directions.
- This is a safe prerequisite check before `note_move` or `note_delete` — the AI can warn the user that N notes will need link updates before proceeding.

---

### Story 4.2: `links_outgoing`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all notes that a specific note links to,
So that I can understand what a note references and navigate to related content.

**Acceptance Criteria:**
- [ ] Tool `links_outgoing` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a list of vault-relative file paths for all notes the target links to
- [ ] Output is newline-separated file paths
- [ ] External links (URLs, `http://...`) are excluded — only internal note links are returned
- [ ] When the target note links to no other notes, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Both resolved and unresolved links are returned (unresolved links point to notes that don't exist yet)

**Tool Definition:**
- **Name:** `links_outgoing`
- **Description:** `"List all notes that a specific note links to (outgoing links). Returns a newline-separated list of vault-relative paths for all internal links found in the note. External URLs are excluded. Returns empty if the note contains no internal links. Use this to map a note's dependencies or navigate its reference graph."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path to inspect for outgoing links.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Newline-separated list of vault-relative paths that the note links to. Empty string if none.

**CLI Mapping:**
```
obsidian links file=<path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "links",
  `file=${params.file}`,
  ...buildGlobalArgs(params),
],
```

**Notes:**
- Note the CLI command is `links` (not `outgoing` or `links:outgoing`). The MCP tool name is `links_outgoing` for disambiguation with `links_backlinks`, but the CLI mapping is simply `links`.
- If the CLI returns unresolved links differently from resolved links (e.g. prefixed or annotated), document this in the output description. The tool returns raw CLI output — no transformation needed.

---

### Story 4.3: `links_unresolved`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all broken or unresolved links across the entire vault,
So that I can identify wiki-links that point to notes that don't exist yet.

**Acceptance Criteria:**
- [ ] Tool `links_unresolved` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a list of unresolved link references across the vault
- [ ] Output includes both the source file and the unresolved link target for each broken link
- [ ] When no unresolved links exist, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] This is a vault-wide scan — no `file` parameter

**Tool Definition:**
- **Name:** `links_unresolved`
- **Description:** `"List all unresolved (broken) internal links across the vault. These are wiki-links that reference notes that do not exist. Returns each broken link and the file it appears in. Returns empty if all links resolve. Use this to find stubs to create or typos in link targets."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** List of unresolved link references including the source file and the broken target. Format depends on CLI default. Empty string if all links are resolved.

**CLI Mapping:**
```
obsidian unresolved [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "unresolved",
  ...buildGlobalArgs(params),
],
```

**Notes:**
- This is a vault-wide operation with no required parameters. On a large vault with many stubs or drafts, this may return substantial output.
- The output format (how the CLI presents "source file → broken target") should be documented in a comment in the tool definition once observed during implementation. The AI will need to parse this format if it processes the results programmatically.
- This tool pairs naturally with `note_create` — the AI can identify unresolved links and offer to create the missing notes.

---

## Implementation Checklist

- [ ] `src/tools/links.ts` created and exports `linkTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `linkTools`
- [ ] All 3 tool names appear in the MCP tool list: `links_backlinks`, `links_outgoing`, `links_unresolved`
- [ ] `file` parameter builds as `file=<value>` in CLI args for file-scoped tools
- [ ] `links_unresolved` has no required parameters — `buildArgs` only uses `buildGlobalArgs`
- [ ] Empty results return empty string (not error) for all 3 tools
- [ ] `FILE_NOT_FOUND` returned when target file does not exist (for `links_backlinks` and `links_outgoing`)
- [ ] Tool descriptions clearly distinguish "backlinks" (what links TO this) from "outgoing" (what this links TO)
- [ ] TypeScript strict mode: no `any`
