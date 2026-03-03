# Epic 8: Bookmarks & Templates

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/bookmarks.ts`
**Tool count:** 2

---

## Overview

Bookmarks and Templates are read-only discovery tools. They expose two of Obsidian's organizational features to the AI: the bookmarks sidebar (saved note references) and the Templates plugin folder (reusable note starters). Both tools are informational — no writes occur. Both tools return empty lists when no items exist, except `templates_list` which errors if the Templates plugin is not configured at all.

These tools let the AI know what the user has bookmarked (useful for understanding high-priority notes) and what templates are available before calling `note_create` with a `template` parameter.

---

## Stories

---

### Story 8.1: bookmarks_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve all bookmarked notes from the user's vault,
So that I can identify which notes the user considers important or frequently-accessed without needing to ask.

**Acceptance Criteria:**
- [ ] Tool `bookmarks_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of bookmarked note paths, one per line, or a structured empty response if no bookmarks exist
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When `vault` is provided, command is invoked with `vault=<name>` flag and only that vault's bookmarks are returned
- [ ] When no bookmarks exist, returns an empty list (not an error) with a message such as `"No bookmarks found in this vault."`
- [ ] `buildArgs` produces `["bookmarks"]` with no vault, or `["bookmarks", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout; no transformation required
- [ ] Tool definition is exported as part of the `bookmarks.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `bookmarks_list`
- **Description:** "Lists all bookmarked notes in the Obsidian vault. Returns file paths of every note the user has bookmarked in Obsidian's Bookmarks sidebar. Returns an empty list if no bookmarks exist. Useful before referencing user priorities or frequently-accessed notes. Example output: 'Projects/Active/website-redesign.md\nJournal/2026-03-01.md'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Newline-separated list of bookmarked note file paths relative to the vault root. Empty list message if no bookmarks exist.

**CLI Mapping:**
```
obsidian bookmarks [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No bookmarks → empty list (not an error): `"No bookmarks found in this vault."`

**Notes:** Bookmarks in Obsidian can include folders and searches, not just notes. If the CLI returns non-note entries, pass them through without filtering — the AI can decide what to do with them. Do not attempt to filter or validate the paths returned.

---

### Story 8.2: templates_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all available note templates in the vault,
So that I can offer the user valid template names when creating a new note, without guessing or hallucinating template names.

**Acceptance Criteria:**
- [ ] Tool `templates_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of template file paths available in the configured templates folder
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the Templates plugin (core or Templater) is not enabled or the templates folder is not configured, returns `TEMPLATES_NOT_CONFIGURED` structured error with actionable setup instructions
- [ ] When the templates folder exists but is empty, returns an empty list (not an error)
- [ ] When `vault` is provided, targets that vault's template configuration
- [ ] `buildArgs` produces `["templates"]` with no vault, or `["templates", "vault=<name>"]` with vault
- [ ] The `TEMPLATES_NOT_CONFIGURED` error includes instructions: point to Settings → Templates → Template folder location (core plugin) or Templater plugin settings
- [ ] Tool definition is exported as part of the `bookmarks.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `templates_list`
- **Description:** "Lists all available note templates in the Obsidian vault's configured templates folder. Returns file paths of every template the user can apply when creating a new note. Use this before calling note_create with a template parameter to confirm the template name is valid. Returns TEMPLATES_NOT_CONFIGURED error if the Templates plugin is not enabled or no template folder is set. Example output: 'Templates/Daily Note.md\nTemplates/Meeting.md\nTemplates/Project Kickoff.md'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Newline-separated list of template file paths relative to the vault root. Empty list message if templates folder is empty. `TEMPLATES_NOT_CONFIGURED` error if plugin not set up.

**CLI Mapping:**
```
obsidian templates [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Templates not configured → `{"error": "Templates plugin is not configured. Enable the Templates core plugin in Obsidian Settings → Core plugins → Templates, then set a template folder under Settings → Templates → Template folder location. Alternatively, install and configure the Templater community plugin.", "code": "TEMPLATES_NOT_CONFIGURED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- Templates folder empty → empty list (not an error): `"No templates found in the configured templates folder."`

**Notes:** The Templates plugin (core) and Templater (community) both use a configured folder. The CLI surfaces a single `TEMPLATES_NOT_CONFIGURED` state if neither is active. This tool does not distinguish between core Templates and Templater — it reports whatever the CLI returns. The template names returned here are the exact values that should be passed to `note_create`'s `template` parameter.
