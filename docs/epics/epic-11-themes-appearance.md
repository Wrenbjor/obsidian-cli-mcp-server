# Epic 11: Themes & Appearance

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/themes.ts`
**Tool count:** 3

---

## Overview

Theme and appearance tools give the AI visibility into the user's visual configuration and the ability to change it. Obsidian supports community-developed themes (installed via the community theme browser) and CSS snippets (user-authored `.css` files placed in `.obsidian/snippets/` that can be toggled on or off).

Three tools are provided:
- `themes_list` — lists all installed themes, indicating which one is currently active
- `theme_set` — activates a theme by name
- `snippets_list` — lists all CSS snippets with their enabled/disabled state

There is no `snippet_enable` or `snippet_disable` tool in this epic. If those CLI commands exist, they are deferred. Do not implement them here.

Theme names are the display names shown in Obsidian's Appearance settings, not directory names or file paths. The theme name used in `theme_set` must exactly match a name returned by `themes_list`.

---

## Stories

---

### Story 11.1: themes_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all installed themes with their names and identify the currently active theme,
So that I can show the user what themes are available and confirm which theme is currently applied before suggesting a change.

**Acceptance Criteria:**
- [ ] Tool `themes_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of all installed themes, each with its name and an indication of whether it is currently active
- [ ] The currently active theme is clearly distinguished in the output (e.g., marked with `[active]` or equivalent indicator from CLI)
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no additional themes are installed (only the default Obsidian theme is present), returns at minimum the default theme as active
- [ ] When `vault` is provided, targets that vault's theme configuration
- [ ] `buildArgs` produces `["themes"]` with no vault, or `["themes", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `themes.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `themes_list`
- **Description:** "Lists all installed Obsidian themes with their display names and indicates which theme is currently active. Theme names returned here are the exact strings to use with theme_set. The active theme is marked in the output. Use this before calling theme_set to confirm the theme name is correct. Example output: 'Minimal [active]\nThings\nObsidianite\nAnuPpuccin'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of installed theme names. The currently active theme is indicated. Theme names in this list are the exact values to pass to `theme_set`.

**CLI Mapping:**
```
obsidian themes [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** There is always at least one theme (the built-in default). This tool should never return an empty list. If the CLI returns an empty list, pass it through — do not manufacture a fallback response. Theme names may contain spaces, capital letters, and special characters. Do not sanitize them.

---

### Story 11.2: theme_set

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to activate a specific installed theme by name,
So that I can change the user's Obsidian appearance on request without them needing to navigate settings manually.

**Acceptance Criteria:**
- [ ] Tool `theme_set` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `name`: string (required) — the theme's display name
  - `vault`: string (optional)
- [ ] Successful execution applies the theme and returns a confirmation message
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the theme name does not match any installed theme, returns `THEME_NOT_FOUND` structured error
- [ ] The theme name comparison should match the CLI's behavior exactly — do not implement fuzzy matching or case normalization on the server side
- [ ] `buildArgs` produces `["theme:set", "name=<name>"]` plus `"vault=<name-of-vault>"` if vault provided
- [ ] Theme names containing spaces must be passed as a single argument value (the CLI receives `name=My Theme Name`, not split tokens)
- [ ] Tool definition is exported as part of the `themes.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `theme_set`
- **Description:** "Sets the active Obsidian theme by display name. The theme must already be installed — use themes_list to see available theme names and confirm the exact spelling. Theme names are case-sensitive. Returns THEME_NOT_FOUND if the name does not match any installed theme. Example: name='Minimal' or name='Things' or name='AnuPpuccin'."
- **Parameters:**
  - `name`: `string` (required) — The exact display name of the theme to activate (e.g., `Minimal`, `Things`, `AnuPpuccin`). Use `themes_list` to find the correct name. Case-sensitive.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the theme was successfully applied.

**CLI Mapping:**
```
obsidian theme:set name=<theme-name> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Theme not found → `{"error": "Theme '<name>' was not found in installed themes. Use themes_list to see available theme names.", "code": "THEME_NOT_FOUND"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Theme names with spaces should be passed to `execFile` as a single array element (e.g., `["theme:set", "name=My Theme Name"]`). Because `execFile` does not use a shell, no quoting is necessary — the array element is passed verbatim. Do not split on spaces or add quotes around the value.

---

### Story 11.3: snippets_list

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to list all CSS snippets in the vault with their enabled/disabled state,
So that I can report the user's snippet configuration and identify which snippets are active without navigating Obsidian's Appearance settings.

**Acceptance Criteria:**
- [ ] Tool `snippets_list` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns a list of all CSS snippets with each snippet's name and its enabled/disabled state
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no snippets exist in `.obsidian/snippets/`, returns an empty list (not an error)
- [ ] When `vault` is provided, targets that vault's snippets folder
- [ ] `buildArgs` produces `["snippets"]` with no vault, or `["snippets", "vault=<name>"]` with vault
- [ ] Output is passed through as-is from CLI stdout
- [ ] Tool definition is exported as part of the `themes.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `snippets_list`
- **Description:** "Lists all CSS snippets installed in the Obsidian vault with their names and enabled/disabled status. CSS snippets are user-authored .css files placed in .obsidian/snippets/ that can be toggled on or off. Returns an empty list if no snippets are installed. Example output: 'custom-fonts [enabled]\nhide-scrollbars [disabled]\ncolored-headers [enabled]'"
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of CSS snippet names and their current enabled/disabled status. Empty list message if no snippets are installed.

**CLI Mapping:**
```
obsidian snippets [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No snippets → empty list (not an error): `"No CSS snippets found in this vault."`

**Notes:** Snippets are stored as `.css` files in `.obsidian/snippets/`. The snippet name is typically the filename without the `.css` extension. The CLI may or may not strip the extension — pass through whatever it returns without modification. There is no enable/disable tool in this epic; `snippets_list` is read-only.
