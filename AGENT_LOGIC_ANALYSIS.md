# Agent Logic Analysis

This document explains how the Codex agent works, including its operational logic, available tools, context gathering mechanisms, and token optimization strategies.

## 1. Agent Operational Logic

### High-Level Flow

The agent operates in a **conversation-based loop** where each turn involves:

1. **User Input Processing**: User messages are converted to `ResponseItem`s and recorded in conversation history
2. **Context Building**: The agent builds a prompt from conversation history
3. **Model Interaction**: The prompt is sent to the LLM (OpenAI API or similar)
4. **Response Processing**: The model's response is processed:
   - **Function/Tool Calls**: Executed and results fed back to the model
   - **Assistant Messages**: Recorded and the turn completes
5. **Token Management**: Token usage is tracked and auto-compaction may trigger
6. **Loop Continuation**: If tool calls were made, the loop continues with updated context

### Main Entry Points

- **`run_task()`** (`codex-rs/core/src/codex.rs:1731`): Main task execution loop
  - Processes user input
  - Manages the turn-by-turn conversation loop
  - Handles auto-compaction when token limits are reached
  - Tracks conversation state

- **`run_turn()`** (`codex-rs/core/src/codex.rs:1869`): Single turn execution
  - Builds the prompt from conversation history
  - Sends to model via streaming API
  - Processes tool calls and responses
  - Returns turn results

### Key Components

1. **Session** (`Session` struct): Manages conversation state, history, and services
2. **TurnContext**: Contains per-turn configuration (model, tools, working directory, etc.)
3. **ContextManager**: Maintains conversation history as `ResponseItem`s
4. **ToolRouter**: Routes tool calls to appropriate handlers
5. **ToolRegistry**: Maps tool names to handler implementations

### Turn Processing Flow

```
User Input
  ↓
record_input_and_rollout_usermsg() → Adds to history
  ↓
Loop:
  ├─ Build prompt from history (get_history_for_prompt())
  ├─ Create Prompt with tools, instructions, etc.
  ├─ Stream to model
  ├─ Process response:
  │   ├─ Tool calls → Execute → Add results to history
  │   └─ Messages → Record → Check if done
  ├─ Check token limits → Auto-compact if needed
  └─ Continue loop if tool calls were made
```

## 2. Available Tools

### Core Tools (Always Available)

1. **`shell`** / **`local_shell`** / **`container.exec`**
   - Executes shell commands
   - Parameters: `command` (array), `workdir`, `timeout_ms`, `with_escalated_permissions`, `justification`
   - Handler: `ShellHandler`

2. **`update_plan`**
   - Updates the step-by-step plan for the current task
   - Supports parallel tool calls
   - Handler: `PlanHandler`

3. **`list_mcp_resources`**
   - Lists resources from MCP servers
   - Parameters: `server` (optional), `cursor` (optional)
   - Supports parallel tool calls
   - Handler: `McpResourceHandler`

4. **`list_mcp_resource_templates`**
   - Lists parameterized resource templates from MCP servers
   - Parameters: `server` (optional), `cursor` (optional)
   - Supports parallel tool calls
   - Handler: `McpResourceHandler`

5. **`read_mcp_resource`**
   - Reads a specific resource from an MCP server
   - Parameters: `server`, `uri`
   - Supports parallel tool calls
   - Handler: `McpResourceHandler`

### Conditional Tools (Based on Configuration)

6. **`exec_command`** / **`write_stdin`** (UnifiedExec mode)
   - PTY-based command execution with interactive stdin
   - Parameters: `cmd`, `shell`, `login`, `yield_time_ms`, `max_output_tokens`
   - Handler: `UnifiedExecHandler`

7. **`apply_patch`** (if ApplyPatch feature enabled)
   - Applies code changes to files
   - Can be Freeform (text-based) or Function (JSON-based)
   - Handler: `ApplyPatchHandler`

8. **`grep_files`** (if in `experimental_supported_tools`)
   - Searches files by regex pattern
   - Parameters: `pattern`, `include` (glob), `path`, `limit` (default: 100)
   - Supports parallel tool calls
   - Handler: `GrepFilesHandler`

9. **`read_file`** (if in `experimental_supported_tools`)
   - Reads file contents with line numbers
   - Parameters: `file_path`, `offset`, `limit`, `mode` (slice/indentation), `indentation` (object)
   - Supports parallel tool calls
   - Handler: `ReadFileHandler`

10. **`list_dir`** (if in `experimental_supported_tools`)
    - Lists directory entries
    - Parameters: `dir_path`, `offset`, `limit`, `depth`
    - Supports parallel tool calls
    - Handler: `ListDirHandler`

11. **`view_image`** (if ViewImageTool feature enabled)
    - Attaches local images to conversation context
    - Parameters: `path`
    - Supports parallel tool calls
    - Handler: `ViewImageHandler`

