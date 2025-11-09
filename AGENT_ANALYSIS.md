# Agent Operational Logic Analysis

## 1. Agent Working Logic

### High-Level Flow

The agent operates as a **conversation loop** where:

1. **User submits input** → `Op::UserTurn` is sent to the agent
2. **Agent processes turn** → `run_task()` is called which:
   - Records user input into conversation history
   - Builds a prompt from conversation history
   - Sends prompt to the model (via `run_turn()`)
   - Model responds with either:
     - **Tool calls** (function calls, custom tool calls, shell commands)
     - **Assistant messages** (text responses)
   - If tool calls are made:
     - Tools are executed via `ToolRouter` and `ToolRegistry`
     - Tool outputs are recorded back into history
     - Loop continues with updated history
   - If only assistant message:
     - Task completes, loop exits

### Key Components

- **`Codex`**: Main entry point, manages submission queue and event stream
- **`Session`**: Manages conversation state, history, and services
- **`TurnContext`**: Context for a single turn (model, tools config, cwd, etc.)
- **`ContextManager`**: Manages conversation history (items stored chronologically)
- **`ToolRouter`**: Routes tool calls to appropriate handlers
- **`ToolRegistry`**: Maps tool names to handler implementations

### Turn Processing Flow

```
User Input
  ↓
run_task()
  ↓
Loop:
  ├─ Build prompt from history (get_history_for_prompt())
  ├─ run_turn() → sends to model
  ├─ Model responds with items
  ├─ Process items:
  │   ├─ Tool calls → execute via ToolRouter
  │   ├─ Tool outputs → record in history
  │   └─ Messages → record in history
  └─ If no more tool calls → break (task complete)
```

## 2. List of All Tools

### Built-in Tools

1. **`shell`** / **`local_shell`** / **`container.exec`**
   - Executes shell commands
   - Handler: `ShellHandler`
   - Supports parallel: No

2. **`unified_exec`** / **`exec_command`** / **`write_stdin`**
   - Unified execution system (alternative to shell)
   - Handler: `UnifiedExecHandler`
   - Supports parallel: No

3. **`apply_patch`**
   - Applies code patches (freeform or JSON format)
   - Handler: `ApplyPatchHandler`
   - Supports parallel: No

4. **`update_plan`**
   - Updates step-by-step plan for the task
   - Handler: `PlanHandler`
   - Supports parallel: No

5. **`grep_files`** (experimental)
   - Searches for patterns across files
   - Handler: `GrepFilesHandler`
   - Supports parallel: **Yes**

6. **`read_file`** (experimental)
   - Reads file contents
   - Handler: `ReadFileHandler`
   - Supports parallel: **Yes**

7. **`list_dir`** (experimental)
   - Lists directory contents
   - Handler: `ListDirHandler`
   - Supports parallel: **Yes**

8. **`test_sync_tool`** (experimental)
   - Synchronous test execution
   - Handler: `TestSyncHandler`
   - Supports parallel: **Yes**

9. **`view_image`**
   - Views image files
   - Handler: `ViewImageHandler`
   - Supports parallel: **Yes**

10. **`web_search`**
    - Performs web searches
    - Handler: Built-in (no custom handler)

### MCP (Model Context Protocol) Tools

- **`list_mcp_resources`** - Lists available MCP resources
- **`list_mcp_resource_templates`** - Lists MCP resource templates
- **`read_mcp_resource`** - Reads MCP resources
- **Custom MCP tools** - Dynamically loaded from configured MCP servers
  - Fully qualified names: `{server}__{tool_name}`
  - Handler: `McpHandler` or `McpResourceHandler`
  - Supports parallel: Varies by tool

### Tool Registration

Tools are registered in `codex-rs/core/src/tools/spec.rs` in the `build_specs()` function. The tool registry maps tool names to handler implementations.

## 3. Context Gathering

### How Context is Gathered

1. **Initial Context** (`build_initial_context()`):
   - System instructions (base instructions, user instructions, developer instructions)
   - Environment context (cwd, git info, project docs)
   - Session configuration metadata

2. **Conversation History** (`ContextManager`):
   - All messages, tool calls, and tool outputs are stored chronologically
   - History is retrieved via `get_history_for_prompt()`
   - Items are normalized to ensure:
     - Every tool call has a corresponding output
     - Every output has a corresponding call

3. **History Processing**:
   - Items are filtered (ghost snapshots removed for prompts)
   - Tool outputs are truncated before being sent to model (see token optimization)
   - History is ordered oldest → newest

### Context Structure

```rust
ContextManager {
    items: Vec<ResponseItem>,  // Chronological conversation items
    token_info: Option<TokenUsageInfo>,  // Token usage tracking
}
```

Items include:
- `Message` (user/assistant messages)
- `FunctionCall` / `FunctionCallOutput`
- `CustomToolCall` / `CustomToolCallOutput`
- `LocalShellCall`
- `WebSearchCall`
- `Reasoning` (model reasoning)
- `GhostSnapshot` (filtered out for prompts)

## 4. Token Optimization Mechanisms

### Current Token-Saving Mechanisms

The system has **several mechanisms** to reduce token usage:

#### A. Tool Output Truncation

