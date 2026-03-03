# Epic 0: Core Infrastructure

**Status:** Ready for Implementation
**Depends On:** Nothing — this epic is the foundation
**Blocks:** All other epics
**Implementer:** Amelia (dev agent)

---

## Overview

This epic establishes everything that every other tool depends on. It must be completed and verified before any other epic begins. The deliverable is a running MCP server that Claude Desktop can connect to, with a CLI executor, a structured error system, a tool registry pattern, and global flag support. No actual Obsidian tools are registered in this epic — only the infrastructure that tools will use.

All code lives in `src/`. Entry point is `src/main.ts`. The server uses stdio transport exclusively.

---

## Stories

---

### Story 0.1: MCP Server Bootstrap

**User Story:**
As an AI assistant using the Obsidian MCP server,
I want the server to start and be discoverable by my MCP client,
So that I can see and invoke Obsidian tools.

**Acceptance Criteria:**
- [ ] `package.json` has a `bin` entry: `"obsidian-mcp": "dist/main.js"`
- [ ] Running `node dist/main.js` starts the server on stdio without errors
- [ ] MCP client (Claude Desktop) can connect and list tools (empty list is acceptable at this stage)
- [ ] Server name is `"obsidian-mcp"` and version matches `package.json`
- [ ] Server stays alive until the MCP client disconnects or sends SIGTERM
- [ ] Server writes nothing to stdout except valid JSON-RPC messages (all debug logging goes to stderr)
- [ ] `package.json` `scripts.build` produces a working `dist/main.js` via esbuild

**Implementation Notes:**

Use `@modelcontextprotocol/sdk`. The server is created with `McpServer` and connected to a `StdioServerTransport`. The entry point must not start the server synchronously at module load — wrap startup in an async IIFE or `main()` function to allow clean error handling during initialization.

```typescript
// src/main.ts skeleton
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "obsidian-mcp",
  version: "1.0.0", // read from package.json at build time or hardcode
});

// Tool registration goes here (other epics)

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  // Server now running; process stays alive until transport closes
}

main().catch((err) => {
  process.stderr.write(`Fatal: ${err}\n`);
  process.exit(1);
});
```

The `esbuild.config.mjs` already exists. Verify it produces a single-file bundle with `--platform=node` and `--format=cjs`. The bundle must be executable by Node.js 18+ without additional dependencies.

**Claude Desktop config for local development testing:**
```json
{
  "mcpServers": {
    "obsidian": {
      "command": "node",
      "args": ["/absolute/path/to/dist/main.js"]
    }
  }
}
```

**Error States:** None — if the server fails to start, it exits with code 1 and writes to stderr.

---

### Story 0.2: CLI Executor Module

**User Story:**
As the MCP server executing tool calls,
I want a single, safe, reusable function that runs `obsidian` CLI commands,
So that all tools share consistent execution, timeout, and error behavior.

**Acceptance Criteria:**
- [ ] `execObsidian(args: string[]): Promise<ObsidianResult>` is exported from `src/executor.ts`
- [ ] Uses `child_process.execFile` (not `exec`, not `spawn`) — no shell interpretation of arguments
- [ ] Captures stdout, stderr, and exit code in `ObsidianResult`
- [ ] Default timeout is 30 seconds; configurable via `OBSIDIAN_TIMEOUT_MS` environment variable
- [ ] When timeout is exceeded, the child process is killed and a `TIMEOUT` error result is returned
- [ ] CLI binary path defaults to `"obsidian"` (system PATH); overridable via `OBSIDIAN_CLI_PATH` environment variable
- [ ] When the binary cannot be found (ENOENT), returns a `CLI_NOT_FOUND` error result
- [ ] Function never throws — all error conditions are captured and returned as structured results
- [ ] Unit-testable: binary path and timeout are injectable parameters (with env var fallback)

**TypeScript Interface:**

```typescript
// src/types.ts

export interface ObsidianResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  timedOut?: boolean;
}

export interface McpErrorResult {
  error: string;   // Human-readable message
  code: string;    // Machine-readable error code (see Story 0.3)
  detail?: string; // Raw stderr or additional context
}

export type ExecResult = ObsidianResult | McpErrorResult;

export function isError(result: ExecResult): result is McpErrorResult {
  return "code" in result;
}
```

**Implementation:**