12. **Web Search** (if WebSearchRequest feature enabled)
    - ToolSpec::WebSearch (no handler, handled by API)

13. **`test_sync_tool`** (if in `experimental_supported_tools`)
    - Internal synchronization helper for tests
    - Handler: `TestSyncHandler`

### MCP Tools (Dynamic)

- Tools from configured MCP servers are dynamically registered
- Fully-qualified names: `"<server>__<tool>"` (e.g., `"mcp__server_name__tool_name"`)
- Handler: `McpHandler`
- Tools are sorted by name and registered at session initialization

### Tool Registration

Tools are registered in `build_specs()` (`codex-rs/core/src/tools/spec.rs:856`):
- Tool specs define the API schema (JSON Schema)
- Handlers implement the execution logic
- Registry maps tool names to handlers
- Router selects tools based on configuration

## 3. Context Gathering

### Initial Context (`build_initial_context()`)

Every turn starts with initial context items:

1. **Developer Instructions** (if present)
   - From `turn_context.developer_instructions`
   - Wrapped in `DeveloperInstructions` response item

2. **User Instructions** (if present)
   - From `turn_context.user_instructions`
   - Includes working directory
   - Wrapped in `UserInstructions` response item

3. **Environment Context**
   - Current working directory
   - Approval policy
   - Sandbox policy
   - User shell configuration
   - Wrapped in `EnvironmentContext` response item

### Conversation History (`get_history_for_prompt()`)

The history is built from `ContextManager`:

1. **Normalization** (`normalize_history()`):
   - Ensures every function/tool call has a corresponding output
   - Removes orphan outputs (outputs without calls)
   - Processes items to truncate large outputs

2. **Processing** (`process_item()`):
   - **FunctionCallOutput**: Truncated using `format_output_for_model_body()`
   - **CustomToolCallOutput**: Truncated using `format_output_for_model_body()`
   - Other items passed through unchanged

3. **Ghost Snapshot Removal**:
   - Ghost snapshots (for git commits) are removed from prompt history
   - They remain in full history but not sent to model

### History Structure

History is stored as a `Vec<ResponseItem>` ordered from oldest to newest:

- **Message**: User/assistant messages
- **FunctionCall**: Tool invocation request
- **FunctionCallOutput**: Tool execution result
- **CustomToolCall**: Custom tool invocation
- **CustomToolCallOutput**: Custom tool result
- **LocalShellCall**: Shell command execution
- **Reasoning**: Model reasoning (if enabled)
- **WebSearchCall**: Web search request
- **GhostSnapshot**: Git commit snapshots (filtered from prompts)

### Context Building Process

```
build_initial_context()
  ↓
[DeveloperInstructions, UserInstructions, EnvironmentContext]
  ↓
get_history_for_prompt()
  ↓
normalize_history() → Ensure call/output pairs
  ↓
process_item() → Truncate large outputs
  ↓
remove_ghost_snapshots() → Filter git snapshots
  ↓
[Initial Context] + [History Items] → Prompt
```

## 4. Token Optimization Mechanisms

The agent uses several mechanisms to reduce token usage:

### 1. Output Truncation (`format_output_for_model_body()`)

**Location**: `codex-rs/core/src/context_manager/truncate.rs`

**Limits**:
- `MODEL_FORMAT_MAX_BYTES`: 10 KiB (10,240 bytes)
- `MODEL_FORMAT_MAX_LINES`: 256 lines
- `MODEL_FORMAT_HEAD_LINES`: 128 lines (first half)
- `MODEL_FORMAT_TAIL_LINES`: 128 lines (last half)
- `MODEL_FORMAT_HEAD_BYTES`: 5 KiB (first half)

**Strategy**: Head+tail truncation
- Shows first 128 lines (up to 5 KiB)
- Shows last 128 lines (up to remaining budget)
- Adds elision marker: `[... omitted N of M lines ...]` or `[... output truncated to fit N bytes ...]`
- Includes total line count: `Total output lines: M`

**Applied To**:
- Function call outputs (`FunctionCallOutput`)
- Custom tool call outputs (`CustomToolCallOutput`)

**Note**: Full output is still sent to clients via events; only the model receives truncated versions.

### 2. Global Function Output Truncation (`globally_truncate_function_output_items()`)

**Location**: `codex-rs/core/src/context_manager/truncate.rs`

**Limit**: 10 KiB total across all output items

**Strategy**: Sequential truncation
- Processes items in order
- Truncates text items to fit within 10 KiB budget
- Omits items that exceed budget
- Adds summary: `[omitted N text items ...]`

**Applied To**: `FunctionCallOutput.content_items` (structured content)

### 3. Auto-Compaction (`run_inline_auto_compact_task()`)

