# PRD: Obsidian CLI MCP Server

**Version:** 1.0
**Status:** Draft — Pending Review
**Author:** Wayne Renbjor
**Date:** 2026-03-03

---

## Problem Statement

Obsidian released official CLI commands. Existing MCP integrations expose a single passthrough tool and instruct the AI to consult help documentation to determine syntax. The AI guesses at every invocation. Errors are silent or unstructured. The pattern is fundamentally broken.

The gap is not technical difficulty. Nobody has built it correctly.

---

## Solution

A first-class MCP server that exposes every Obsidian CLI command as its own discrete, typed, documented tool. The AI never guesses at syntax. Every parameter is typed. Every error is structured and actionable.

**Delivered as:** Standalone TypeScript/Node.js npm package, configured in Claude Desktop (or any MCP host) in one config block.

---

## Target Users

**Primary:** AI power users controlling Obsidian via Claude Desktop, Cursor, or any MCP-compatible client.

**Secondary:** Developers building AI workflows that interact with a local Obsidian knowledge base.

---

## Success Metrics

- All 52 CLI commands exposed as discrete MCP tools with typed input schemas
- Zero tools use passthrough / generic command execution
- AI correctly invokes any tool on first attempt without documentation lookup
- All error states return structured, human-readable messages (not raw exceptions)
- Server installs and configures in under 5 minutes

---

## Setup Requirements

1. Obsidian v1.12+ with Installer v1.11.7+
2. CLI enabled: **Settings → General → Command Line Interface → toggle on → Install Path**
3. Obsidian must be running when tools are invoked
4. Node.js 18+ installed
5. Claude Desktop config updated with server entry (one JSON block, documented in README)

---

## Functional Requirements

### FR-0: Core Infrastructure

| ID | Requirement |
|---|---|
| FR-0.1 | Server starts via stdio transport; Claude Desktop manages the process |
| FR-0.2 | All tools execute via a single CLI executor module (`execFile`, not `exec`) |
| FR-0.3 | All errors return structured MCP content — never raw exceptions |
| FR-0.4 | `OBSIDIAN_NOT_RUNNING` error returned when Obsidian is not open |
| FR-0.5 | `CLI_NOT_FOUND` error returned with setup instructions when CLI binary absent |
| FR-0.6 | Configurable timeout per invocation (default 30s); `TIMEOUT` error on breach |
| FR-0.7 | All tools support optional `vault` parameter for multi-vault targeting |
| FR-0.8 | All tools support optional `copy` flag to copy output to clipboard |

### FR-1: Files & Folders (10 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `files_list` | `files` | `vault?` | Newline-separated file paths; with `total` returns count |
| `files_count` | `files total` | `vault?` | Integer count of files |
| `note_create` | `create` | `name`, `content?`, `template?`, `silent?`, `vault?` | Confirmation with file path |
| `note_read` | `read` | `file`, `vault?`, `silent?` | Note contents as string |
| `note_append` | `append` | `file`, `content`, `vault?` | Confirmation |
| `note_prepend` | `prepend` | `file`, `content`, `vault?` | Confirmation |
| `note_move` | `move` | `file`, `path`, `vault?` | Confirmation; internal links updated |
| `note_delete` | `delete` | `file`, `vault?` | Confirmation |
| `note_outline` | `outline` | `file`, `vault?`, `format?` | Heading tree |
| `notes_orphans` | `orphans` | `vault?` | List of notes with no incoming links |

**Error states:**
- `note_create` with duplicate name: `FILE_EXISTS` error
- `note_read/append/prepend/move/delete/outline` with missing file: `FILE_NOT_FOUND` error
- `note_move` with occupied destination: `DESTINATION_EXISTS` error

### FR-2: Search (2 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `search` | `search` | `query`, `limit?`, `format?`, `vault?` | Matching file paths/content |
| `search_open` | `search:open` | `query`, `vault?` | Opens search in Obsidian UI; confirmation |

**Error states:**
- Zero results: empty list (not an error)
- Invalid query syntax: `INVALID_QUERY` error with message

### FR-3: Properties (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `properties_list` | `properties` | `file`, `format?`, `vault?` | All frontmatter properties |
| `property_set` | `property:set` | `name`, `value`, `file`, `vault?` | Confirmation |
| `property_remove` | `property:remove` | `name`, `file`, `vault?` | Confirmation |

**Error states:**
- File not found: `FILE_NOT_FOUND`
- `property_remove` on nonexistent property: `PROPERTY_NOT_FOUND` error

### FR-4: Links (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `links_backlinks` | `backlinks` | `file`, `vault?` | List of files linking to target |
| `links_outgoing` | `links` | `file`, `vault?` | List of files this note links to |
| `links_unresolved` | `unresolved` | `vault?` | All broken/unresolved links across vault |

**Error states:**
- File not found: `FILE_NOT_FOUND`
- Zero backlinks/links: empty list (not an error)

### FR-5: Tags (2 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `tags_list` | `tags` | `sort?`, `counts?`, `vault?` | All tags; optionally with usage counts |
| `tag_filter` | `tag` | `name`, `vault?` | Notes containing the specified tag |

**Error states:**
- Zero results: empty list (not an error)

### FR-6: Tasks (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `tasks_all` | `tasks` | `vault?` | All open tasks across vault |
| `tasks_daily` | `tasks daily` | `vault?` | Tasks in today's daily note |
| `tasks_todo` | `tasks todo` | `vault?` | Incomplete tasks only |

