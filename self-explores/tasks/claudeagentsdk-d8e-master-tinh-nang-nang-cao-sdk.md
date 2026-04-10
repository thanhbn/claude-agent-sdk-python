---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-d8e
title: "[MAS-3] Master các tính năng nâng cao SDK"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
depends_on: [MAS-1]
blocks: [MAS-4]
tags: [mas, sdk-advanced, hooks, permissions, agents, dynamic-control, claude-agent-sdk-python]
---

# [MAS-3] Master các tính năng nâng cao SDK — Detailed Design

## 1. Objective
Nắm vững 4 tính năng THIẾT YẾU của claude-agent-sdk-python thông qua việc viết 4 example files chạy thực tế. Mỗi feature = 1 file hoàn chỉnh, có error handling, có output verification. Đây là prerequisite bắt buộc cho MAS-4 (Orchestrator) vì orchestrator sẽ kết hợp hooks + permissions + agents + dynamic control.

**4 Features:**
1. **Hooks** — PreToolUse + PostToolUse callbacks intercept tool execution lifecycle
2. **can_use_tool** — Permission callback gate mọi tool call
3. **AgentDefinition** — Khai báo sub-agents với tools, model, prompt riêng
4. **Dynamic control** — `client.set_model()` + `client.interrupt()` runtime

## 2. Scope
**In-scope:**
- 4 example files trong `examples/advanced/`
- Mỗi file: self-contained, chạy được với `python examples/advanced/<file>.py`
- Sử dụng REAL SDK imports: `claude_agent_sdk.types`, `claude_agent_sdk`
- Error handling cho từng feature (timeout, missing CLI, malformed output)
- Output verification: mỗi file print status/result để confirm thành công

**Out-of-scope:**
- Kết hợp nhiều features trong 1 file (đó là MAS-4)
- Production patterns (rate limiting, retry queues, circuit breakers)
- CLI internal behavior testing
- Performance benchmarking

## 3. Input / Output
**Input:**
- SDK source code: [`types.py`](../../src/claude_agent_sdk/types.py) (HookMatcher, HookCallback, HookInput, HookContext, HookJSONOutput, PreToolUseHookInput, PostToolUseHookInput, CanUseTool, ToolPermissionContext, PermissionResultAllow, PermissionResultDeny, AgentDefinition)
- SDK public API: `src/claude_agent_sdk/__init__.py` (@tool, create_sdk_mcp_server, query, ClaudeSDKClient, ClaudeAgentOptions)
- SDK client: [`client.py`](../../src/claude_agent_sdk/_internal/client.py) (set_model, set_permission_mode, interrupt, get_mcp_status)
- MAS learning guide: `self-explores/tasks/mas-learning-guide.md` (sections 4.1-4.4)
- Prior task outputs: claudeagentsdk-ciw (design principles), claudeagentsdk-brq (shortcuts/exercises)

**Output:**
- `examples/advanced/01_hooks_pretool_posttool.py` — Hook system demo
- `examples/advanced/02_permission_callback.py` — can_use_tool demo
- `examples/advanced/03_agent_definitions.py` — AgentDefinition demo
- `examples/advanced/04_dynamic_control.py` — set_model + interrupt demo
- Worklog update with results per file

## 4. Dependencies
- **MAS-1** (SDK basics: query + ClaudeSDKClient + @tool) phải xong trước — cần hiểu basic flows trước khi layer advanced features
- **Blocks MAS-4** — Orchestrator cần tất cả 4 features này

## 5. Flow xu ly

### Step 1: Verify SDK types availability (~5 phut)
Confirm all required types exist in [`types.py`](../../src/claude_agent_sdk/types.py) and are exported from `__init__.py`:

```bash
# Verify hook types
grep -n "class HookMatcher\|class HookCallback\|HookInput\|HookContext\|HookJSONOutput" src/claude_agent_sdk/types.py
grep -n "PreToolUseHookInput\|PostToolUseHookInput" src/claude_agent_sdk/types.py

# Verify permission types
grep -n "CanUseTool\|ToolPermissionContext\|PermissionResultAllow\|PermissionResultDeny" src/claude_agent_sdk/types.py

# Verify agent types
grep -n "class AgentDefinition" src/claude_agent_sdk/types.py

# Verify dynamic control methods
grep -n "def set_model\|def set_permission_mode\|def interrupt\|def get_mcp_status" src/claude_agent_sdk/client.py
```