```typescript
// src/executor.ts
import { execFile } from "child_process";
import { promisify } from "util";

const execFileAsync = promisify(execFile);

export async function execObsidian(
  args: string[],
  options?: { timeoutMs?: number; cliPath?: string }
): Promise<ExecResult> {
  const cliPath = options?.cliPath ?? process.env.OBSIDIAN_CLI_PATH ?? "obsidian";
  const timeoutMs = options?.timeoutMs ?? Number(process.env.OBSIDIAN_TIMEOUT_MS ?? 30000);

  try {
    const { stdout, stderr } = await execFileAsync(cliPath, args, {
      timeout: timeoutMs,
      maxBuffer: 10 * 1024 * 1024, // 10MB output buffer
    });
    return { stdout: stdout.trim(), stderr: stderr.trim(), exitCode: 0 };
  } catch (err: unknown) {
    // Handle structured errors from child_process
    // Map to McpErrorResult — see Story 0.3
  }
}
```

**Notes:**
- `maxBuffer` is 10MB — large vaults may produce substantial output from list commands.
- `args` are passed directly to `execFile` as an array — never interpolated into a shell string. This is the primary shell injection protection.
- The `trim()` calls normalize trailing newlines from CLI output.

---

### Story 0.3: Error Handling & Error Codes

**User Story:**
As an AI assistant receiving tool results,
I want all error conditions to return structured, machine-readable error objects,
So that I can communicate the specific problem to the user without parsing raw exception messages.

**Acceptance Criteria:**
- [ ] All 7 error codes are defined as a TypeScript const enum or string literal union in `src/types.ts`
- [ ] `mapCliError(err: unknown, context?: Record<string, string>): McpErrorResult` is exported from `src/executor.ts`
- [ ] `OBSIDIAN_NOT_RUNNING` is detected from stderr containing "not running", "not connected", or "connection refused" (case-insensitive)
- [ ] `CLI_NOT_FOUND` is detected when `execFile` throws with `code === "ENOENT"`
- [ ] `FILE_NOT_FOUND` is detected from stderr containing "not found", "does not exist", or "no such file" (case-insensitive) combined with exit code ≠ 0
- [ ] `VAULT_NOT_FOUND` is detected from stderr containing "vault not found" or "unknown vault" (case-insensitive)
- [ ] `TIMEOUT` is detected when `execFile` throws with `killed === true` or `signal === "SIGTERM"` after timeout
- [ ] `COMMAND_FAILED` is the fallback for any exit code ≠ 0 not matching a more specific pattern
- [ ] `PARSE_ERROR` is returned when `format=json` output cannot be parsed as valid JSON
- [ ] All error messages use the exact wording from the architecture document
- [ ] Errors are returned as MCP content (text type), not thrown as JavaScript exceptions
- [ ] `formatMcpError(err: McpErrorResult): CallToolResult` converts an error to valid MCP tool result format

**Error Code Definitions:**

```typescript
// src/types.ts
export const ErrorCode = {
  OBSIDIAN_NOT_RUNNING: "OBSIDIAN_NOT_RUNNING",
  CLI_NOT_FOUND: "CLI_NOT_FOUND",
  FILE_NOT_FOUND: "FILE_NOT_FOUND",
  VAULT_NOT_FOUND: "VAULT_NOT_FOUND",
  TIMEOUT: "TIMEOUT",
  COMMAND_FAILED: "COMMAND_FAILED",
  PARSE_ERROR: "PARSE_ERROR",
  // Domain-specific codes (used by individual tools)
  FILE_EXISTS: "FILE_EXISTS",
  DESTINATION_EXISTS: "DESTINATION_EXISTS",
  PROPERTY_NOT_FOUND: "PROPERTY_NOT_FOUND",
  NO_DAILY_NOTE: "NO_DAILY_NOTE",
  DAILY_NOTE_NOT_CONFIGURED: "DAILY_NOTE_NOT_CONFIGURED",
  PLUGIN_NOT_FOUND: "PLUGIN_NOT_FOUND",
  THEME_NOT_FOUND: "THEME_NOT_FOUND",
  TEMPLATES_NOT_CONFIGURED: "TEMPLATES_NOT_CONFIGURED",
  BASES_NOT_ENABLED: "BASES_NOT_ENABLED",
  NOT_A_BASE: "NOT_A_BASE",
  SYNC_NOT_ENABLED: "SYNC_NOT_ENABLED",
  VERSION_NOT_FOUND: "VERSION_NOT_FOUND",
  PUBLISH_NOT_CONFIGURED: "PUBLISH_NOT_CONFIGURED",
  INVALID_QUERY: "INVALID_QUERY",
  PATH_NOT_WRITABLE: "PATH_NOT_WRITABLE",
  EVAL_ERROR: "EVAL_ERROR",
} as const;

export type ErrorCode = typeof ErrorCode[keyof typeof ErrorCode];
```

**Standard Error Messages (exact wording — do not deviate):**

