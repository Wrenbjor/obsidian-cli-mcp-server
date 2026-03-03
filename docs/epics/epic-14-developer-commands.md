# Epic 14: Developer Commands

**Status:** Ready for Implementation
**Assignee:** Amelia
**Dependencies:** Epic 0 (Core Infrastructure — executor.ts, registry.ts, types.ts)
**Source file:** `src/tools/developer.ts`
**Tool count:** 9

---

## Overview

Developer tools expose Obsidian's built-in debugging and introspection capabilities via the CLI. These are not note-taking tools — they target developers building or debugging Obsidian plugins, themes, and workflows.

The tools cover: toggling DevTools, attaching/detaching the debugger, capturing screenshots, reading console logs and JS errors, inspecting CSS rules, querying the DOM, toggling mobile emulation, and executing arbitrary JavaScript in the Obsidian app context.

**Security note for `dev_eval`:** This tool executes arbitrary JavaScript inside the Obsidian app's Electron/Chromium context. It has full access to Obsidian's internals, the Node.js runtime, and the filesystem. This is intentional — it is a developer power tool. The tool description must make this explicit so the AI understands the scope of what it is invoking. No additional restrictions are imposed by this server; the user configured this tool intentionally.

**Implementation note:** All 9 tools in this epic are implemented in `src/tools/developer.ts` and registered in `main.ts` alongside all other tool domains.

---

## Stories

---

### Story 14.1: dev_devtools

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to toggle the Electron DevTools panel in Obsidian,
So that I can open developer tools for the user during a debugging session without requiring them to use a keyboard shortcut or menu.