**Verify:** All 4 feature areas have corresponding types/methods in SDK

### Step 2: Write Example 1 — Hooks (PreToolUse + PostToolUse) (~20 phut)

**File:** `examples/advanced/01_hooks_pretool_posttool.py`

**Design:**
- PreToolUse hook: log tool name + input BEFORE execution, optionally block dangerous commands
- PostToolUse hook: log tool name + output AFTER execution, measure duration
- Both hooks registered via HookMatcher in ClaudeAgentOptions.hooks dict
- Demo prompt triggers 2-3 tool uses to show hooks firing

```python
"""
Example: Hooks — PreToolUse + PostToolUse

Demonstrates:
- Registering PreToolUse hook to intercept tool calls before execution
- Registering PostToolUse hook to process results after execution
- Hook callback signature: async (input, tool_use_id, context) -> HookJSONOutput
- GOTCHA: Use continue_ (not continue) and async_ (not async) — Python keywords
"""
import anyio
import time
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import (
    HookMatcher, HookCallback,
    HookInput, HookContext, HookJSONOutput,
    PreToolUseHookInput, PostToolUseHookInput,
    AssistantMessage, ResultMessage, TextBlock, ToolUseBlock,
)

# Track timing for PostToolUse
_tool_start_times: dict[str, float] = {}

async def pre_tool_use_hook(
    input: HookInput,
    tool_use_id: str | None,
    context: HookContext,
) -> HookJSONOutput:
    """Fires BEFORE every tool execution."""
    tool_name = input.get("tool_name", "unknown")
    tool_input = input.get("tool_input", {})

    print(f"  [PRE]  Tool: {tool_name}")
    print(f"         Input: {str(tool_input)[:120]}...")

    # Track start time for duration measurement
    if tool_use_id:
        _tool_start_times[tool_use_id] = time.monotonic()

    # Block dangerous commands (example safety gate)
    if tool_name == "Bash":
        cmd = tool_input.get("command", "")
        if "rm -rf" in cmd or "sudo" in cmd:
            return {
                "decision": "block",
                "reason": f"Blocked dangerous command: {cmd[:50]}",
            }

    # Allow everything else — MUST use continue_ (trailing underscore)
    return {"continue_": True}


async def post_tool_use_hook(
    input: HookInput,
    tool_use_id: str | None,
    context: HookContext,
) -> HookJSONOutput:
    """Fires AFTER every tool execution."""
    tool_name = input.get("tool_name", "unknown")

    # Measure duration
    duration_ms = 0
    if tool_use_id and tool_use_id in _tool_start_times:
        duration_ms = (time.monotonic() - _tool_start_times.pop(tool_use_id)) * 1000

    print(f"  [POST] Tool: {tool_name} completed in {duration_ms:.0f}ms")

    return {"continue_": True}


async def main():
    options = ClaudeAgentOptions(
        hooks={
            "PreToolUse": [
                HookMatcher(
                    matcher=None,  # None = match ALL tools
                    hooks=[pre_tool_use_hook],
                    timeout=30.0,
                ),
            ],
            "PostToolUse": [
                HookMatcher(
                    matcher=None,
                    hooks=[post_tool_use_hook],
                    timeout=30.0,
                ),
            ],
        },
        permission_mode="acceptEdits",
        max_turns=5,
        max_budget_usd=0.10,
    )

    print("=== Hook Demo: PreToolUse + PostToolUse ===\n")

    async for msg in query(
        prompt="List the Python files in the current directory using Bash, then read the first 5 lines of any .py file you find.",
        options=options,
    ):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    print(f"\n[Claude] {block.text[:200]}")
        elif isinstance(msg, ResultMessage):
            print(f"\n=== Done. Cost: ${msg.total_cost_usd:.4f}, Turns: {msg.num_turns} ===")

if __name__ == "__main__":
    anyio.run(main)
```

**Verify:**
- PreToolUse prints BEFORE each tool execution
- PostToolUse prints AFTER with duration
- `continue_` used (not `continue`)
- Dangerous command blocking logic present

