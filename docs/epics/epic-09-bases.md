# Epic 9: Bases (Database Views)

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/bases.ts`
**Tool count:** 2

---

## Overview

Obsidian Bases is Obsidian's native database view feature (introduced as a first-party plugin). It allows users to define structured data views over their vault, similar to Notion databases or Dataview tables, but built into Obsidian. A Base file (`.base` extension or a note with a bases code block) defines a query against vault properties.

These two tools give the AI the ability to discover what Bases exist in the vault and to execute queries against them, returning structured results. The query tool supports multiple output formats, defaulting to JSON for programmatic use.

`bases_list` is a discovery tool. `base_query` is an execution tool. Both require the Bases plugin to be enabled.

---

## Stories

---

### Story 9.1: bases_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all Base files in the vault,
So that I can know what database views are available before querying them, without guessing filenames.

**Acceptance Criteria:**
- [ ] Tool `bases_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of Base file paths in the vault, one per line
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the Bases plugin is not enabled, returns `BASES_NOT_ENABLED` structured error with enablement instructions
- [ ] When no Base files exist in the vault, returns an empty list (not an error)
- [ ] When `vault` is provided, scopes the listing to that vault
- [ ] `buildArgs` produces `["bases"]` with no vault, or `["bases", "vault=<name>"]` with vault
- [ ] Tool definition is exported as part of the `bases.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `bases_list`
- **Description:** "Lists all Base files (Obsidian database views) in the vault. Bases are Obsidian's native structured data views, similar to Notion databases. Returns file paths of every .base file or note containing a bases query. Use this before calling base_query to confirm the file path. Returns BASES_NOT_ENABLED error if the Bases core plugin is not enabled. Returns an empty list if no Bases exist. Example output: 'Projects/Project Tracker.base\nFinance/Budget View.base'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Newline-separated list of Base file paths relative to the vault root. Empty list message if none exist.

**CLI Mapping:**
```
obsidian bases [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Bases plugin not enabled → `{"error": "The Bases plugin is not enabled. Enable it in Obsidian Settings → Core plugins → Bases.", "code": "BASES_NOT_ENABLED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No bases in vault → empty list (not an error): `"No Base files found in this vault."`

**Notes:** Bases is a first-party Obsidian plugin that may not be enabled in all vaults. Always handle `BASES_NOT_ENABLED` gracefully with setup instructions. The file paths returned here are the exact values to pass as the `file` parameter to `base_query`.

---

### Story 9.2: base_query

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to execute a query against a Base view and receive the results in a structured format,
So that I can extract and work with the user's structured vault data without requiring them to manually export it.

**Acceptance Criteria:**
- [ ] Tool `base_query` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `file`: string (required) — path to the Base file
  - `format`: enum (optional) — one of `json`, `csv`, `tsv`, `md`, `paths`; defaults to `json`
  - `vault`: string (optional)
- [ ] Successful execution returns query results in the requested format as a string
- [ ] Default format is `json` when `format` is not provided
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When the specified file exists but is not a Base file, returns `NOT_A_BASE` structured error
- [ ] When the Bases plugin is not enabled, returns `BASES_NOT_ENABLED` structured error
- [ ] When `format=json` is used (default), executor validates JSON parseability; malformed output returns `PARSE_ERROR` with raw output in `detail`
- [ ] `buildArgs` produces `["base:query", "file=<path>"]` plus `"format=<format>"` if provided, plus `"vault=<name>"` if provided
- [ ] When format is omitted, `buildArgs` does NOT append a format arg (CLI uses its own default, which must be json)
- [ ] Tool definition is exported as part of the `bases.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `base_query`
- **Description:** "Executes a query against an Obsidian Base (database view) file and returns the results. Bases are structured data views over vault notes. Provide the path to a .base file (use bases_list to find valid paths). Results are returned in the requested format: json (default, structured array of objects), csv (comma-separated), tsv (tab-separated), md (markdown table), or paths (file paths only). Use json format when you need to process the data programmatically. Use md when presenting results to the user. Returns FILE_NOT_FOUND if the file does not exist, NOT_A_BASE if the file is not a Base file. Example: file='Projects/Project Tracker.base' format='json'"
- **Parameters:**
  - `file`: `string` (required) — Path to the Base file relative to the vault root. Use `bases_list` to discover valid paths.
  - `format`: `string` (optional) — Output format. One of: `json` (default), `csv`, `tsv`, `md`, `paths`. Use `json` for programmatic processing, `md` for display.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Query results as a string in the specified format. JSON format returns a JSON array of objects, one per row. CSV/TSV include headers. MD returns a markdown table. Paths returns one file path per line.

**CLI Mapping:**
```
obsidian base:query file=<path> [format=json|csv|tsv|md|paths] [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '<file>' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Not a Base file → `{"error": "File '<file>' is not a Base file. Only .base files or notes with a bases query block can be queried.", "code": "NOT_A_BASE"}`
- Bases plugin not enabled → `{"error": "The Bases plugin is not enabled. Enable it in Obsidian Settings → Core plugins → Bases.", "code": "BASES_NOT_ENABLED"}`
- JSON parse failure → `{"error": "CLI returned malformed JSON.", "code": "PARSE_ERROR", "detail": "<raw stdout>"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** The `format` parameter controls what the CLI serializes — it is passed directly to the CLI, not applied by this server. When `format=json` (or format is omitted and the default is json), the executor must validate that stdout is valid JSON before returning. If validation fails, return `PARSE_ERROR` with the raw output in `detail` so the caller can inspect what the CLI actually returned. Do not attempt to repair or re-parse malformed output.
