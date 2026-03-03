# Architecture: Obsidian CLI MCP Server

## Decision Summary

**Architecture:** Standalone TypeScript/Node.js MCP server
**Transport:** stdio (Claude Desktop manages process lifecycle)
**Obsidian integration:** Subprocess calls to `obsidian` CLI
**Distribution:** npm package

The Obsidian plugin approach was evaluated and rejected for v1. The CLI already handles all Obsidian integration via IPC — there is no value in embedding a second layer inside Obsidian itself. A standalone server is simpler, faster to ship, and easier to maintain.

---

## System Overview

```
MCP Client (Claude Desktop / Cursor / any MCP host)
    │  stdio (JSON-RPC over stdin/stdout)
    ▼
obsidian-mcp  (this server, Node.js process)
    │  child_process.execFile()
    ▼
obsidian CLI  (ships with Obsidian v1.12+)
    │  IPC socket
    ▼
Running Obsidian app
```

The MCP client starts this server as a subprocess. Communication is stdin/stdout JSON-RPC (stdio transport). Each tool invocation shells out to the `obsidian` CLI binary, which communicates with the running Obsidian app over its IPC socket.

---

## Technology Stack

| Concern | Choice | Rationale |
|---|---|---|
| Language | TypeScript | Project already scaffolded; strong MCP SDK support |
| Runtime | Node.js 18+ | Required for MCP SDK; ships with modern npm |
| MCP SDK | `@modelcontextprotocol/sdk` | Official TypeScript SDK |
| Transport | stdio | Standard for Claude Desktop-managed servers |
| CLI execution | `child_process.execFile` | Secure; avoids shell injection |
| Build | esbuild (existing) | Already configured |
| Distribution | npm (`obsidian-mcp`) | Standard; `npx obsidian-mcp` zero-install usage |

---

## Prerequisites (User)

- Obsidian v1.12+ with Installer v1.11.7+
- CLI enabled: **Settings → General → Command Line Interface → toggle on → Install Path**
- Obsidian must be running when tools are invoked
- Node.js 18+ (for npx / global install)

---

## Project Structure

```
src/
  main.ts              # Entry point — creates server, registers all tools, starts stdio
  executor.ts          # CLI subprocess wrapper (execObsidian, parseOutput)
  types.ts             # Shared TypeScript types and interfaces
  tools/
    registry.ts        # Tool factory and batch-registration helpers
    files.ts           # Epic 1: Files & Folders tools (10 tools)
    search.ts          # Epic 2: Search tools (2 tools)
    properties.ts      # Epic 3: Properties tools (3 tools)
    links.ts           # Epic 4: Links tools (3 tools)
    tags.ts            # Epic 5: Tags tools (2 tools)
    tasks.ts           # Epic 6: Tasks tools (3 tools)
    daily.ts           # Epic 7: Daily Notes tools (4 tools)
    bookmarks.ts       # Epic 8: Bookmarks & Templates tools (2 tools)
    bases.ts           # Epic 9: Bases tools (2 tools)
    plugins.ts         # Epic 10: Plugins tools (3 tools)
    themes.ts          # Epic 11: Themes & Appearance tools (3 tools)
    sync.ts            # Epic 12: Sync tools (3 tools)
    publish.ts         # Epic 13: Publish tools (3 tools)
    developer.ts       # Epic 14: Developer tools (9 tools)
docs/
  prd.md
  architecture.md      # this file
  epics/               # 15 epic files with stories
```

---

## Core Module: executor.ts

All tool handlers go through a single CLI execution layer. This is the only place `child_process` is called.

```typescript
interface ObsidianResult {
  stdout: string;
  stderr: string;
  exitCode: number;
}

async function execObsidian(args: string[]): Promise<ObsidianResult>
```

**Responsibilities:**
- Locate the `obsidian` CLI binary (configurable; default: system PATH)
- Execute via `execFile` (not `exec`) to prevent shell injection
- Capture stdout, stderr, exit code
- Apply configurable timeout (default: 30s)
- Map known error patterns to structured error objects