### Step 3: Write Example 2 — Permission Callback (can_use_tool) (~20 phut)

**File:** `examples/advanced/02_permission_callback.py`

**Design:**
- Implement `can_use_tool` callback that gates ALL tool usage
- Whitelist: Read, Glob, Grep always allowed
- Blacklist: Write, Edit to system paths denied
- Bash: inspect command content, deny dangerous patterns
- Show both Allow and Deny paths with logging

```python
"""
Example: Permission Callback (can_use_tool)

Demonstrates:
- Implementing can_use_tool to gate every tool call
- PermissionResultAllow vs PermissionResultDeny
- Path-based and command-based permission logic
- Callback signature: async (tool_name, tool_input, context) -> PermissionResult
"""
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import (
    ToolPermissionContext,
    PermissionResultAllow,
    PermissionResultDeny,
    AssistantMessage, ResultMessage, TextBlock,
)

# Counters for verification
_allowed_count = 0
_denied_count = 0

SAFE_READ_TOOLS = {"Read", "Glob", "Grep"}
BLOCKED_PATHS = {"/etc/", "/usr/", "/sys/", "/boot/"}
DANGEROUS_COMMANDS = {"rm -rf", "sudo ", "chmod 777", "mkfs", "> /dev/"}


async def permission_handler(
    tool_name: str,
    tool_input: dict,
    context: ToolPermissionContext,
) -> PermissionResultAllow | PermissionResultDeny:
    """Gate every tool call. Returns Allow or Deny."""
    global _allowed_count, _denied_count

    # Read-only tools: always allow
    if tool_name in SAFE_READ_TOOLS:
        _allowed_count += 1
        print(f"  [ALLOW] {tool_name}")
        return PermissionResultAllow()

    # Bash: inspect command content
    if tool_name == "Bash":
        cmd = tool_input.get("command", "")
        for danger in DANGEROUS_COMMANDS:
            if danger in cmd:
                _denied_count += 1
                print(f"  [DENY]  Bash: dangerous pattern '{danger}' in: {cmd[:60]}")
                return PermissionResultDeny(
                    message=f"Command contains blocked pattern: {danger}",
                    interrupt=False,  # False = Claude tries alternative, True = stop conversation
                )
        _allowed_count += 1
        print(f"  [ALLOW] Bash: {cmd[:60]}")
        return PermissionResultAllow()

    # Write/Edit: check target path
    if tool_name in {"Write", "Edit"}:
        path = tool_input.get("file_path", "")
        for blocked in BLOCKED_PATHS:
            if path.startswith(blocked):
                _denied_count += 1
                print(f"  [DENY]  {tool_name}: blocked path {path}")
                return PermissionResultDeny(message=f"Cannot write to {path}")
        _allowed_count += 1
        print(f"  [ALLOW] {tool_name}: {path}")
        return PermissionResultAllow()

    # Default: allow unknown tools
    _allowed_count += 1
    print(f"  [ALLOW] {tool_name} (default)")
    return PermissionResultAllow()


async def main():
    options = ClaudeAgentOptions(
        can_use_tool=permission_handler,
        max_turns=5,
        max_budget_usd=0.10,
    )

    print("=== Permission Callback Demo ===\n")

    async for msg in query(
        prompt="Read the current directory listing, then try to read /etc/passwd contents.",
        options=options,
    ):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    print(f"\n[Claude] {block.text[:200]}")
        elif isinstance(msg, ResultMessage):
            print(f"\n=== Done. Allowed: {_allowed_count}, Denied: {_denied_count} ===")
            print(f"    Cost: ${msg.total_cost_usd:.4f}, Turns: {msg.num_turns}")

if __name__ == "__main__":
    anyio.run(main)
```

**Verify:**
- Both Allow and Deny paths exercised
- Counters track permission decisions
- `interrupt=False` used for soft deny (Claude retries)
- Path-based and command-based checks present

### Step 4: Write Example 3 — AgentDefinition (~20 phut)

**File:** `examples/advanced/03_agent_definitions.py`

**Design:**
- Define 2 agents: code-reviewer (read-only, sonnet) and doc-writer (read+write, haiku)
- Use ClaudeSDKClient with agents dict in options
- Prompt orchestrates between agents naturally
- Show model and tools constraints per agent