| Code | Message Template |
|---|---|
| `OBSIDIAN_NOT_RUNNING` | `"Obsidian must be open for this tool to work. Launch Obsidian and try again."` |
| `CLI_NOT_FOUND` | `"Obsidian CLI not found. Enable it in Obsidian: Settings → General → Command Line Interface."` |
| `FILE_NOT_FOUND` | `"File '{file}' was not found in the vault."` (substitute actual filename) |
| `VAULT_NOT_FOUND` | `"Vault '{vault}' was not found. Check the vault name and try again."` (substitute vault name) |
| `TIMEOUT` | `"Command timed out after {n}s. Obsidian may be busy."` (substitute seconds) |
| `COMMAND_FAILED` | `"Command failed: {stderr}"` (substitute raw stderr) |
| `PARSE_ERROR` | `"CLI returned malformed output that could not be parsed."` |

**MCP Result Formatting:**

Errors must be returned as valid `CallToolResult` objects:

```typescript
export function formatMcpError(err: McpErrorResult): CallToolResult {
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(err, null, 2),
      },
    ],
    isError: true,
  };
}

export function formatMcpSuccess(text: string): CallToolResult {
  return {
    content: [
      {
        type: "text",
        text,
      },
    ],
  };
}
```

**Notes:**
- Domain-specific error codes (e.g. `NO_DAILY_NOTE`, `FILE_EXISTS`) are detected by the individual tool handlers that call `execObsidian`, not by the core error mapper. The core mapper handles only the codes it can detect from exit code and stderr patterns alone.
- `isError: true` on the `CallToolResult` signals to the MCP client that the tool call failed, while still delivering structured content the AI can read.

---

### Story 0.4: Tool Registry Pattern

**User Story:**
As a developer adding a new Obsidian tool,
I want a declarative registration pattern,
So that I can define a tool's schema, argument builder, and output transformer in one place without duplicating error handling or MCP boilerplate.

**Acceptance Criteria:**
- [ ] `ToolDefinition` interface is exported from `src/tools/registry.ts`
- [ ] `registerTools(server: McpServer, defs: ToolDefinition[]): void` is exported from `src/tools/registry.ts`
- [ ] Each registered tool appears in the MCP tool list with correct name, description, and input schema
- [ ] Tool handler calls `execObsidian(def.buildArgs(params))` and returns `formatMcpSuccess` or `formatMcpError`
- [ ] If `transformOutput` is defined on the definition, it is called on the raw stdout before returning
- [ ] If `format=json` is in the params and the output is not valid JSON, `PARSE_ERROR` is returned
- [ ] All 52 tools across all epics are registered by calling `registerTools` once per domain in `main.ts`
- [ ] Domain files export a `const tools: ToolDefinition[]` array — they do not call `registerTools` themselves

**ToolDefinition Interface:**

```typescript
// src/tools/registry.ts
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: {
    type: "object";
    properties: Record<string, unknown>;
    required?: string[];
  };
  buildArgs: (params: Record<string, unknown>) => string[];
  transformOutput?: (raw: string, params: Record<string, unknown>) => string;
}

export function registerTools(server: McpServer, defs: ToolDefinition[]): void {
  for (const def of defs) {
    server.tool(def.name, def.description, def.inputSchema.properties, async (params) => {
      const args = def.buildArgs(params);
      const result = await execObsidian(args);

      if (isError(result)) {
        return formatMcpError(result);
      }

      let output = result.stdout;

      if (params.format === "json") {
        try {
          JSON.parse(output); // validate
        } catch {
          return formatMcpError({
            error: "CLI returned malformed output that could not be parsed.",
            code: ErrorCode.PARSE_ERROR,
            detail: output,
          });
        }
      }

      if (def.transformOutput) {
        output = def.transformOutput(output, params);
      }

      return formatMcpSuccess(output);
    });
  }
}
```

**main.ts structure after all epics are complete:**

```typescript
import { registerTools } from "./tools/registry.js";
import { fileTools } from "./tools/files.js";
import { searchTools } from "./tools/search.js";
import { propertyTools } from "./tools/properties.js";
// ... etc

registerTools(server, fileTools);
registerTools(server, searchTools);
registerTools(server, propertyTools);
// ... etc
```

**Notes:**
- The `server.tool()` API from `@modelcontextprotocol/sdk` accepts the tool name, description, input schema properties, and a handler function. Check the SDK version in `package.json` for exact API surface — it may differ slightly between minor versions.
- Do not use `server.setRequestHandler` directly. Use the `server.tool()` convenience method.
- The `inputSchema.properties` passed to `server.tool` should be a Zod schema object or a plain JSON Schema object, depending on which SDK version is installed. Check the installed SDK API before implementing.

---

### Story 0.5: Global Flags

**User Story:**
As an AI assistant invoking any Obsidian tool,
I want consistent optional parameters available on every tool,
So that I can target specific vaults, request specific output formats, and control clipboard behavior without tool-specific documentation.