**Location**: `codex-rs/core/src/context_manager/truncate.rs`

**Limits**:
- `MODEL_FORMAT_MAX_BYTES = 10 KiB` (10,240 bytes)
- `MODEL_FORMAT_MAX_LINES = 256 lines`
- `MODEL_FORMAT_HEAD_LINES = 128` (first half)
- `MODEL_FORMAT_TAIL_LINES = 128` (last half)

**How it works**:
- Tool outputs are truncated using **head+tail** strategy
- Shows first 128 lines + last 128 lines
- Middle content is replaced with: `[... omitted N of M lines ...]`
- If content exceeds 10 KiB, also truncates by bytes
- **Note**: Full output is still available to clients; only model sees truncated version

**Functions**:
- `format_output_for_model_body()` - Truncates single output string
- `globally_truncate_function_output_items()` - Truncates content items array

#### B. Conversation Compaction

**Location**: `codex-rs/core/src/codex/compact.rs`

**How it works**:
- When token usage exceeds `auto_compact_token_limit`, automatically triggers
- Uses model to summarize conversation history
- Replaces old history with:
  - Initial context (system instructions, etc.)
  - Recent user messages (last N messages, up to 80K tokens)
  - Summary message from model

**Limits**:
- `COMPACT_USER_MESSAGE_MAX_TOKENS = 20,000` per message
- Total user messages limit: `20,000 * 4 = 80,000 tokens`

**Auto-compact trigger**:
- Default limit: `context_window * 0.9` (90% of context window)
- Configurable via `model_auto_compact_token_limit` in config
- Model-specific defaults in `openai_model_info.rs`

#### C. History Item Removal

**Location**: `codex-rs/core/src/context_manager/history.rs`

- If context window is exceeded during compaction, oldest items are removed
- `remove_first_item()` removes oldest conversation item
- Ensures prompt fits within context window

#### D. User Message Truncation in Compaction

**Location**: `codex-rs/core/src/codex/compact.rs::build_compacted_history_with_limit()`

- When building compacted history, user messages are truncated if they exceed the limit
- Uses `truncate_middle()` to preserve beginning and end

### Token Counting

- Token usage is tracked via `TokenUsageInfo`
- Uses tokenizer (o200k_base, falls back to cl100k_base, then 4-bytes-per-token estimate)
- Tracks: input tokens, output tokens, total tokens in context window

## 5. How to Increase Quality (Disable Token Optimization)

Since you want to **increase quality and don't care about token cost**, here are the mechanisms to disable/modify:

### A. Disable Tool Output Truncation

**File**: `codex-rs/core/src/context_manager/truncate.rs`

**Current limits**:
```rust
pub(crate) const MODEL_FORMAT_MAX_BYTES: usize = 10 * 1024; // 10 KiB
pub(crate) const MODEL_FORMAT_MAX_LINES: usize = 256;
```

**To disable**: Increase these constants significantly or remove truncation logic:
- Set `MODEL_FORMAT_MAX_BYTES` to a very large value (e.g., `usize::MAX`)
- Set `MODEL_FORMAT_MAX_LINES` to a very large value (e.g., `usize::MAX`)

**Note**: The truncation happens in `ContextManager::process_item()` which is called when items are recorded.

### B. Disable Auto-Compaction

**File**: `codex-rs/core/src/config/mod.rs` or via config file

**Current behavior**: Auto-compacts when token usage exceeds limit

**To disable**:
- Set `model_auto_compact_token_limit` to `None` or a very high value
- Or modify `codex-rs/core/src/openai_model_info.rs` to set `auto_compact_token_limit: None`

### C. Increase Compaction Limits

**File**: `codex-rs/core/src/codex/compact.rs`

**Current limits**:
```rust
const COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000;
// Used as: COMPACT_USER_MESSAGE_MAX_TOKENS * 4 = 80,000 tokens
```

**To increase**: Raise these values to preserve more history during compaction

### D. Remove History Item Removal

**File**: `codex-rs/core/src/codex/compact.rs::run_compact_task_inner()`

The code removes oldest items if context window is exceeded. To disable, you'd need to modify the error handling to fail instead of removing items.

### E. Project Documentation Truncation

**Location**: `codex-rs/core/src/project_doc.rs`

**How it works**:
- `AGENTS.md` files are read and included in initial context
- Total size is limited by `project_doc_max_bytes` config
- Files are truncated if they exceed the remaining budget
- Default limit: `32 KiB` (32,768 bytes) - defined in `config/mod.rs`

**To increase**: Set `project_doc_max_bytes` to a higher value in config, or modify `PROJECT_DOC_MAX_BYTES` constant

### Summary of Token Optimization Points

1. ✅ **Tool output truncation** - `context_manager/truncate.rs` (10 KiB, 256 lines)
2. ✅ **Auto-compaction** - `codex/compact.rs` (triggers at 90% of context window)
3. ✅ **User message truncation in compaction** - `codex/compact.rs` (20K tokens per message)
4. ✅ **History item removal** - `codex/compact.rs` (removes oldest if still too large)
5. ✅ **Project documentation truncation** - `project_doc.rs` (limited by `project_doc_max_bytes`)

All of these can be adjusted or disabled to maximize context quality at the cost of higher token usage.