**Error detection:**
- Exit code ≠ 0 → surface stderr as structured error
- "not running" in stderr → `OBSIDIAN_NOT_RUNNING` error code
- "not found" / "does not exist" → `FILE_NOT_FOUND` error code
- Timeout exceeded → `TIMEOUT` error code

---

## Core Module: registry.ts

Tools are registered via a declarative factory rather than 52 individual handler functions.

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  buildArgs: (params: Record<string, unknown>) => string[];
  transformOutput?: (raw: string, params: Record<string, unknown>) => string;
}

function registerTool(server: McpServer, def: ToolDefinition): void
function registerTools(server: McpServer, defs: ToolDefinition[]): void
```

Each domain file exports an array of `ToolDefinition` objects. `main.ts` calls `registerTools` once per domain. This keeps tool additions to a single location per domain, and the execution contract (error handling, timeout, output formatting) is enforced uniformly by the registry.

---

## Transport: stdio

The server uses stdio transport. Claude Desktop starts and manages the process:

**claude_desktop_config.json:**
```json
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": ["-y", "obsidian-mcp"]
    }
  }
}
```

Or, for global install:
```json
{
  "mcpServers": {
    "obsidian": {
      "command": "obsidian-mcp"
    }
  }
}
```

No port management. No firewall rules. No startup scripts.

---

## Error Contract

Every tool must surface errors as readable MCP content, not raw exceptions. The executor maps errors to this structure:

```typescript
interface McpError {
  error: string;       // Human-readable message
  code: string;        // Machine-readable code
  detail?: string;     // Original stderr or additional context
}
```

**Standard error messages:**

| Condition | code | message |
|---|---|---|
| Obsidian not running | `OBSIDIAN_NOT_RUNNING` | `"Obsidian must be open for this tool to work. Launch Obsidian and try again."` |
| File not found | `FILE_NOT_FOUND` | `"File '{file}' was not found in the vault."` |
| Vault not found | `VAULT_NOT_FOUND` | `"Vault '{vault}' was not found. Check the vault name and try again."` |
| Command timeout | `TIMEOUT` | `"Command timed out after {n}s. Obsidian may be busy."` |
| CLI not found | `CLI_NOT_FOUND` | `"Obsidian CLI not found. Enable it in Obsidian: Settings → General → Command Line Interface."` |
| Generic failure | `COMMAND_FAILED` | `"Command failed: {stderr}"` |

Errors are returned as valid MCP tool results (not thrown), so the AI receives structured context rather than an exception trace.

---

## Global Flags

These optional parameters apply across multiple tools and must be supported uniformly:

| Parameter | Type | Description | Applicable To |
|---|---|---|---|
| `vault` | `string` | Target vault name | All commands |
| `format` | `enum` | Output format: `json`, `csv`, `tsv`, `md`, `paths`, `yaml`, `tree` | Format-supporting commands |
| `copy` | `boolean` | Copy output to clipboard | All commands |
| `silent` | `boolean` | Suppress file opening in Obsidian | `create`, `read`, others |
| `total` | `boolean` | Return count only | List commands |

The `buildArgs` function in each `ToolDefinition` is responsible for appending these when present in the input params.

---

## Output Handling

Raw CLI output is returned as-is unless `transformOutput` is defined on the tool. For `format=json` outputs, the executor validates JSON parseability before returning. Malformed JSON from the CLI is surfaced as a `PARSE_ERROR` with the raw output in `detail`.

---

## Distribution

- npm package name: `obsidian-mcp`
- Binary entry: `bin.obsidian-mcp` pointing to compiled `main.js`
- Compile target: Node.js 18, CommonJS
- No Obsidian plugin review process required
- README includes Claude Desktop config snippet for zero-friction setup

---

## Future Consideration: Plugin Wrapper

If community plugin distribution becomes a priority (v2), a thin Obsidian plugin can be added that:
1. Detects Claude Desktop installation
2. Auto-injects the `mcpServers` config entry
3. Provides an enable/disable toggle in Obsidian settings

The core MCP server code remains unchanged.

---

## Out of Scope (v1)

- HTTP/SSE transport (not needed for stdio-based clients)
- Plugin API access beyond CLI command surface
- Remote vault or cloud sync integration
- Authentication / API key gating
- GUI settings panel