**Acceptance Criteria:**
- [ ] Tool `dev_devtools` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution toggles the DevTools panel (open → close or close → open) and returns a message indicating the new state after toggle
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] The return value indicates the resulting state (e.g., `"DevTools opened."` or `"DevTools closed."`)
- [ ] `buildArgs` produces `["devtools"]` with no vault, or `["devtools", "vault=<name>"]` with vault
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_devtools`
- **Description:** "Toggles the Electron DevTools panel in Obsidian open or closed. Returns the resulting state after toggling. Useful for opening developer tools during debugging sessions. Each call flips the current state — call once to open, call again to close."
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Message indicating the new state of DevTools after the toggle (opened or closed).

**CLI Mapping:**
```
obsidian devtools [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** DevTools is a toggle — there is no separate open/close command in this epic. The CLI command flips the current state. The return message from the CLI will indicate whether DevTools is now open or closed — pass it through as-is.

---

### Story 14.2: dev_debug

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to attach or detach the Chrome DevTools Protocol debugger to the Obsidian app,
So that I can enable or disable remote debugging for a developer's plugin debugging workflow.

**Acceptance Criteria:**
- [ ] Tool `dev_debug` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `enabled`: boolean (required) — `true` to attach debugger, `false` to detach
  - `vault`: string (optional)
- [ ] When `enabled=true`, `buildArgs` appends `"on"` to the command: `["dev:debug", "on"]`
- [ ] When `enabled=false`, `buildArgs` appends `"off"` to the command: `["dev:debug", "off"]`
- [ ] Successful execution attaches or detaches the debugger and returns a confirmation
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_debug`
- **Description:** "Attaches or detaches the Chrome DevTools Protocol (CDP) debugger to the Obsidian app. Set enabled=true to attach the debugger (allows remote debugging from Chrome DevTools or a CDP client), enabled=false to detach. Used during plugin development for step-through debugging."
- **Parameters:**
  - `enabled`: `boolean` (required) — `true` to attach the Chrome DevTools Protocol debugger, `false` to detach it.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that the debugger was attached or detached.

**CLI Mapping:**
```
obsidian dev:debug on|off [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** The MCP parameter is `enabled: boolean` for type clarity. The `buildArgs` function translates this to the string `"on"` or `"off"` for the CLI call. This translation must happen in `buildArgs`, not in the executor.

---

### Story 14.3: dev_screenshot

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to capture a full screenshot of the Obsidian app and receive it as a base64 PNG,
So that I can visually inspect the current Obsidian UI state during debugging without requiring the user to manually take and share a screenshot.

**Acceptance Criteria:**
- [ ] Tool `dev_screenshot` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `path`: string (required) — filesystem path where the screenshot will be saved
  - `vault`: string (optional)
- [ ] Successful execution saves a PNG file at the specified path and returns the screenshot as a base64-encoded PNG string
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the specified path is not writable (directory does not exist, permission denied), returns `PATH_NOT_WRITABLE` structured error
- [ ] `buildArgs` produces `["dev:screenshot", "path=<path>"]` plus `"vault=<name>"` if provided
- [ ] The base64 PNG is returned in the tool result; the saved file at `path` is a side effect
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_screenshot`
- **Description:** "Captures a full screenshot of the Obsidian app window and saves it as a PNG file at the specified path. Also returns the screenshot as a base64-encoded PNG string so it can be viewed directly. Useful for visually inspecting Obsidian's UI state during debugging. Returns PATH_NOT_WRITABLE if the target path is not accessible."
- **Parameters:**
  - `path`: `string` (required) — Absolute filesystem path where the PNG screenshot should be saved (e.g., `/tmp/obsidian-screenshot.png` or `C:\Users\user\Desktop\screenshot.png`).
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Base64-encoded PNG string of the screenshot. The PNG file is also saved at the specified path.

**CLI Mapping:**
```
obsidian dev:screenshot path=<file-path> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Path not writable → `{"error": "Cannot write to path '<path>'. Check that the directory exists and you have write permission.", "code": "PATH_NOT_WRITABLE"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** The CLI saves the file and outputs the base64 to stdout (or the path to the saved file — verify CLI behavior). If the CLI only outputs the saved file path rather than base64, the server should read the file and base64-encode it before returning. If the CLI outputs base64 directly to stdout, return it as-is. The path parameter should be an absolute path to avoid ambiguity about the working directory.

---

### Story 14.4: dev_console

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve captured browser console messages from the Obsidian app,
So that I can inspect console output during plugin or theme debugging without the user needing to open DevTools.

**Acceptance Criteria:**
- [ ] Tool `dev_console` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `limit`: number (optional) — maximum number of messages to return; no default enforced server-side
  - `level`: string (optional) — filter by log level; must be one of: `error`, `warn`, `log`
  - `vault`: string (optional)
- [ ] Successful execution returns captured console messages, optionally filtered by level and limited in count
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no messages have been captured, returns an empty list (not an error)
- [ ] When `limit` is provided, `buildArgs` appends `"limit=<N>"` to the command
- [ ] When `level` is provided, `buildArgs` appends `"level=<level>"` to the command
- [ ] `buildArgs` base command: `["dev:console"]` plus optional `limit=` and `level=` args
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_console`
- **Description:** "Returns captured browser console messages from the Obsidian app, similar to what you would see in the Console tab of DevTools. Optionally filter by log level (error, warn, log) and limit the number of results. Messages are returned in chronological order. Returns an empty list if no messages have been captured. Use level='error' to focus on errors only."
- **Parameters:**
  - `limit`: `number` (optional) — Maximum number of messages to return. If omitted, returns all captured messages.
  - `level`: `string` (optional) — Filter messages by log level. One of: `error`, `warn`, `log`. If omitted, returns all levels.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of captured console messages with timestamp, level, and message content. Empty list if no messages captured.

**CLI Mapping:**
```
obsidian dev:console [limit=<N>] [level=error|warn|log] [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No messages captured → empty list (not an error): `"No console messages captured."`
- Invalid level value → `{"error": "Invalid level '<level>'. Must be one of: error, warn, log.", "code": "INVALID_PARAMETER"}`

**Notes:** Input validation for `level` should happen in `buildArgs` before the CLI call. If the value is not one of `error`, `warn`, `log`, return `INVALID_PARAMETER` without invoking the CLI. The `limit` parameter must be a positive integer; if a non-positive value is provided, return `INVALID_PARAMETER`.

---

### Story 14.5: dev_errors

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve all captured JavaScript errors from the Obsidian app,
So that I can diagnose plugin or theme errors without the user needing to open DevTools or share screenshots.

**Acceptance Criteria:**
- [ ] Tool `dev_errors` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly: `vault` is optional string; no other parameters
- [ ] Successful execution returns all captured JavaScript errors with their message, stack trace (if available), and timestamp
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no errors have been captured, returns an empty list (not an error)
- [ ] When `vault` is provided, targets that vault
- [ ] `buildArgs` produces `["dev:errors"]` with no vault, or `["dev:errors", "vault=<name>"]` with vault
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_errors`
- **Description:** "Returns all captured JavaScript errors from the Obsidian app, including unhandled exceptions and rejected promises. Each error includes the error message, stack trace when available, and timestamp. Returns an empty list if no errors have occurred. Useful for diagnosing plugin failures or uncaught exceptions without opening DevTools."
- **Parameters:**
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** List of captured JavaScript errors. Each entry includes: error message, stack trace (if available), and timestamp. Empty list if no errors have been captured.

**CLI Mapping:**
```
obsidian dev:errors [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No errors captured → empty list (not an error): `"No JavaScript errors captured."`

**Notes:** This is a simpler, focused subset of `dev_console` — errors only, no level filtering needed, no limit parameter. This distinction exists because error-checking is the most common single-use debugging action. Implement as a separate tool, not a wrapper around `dev_console`.

---

### Story 14.6: dev_css

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to retrieve CSS rules matching a given selector from the Obsidian app's computed styles,
So that I can inspect how a theme or snippet is styling a particular element without requiring the user to manually navigate DevTools.

**Acceptance Criteria:**
- [ ] Tool `dev_css` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `selector`: string (required) — CSS selector to inspect
  - `vault`: string (optional)
- [ ] Successful execution returns CSS rules matching the selector, including the source file and line number for each rule
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no CSS rules match the selector, returns an empty result (not an error)
- [ ] `buildArgs` produces `["dev:css", "selector=<selector>"]` plus `"vault=<name>"` if provided
- [ ] CSS selectors with spaces or special characters are passed as a single `execFile` array element (no shell quoting needed)
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_css`
- **Description:** "Returns CSS rules that match a given CSS selector in the Obsidian app, along with the source file and line number for each rule. Useful for theme debugging — find out what stylesheet is responsible for a particular style. Selector examples: '.markdown-rendered', '.workspace-leaf', '.theme-dark .nav-file-title'. Returns empty result if no rules match."
- **Parameters:**
  - `selector`: `string` (required) — CSS selector to inspect (e.g., `.markdown-rendered`, `.workspace-leaf-content`, `.theme-dark .nav-file-title`).
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** CSS rules matching the selector, each with the rule text, source stylesheet filename, and line number.

**CLI Mapping:**
```
obsidian dev:css selector=<css-selector> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No matching rules → empty result (not an error): `"No CSS rules match selector '<selector>'."`

**Notes:** CSS selectors may include spaces (e.g., `.theme-dark .nav-file-title`). When constructing the `buildArgs` array, the entire selector string is one element: `["dev:css", "selector=.theme-dark .nav-file-title"]`. Because `execFile` does not invoke a shell, no quoting is needed — the string is passed verbatim.

---

### Story 14.7: dev_dom

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to query the Obsidian app's DOM for elements matching a CSS selector,
So that I can inspect the rendered UI structure during plugin or theme development without requiring DevTools access.

**Acceptance Criteria:**
- [ ] Tool `dev_dom` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `selector`: string (required) — CSS selector to match against DOM elements
  - `total`: boolean (optional) — when `true`, returns only the count of matching elements; default is `false`
  - `vault`: string (optional)
- [ ] When `total` is `false` or omitted, returns the matched DOM elements (or a representation thereof)
- [ ] When `total` is `true`, returns only the integer count of matching elements
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When no elements match the selector, returns an empty result or count of 0 (not an error)
- [ ] `buildArgs` when `total=false` (default): `["dev:dom", "selector=<selector>"]`
- [ ] `buildArgs` when `total=true`: `["dev:dom", "selector=<selector>", "total"]`
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_dom`
- **Description:** "Queries the Obsidian app's DOM for elements matching a CSS selector. Returns the matching elements and their attributes/content by default. Set total=true to return only the count of matching elements (useful for checking existence or counting without full element data). Selector examples: '.workspace-leaf', '.markdown-preview-view', '#app'. Returns empty result if no elements match."
- **Parameters:**
  - `selector`: `string` (required) — CSS selector to match against the Obsidian DOM (e.g., `.workspace-leaf`, `#app`, `.markdown-rendered h1`).
  - `total`: `boolean` (optional) — When `true`, returns only the count of matching elements. Default is `false`.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** When `total=false`: representation of matching DOM elements (structure, attributes, and text content). When `total=true`: integer count of matching elements.

**CLI Mapping:**
```
obsidian dev:dom selector=<css-selector> [total] [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`
- No matching elements → empty result or count of 0 (not an error)

**Notes:** The `total` flag is passed to the CLI as a bare flag (`total`), not as `total=true`. When `total` is `false` or omitted, do not include the flag in `buildArgs` at all. Like `dev_css`, selectors with spaces must be passed as a single `execFile` array element.

---

### Story 14.8: dev_mobile

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to toggle Obsidian's mobile emulation mode on or off,
So that I can help a developer test how their plugin or theme renders in mobile layout without requiring Obsidian to be run on a physical device.

**Acceptance Criteria:**
- [ ] Tool `dev_mobile` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `enabled`: boolean (required) — `true` to enable mobile emulation, `false` to disable
  - `vault`: string (optional)
- [ ] When `enabled=true`, `buildArgs` appends `"on"` to the command: `["dev:mobile", "on"]`
- [ ] When `enabled=false`, `buildArgs` appends `"off"` to the command: `["dev:mobile", "off"]`
- [ ] Successful execution toggles mobile emulation mode and returns a confirmation
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_mobile`
- **Description:** "Toggles mobile emulation mode in Obsidian, which simulates how Obsidian renders on a mobile device. Set enabled=true to activate mobile layout, enabled=false to return to desktop layout. Useful for testing plugin or theme mobile compatibility without a physical device."
- **Parameters:**
  - `enabled`: `boolean` (required) — `true` to enable mobile emulation (mobile layout), `false` to disable it (return to desktop layout).
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** Confirmation that mobile emulation was enabled or disabled.

**CLI Mapping:**
```
obsidian dev:mobile on|off [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:** Same `boolean → on/off` translation pattern as `dev_debug`. The `buildArgs` function handles this mapping. The resulting `buildArgs` is `["dev:mobile", "on"]` or `["dev:mobile", "off"]` — never a `vault` before the `on/off` arg; vault always comes last.

---

### Story 14.9: dev_eval

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want to execute arbitrary JavaScript code in the Obsidian app context and receive the result,
So that I can directly interact with Obsidian's API, inspect internal state, or perform operations not covered by other CLI commands during a development session.

**Acceptance Criteria:**
- [ ] Tool `dev_eval` is registered and visible in MCP tool list
- [ ] Input schema matches spec exactly:
  - `code`: string (required) — JavaScript code to execute
  - `vault`: string (optional)
- [ ] Successful execution runs the provided JavaScript in the Obsidian app context and returns the result as a string
- [ ] When Obsidian is not running, returns `OBSIDIAN_NOT_RUNNING` structured error
- [ ] When the provided JavaScript has a syntax error or throws a runtime exception, returns `EVAL_ERROR` structured error with the JavaScript error message and type in the `detail` field
- [ ] `buildArgs` produces `["eval", "code=<javascript>"]` plus `"vault=<name>"` if provided
- [ ] The `code` parameter value, however long or complex, is passed as a single `execFile` array element
- [ ] The tool description explicitly states that this executes code in the Obsidian app context — this is a hard requirement, not optional wording
- [ ] Tool definition is exported as part of the `developer.ts` tool array registered in `main.ts`

**Tool Definition:**
- **Name:** `dev_eval`
- **Description:** "EXECUTES JAVASCRIPT CODE IN THE OBSIDIAN APP CONTEXT. This runs arbitrary JavaScript with full access to Obsidian's internal API, the Node.js runtime, and the filesystem — it is not sandboxed. Use this to interact with Obsidian internals, inspect state, or perform operations not available through other tools. Returns the result of the evaluated expression. Returns EVAL_ERROR if the code has a syntax error or throws a runtime exception. Example: code='app.vault.getName()' or code='Object.keys(app.plugins.plugins)'"
- **Parameters:**
  - `code`: `string` (required) — JavaScript code to execute in the Obsidian app context. Has full access to Obsidian internals via the global `app` object, Node.js APIs, and the filesystem. Not sandboxed.
  - `vault`: `string` (optional) — Name of the vault to target. If omitted, uses the default or currently active vault.
- **Returns:** The string representation of the JavaScript expression's return value. For objects, returns a JSON-serialized or inspect-formatted string.

**CLI Mapping:**
```
obsidian eval code=<javascript-code> [vault=<name>]
```

**Error States:**
- Obsidian not running → `{"error": "Obsidian must be open for this tool to work. Launch Obsidian and try again.", "code": "OBSIDIAN_NOT_RUNNING"}`
- JavaScript syntax or runtime error → `{"error": "JavaScript execution failed: <error message>", "code": "EVAL_ERROR", "detail": "<full error type and stack if available>"}`
- Vault not found → `{"error": "Vault '<name>' was not found. Check the vault name and try again.", "code": "VAULT_NOT_FOUND"}`

**Notes:**

The description wording `"EXECUTES JAVASCRIPT CODE IN THE OBSIDIAN APP CONTEXT"` in all caps at the start is intentional and required. The AI must be unambiguous about what this tool does before invoking it.

The `code` parameter may contain any valid JavaScript, including multi-line code, async/await expressions, and calls to `app.vault`, `app.plugins`, `app.workspace`, and Node.js built-ins. The string is passed as a single element to `execFile` — no escaping or shell-quoting is needed.

The exit code from the CLI distinguishes between successful eval (exit 0) and eval error (non-zero exit). The stderr content on eval failure should be surfaced as the `detail` field of the `EVAL_ERROR` response.

Example valid inputs:
- `app.vault.getName()` — returns the vault name
- `Object.keys(app.plugins.plugins)` — returns array of enabled plugin IDs
- `app.workspace.getActiveFile()?.path` — returns the currently open file path
- `await app.vault.read(app.workspace.getActiveFile())` — reads current file content
