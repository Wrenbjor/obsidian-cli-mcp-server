# Epic 12: Sync

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/sync.ts`
**Tool count:** 3

---

## Overview

Obsidian Sync is Obsidian's official paid sync service. These tools expose three Sync-specific operations: checking sync health and status, viewing the version history of a file, and restoring a file to a prior version.

All three tools require an active Obsidian Sync subscription and for Sync to be configured and active in the vault. Any tool called when Sync is not configured returns `SYNC_NOT_ENABLED` with clear instructions.

These tools are particularly valuable for the AI in recovery scenarios — when a user reports that a note was accidentally modified or deleted, the AI can check sync history and offer to restore it.

---

## Stories

---

### Story 12.1: sync_status

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to check the current Obsidian Sync status and health of a vault,
So that I can determine whether sync is functioning correctly, report on pending changes, and identify the last time the vault synced successfully.

**Acceptance Criteria:**
- [ ] Tool `sync_status` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns sync health status, last sync timestamp, and count of pending changes (or indication that vault is fully synced)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When Obsidian Sync is not configured or not active in the vault, returns `SYNC_NOT_ENABLED` structured error with instructions
- [ ] When `vault` is provided, targets that vault's sync status
- [ ] `buildArgs` produces `["sync:status"]` with no vault, or `["sync:status", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `sync.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `sync_status`
- **Description:** "Returns the current Obsidian Sync status for the vault, including sync health, the timestamp of the last successful sync, and the number of pending changes. Use this to verify Sync is working, diagnose sync issues, or confirm a vault is up to date before making changes. Requires an active Obsidian Sync subscription and Sync configured for the vault. Returns SYNC_NOT_ENABLED if Sync is not set up."
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Sync status report including: health indicator (synced/syncing/error), last sync timestamp, and count of pending changes (files waiting to upload or download).

**CLI Mapping:**
```
obsidian sync:status [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Sync not configured → `{"error": "Obsidian Sync is not configured for this vault. Set up Obsidian Sync in Settings → Sync, then log in with your Obsidian account.", "code": "SYNC_NOT_ENABLED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Sync status may include a live sync-in-progress state if the vault is actively syncing at the time of the call. Pass through the CLI output verbatim — do not attempt to parse or reformat status fields. If the CLI uses a non-zero exit code for sync error states (e.g., sync conflict), surface this as a `COMMAND_FAILED` error with the stderr content.

---

### Story 12.2: sync_history

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve the version history of a specific file as recorded by Obsidian Sync,
So that I can show the user available restore points and help them recover an earlier version of a note.

**Acceptance Criteria:**
- [ ] Tool `sync_history` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `path`: string (required) — vault-relative path to the file
  - `vault`: string (optional)
- [ ] Successful execution returns a list of historical versions for the file, each with at minimum: version ID (or number), timestamp, and author/device name
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist in the vault, returns `FILE_NOT_FOUND` structured error
- [ ] When Obsidian Sync is not configured, returns `SYNC_NOT_ENABLED` structured error
- [ ] When `vault` is provided, targets that vault
- [ ] `buildArgs` produces `["sync:history", "path=<path>"]` plus `"vault=<name>"` if provided
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `sync.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `sync_history`
- **Description:** "Returns the version history for a specific file as recorded by Obsidian Sync. Each version includes a version identifier (used with sync_restore), the timestamp when that version was saved, and the author or device that saved it. Use this to identify available restore points before calling sync_restore. Requires Obsidian Sync. Example: path='Projects/website-redesign.md'"
- **Parameters:**
  - `path`: `string` (required) — Vault-relative path to the file (e.g., `Projects/website-redesign.md`). The file must exist in the vault.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of version history entries for the file. Each entry includes: version ID or number (used as the `version` parameter in `sync_restore`), timestamp of the version, and author/device identifier.

**CLI Mapping:**
```
obsidian sync:history path=<file-path> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '<path>' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Sync not configured → `{"error": "Obsidian Sync is not configured for this vault. Set up Obsidian Sync in Settings → Sync, then log in with your Obsidian account.", "code": "SYNC_NOT_ENABLED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** The version identifiers returned in this list are the exact values to pass as the `version` parameter to `sync_restore`. The version format (integer, UUID, timestamp string) is determined by the CLI output — do not assume a specific format. If a file has no sync history yet (newly created and not yet synced), the CLI may return an empty list or a single current-version entry. Pass through whatever the CLI returns.

---

### Story 12.3: sync_restore

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to restore a file to a specific historical version from Obsidian Sync,
So that I can help the user recover a note from an earlier state without them needing to use Obsidian's version history UI manually.

**Acceptance Criteria:**
- [ ] Tool `sync_restore` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `path`: string (required) — vault-relative path to the file
  - `version`: string (required) — version ID from sync history
  - `vault`: string (optional)
- [ ] Successful execution restores the file to the specified version and returns a confirmation message
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified file does not exist, returns `FILE_NOT_FOUND` structured error
- [ ] When the version ID does not match any recorded version for the file, returns `VERSION_NOT_FOUND` structured error
- [ ] When Obsidian Sync is not configured, returns `SYNC_NOT_ENABLED` structured error
- [ ] `buildArgs` produces `["sync:restore", "path=<path>", "version=<version>"]` plus `"vault=<name>"` if provided
- [ ] Tool definition is exported as part of the `sync.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `sync_restore`
- **Description:** "Restores a file to a specific historical version recorded by Obsidian Sync. Use sync_history first to get available version IDs for the file. This overwrites the current file content with the historical version — the operation is reversible only by restoring another version from Sync history. Requires Obsidian Sync. Returns VERSION_NOT_FOUND if the version ID is not valid for the specified file. Example: path='Projects/website-redesign.md' version='42'"
- **Parameters:**
  - `path`: `string` (required) — Vault-relative path to the file to restore (e.g., `Projects/website-redesign.md`).
  - `version`: `string` (required) — Version identifier from `sync_history` output. Use `sync_history` to retrieve valid version IDs for this file.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the file was successfully restored to the specified version.

**CLI Mapping:**
```
obsidian sync:restore path=<file-path> version=<version-id> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- File not found → `{"error": "File '<path>' was not found in the vault.", "code": "FILE_NOT_FOUND"}`
- Version not found → `{"error": "Version '<version>' was not found in the sync history for '<path>'. Use sync_history to see available versions.", "code": "VERSION_NOT_FOUND"}`
- Sync not configured → `{"error": "Obsidian Sync is not configured for this vault. Set up Obsidian Sync in Settings → Sync, then log in with your Obsidian account.", "code": "SYNC_NOT_ENABLED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** This is a destructive operation in the sense that it overwrites current file content. The AI should inform the user before calling this tool in an agentic context. However, since Obsidian Sync maintains full history, the operation is effectively reversible via another restore call. Do not add any confirmation prompting inside the tool itself — that is the AI's responsibility, not the MCP server's. The `version` parameter type is `string` to accommodate version formats that may not be numeric (UUIDs, timestamp strings, etc.).
