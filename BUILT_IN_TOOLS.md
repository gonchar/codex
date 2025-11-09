# Complete List of Built-in Tools

## Always Available Tools

### 1. **shell** (and aliases: `local_shell`, `container.exec`)
- **Type**: Function tool
- **Description**: Runs a shell command and returns its output
- **Handler**: `ShellHandler`
- **Supports Parallel**: No
- **Parameters**:
  - `command` (required): Array of strings - The command to execute
  - `workdir` (optional): String - The working directory to execute the command in
  - `timeout_ms` (optional): Number - The timeout for the command in milliseconds
  - `with_escalated_permissions` (optional): Boolean - Whether to request escalated permissions
  - `justification` (optional): String - 1-sentence explanation if escalated permissions needed
- **Availability**: Always available (default shell tool)

### 2. **update_plan**
- **Type**: Function tool
- **Description**: Updates the task plan. Provides an optional explanation and a list of plan items, each with a step and status. At most one step can be in_progress at a time.
- **Handler**: `PlanHandler`
- **Supports Parallel**: No
- **Parameters**:
  - `plan` (required): Array of plan items, each with:
    - `step` (required): String
    - `status` (required): String - One of: "pending", "in_progress", "completed"
  - `explanation` (optional): String
- **Availability**: Always available

### 3. **list_mcp_resources**
- **Type**: Function tool
- **Description**: Lists resources provided by MCP servers. Resources allow servers to share data that provides context to language models, such as files, database schemas, or application-specific information. Prefer resources over web search when possible.
- **Handler**: `McpResourceHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `server` (optional): String - Optional MCP server name. When omitted, lists resources from every configured server
  - `cursor` (optional): String - Opaque cursor returned by a previous list_mcp_resources call for the same server
- **Availability**: Always available

### 4. **list_mcp_resource_templates**
- **Type**: Function tool
- **Description**: Lists resource templates provided by MCP servers. Parameterized resource templates allow servers to share data that takes parameters and provides context to language models, such as files, database schemas, or application-specific information. Prefer resource templates over web search when possible.
- **Handler**: `McpResourceHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `server` (optional): String - Optional MCP server name. When omitted, lists resource templates from all configured servers
  - `cursor` (optional): String - Opaque cursor returned by a previous list_mcp_resource_templates call for the same server
- **Availability**: Always available

### 5. **read_mcp_resource**
- **Type**: Function tool
- **Description**: Reads a resource provided by an MCP server. The server name must match exactly as configured and must match the 'server' field returned by list_mcp_resources.
- **Handler**: `McpResourceHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `server` (required): String - MCP server name exactly as configured
  - `uri` (required): String - Resource URI exactly as returned by list_mcp_resources
- **Availability**: Always available

## Conditional Tools (Based on Configuration)

### 6. **local_shell** (as primary tool)
- **Type**: LocalShell tool
- **Description**: Alternative shell execution method
- **Handler**: `ShellHandler` (shared with `shell`)
- **Supports Parallel**: No
- **Availability**: Only when `shell_type` config is set to `Local`

### 7. **exec_command** (UnifiedExec mode)
- **Type**: Function tool
- **Description**: Runs a command in a PTY, returning output or a session ID for ongoing interaction
- **Handler**: `UnifiedExecHandler`
- **Supports Parallel**: No
- **Parameters**:
  - `cmd` (required): String - Shell command to execute
  - `shell` (optional): String - Shell binary to launch. Defaults to /bin/bash
  - `login` (optional): Boolean - Whether to run the shell with -l/-i semantics. Defaults to true
  - `yield_time_ms` (optional): Number - How long to wait (in milliseconds) for output before yielding
  - `max_output_tokens` (optional): Number - Maximum number of tokens to return. Excess output will be truncated
- **Availability**: Only when `shell_type` config is set to `UnifiedExec`

### 8. **write_stdin** (UnifiedExec mode)
- **Type**: Function tool
- **Description**: Writes characters to an existing unified exec session and returns recent output
- **Handler**: `UnifiedExecHandler`
- **Supports Parallel**: No
- **Parameters**:
  - `session_id` (required): Number - Identifier of the running unified exec session
  - `chars` (optional): String - Bytes to write to stdin (may be empty to poll)
  - `yield_time_ms` (optional): Number - How long to wait (in milliseconds) for output before yielding
  - `max_output_tokens` (optional): Number - Maximum number of tokens to return. Excess output will be truncated
- **Availability**: Only when `shell_type` config is set to `UnifiedExec`

### 9. **apply_patch**
- **Type**: Freeform tool (default) or Function tool (JSON mode)
- **Description**: Edits files using a patch format. Can be freeform (grammar-based) or JSON format
- **Handler**: `ApplyPatchHandler`
- **Supports Parallel**: No
- **Parameters** (Freeform mode):
  - Uses Lark grammar format for patch syntax
- **Parameters** (JSON mode):
  - `input` (required): String - The entire contents of the apply_patch command
- **Availability**: Only when `apply_patch_tool_type` config is set (Freeform or Function)

### 10. **web_search**
- **Type**: WebSearch tool
- **Description**: Performs web searches
- **Handler**: Built-in (no custom handler)
- **Supports Parallel**: Unknown
- **Availability**: Only when `web_search_request` config is enabled

### 11. **view_image**
- **Type**: Function tool
- **Description**: Attach a local image (by filesystem path) to the conversation context for this turn
- **Handler**: `ViewImageHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `path` (required): String - Local filesystem path to an image file
- **Availability**: Only when `include_view_image_tool` config is enabled