```python
"""
Example: Agent Definitions

Demonstrates:
- Defining specialist agents with AgentDefinition
- Each agent has: description, prompt, tools, model
- Model options: "sonnet", "opus", "haiku", "inherit"
- Claude automatically routes sub-tasks to defined agents
"""
import anyio
import json
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
from claude_agent_sdk.types import (
    AgentDefinition,
    AssistantMessage, ResultMessage, TextBlock, ToolUseBlock,
)


async def main():
    options = ClaudeAgentOptions(
        agents={
            "code-reviewer": AgentDefinition(
                description="Reviews Python code for bugs, security issues, and style problems",
                prompt=(
                    "You are a senior Python code reviewer. "
                    "Focus on: type safety, error handling gaps, security vulnerabilities, "
                    "and Pythonic idioms. Be concise — bullet points, not paragraphs."
                ),
                tools=["Read", "Glob", "Grep"],  # Read-only: cannot modify code
                model="sonnet",
            ),
            "doc-writer": AgentDefinition(
                description="Writes concise documentation and docstrings for Python code",
                prompt=(
                    "You are a technical documentation writer. "
                    "Write clear, Google-style docstrings and README sections. "
                    "Include type hints in examples. Keep it under 200 words per item."
                ),
                tools=["Read", "Glob"],  # Read-only for this demo
                model="haiku",
            ),
        },
        permission_mode="acceptEdits",
        max_turns=10,
        max_budget_usd=0.20,
    )

    print("=== Agent Definitions Demo ===")
    print("  Agents: code-reviewer (sonnet, read-only), doc-writer (haiku, read-only)\n")

    async with ClaudeSDKClient(options=options) as client:
        async for msg in client.send_message(
            "Review the file src/claude_agent_sdk/_errors.py for code quality, "
            "then write a brief docstring summary for each error class found."
        ):
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(f"[Agent] {block.text[:300]}")
                    elif isinstance(block, ToolUseBlock):
                        print(f"  >> Tool: {block.name}({json.dumps(block.input)[:80]}...)")
            elif isinstance(msg, ResultMessage):
                print(f"\n=== Done. Cost: ${msg.total_cost_usd:.4f}, Turns: {msg.num_turns} ===")

if __name__ == "__main__":
    anyio.run(main)
```

**Key points:**
- `AgentDefinition.model` constrains which model the sub-agent uses
- `AgentDefinition.tools` constrains which tools the sub-agent can access
- The main Claude instance routes sub-tasks to agents based on `description`

**Verify:**
- 2 agents defined with different tools/models
- Read-only constraint enforced (no Write/Edit in tools list)
- model field uses SDK-valid values ("sonnet", "haiku")

### Step 5: Write Example 4 — Dynamic Control (~20 phut)

**File:** `examples/advanced/04_dynamic_control.py`

**Design:**
- Start with sonnet model, switch to opus mid-conversation via `client.set_model()`
- Use `client.interrupt()` to stop a long-running task
- Check MCP status with `client.get_mcp_status()`
- Show ClaudeSDKClient as async context manager for stateful session

