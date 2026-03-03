# Epic 1: Files & Folders

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/files.ts`

---

## Overview

Files & Folders is the most frequently used tool domain. It covers the complete lifecycle of notes: listing, creating, reading, modifying, moving, deleting, and inspecting. All 10 tools are exported as a single `fileTools: ToolDefinition[]` array and registered in `main.ts` via `registerTools(server, fileTools)`.

These tools map directly to the `files`, `create`, `read`, `append`, `prepend`, `move`, `delete`, `outline`, and `orphans` CLI commands.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 1.1: `files_list`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all files in the active vault,
So that I can enumerate available notes before reading, searching, or operating on them.

**Acceptance Criteria:**
- [ ] Tool `files_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a newline-separated list of file paths relative to vault root
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Empty vault returns an empty string (not an error)
- [ ] File paths include folder structure (e.g. `Projects/MyNote.md`, not just `MyNote.md`)

**Tool Definition:**
- **Name:** `files_list`
- **Description:** `"List all files in the Obsidian vault. Returns a newline-separated list of file paths relative to the vault root (e.g. 'Projects/MyNote.md'). Use this to enumerate available notes before reading or searching. Returns an empty result if the vault contains no files."`
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. Omit to use the default vault.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Newline-separated string of vault-relative file paths. Empty string if vault has no files.