## Experimental Tools (Require Explicit Enablement)

### 12. **grep_files**
- **Type**: Function tool
- **Description**: Finds files whose contents match the pattern and lists them by modification time
- **Handler**: `GrepFilesHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `pattern` (required): String - Regular expression pattern to search for
  - `include` (optional): String - Optional glob that limits which files are searched (e.g. "*.rs" or "*.{ts,tsx}")
  - `path` (optional): String - Directory or file path to search. Defaults to the session's working directory
  - `limit` (optional): Number - Maximum number of file paths to return (defaults to 100)
- **Availability**: Only when `experimental_supported_tools` contains `"grep_files"`

### 13. **read_file**
- **Type**: Function tool
- **Description**: Reads a local file with 1-indexed line numbers, supporting slice and indentation-aware block modes
- **Handler**: `ReadFileHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `file_path` (required): String - Absolute path to the file
  - `offset` (optional): Number - The line number to start reading from. Must be 1 or greater
  - `limit` (optional): Number - The maximum number of lines to return
  - `mode` (optional): String - Optional mode selector: "slice" for simple ranges (default) or "indentation" to expand around an anchor line
  - `indentation` (optional): Object - When mode is "indentation", contains:
    - `anchor_line` (optional): Number - Anchor line to center the indentation lookup on (defaults to offset)
    - `max_levels` (optional): Number - How many parent indentation levels (smaller indents) to include
    - `include_siblings` (optional): Boolean - When true, include additional blocks that share the anchor indentation
    - `include_header` (optional): Boolean - Include doc comments or attributes directly above the selected block
    - `max_lines` (optional): Number - Hard cap on the number of lines returned when using indentation mode
- **Availability**: Only when `experimental_supported_tools` contains `"read_file"`

### 14. **list_dir**
- **Type**: Function tool
- **Description**: Lists entries in a local directory with 1-indexed entry numbers and simple type labels
- **Handler**: `ListDirHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `dir_path` (required): String - Absolute path to the directory to list
  - `offset` (optional): Number - The entry number to start listing from. Must be 1 or greater
  - `limit` (optional): Number - The maximum number of entries to return
  - `depth` (optional): Number - The maximum directory depth to traverse. Must be 1 or greater
- **Availability**: Only when `experimental_supported_tools` contains `"list_dir"`

### 15. **test_sync_tool**
- **Type**: Function tool
- **Description**: Internal synchronization helper used by Codex integration tests
- **Handler**: `TestSyncHandler`
- **Supports Parallel**: Yes
- **Parameters**:
  - `sleep_before_ms` (optional): Number - Optional delay in milliseconds before any other action
  - `sleep_after_ms` (optional): Number - Optional delay in milliseconds after completing the barrier
  - `barrier` (optional): Object - Synchronization barrier:
    - `id` (required): String - Identifier shared by concurrent calls that should rendezvous
    - `participants` (required): Number - Number of tool calls that must arrive before the barrier opens
    - `timeout_ms` (optional): Number - Maximum time in milliseconds to wait at the barrier
- **Availability**: Only when `experimental_supported_tools` contains `"test_sync_tool"`

## Dynamic Tools (MCP)

### 16. **MCP Tools** (Dynamic)
- **Type**: Function tools (converted from MCP tools)
- **Description**: Tools provided by configured MCP (Model Context Protocol) servers
- **Handler**: `McpHandler`
- **Supports Parallel**: Varies by tool
- **Naming**: Fully qualified as `{server}__{tool_name}` (e.g., `my_server__my_tool`)
- **Parameters**: Defined by each MCP server
- **Availability**: Dynamically loaded from configured MCP servers

## Tool Aliases

The following tool names are aliases that map to the same handler:

- `shell` → `ShellHandler`
- `container.exec` → `ShellHandler` (legacy compatibility)
- `local_shell` → `ShellHandler` (when not used as primary tool type)

## Summary

**Total Built-in Tools**: 15 core tools + dynamic MCP tools

**By Category**:
- **Always Available**: 5 tools (shell, update_plan, 3 MCP resource tools)
- **Conditional**: 6 tools (based on config flags)
- **Experimental**: 4 tools (require explicit enablement)
- **Dynamic**: MCP tools (loaded from configured servers)

**Parallel Execution Support**:
- **Supports Parallel**: 8 tools (list_mcp_resources, list_mcp_resource_templates, read_mcp_resource, view_image, grep_files, read_file, list_dir, test_sync_tool)
- **No Parallel Support**: 7 tools (shell variants, exec_command, write_stdin, apply_patch, update_plan, web_search)

## Notes

1. **Tool Availability**: Tools are registered in `codex-rs/core/src/tools/spec.rs` in the `build_specs()` function
2. **Handler Registration**: Handlers are registered separately from tool specs, allowing multiple tool names to share the same handler
3. **MCP Tools**: MCP tools are dynamically converted from MCP protocol format to OpenAI function format
4. **Experimental Tools**: Must be explicitly enabled in config via `experimental_supported_tools` array
5. **Shell Tool Variants**: The shell tool has multiple variants depending on configuration, but all use the same `ShellHandler`