```python
"""
Example: Dynamic Control — set_model + interrupt

Demonstrates:
- Changing model mid-conversation with client.set_model()
- Interrupting a running task with client.interrupt()
- Checking MCP server status with client.get_mcp_status()
- ClaudeSDKClient as async context manager for stateful sessions
"""
import anyio
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions
from claude_agent_sdk.types import (
    AssistantMessage, ResultMessage, TextBlock, ToolUseBlock, RateLimitEvent,
)


async def main():
    options = ClaudeAgentOptions(
        permission_mode="acceptEdits",
        max_turns=15,
        max_budget_usd=0.30,
    )

    print("=== Dynamic Control Demo ===\n")

    async with ClaudeSDKClient(options=options) as client:
        # --- Phase 1: Quick task with default model ---
        print("[Phase 1] Default model — quick task")
        turn_count = 0
        async for msg in client.send_message(
            "What is 2 + 2? Answer in one word."
        ):
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(f"  Response: {block.text}")
            elif isinstance(msg, ResultMessage):
                print(f"  Cost so far: ${msg.total_cost_usd:.4f}")
                break

        # --- Phase 2: Switch model mid-conversation ---
        print("\n[Phase 2] Switching model to opus...")
        await client.set_model("claude-opus-4-6")
        print("  Model switched.")

        async for msg in client.send_message(
            "Now explain the significance of that number in mathematics, briefly."
        ):
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(f"  Opus says: {block.text[:200]}")
            elif isinstance(msg, ResultMessage):
                print(f"  Cost so far: ${msg.total_cost_usd:.4f}")
                break

        # --- Phase 3: Interrupt demo ---
        print("\n[Phase 3] Starting long task, will interrupt after 3 messages...")
        msg_count = 0
        async for msg in client.send_message(
            "List every file in the src/ directory recursively with descriptions."
        ):
            if isinstance(msg, AssistantMessage):
                msg_count += 1
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(f"  [{msg_count}] {block.text[:100]}...")
                if msg_count >= 3:
                    print("\n  >>> Interrupting! <<<")
                    await client.interrupt()
                    break
            elif isinstance(msg, ResultMessage):
                break

        # --- Phase 4: Check MCP status ---
        print("\n[Phase 4] Checking MCP server status...")
        try:
            status = await client.get_mcp_status()
            servers = status.get("mcpServers", [])
            if servers:
                for srv in servers:
                    print(f"  MCP: {srv.get('name', '?')} — {srv.get('status', '?')}")
            else:
                print("  No MCP servers configured (expected for this demo)")
        except Exception as e:
            print(f"  MCP status check: {e}")

        print("\n=== Dynamic Control Demo Complete ===")

if __name__ == "__main__":
    anyio.run(main)
```

**Verify:**
- `set_model()` called between turns (not during streaming)
- `interrupt()` called during active streaming
- `get_mcp_status()` called for status check
- Multi-turn conversation maintains state across model switch

### Step 6: Verify all 4 examples (~10 phut)

```bash
# Check files exist
ls -la examples/advanced/

# Syntax check (no runtime — just parse)
python -c "import ast; ast.parse(open('examples/advanced/01_hooks_pretool_posttool.py').read())"
python -c "import ast; ast.parse(open('examples/advanced/02_permission_callback.py').read())"
python -c "import ast; ast.parse(open('examples/advanced/03_agent_definitions.py').read())"
python -c "import ast; ast.parse(open('examples/advanced/04_dynamic_control.py').read())"

# If CLI is available, run each:
python examples/advanced/01_hooks_pretool_posttool.py
python examples/advanced/02_permission_callback.py
python examples/advanced/03_agent_definitions.py
python examples/advanced/04_dynamic_control.py
```

**Verify per file:**
| File | Key Output | Success Criteria |
|------|-----------|------------------|
| 01_hooks | `[PRE]` and `[POST]` lines printed | Both hooks fire for every tool call |
| 02_permission | `[ALLOW]` and `[DENY]` lines | Both paths exercised, counters accurate |
| 03_agents | Tool calls with agent routing | 2 agents used, different tools per agent |
| 04_dynamic | Model switch + interrupt | set_model succeeds, interrupt stops stream |

## 6. Edge Cases & Error Handling

| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| CLI binary not found | No claude CLI installed | `CLINotFoundError` raised | Catch and print install instructions |
| Hook callback raises exception | Bug in hook function | SDK catches, logs, continues | Wrap hook body in try/except |
| Hook returns wrong field name | `continue` instead of `continue_` | `SyntaxError` at Python level | Document: always use trailing underscore |
| Permission callback timeout | Slow external service check | SDK may timeout the callback | Set reasonable `timeout` in HookMatcher |
| set_model with invalid model | Typo in model name | CLI returns error | Catch and log, fallback to previous model |
| interrupt() when no task running | Called between turns | No-op or error | Guard with try/except |
| AgentDefinition with empty tools | `tools=[]` | Agent has no tool access | Validate tools list before passing |
| can_use_tool returns None | Forgot to return | TypeError | Always return Allow or Deny explicitly |
| RateLimitEvent during streaming | High usage | Message stream includes rate limit | Check `isinstance(msg, RateLimitEvent)` |
| PostToolUse with tool failure | Tool raises exception | PostToolUseFailure hook fires instead | Register both PostToolUse and PostToolUseFailure |

## 7. Acceptance Criteria