**Location**: `codex-rs/core/src/codex/compact.rs`

**Trigger**: When token usage exceeds `auto_compact_token_limit`
- Default: 90% of context window (e.g., 180k for 200k window)
- Configurable via `model_auto_compact_token_limit`

**Strategy**: Summarization
1. Collects all user messages from history
2. Sends to model with summarization prompt (`SUMMARIZATION_PROMPT`)
3. Model generates a summary of the conversation
4. Replaces history with:
   - Initial context (always preserved)
   - Recent user messages (up to `COMPACT_USER_MESSAGE_MAX_TOKENS * 4` bytes)
   - Summary message (model's summary)

**Process**:
```
Token limit reached
  ↓
Collect user messages
  ↓
Send to model with compact prompt
  ↓
Get summary
  ↓
build_compacted_history():
  - Initial context (preserved)
  - Recent user messages (truncated if needed)
  - Summary message
  ↓
Replace history
```

**Limits**:
- `COMPACT_USER_MESSAGE_MAX_TOKENS`: 20,000 tokens
- User messages truncated to `max_bytes` (default: 80,000 bytes)

**Fallback**: If compaction fails or still exceeds limit:
- Removes oldest history items one by one
- Retries compaction
- If still fails, shows error and stops

### 4. History Normalization

**Location**: `codex-rs/core/src/context_manager/normalize.rs`

**Purpose**: Ensures history invariants without duplicating data

**Actions**:
- Removes orphan outputs (outputs without corresponding calls)
- Ensures every call has an output (adds empty output if missing)

### 5. Tool Output Limits

Some tools have built-in limits:

- **`read_file`**: `limit` parameter (default: varies)
- **`grep_files`**: `limit` parameter (default: 100 files)
- **`list_dir`**: `limit` parameter (default: varies)
- **`exec_command`**: `max_output_tokens` parameter

### 6. Project Doc Truncation

**Location**: `codex-rs/core/src/project_doc.rs`

**Limit**: `project_doc_max_bytes` (configurable)

**Strategy**: Truncates project documentation files if they exceed the limit

## 5. How to Increase Quality (Disable Token Optimization)

Since you want to increase quality and don't care about cost, here are the mechanisms you can disable or adjust:

### 1. Disable Output Truncation

**File**: `codex-rs/core/src/context_manager/truncate.rs`

**Constants to increase**:
- `MODEL_FORMAT_MAX_BYTES`: Currently 10 KiB → Increase to very large value (e.g., 1 MiB)
- `MODEL_FORMAT_MAX_LINES`: Currently 256 → Increase to very large value (e.g., 100,000)

**Note**: This affects all tool outputs sent to the model.

### 2. Disable Auto-Compaction

**File**: `codex-rs/core/src/config/mod.rs` or `codex-rs/core/src/openai_model_info.rs`

**Options**:
- Set `model_auto_compact_token_limit` to `None` in config
- Or set it to a very high value (e.g., `i64::MAX`)
- Or modify `default_auto_compact_limit()` to return a higher percentage

**Config example**:
```toml
[model]
model_auto_compact_token_limit = null  # Disable
# OR
model_auto_compact_token_limit = 999999999  # Very high limit
```

### 3. Increase Tool Output Limits

**Files**: Tool handler files

**Parameters to increase**:
- `read_file`: Increase default `limit`
- `grep_files`: Increase default `limit` (currently 100)
- `list_dir`: Increase default `limit`
- `exec_command`: Increase or remove `max_output_tokens` limit

### 4. Increase Compact User Message Limit

**File**: `codex-rs/core/src/codex/compact.rs`

**Constant**: `COMPACT_USER_MESSAGE_MAX_TOKENS` (currently 20,000)

**Note**: This only affects compaction, not normal operation.

### 5. Disable Global Function Output Truncation

**File**: `codex-rs/core/src/context_manager/truncate.rs`

**Function**: `globally_truncate_function_output_items()`

**Option**: Increase `MODEL_FORMAT_MAX_BYTES` or modify the function to not truncate

### 6. Increase Project Doc Limit

**Config**: `project_doc_max_bytes`

**Example**:
```toml
project_doc_max_bytes = 10485760  # 10 MiB instead of default
```

## Summary

The agent operates as a conversation loop that:
1. Builds context from initial items + conversation history
2. Sends prompts to the LLM with available tools
3. Executes tool calls and feeds results back
4. Manages token usage through truncation and compaction

**Key Token Optimization Mechanisms**:
1. **Output truncation**: 10 KiB, 256 lines (head+tail)
2. **Auto-compaction**: At 90% of context window
3. **Tool limits**: Various per-tool limits
4. **Global truncation**: 10 KiB for structured outputs

**To maximize quality**: Increase or disable these limits in the files mentioned above.
