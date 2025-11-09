# Tools Reference

Quick reference for all available tools in the Codex agent.

## Core Tools (Always Available)

| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `shell` / `local_shell` / `container.exec` | Execute shell commands | `command` (array), `workdir`, `timeout_ms`, `with_escalated_permissions`, `justification` | No |
| `update_plan` | Update step-by-step task plan | Plan-specific | Yes |
| `list_mcp_resources` | List MCP server resources | `server` (opt), `cursor` (opt) | Yes |
| `list_mcp_resource_templates` | List MCP resource templates | `server` (opt), `cursor` (opt) | Yes |
| `read_mcp_resource` | Read specific MCP resource | `server`, `uri` | Yes |

## Conditional Tools

### UnifiedExec Mode
| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `exec_command` | PTY-based command execution | `cmd`, `shell`, `login`, `yield_time_ms`, `max_output_tokens` | No |
| `write_stdin` | Write to unified exec session | `session_id`, `chars` | No |

### ApplyPatch Feature
| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `apply_patch` | Apply code changes | Freeform text or JSON | No |

### Experimental Tools (`experimental_supported_tools`)
| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `grep_files` | Search files by regex | `pattern`, `include` (glob), `path`, `limit` (default: 100) | Yes |
| `read_file` | Read file with line numbers | `file_path`, `offset`, `limit`, `mode` (slice/indentation), `indentation` (object) | Yes |
| `list_dir` | List directory entries | `dir_path`, `offset`, `limit`, `depth` | Yes |
| `test_sync_tool` | Test synchronization helper | `sleep_before_ms`, `sleep_after_ms`, `barrier` | Yes |

### ViewImage Feature
| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `view_image` | Attach local image to context | `path` | Yes |

### WebSearch Feature
| Tool Name | Description | Parameters | Parallel? |
|-----------|-------------|------------|-----------|
| `web_search` | Web search (handled by API) | API-specific | No |

## MCP Tools

- Dynamically registered from configured MCP servers
- Fully-qualified name format: `"<server>__<tool>"` (e.g., `"mcp__server_name__tool_name"`)
- Tools are sorted alphabetically by name
- All MCP tools use `McpHandler`

## Tool Registration

Tools are registered in `codex-rs/core/src/tools/spec.rs` in the `build_specs()` function:

1. **Shell tools**: Based on `config.shell_type` (Default/Local/UnifiedExec)
2. **MCP tools**: From `mcp_connection_manager.list_all_tools()`
3. **Feature flags**: Check `features.enabled(Feature::*)`
4. **Experimental tools**: Check `config.experimental_supported_tools`

## Tool Handlers

| Handler | Tools |
|---------|-------|
| `ShellHandler` | `shell`, `local_shell`, `container.exec` |
| `UnifiedExecHandler` | `exec_command`, `write_stdin` |
| `PlanHandler` | `update_plan` |
| `ApplyPatchHandler` | `apply_patch` |
| `GrepFilesHandler` | `grep_files` |
| `ReadFileHandler` | `read_file` |
| `ListDirHandler` | `list_dir` |
| `ViewImageHandler` | `view_image` |
| `McpHandler` | All MCP tools |
| `McpResourceHandler` | `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` |
| `TestSyncHandler` | `test_sync_tool` |

## Tool Execution Flow

```
Model requests tool call
  ↓
ToolRouter routes to handler
  ↓
ToolRegistry.dispatch()
  ↓
Handler.handle() → ToolOutput
  ↓
ToolOutput → ResponseInputItem
  ↓
Added to conversation history
  ↓
Sent back to model in next turn
```

## Parallel Tool Calls

Tools marked with "Parallel?" = Yes can be executed concurrently when:
- Model supports parallel tool calls (`model_family.supports_parallel_tool_calls`)
- Multiple tool calls are requested in the same turn
- Tool is registered with `push_spec_with_parallel_support()`

Parallel execution is handled by `ToolCallRuntime` in `codex-rs/core/src/tools/parallel.rs`.