### Happy Path
- **AC-1:** Given `01_hooks_pretool_posttool.py`, When run with CLI available, Then PreToolUse fires before EVERY tool call AND PostToolUse fires after EVERY tool call (verified by `[PRE]`/`[POST]` console output)
- **AC-2:** Given `02_permission_callback.py`, When run, Then at least 1 tool call is ALLOWED and at least 1 is DENIED (verified by counters: `_allowed_count > 0` AND `_denied_count > 0`)
- **AC-3:** Given `03_agent_definitions.py`, When run, Then 2 AgentDefinitions are configured with different model/tools constraints (verified by options dict inspection)
- **AC-4:** Given `04_dynamic_control.py`, When run, Then model is switched mid-conversation AND interrupt stops active streaming (verified by phase outputs)

### Negative Path
- **AC-5:** Given hook returns `{"continue": True}` (missing underscore), When Python parses, Then SyntaxError is raised — example code MUST use `continue_`
- **AC-6:** Given CLI binary is missing, When any example runs, Then `CLINotFoundError` is raised with install instructions (not a cryptic subprocess error)

### Integration
- **AC-7:** Given all 4 examples pass syntax check (`ast.parse`), When files are created, Then each file is self-contained (no cross-file imports between examples)

## 8. Technical Notes

### Hook Callback Signature (from types.py)
```python
# Callback signature
async def hook(
    input: HookInput,        # Dict with tool_name, tool_input, etc.
    tool_use_id: str | None, # Unique ID for this tool invocation
    context: HookContext,     # Session context (conversation_id, etc.)
) -> HookJSONOutput           # Dict with continue_, decision, reason, etc.
```

### Hook Output Wire Format Conversion
```python
# Python (what you write)          →  Wire (what CLI receives)
{"continue_": True}                →  {"continue": true}
{"async_": True}                   →  {"async": true}
{"decision": "block", "reason": X} →  {"decision": "block", "reason": X}
```
Conversion happens in `_convert_hook_output_for_cli()` at [`query.py:34-50`](../../src/claude_agent_sdk/_internal/query.py#L34-L50).

### Permission Callback Signature (from types.py)
```python
async def perm(
    tool_name: str,                    # "Bash", "Read", "Write", etc.
    tool_input: dict,                  # Tool arguments
    context: ToolPermissionContext,     # Session context
) -> PermissionResultAllow | PermissionResultDeny
```

### AgentDefinition Fields (from types.py)
```python
@dataclass
class AgentDefinition:
    description: str          # What this agent does (for routing)
    prompt: str               # System prompt for this agent
    tools: list[str]          # Tool names this agent can use
    model: str = "inherit"    # "sonnet" | "opus" | "haiku" | "inherit"
```

### Dynamic Control Methods (from client.py)
```python
await client.set_model("claude-opus-4-6")          # Switch model
await client.set_permission_mode("bypassPermissions")  # Change permissions
await client.interrupt()                             # Stop current task
status = await client.get_mcp_status()               # Check MCP servers
await client.reconnect_mcp_server("server-name")     # Reconnect failed MCP
await client.toggle_mcp_server("name", enabled=False) # Toggle MCP on/off
```

### SDK MCP Tool Naming Convention
When tools are registered via MCP servers, Claude sees them as: `mcp__{server_name}__{tool_name}`
Example: server "calc" with tool "add" becomes `mcp__calc__add` in allowed_tools.

## 9. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| CLI not installed on dev machine | Medium | Blocks all examples | Document install: `npm install -g @anthropic-ai/claude-code` |
| Hook callback API changed since v0.1.48 | Low | Examples break | Pin to SDK version, verify against types.py before running |
| AgentDefinition not working in current CLI version | Medium | Example 03 may fail | Test with `max_turns=1` first to verify agent registration |
| `interrupt()` timing race condition | Medium | Interrupt may miss or double-fire | Add small delay after interrupt, handle gracefully |
| Budget overrun during testing | Low | Unexpected costs | Set `max_budget_usd` low on all examples ($0.10-$0.30) |
| Permission callback not called for allowed_tools | Medium | Example 02 shows no denies | Don't use allowed_tools in permission example |

## Worklog

### [PENDING] Awaiting MAS-1 completion