**Error states:**
- No daily note configured or found: `NO_DAILY_NOTE` error
- Zero results: empty list (not an error)

### FR-7: Daily Notes (4 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `daily_open` | `daily` | `vault?` | Opens today's daily note; confirmation |
| `daily_read` | `daily:read` | `vault?` | Today's daily note contents |
| `daily_append` | `daily:append` | `content`, `vault?` | Confirmation |
| `daily_prepend` | `daily:prepend` | `content`, `vault?` | Confirmation |

**Error states:**
- Daily note plugin not configured: `DAILY_NOTE_NOT_CONFIGURED` error with instructions

### FR-8: Bookmarks & Templates (2 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `bookmarks_list` | `bookmarks` | `vault?` | All bookmarked notes |
| `templates_list` | `templates` | `vault?` | All available templates |

**Error states:**
- Zero results: empty list (not an error)
- Templates folder not configured: `TEMPLATES_NOT_CONFIGURED` error

### FR-9: Bases (2 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `bases_list` | `bases` | `vault?` | All Bases in vault |
| `base_query` | `base:query` | `file`, `format?`, `vault?` | Base query results in requested format |

**Error states:**
- File not found: `FILE_NOT_FOUND`
- Not a valid Base file: `NOT_A_BASE` error
- Bases plugin not enabled: `BASES_NOT_ENABLED` error

### FR-10: Plugins (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `plugins_list` | `plugins` | `vault?` | All installed plugins with status |
| `plugin_enable` | `plugin:enable` | `id`, `vault?` | Confirmation |
| `plugin_reload` | `plugin:reload` | `id`, `vault?` | Confirmation |

**Error states:**
- Plugin ID not found: `PLUGIN_NOT_FOUND` error
- Plugin already in target state: informational message (not an error)

### FR-11: Themes & Appearance (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `themes_list` | `themes` | `vault?` | All installed themes |
| `theme_set` | `theme:set` | `name`, `vault?` | Confirmation |
| `snippets_list` | `snippets` | `vault?` | All CSS snippets with enabled status |

**Error states:**
- Theme name not found: `THEME_NOT_FOUND` error

### FR-12: Sync (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `sync_status` | `sync:status` | `vault?` | Sync health and status |
| `sync_history` | `sync:history` | `path`, `vault?` | Version history for file |
| `sync_restore` | `sync:restore` | `path`, `version`, `vault?` | Confirmation of restore |

**Error states:**
- Obsidian Sync not enabled/configured: `SYNC_NOT_ENABLED` error
- Version not found: `VERSION_NOT_FOUND` error
- File not found: `FILE_NOT_FOUND`

### FR-13: Publish (3 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `publish_list` | `publish:list` | `vault?` | All published notes |
| `publish_add` | `publish:add` | `file`, `vault?` | Confirmation |
| `publish_remove` | `publish:remove` | `file`, `vault?` | Confirmation |

**Error states:**
- Obsidian Publish not configured: `PUBLISH_NOT_CONFIGURED` error
- File not found: `FILE_NOT_FOUND`

### FR-14: Developer Commands (9 tools)

| Tool | Command | Parameters | Returns |
|---|---|---|---|
| `dev_devtools` | `devtools` | `vault?` | Toggles Electron DevTools; confirmation |
| `dev_debug` | `dev:debug` | `enabled` (bool), `vault?` | Confirmation |
| `dev_screenshot` | `dev:screenshot` | `path`, `vault?` | Base64 PNG |
| `dev_console` | `dev:console` | `limit?`, `level?`, `vault?` | Captured console messages |
| `dev_errors` | `dev:errors` | `vault?` | Captured JavaScript errors |
| `dev_css` | `dev:css` | `selector`, `vault?` | CSS rules with source locations |
| `dev_dom` | `dev:dom` | `selector`, `total?`, `vault?` | Matching DOM elements |
| `dev_mobile` | `dev:mobile` | `enabled` (bool), `vault?` | Confirmation |
| `dev_eval` | `eval` | `code`, `vault?` | JavaScript execution result |

**Error states:**
- `dev_eval` with invalid JavaScript: `EVAL_ERROR` with JS error message
- `dev_screenshot` path unwritable: `PATH_NOT_WRITABLE` error

**Security note:** `dev_eval` executes arbitrary JavaScript in the Obsidian app context. The tool description must clearly communicate this to the AI. No additional restrictions — the user configured this tool intentionally.

---

## Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR-1 | Tool invocations complete within 30s under normal conditions |
| NFR-2 | Server startup time under 500ms |
| NFR-3 | Zero shell injection vectors — all args passed via `execFile` array |
| NFR-4 | TypeScript strict mode enabled |
| NFR-5 | All tools documented with descriptions an AI can parse unambiguously |
| NFR-6 | Works on macOS, Windows, Linux (wherever Obsidian CLI is supported) |
| NFR-7 | No telemetry, no network calls beyond CLI subprocess |
| NFR-8 | npm package published; usable via `npx obsidian-mcp` without global install |

---

## Out of Scope (v1)

- GUI settings panel beyond minimal config (timeout, CLI path override)
- Obsidian API surface beyond CLI command set
- Cloud / remote vault support
- HTTP/SSE transport
- Authentication or access control
- Plugin submission to community directory (evaluated for v2)