**Acceptance Criteria:**
- [ ] `vault?: string` parameter is present in every tool's input schema (optional on all tools)
- [ ] `copy?: boolean` parameter is present in every tool's input schema (optional on all tools)
- [ ] `format?: string` parameter (enum of valid values) is present on all tools that support output format selection
- [ ] `total?: boolean` parameter is present on all list-type tools
- [ ] `silent?: boolean` parameter is present on tools where silence is meaningful (`note_create`, `note_read`)
- [ ] When `vault` is provided, `vault=<name>` is appended to CLI args
- [ ] When `copy` is `true`, `copy` flag is appended to CLI args
- [ ] When `format` is provided, `format=<value>` is appended to CLI args
- [ ] When `total` is `true`, `total` flag is appended to CLI args
- [ ] Helper function `buildGlobalArgs(params)` is exported from `src/tools/registry.ts` for use in `buildArgs` implementations

**Global Flags Helper:**

```typescript
// src/tools/registry.ts

export function buildGlobalArgs(params: Record<string, unknown>): string[] {
  const args: string[] = [];
  if (typeof params.vault === "string" && params.vault) {
    args.push(`vault=${params.vault}`);
  }
  if (params.copy === true) {
    args.push("copy");
  }
  if (typeof params.format === "string" && params.format) {
    args.push(`format=${params.format}`);
  }
  if (params.total === true) {
    args.push("total");
  }
  if (params.silent === true) {
    args.push("silent");
  }
  return args;
}
```

**Global Schema Fragments:**

Define reusable schema fragments to avoid repetition across 52 tool definitions:

```typescript
// src/tools/registry.ts

export const globalSchemaProperties = {
  vault: {
    type: "string" as const,
    description: "Target vault name. Omit to use the default (most recently active) vault.",
  },
  copy: {
    type: "boolean" as const,
    description: "Copy tool output to clipboard.",
  },
} as const;

export const formatSchemaProperty = (values: string[]) => ({
  type: "string" as const,
  enum: values,
  description: `Output format. Options: ${values.join(", ")}.`,
});

export const totalSchemaProperty = {
  type: "boolean" as const,
  description: "Return only the count of results instead of the full list.",
} as const;

export const silentSchemaProperty = {
  type: "boolean" as const,
  description: "Execute without opening the note in Obsidian's UI.",
} as const;
```

**Usage in a tool definition:**

```typescript
// Example from files.ts
import { buildGlobalArgs, globalSchemaProperties, totalSchemaProperty } from "./registry.js";

{
  name: "files_list",
  description: "...",
  inputSchema: {
    type: "object",
    properties: {
      ...globalSchemaProperties,
    },
    required: [],
  },
  buildArgs: (params) => ["files", ...buildGlobalArgs(params)],
}
```

**Notes:**
- `vault` is the canonical way to target a non-default vault. If a user has a single vault, this parameter can always be omitted.
- `copy` is a pass-through to the Obsidian CLI's built-in clipboard feature. It does not affect what the MCP tool returns.
- The order of args matters: command first, then named parameters, then flags (e.g. `copy`, `silent`, `total`). Follow the pattern established in the CLI Mapping sections of each story.

---

## Verification Checklist

Before marking Epic 0 complete, verify all of the following:

- [ ] `npm run build` succeeds with zero TypeScript errors
- [ ] `node dist/main.js` starts without errors and accepts stdin without crashing
- [ ] Claude Desktop connects and sees the server (tool list may be empty)
- [ ] `execObsidian(["--version"])` returns stdout with Obsidian version string (when Obsidian is running)
- [ ] `execObsidian(["--version"])` returns `OBSIDIAN_NOT_RUNNING` when Obsidian is not running
- [ ] `execObsidian([])` with an invalid binary path returns `CLI_NOT_FOUND`
- [ ] All TypeScript types in `src/types.ts` are strict (no `any`)
- [ ] `src/tools/registry.ts` exports `ToolDefinition`, `registerTools`, `buildGlobalArgs`, and the schema fragment helpers

---

## File Checklist

| File | Created By This Epic |
|---|---|
| `src/main.ts` | Yes |
| `src/executor.ts` | Yes |
| `src/types.ts` | Yes |
| `src/tools/registry.ts` | Yes |
| `src/tools/files.ts` | Epic 1 |
| `src/tools/search.ts` | Epic 2 |
| `src/tools/properties.ts` | Epic 3 |
| `src/tools/links.ts` | Epic 4 |
| `src/tools/tags.ts` | Epic 5 |
| `src/tools/tasks.ts` | Epic 6 |
| `src/tools/daily.ts` | Epic 7 |
