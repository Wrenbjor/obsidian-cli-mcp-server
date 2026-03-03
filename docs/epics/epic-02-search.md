# Epic 2: Search

**Status:** Ready for Implementation
**Depends On:** Epic 0 (Core Infrastructure)
**Blocks:** Nothing
**Implementer:** Amelia (dev agent)
**Source File:** `src/tools/search.ts`

---

## Overview

Two tools covering vault-wide full-text search. The first (`search`) returns results programmatically for the AI to process. The second (`search_open`) opens Obsidian's built-in search UI with a pre-filled query — useful when the user wants to browse results themselves.

Zero results is never an error — both tools return empty output when nothing matches. The only error conditions are infrastructure failures and invalid query syntax.

All tools support the global `vault` and `copy` flags from Epic 0.

---

## Stories

---

### Story 2.1: `search`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to search the vault for notes matching a query,
So that I can find relevant notes by content and return them to the user or use them in subsequent operations.

**Acceptance Criteria:**
- [ ] Tool `search` is registered and visible in MCP tool list
- [ ] Input schema: `query` (string, required), `limit` (integer, optional), `format` (enum, optional), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution returns matching file paths and/or content excerpts
- [ ] When zero notes match the query, returns an empty string (not an error)
- [ ] When `limit` is specified, at most that many results are returned
- [ ] When `format=json`, returns a JSON array of result objects; validates JSON before returning, returns `PARSE_ERROR` if malformed
- [ ] When `format=text`, returns a plain-text formatted list with file paths and excerpts
- [ ] When `format=paths`, returns only a newline-separated list of matching file paths
- [ ] When `format` is omitted, CLI default format is returned
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] When the query syntax is invalid, returns `INVALID_QUERY` structured error with the CLI's error message
- [ ] `limit` must be a positive integer; omit validation beyond type-checking (let the CLI handle range errors)

**Tool Definition:**
- **Name:** `search`
- **Description:** `"Search the Obsidian vault for notes matching a query. Returns matching file paths and content excerpts. Zero results returns empty output (not an error). Use format='json' for structured results, format='paths' for just the file paths, or format='text' for a human-readable list. Use limit to cap the number of results. Supports Obsidian query syntax: 'tag:#project', 'file:Sprint', 'path:Projects/', full-text keywords, and boolean operators."`
- **Parameters:**
  - `query`: `string` (required) — Search query. Supports Obsidian query syntax including `tag:`, `file:`, `path:`, and full-text keywords.
  - `limit`: `integer` (optional) — Maximum number of results to return.
  - `format`: `string` (optional, enum: `"json"`, `"text"`, `"paths"`) — Output format.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Matching notes with file paths and excerpts. Format depends on `format` parameter. Empty string if no results.

**CLI Mapping:**
```
obsidian search query=<text> [limit=<n>] [format=<json|text|paths>] [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Invalid query syntax → `{"error": "Invalid search query: {message}", "code": "INVALID_QUERY", "detail": "<cli stderr>"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`
- JSON format, malformed output → `{"error": "CLI returned malformed output...", "code": "PARSE_ERROR", "detail": "<raw output>"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => {
  const args = [`query=${params.query}`];
  if (typeof params.limit === "number") {
    args.push(`limit=${params.limit}`);
  }
  return ["search", ...args, ...buildGlobalArgs(params)];
},
```

**`INVALID_QUERY` Detection:**
Detect by checking stderr for query-related error patterns (e.g. "invalid query", "parse error", "syntax error") with a non-zero exit code. Add this detection to the tool's post-processing step or to the error mapper. This code is not auto-detectable by the core executor since it requires domain knowledge about what constitutes an invalid query vs. a general command failure.

**Notes:**
- The Obsidian query syntax supports operators like `tag:#tagname`, `file:filename`, `path:folder/`, `"exact phrase"`, and boolean `-exclude` prefixes. Include examples of this in the description so the AI can construct meaningful queries without documentation.
- Large vaults may return substantial results — the `limit` parameter is the AI's primary tool for managing result set size.
- When `format=json`, the `transformOutput` function should validate JSON parseability and surface `PARSE_ERROR` if validation fails.

```typescript
transformOutput: (raw, params) => {
  if (params.format === "json") {
    try {
      JSON.parse(raw); // validate; if it throws, caller returns PARSE_ERROR
      return raw;
    } catch {
      // Registry layer handles PARSE_ERROR based on JSON parse failure
      throw new Error("PARSE_ERROR");
    }
  }
  return raw;
},
```

---

### Story 2.2: `search_open`

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to open Obsidian's search panel with a pre-filled query,
So that the user can browse search results in Obsidian's UI directly.

**Acceptance Criteria:**
- [ ] Tool `search_open` is registered and visible in MCP tool list
- [ ] Input schema: `query` (string, required), `vault` (string, optional), `copy` (boolean, optional)
- [ ] Successful execution opens Obsidian's search panel with the query pre-filled
- [ ] Returns a confirmation string (does not return the search results themselves)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is specified and not found, returns `VAULT_NOT_FOUND` structured error
- [ ] The tool does not wait for the user to interact with the search panel — it fires and confirms

**Tool Definition:**
- **Name:** `search_open`
- **Description:** `"Open Obsidian's built-in search panel with a pre-filled query, allowing the user to browse results in the Obsidian UI. This tool does not return search results — it opens the Obsidian interface. Use the 'search' tool instead when you need to process results programmatically."`
- **Parameters:**
  - `query`: `string` (required) — Search query to pre-fill in the Obsidian search panel.
  - `vault`: `string` (optional) — Target vault name.
  - `copy`: `boolean` (optional) — Copy output to clipboard.
- **Returns:** Confirmation string that the search panel was opened.

**CLI Mapping:**
```
obsidian search:open query=<text> [vault=<name>] [copy]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open...", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '{vault}' was not found...", "code": "VAULT_NOT_FOUND"}`

**`buildArgs` Implementation:**

```typescript
buildArgs: (params) => [
  "search:open",
  `query=${params.query}`,
  ...buildGlobalArgs(params),
],
```

**Notes:**
- This tool is intentionally UI-focused. It is useful when a user asks the AI to "open search for X in Obsidian" rather than asking the AI to process the results. Make the distinction between `search` (programmatic) and `search_open` (UI) clear in both tool descriptions so the AI selects the correct one.
- No `format` parameter applies to this tool — it does not return content.

---

## Implementation Checklist

- [ ] `src/tools/search.ts` created and exports `searchTools: ToolDefinition[]`
- [ ] `main.ts` imports and registers `searchTools`
- [ ] Both tool names appear in the MCP tool list: `search`, `search_open`
- [ ] `query` parameter builds as `query=<value>` in CLI args
- [ ] `limit` parameter builds as `limit=<value>` in CLI args (only when defined)
- [ ] `format=json` output is validated for JSON parseability; `PARSE_ERROR` returned on failure
- [ ] `INVALID_QUERY` error detection implemented (per detection notes above)
- [ ] Empty result set returns empty string (not error) for `search`
- [ ] `search_open` returns confirmation (not results)
- [ ] TypeScript strict mode: no `any`
