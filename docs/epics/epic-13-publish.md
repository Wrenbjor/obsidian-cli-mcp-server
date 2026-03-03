# Epic 13: Publish

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/publish.ts`
**Tool count:** 3

---

## Overview

Obsidian Publish is Obsidian's official paid publishing service for sharing notes as a public website. These tools give the AI the ability to inspect the current publish state of the vault, add notes to the publish queue, and remove notes from it.

All three tools require an active Obsidian Publish subscription and for Publish to be configured for the vault. When Publish is not set up, all tools return `PUBLISH_NOT_CONFIGURED` with setup instructions.

The publish queue in Obsidian controls which notes are included in the published site. Adding a note to the queue marks it for inclusion; removing it marks it for exclusion. Actual publication (syncing changes to the live site) may be triggered separately through Obsidian's Publish panel — the CLI tools manage the queue, not the final publish action, unless the CLI provides a publish-commit command (which is not in scope for this epic).

---

## Stories

---

### Story 13.1: publish_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all notes currently included in the Obsidian Publish site,
So that I can report what is published, identify notes that should be added or removed, and verify publish state without opening Obsidian's Publish panel.

**Acceptance Criteria:**
- [ ] Tool `publish_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of all note paths currently marked for publishing
- [ ] Each entry should include at minimum the file path and its publish status (published, unpublished-pending, or equivalent states returned by the CLI)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When Obsidian Publish is not configured for the vault, returns `PUBLISH_NOT_CONFIGURED` structured error with setup instructions
- [ ] When no notes are published (empty publish set), returns an empty list (not an error)
- [ ] When `vault` is provided, targets that vault's Publish configuration
- [ ] `buildArgs` produces `["publish:list"]` with no vault, or `["publish:list", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `publish.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `publish_list`
- **Description:** "Lists all notes currently included in the Obsidian Publish site for this vault. Returns file paths of notes marked for publishing, along with their publish status (e.g., published, pending changes). Use this to audit what is publicly visible, or to identify notes before calling publish_add or publish_remove. Requires Obsidian Publish. Returns PUBLISH_NOT_CONFIGURED if Publish is not set up for this vault."
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of note file paths included in the Publish site, with status for each. Empty list if no notes are published.

**CLI Mapping:**
```
obsidian publish:list [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Publish not configured → `{"error": "Obsidian Publish is not configured for this vault. Set up Publish in Obsidian Settings → Publish, then log in with your Obsidian account.", "code": "PUBLISH_NOT_CONFIGURED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No published notes → empty list (not an error): `"No notes are currently in the publish set for this vault."`

**Notes:** Obsidian Publish tracks which notes should be included on the site. Some notes may be in a "pending changes" state if they have been modified locally since last published. The CLI output may include status distinctions (published vs. changed vs. pending) — pass them through without filtering.

---

### Story 13.2: publish_add

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to add a note to the Obsidian Publish queue,
So that the user's note will be included in their published site when they next sync their publish changes.

**Acceptance Criteria:**
- [ ] Tool `publish_add` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `file`: string (required) — vault-relative path to the note
  - `vault`: string (optional)
- [ ] Successful execution marks the file for publishing and returns a confirmation message
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist in the vault, returns `FILE_NOT_FOUND` structured error
- [ ] When Obsidian Publish is not configured, returns `PUBLISH_NOT_CONFIGURED` structured error
- [ ] When the file is already in the publish set, returns an informational success message (not an error): `"File '<file>' is already included in the publish set."`
- [ ] `buildArgs` produces `["publish:add", "file=<path>"]` plus `"vault=<name>"` if provided
- [ ] Tool definition is exported as part of the `publish.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `publish_add`
- **Description:** "Adds a note to the Obsidian Publish site queue, marking it for inclusion when the user next publishes their site. The note must exist in the vault. If the note is already in the publish set, returns an informational message rather than an error. Use publish_list to check current publish state. Requires Obsidian Publish. Returns FILE_NOT_FOUND if the file doesn't exist. Example: file='Blog/my-new-post.md'"
- **Parameters:**
  - `file`: `string` (required) — Vault-relative path to the note to add to the publish set (e.g., `Blog/my-new-post.md`).
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the file was added to the publish queue, or informational message if it was already included.

**CLI Mapping:**
```
obsidian publish:add file=<file-path> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '<file>' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Publish not configured → `{"error": "Obsidian Publish is not configured for this vault. Set up Publish in Obsidian Settings → Publish, then log in with your Obsidian account.", "code": "PUBLISH_NOT_CONFIGURED"}`
- File already published → informational success (not an error): `"File '<file>' is already included in the publish set."`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Adding a file to the publish queue does not immediately push it to the live Obsidian Publish site — it marks it for the next publish sync. The user still needs to trigger a publish sync either through Obsidian's UI or via a separate CLI command (if one exists). Do not imply in the confirmation message that the site has been updated immediately. A safe confirmation message is: `"File '<file>' has been added to the publish queue."`

---

### Story 13.3: publish_remove

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to remove a note from the Obsidian Publish queue,
So that the user's note will be excluded from their published site when they next sync their publish changes.

**Acceptance Criteria:**
- [ ] Tool `publish_remove` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `file`: string (required) — vault-relative path to the note
  - `vault`: string (optional)
- [ ] Successful execution marks the file for removal from the publish set and returns a confirmation message
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist in the vault, returns `FILE_NOT_FOUND` structured error
- [ ] When Obsidian Publish is not configured, returns `PUBLISH_NOT_CONFIGURED` structured error
- [ ] When the file is not currently in the publish set, returns an informational success message (not an error): `"File '<file>' is not in the publish set."`
- [ ] `buildArgs` produces `["publish:remove", "file=<path>"]` plus `"vault=<name>"` if provided
- [ ] Tool definition is exported as part of the `publish.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `publish_remove`
- **Description:** "Removes a note from the Obsidian Publish site queue, marking it for exclusion when the user next publishes their site. The note must exist in the vault. If the note is not currently in the publish set, returns an informational message rather than an error. Use publish_list to check current publish state. Requires Obsidian Publish. Returns FILE_NOT_FOUND if the file doesn't exist in the vault. Example: file='Blog/old-post.md'"
- **Parameters:**
  - `file`: `string` (required) — Vault-relative path to the note to remove from the publish set (e.g., `Blog/old-post.md`).
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the file was removed from the publish queue, or informational message if it was not in the publish set.

**CLI Mapping:**
```
obsidian publish:remove file=<file-path> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '<file>' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Publish not configured → `{"error": "Obsidian Publish is not configured for this vault. Set up Publish in Obsidian Settings → Publish, then log in with your Obsidian account.", "code": "PUBLISH_NOT_CONFIGURED"}`
- File not in publish set → informational success (not an error): `"File '<file>' is not currently in the publish set."`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Like `publish_add`, removing a file from the publish queue does not immediately update the live site — it marks it for the next publish sync. Do not imply in the confirmation message that the site has been updated immediately. A safe confirmation message is: `"File '<file>' has been removed from the publish queue."` The `FILE_NOT_FOUND` error applies when the note itself does not exist in the vault — a note that exists in the vault but is not in the publish set should return the informational message, not `FILE_NOT_FOUND`.
