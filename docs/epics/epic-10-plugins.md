# Epic 10: Plugins

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/plugins.ts`
**Tool count:** 3

---

## Overview

Plugin management tools give the AI visibility into what plugins are installed and the ability to enable or hot-reload them. These tools are primarily useful for developers building or debugging Obsidian plugins, and for power users who want the AI to help manage their plugin configuration.

Three tools are provided:
- `plugins_list` — read-only discovery of all installed plugins and their current status
- `plugin_enable` — enables a specific plugin by its plugin ID
- `plugin_reload` — hot-reloads a running plugin without restarting Obsidian

Note: There is no `plugin_disable` in this epic. If a disable command exists in the CLI, it is deferred to a future epic. Do not implement it here.

Plugin IDs in Obsidian are the folder names under `.obsidian/plugins/` (e.g., `dataview`, `templater-obsidian`, `obsidian-tasks-plugin`). IDs are lowercase, often hyphenated, and distinct from the human-readable plugin names.

---

## Stories

---

### Story 10.1: plugins_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all installed plugins in the vault with their enabled/disabled status and IDs,
So that I can determine which plugins are available and which need to be enabled before performing plugin-dependent operations.

**Acceptance Criteria:**
- [ ] Tool `plugins_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of all installed plugins with at minimum: plugin ID, plugin name (human-readable), and enabled/disabled status
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no plugins are installed (unusual but possible), returns an empty list (not an error)
- [ ] When `vault` is provided, targets that vault's plugin configuration
- [ ] `buildArgs` produces `["plugins"]` with no vault, or `["plugins", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `plugins.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `plugins_list`
- **Description:** "Lists all installed Obsidian plugins with their plugin ID, display name, and enabled/disabled status. Plugin IDs (e.g., 'dataview', 'templater-obsidian') are the identifiers used with plugin_enable and plugin_reload. Use this to discover plugin IDs before enabling or reloading. Example output might show: 'dataview [enabled], templater-obsidian [disabled], obsidian-tasks-plugin [enabled]'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of installed plugins. Each entry includes the plugin ID (used for enable/reload operations), human-readable name, and current status (enabled or disabled).

**CLI Mapping:**
```
obsidian plugins [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No plugins installed → empty list (not an error): `"No community plugins installed in this vault."`

**Notes:** Core plugins (built into Obsidian, like Daily Notes and Templates) may or may not appear in this list depending on how the CLI implements the command. Pass through whatever the CLI returns without filtering. The plugin IDs returned here are the exact strings to use with `plugin_enable` and `plugin_reload`.

---

### Story 10.2: plugin_enable

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to enable a specific Obsidian plugin by its plugin ID,
So that I can activate plugins the user needs without requiring them to navigate Obsidian's settings UI.

**Acceptance Criteria:**
- [ ] Tool `plugin_enable` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `id`: string (required) — the plugin's ID
  - `vault`: string (optional)
- [ ] Successful execution returns a confirmation message indicating the plugin was enabled
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the plugin ID does not exist in the vault, returns `PLUGIN_NOT_FOUND` structured error
- [ ] When the plugin is already enabled, returns an informational message (not an error): the tool succeeds with a message like `"Plugin '<id>' is already enabled."`
- [ ] `buildArgs` produces `["plugin:enable", "id=<id>"]` plus `"vault=<name>"` if provided
- [ ] Tool definition is exported as part of the `plugins.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `plugin_enable`
- **Description:** "Enables an installed Obsidian plugin by its plugin ID. The plugin ID is the technical identifier (e.g., 'dataview', 'templater-obsidian'), not the display name. Use plugins_list to find the correct ID. If the plugin is already enabled, returns an informational message rather than an error. Returns PLUGIN_NOT_FOUND if no plugin with that ID is installed."
- **Parameters:**
  - `id`: `string` (required) — The plugin's ID (e.g., `dataview`, `templater-obsidian`). Use `plugins_list` to find valid IDs. This is case-sensitive.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation message that the plugin was enabled, or informational message if it was already enabled.

**CLI Mapping:**
```
obsidian plugin:enable id=<plugin-id> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin ID not found → `{"error": "Plugin '<id>' was not found. Use plugins_list to see installed plugin IDs.", "code": "PLUGIN_NOT_FOUND"}`
- Plugin already enabled → informational success (not an error): `"Plugin '<id>' is already enabled."`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Plugin IDs are case-sensitive and must match exactly (e.g., `dataview` not `Dataview`). The "already enabled" case must be handled as a successful tool result, not as an error, so the AI does not treat it as a failure. Check the CLI's exit code: if exit code is 0 and stdout contains an "already enabled" indicator, format it as a success message. Do not implement plugin disable — it is out of scope for this epic.

---

### Story 10.3: plugin_reload

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to hot-reload a plugin in Obsidian without restarting the app,
So that I can apply plugin code changes during development without interrupting the user's Obsidian session.

**Acceptance Criteria:**
- [ ] Tool `plugin_reload` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `id`: string (required) — the plugin's ID
  - `vault`: string (optional)
- [ ] Successful execution returns a confirmation that the plugin was reloaded
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the plugin ID does not exist in the vault, returns `PLUGIN_NOT_FOUND` structured error
- [ ] When the plugin exists but is currently disabled, returns `PLUGIN_NOT_ENABLED` structured error (a disabled plugin cannot be hot-reloaded)
- [ ] `buildArgs` produces `["plugin:reload", "id=<id>"]` plus `"vault=<name>"` if provided
- [ ] Tool definition is exported as part of the `plugins.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `plugin_reload`
- **Description:** "Hot-reloads an enabled Obsidian plugin without restarting Obsidian. This is primarily a developer tool — use it after modifying plugin code to apply changes immediately. The plugin must already be enabled; use plugin_enable first if needed. Returns PLUGIN_NOT_FOUND if the ID doesn't exist, PLUGIN_NOT_ENABLED if the plugin is installed but currently disabled. Plugin ID examples: 'dataview', 'templater-obsidian', 'obsidian-tasks-plugin'."
- **Parameters:**
  - `id`: `string` (required) — The plugin's ID (e.g., `dataview`, `templater-obsidian`). Use `plugins_list` to find valid IDs. This is case-sensitive.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the plugin was successfully reloaded.

**CLI Mapping:**
```
obsidian plugin:reload id=<plugin-id> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Plugin ID not found → `{"error": "Plugin '<id>' was not found. Use plugins_list to see installed plugin IDs.", "code": "PLUGIN_NOT_FOUND"}`
- Plugin installed but disabled → `{"error": "Plugin '<id>' is installed but not enabled. Enable it first with plugin_enable before reloading.", "code": "PLUGIN_NOT_ENABLED"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Hot-reload is a developer-facing feature. It triggers the plugin's `onunload()` and `onload()` lifecycle methods in sequence. The CLI maps to Obsidian's internal hot-reload mechanism. If the CLI does not distinguish between "plugin not found" and "plugin not enabled" via exit code, use stderr content to differentiate. When in doubt, surface the raw stderr as a `COMMAND_FAILED` error rather than misclassifying.