**CLI Mapping:**
```
obsidian files [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** This is the simplest list command and a good reference implementation for other list tools. No `transformOutput` needed — raw CLI output is the correct format.

---

### Story 1.2: `files_count`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to count all files in the vault without receiving the full list,
So that I can report vault size or check whether a vault has content without processing a potentially large payload.

**Acceptance Criteria:**
- [ ] Tool `files_count` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a plain integer as a string (e.g. `"247"`)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Empty vault returns `"0"`

**Tool Definition:**
- **Name:** `files_count`
- **Description:** `"Count the total number of files in the Obsidian vault. Returns a single integer. Use this to check vault size without receiving the full file list. Example return value: '247'."`
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. Omit to use the default vault.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** A plain integer string representing the total file count.

**CLI Mapping:**
```
obsidian files total [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** The `total` flag on the `files` command makes the CLI return a count instead of the list. This is implemented via `buildArgs` returning `["files", "total"]` rather than using the generic `buildGlobalArgs` `total` flag helper — because `total` here is a subcommand argument, not a trailing flag.

```typescript
buildArgs: (params) => [
  "files",
  "total",
  ...buildGlobalArgs(params),
],
```

---

### Story 1.3: `note_create`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to create a new note in the vault,
So that I can initialize notes for the user with specific content or from a template.

**Acceptance Criteria:**
- [ ] Tool `note_create` is registered and visible in MCP tool list
- [ ] Input schema: `name` (string, required), `content` (string, optional), `template` (string, optional), `silent` (boolean, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string containing the created note's path
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When a note with the same name already exists, returns `FILE_EXISTS` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] When `template` is specified and not found, returns `FILE_NOT_FOUND` structured error with context indicating it was the template
- [ ] `name` may include folder path (e.g. `"Projects/Sprint Notes"`)
- [ ] `content` and `template` are mutually usable; if both provided, behavior follows CLI default (template takes precedence or content is appended — document whichever the CLI does)
- [ ] When `silent` is true, the note is created but not opened in Obsidian's UI

**Tool Definition:**
- **Name:** `note_create`
- **Description:** `"Create a new note in the Obsidian vault. Specify a name (required) and optionally initial content or a template to apply. The name may include a folder path (e.g. 'Projects/Sprint Notes'). Returns the path of the created note. Errors if a note with that name already exists."`
- **Parameters:**
  - `name`: `string` (required) — Note name or path. May include folders: `"Projects/Sprint Notes"`. Do not include `.md` extension.
  - `content`: `string` (optional) — Initial text content for the note body.
  - `template`: `string` (optional) — Name of an existing template note to apply.
  - `silent`: `boolean` (optional) — Create the note without opening it in Obsidian.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string with the created note's vault-relative path.

**CLI Mapping:**
```
obsidian create name=<value> [content=<value>] [template=<value>] [silent] [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File already exists → `{"error": "A note named '{name}' already exists.", "code": "FILE_EXISTS"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`
- Template not found → `{"error": "File '{template}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`

**Notes:**
The `FILE_EXISTS` detection should look for "already exists" in stderr with exit code ≠ 0. Add this pattern to the error mapper in `executor.ts` or handle it in the tool's post-processing step. The `name` parameter with spaces must be passed as a single argument: `name=My Note Name` (no quoting needed since `execFile` handles the array directly without shell interpolation).

```typescript
buildArgs: (params) => {
  const args = ["create", `name=${params.name}`];
  if (typeof params.content === "string") args.push(`content=${params.content}`);
  if (typeof params.template === "string") args.push(`template=${params.template}`);
  if (params.silent === true) args.push("silent");
  return [...args, ...buildGlobalArgs(params)];
},
```

---

### Story 1.4: `note_read`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to read the full contents of a note,
So that I can analyze, summarize, or work with the note's content.

**Acceptance Criteria:**
- [ ] Tool `note_read` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `silent` (boolean, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns the full text content of the note as a string
- [ ] Returned content includes frontmatter if present
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] When `silent` is false (default), the note is opened in Obsidian's UI; when `silent` is true, it is not
- [ ] Empty notes return empty string (not an error)
- [ ] `file` may be a partial path match or full vault-relative path

**Tool Definition:**
- **Name:** `note_read`
- **Description:** `"Read the full contents of a note from the Obsidian vault. Returns the complete note text including frontmatter. The 'file' parameter is the note name or vault-relative path (e.g. 'Projects/Sprint Notes.md'). Use silent=true to read without opening the note in Obsidian's UI."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path (e.g. `"Projects/Sprint Notes.md"`).
  - `silent`: `boolean` (optional) — Read without opening the note in Obsidian. Default: false.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Full note content as a plain string, including frontmatter if present.

**CLI Mapping:**
```
obsidian read file=<path> [silent] [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** This is the primary tool for retrieving note content. The AI should use `note_read` before any operation that requires knowing the note's current state (e.g. before appending or checking for existing content). `FILE_NOT_FOUND` detection relies on the stderr pattern matching in `executor.ts` — ensure the pattern is broad enough to catch the CLI's exact wording.

---

### Story 1.5: `note_append`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to append content to the end of an existing note,
So that I can add new information without overwriting existing content.

**Acceptance Criteria:**
- [ ] Tool `note_append` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `content` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] Content is added to the very end of the file, after all existing content
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Multi-line content passed in `content` is preserved as-is

**Tool Definition:**
- **Name:** `note_append`
- **Description:** `"Append text to the end of an existing note. The content is added after all existing text. Use this to add new sections, log entries, or additional information without disturbing existing note content. Errors if the note does not exist — use note_create first if needed."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path (e.g. `"Daily/2026-03-03.md"`).
  - `content`: `string` (required) — Text to append. May be multi-line.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating content was appended.

**CLI Mapping:**
```
obsidian append file=<path> content=<text> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** When content contains newlines, the string is passed as a single argument element in the `execFile` args array. No shell escaping is needed. The CLI should handle the raw string value.

---

### Story 1.6: `note_prepend`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to prepend content to the beginning of a note's body (after frontmatter),
So that I can insert new content at the top without overwriting or shifting frontmatter.

**Acceptance Criteria:**
- [ ] Tool `note_prepend` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `content` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] Content is inserted after any frontmatter block (`--- ... ---`), not before it
- [ ] When the note has no frontmatter, content is inserted at the very beginning
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error

**Tool Definition:**
- **Name:** `note_prepend`
- **Description:** `"Prepend text to the beginning of an existing note's body, after any YAML frontmatter. Use this to add content to the top of a note without disturbing frontmatter or existing body content. If the note has no frontmatter, content is inserted at position 0. Errors if the note does not exist."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path.
  - `content`: `string` (required) — Text to prepend. May be multi-line.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating content was prepended.

**CLI Mapping:**
```
obsidian prepend file=<path> content=<text> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** The frontmatter-awareness behavior is handled by the Obsidian CLI, not by this server. Document in the tool description that this is the expected behavior so the AI knows what to expect.

---

### Story 1.7: `note_move`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to move or rename a note to a new path,
So that I can reorganize the vault structure while preserving internal link integrity.

**Acceptance Criteria:**
- [ ] Tool `note_move` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `path` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string with the new path
- [ ] All internal wiki-links pointing to the moved note are updated automatically (Obsidian handles this)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the source file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When a file already exists at the destination path, returns `DESTINATION_EXISTS` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] `path` is the full destination vault-relative path including filename (e.g. `"Archive/OldSprint.md"`)

**Tool Definition:**
- **Name:** `note_move`
- **Description:** `"Move or rename a note to a new vault-relative path. Obsidian automatically updates all internal wiki-links that reference the moved note. 'file' is the current path; 'path' is the destination (e.g. 'Archive/SprintRetro.md'). Errors if the source does not exist or the destination is already occupied."`
- **Parameters:**
  - `file`: `string` (required) — Current vault-relative path of the note to move.
  - `path`: `string` (required) — Destination vault-relative path including filename (e.g. `"Archive/SprintRetro.md"`).
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string with the new vault-relative path.

**CLI Mapping:**
```
obsidian move file=<current-path> path=<destination-path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Source file not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Destination occupied → `{"error": "A file already exists at '{path}'.", "code": "DESTINATION_EXISTS"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** `DESTINATION_EXISTS` detection should look for "already exists" in stderr (same pattern as `FILE_EXISTS`) but the context is different — the error is about the destination, not the source. The tool handler can inspect which error pattern fired and return the appropriate code. If the CLI does not distinguish between source-not-found and destination-occupied in stderr, document the limitation and default to `COMMAND_FAILED` for the ambiguous case.

---

### Story 1.8: `note_delete`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to delete a note from the vault,
So that I can remove obsolete or unwanted notes as part of vault management tasks.

**Acceptance Criteria:**
- [ ] Tool `note_delete` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a confirmation string
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] Deletion is permanent (or moves to system trash, per Obsidian config) — the tool description must communicate this

**Tool Definition:**
- **Name:** `note_delete`
- **Description:** `"Delete a note from the Obsidian vault. This operation is permanent (or moves to system trash depending on Obsidian's deletion settings). Use with caution — there is no undo via MCP. Errors if the note does not exist."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path to delete.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string indicating the note was deleted.

**CLI Mapping:**
```
obsidian delete file=<path> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** The tool description explicitly mentions permanence. This is important for the AI to communicate to users before executing a delete. The AI should not call this tool without confirming intent when acting on behalf of a user.

---

### Story 1.9: `note_outline`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve the heading structure of a note,
So that I can understand how a note is organized without reading its full content.

**Acceptance Criteria:**
- [ ] Tool `note_outline` is registered and visible in MCP tool list
- [ ] Input schema: `file` (string, required), `format` (enum `["tree", "md"]`, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns the heading hierarchy of the note
- [ ] When `format=tree`, output is a text tree representation with indentation showing heading levels
- [ ] When `format=md`, output is markdown with `#` prefix heading levels
- [ ] When `format` is omitted, CLI default format is returned (document which that is)
- [ ] Notes with no headings return an empty result (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error

**Tool Definition:**
- **Name:** `note_outline`
- **Description:** `"Get the heading structure of a note as a navigable outline. Returns all headings (H1–H6) with their hierarchy. Use format='tree' for an indented tree view or format='md' for markdown headings. Useful for understanding note structure before reading full content or inserting content at a specific section."`
- **Parameters:**
  - `file`: `string` (required) — Note name or vault-relative path.
  - `format`: `string` (optional, enum: `"tree"`, `"md"`) — Output format. Defaults to CLI default.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Heading hierarchy as tree or markdown, depending on format.

**CLI Mapping:**
```
obsidian outline file=<path> [format=<tree|md>] [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '{file}' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** The `format` parameter here is tool-specific (only `tree` and `md` are valid), not the generic global format flag. Use the `formatSchemaProperty(["tree", "md"])` helper from the registry for the schema, but build it into the args as `format=<value>` via `buildGlobalArgs`.

---

### Story 1.10: `notes_orphans`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list notes that have no incoming links,
So that I can identify isolated notes for cleanup, linking, or archiving.

**Acceptance Criteria:**
- [ ] Tool `notes_orphans` is registered and visible in MCP tool list
- [ ] Input schema: `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns a newline-separated list of vault-relative paths for orphaned notes
- [ ] Notes with zero incoming backlinks are considered orphans
- [ ] When no orphans exist, returns an empty string (not an error)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error

**Tool Definition:**
- **Name:** `notes_orphans`
- **Description:** `"List all notes in the vault that have no incoming links (orphans). These are notes that no other note references. Returns a newline-separated list of vault-relative paths. Returns empty if all notes are linked. Use this to find isolated notes that may need to be connected, reviewed, or archived."`
- **Parameters:**
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Newline-separated list of vault-relative paths for orphaned notes. Empty string if none.

**CLI Mapping:**
```
obsidian orphans [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**Notes:** This is a vault-wide operation with no required parameters. It may be slow on large vaults. No timeout special-casing needed — the global 30s timeout applies.

---

## Implementation Checklist

- [ ] `src/tools/files.ts` created and exports `fileTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `fileTools`
- [ ] All 10 tool names appear in the MCP tool list
- [ ] All `file` parameters build as `file=<value>` in CLI args
- [ ] All `name` parameters build as `name=<value>` in CLI args
- [ ] All `path` parameters build as `path=<value>` in CLI args
- [ ] All `content` parameters build as `content=<value>` in CLI args
- [ ] Global flags (`vault`, `copy`, `silent`) handled via `buildGlobalArgs` helper
- [ ] `FILE_EXISTS` and `DESTINATION_EXISTS` error detection added to executor or handled per-tool
- [ ] TypeScript strict mode: no `any`, all params typed as `Record<string, unknown>` and narrowed before use
