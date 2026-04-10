---
date: 2026-03-28
type: task-worklog
task: claudeagentsdk-brq
title: "claude-agent-sdk-python — Skill Transfer (Lối tắt & Thực hành)"
status: open
detailed_at: 2026-03-28
detail_score: ready-for-dev
tags: [system-design, skill-transfer, exercises, claude-agent-sdk-python]
---

# claude-agent-sdk-python — Skill Transfer — Detailed Design

## 1. Objective
Tạo 3-5 mental shortcuts + 2-3 exercises cho SDK INTEGRATORS (developers dùng SDK trong app). Mỗi shortcut/exercise áp dụng nguyên lý cụ thể từ Task 4.

## 2. Scope
**In-scope:**
- Mental shortcuts for: entry point reading, pattern recognition, gotcha avoidance
- Hands-on exercises with CLI binary
- Thought experiment fallbacks (no CLI needed)
- Target: SDK integrators (import claude_agent_sdk, build apps)

**Out-of-scope:**
- SDK contributor guides (modifying SDK internals)
- CLI development guides
- Deployment/production guides

## 3. Input / Output
**Input:**
- Task 3 output: code tinh hoa snippets + file:line mappings
- Task 4 output: design principles + industry references
- SDK public API: query(), ClaudeSDKClient, @tool, types

**Output:**
- 3-5 mental shortcuts (each targeting different angle)
- 2-3 exercises (hands-on + thought experiment mode each)
- Worklog with copy-paste ready content

## 4. Dependencies
- claudeagentsdk-iaq (Task 3) + claudeagentsdk-ciw (Task 4) phải xong trước

## 5. Flow xử lý

### Step 1: Đọc Task 3 + Task 4 output (~10 phút)
Read worklogs → extract: code tinh hoa, design principles, key patterns.
**Verify:** Can list 3 principles and 3 code locations from memory

### Step 2: Design Mental Shortcuts (~15 phút)
3 shortcuts, each targeting DIFFERENT angle:

**Shortcut 1 — Entry Point (đọc gì trước):**
"Bắt đầu từ [`types.py`](../../src/claude_agent_sdk/types.py) — đây là spec sống. Mọi Message, ContentBlock, Option đều là dataclass. Hiểu types = hiểu API."
- File: [`types.py`](../../src/claude_agent_sdk/types.py)
- Why: 1203 LOC, mọi thứ khác reference nó

**Shortcut 2 — Pattern Recognition:**
"Tìm `request_id` trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py) — mỗi lần thấy nó là 1 kiểu message trong control protocol. Đếm = biết SDK nói gì với CLI."
- File: [`query.py`](../../src/claude_agent_sdk/_internal/query.py)
- Why: request_id pattern = map of all SDK↔CLI communication

**Shortcut 3 — Gotcha Avoidance:**
"Python keywords: `async_` không phải `async`, `continue_` không phải `continue`. SDK dùng trailing underscore rồi convert khi gửi CLI. Nếu hook output sai field name → silent failure."
- File: [`query.py:34-50`](../../src/claude_agent_sdk/_internal/query.py#L34-L50) (_convert_hook_output_for_cli)
- Why: Đây là gotcha #1 — IDE autocomplete sẽ gợi ý `async` (wrong), cần `async_` (right)

**Shortcut 4 (optional) — Architecture Mental Model:**
"SDK = Translator: Python async world ↔ CLI subprocess JSON world. Query class = translator. Transport = pipe. Types = dictionary."

**Shortcut 5 (optional) — Debug Strategy:**
"Khi debug SDK issues: check transport stderr first (CLI errors), then query.py message handling (protocol errors), then types.py (parse errors)."

**Verify:** 3+ shortcuts, each targets different angle, no overlap

### Step 3: Design Exercises (~20 phút)

**Exercise 1: Create custom SDK MCP tool (hands-on)**
- Principle applied: Open/Closed (add tools without modifying core)
- Task: Create a simple @tool that returns current timestamp
```python
from claude_agent_sdk import tool, create_sdk_mcp_server, query, ClaudeAgentOptions

@tool(description="Get current UTC timestamp")
def get_time() -> str:
    from datetime import datetime, timezone
    return datetime.now(timezone.utc).isoformat()

mcp_server = create_sdk_mcp_server([get_time])

async def main():
    async for msg in query(
        prompt="What time is it?",
        options=ClaudeAgentOptions(sdk_mcp_servers=[mcp_server]),
    ):
        print(msg)
```
- Verify criteria: Tool appears in response, timestamp is returned
- Estimated time: 30 min
- Thought experiment: "Without running code — which files would be involved? Trace @tool → create_sdk_mcp_server → Query intercepts call → executes in-process → returns result to CLI"

**Exercise 2: Write PreToolUse hook (hands-on)**
- Principle applied: Observer pattern
- Task: Log every tool use before execution
```python
from claude_agent_sdk import ClaudeSDKClient, ClaudeAgentOptions

async def log_tool_use(hook_input):
    print(f"Tool: {hook_input.tool_name}, Input: {hook_input.tool_input}")
    return {"continue_": True}  # Note: continue_, not continue!

async def main():
    options = ClaudeAgentOptions(
        hooks={"PreToolUse": log_tool_use},
    )
    async with ClaudeSDKClient(options) as client:
        async for msg in client.send_message("List files in current directory"):
            pass
```
- Verify criteria: Hook fires before each tool use, logs tool_name
- Gotcha test: What happens if you use `continue` instead of `continue_`? (KeyError or silent failure)
- Estimated time: 45 min
- Thought experiment: "Trace the hook lifecycle: CLI triggers PreToolUse → sends hook_callback request to SDK → Query dispatches to Python function → function returns → Query converts continue_ to continue → sends response back to CLI"

**Exercise 3: Thought Experiment — Error Handling**
- Principle applied: Exception hierarchy
- No CLI needed
- Question: "If the CLI binary is not found, which exception is thrown? Trace the code path: query() → InternalClient → Transport.connect() → _find_cli_binary() → CLINotFoundError. What would you catch in your app?"
- Expected answer: Catch `CLINotFoundError` (specific) or `ClaudeSDKError` (general)
- Follow-up: "What if CLI is found but crashes during execution? Which exception? → ProcessError"
- Estimated time: 15 min

**Verify:** 2+ exercises with hands-on mode, each has thought experiment fallback

## 6. Edge Cases & Error Handling
| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| No CLI binary available | Exercises can't run | Use thought experiment mode | Document expected behavior |
| SDK API changed | Exercise code outdated | Note SDK version in exercise | Pin to current version |
| User is contributor not integrator | Wrong audience | Note "for integrators" in header | Redirect to CONTRIBUTING.md |

## 7. Acceptance Criteria
- **Happy 1:** Given shortcuts designed, When reading, Then a new SDK user can navigate codebase faster
- **Happy 2:** Given exercises designed, When attempted, Then each has: description, code, verify criteria, time estimate, thought experiment fallback
- **Negative:** Given no CLI binary, When user attempts exercise, Then thought experiment mode provides equivalent learning

## 8. Technical Notes
- SDK version: check [`_version.py`](../../src/claude_agent_sdk/_version.py)
- @tool decorator: [`__init__.py`](../../src/claude_agent_sdk/__init__.py)
- Hook types: PreToolUse, PostToolUse, Stop, Notification, PermissionRequest (from types.py)
- Python keyword gotcha: async_, continue_ in hook outputs

## 9. Risks
- Exercises may break with SDK updates → pin version + note "tested with vX.Y.Z"
- Mental shortcuts may be too SDK-internals-focused for integrators → focus on PUBLIC API

## Phản biện (2026-03-28)

### Điểm chất lượng: 7/10 → fixed with audience + fallback
- Target: SDK integrators
- Each exercise: hands-on + thought experiment

## Worklog

### [START] Auto-claimed from Task 4 completion

### Mental Shortcuts (5)

**Shortcut 1 — Entry Point: "Đọc types.py trước"**
[`types.py`](../../src/claude_agent_sdk/types.py) (1203 LOC) là spec sống của SDK. `Message` union type ([line 950](../../src/claude_agent_sdk/types.py#L950)), [`ClaudeAgentOptions`](../../src/claude_agent_sdk/types.py#L1035) (line 1035, 35+ fields), mọi hook type, content block type đều ở đây. Hiểu [`types.py`](../../src/claude_agent_sdk/types.py) = hiểu 80% API surface. Đọc 10 phút, tiết kiệm hàng giờ đoán.

**Shortcut 2 — Pattern Recognition: "Đếm request_id trong query.py"**
Mỗi `request_id` usage trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py) = 1 kiểu message trong control protocol. Grep `request_id` → thấy tất cả cách SDK nói chuyện với CLI: initialize, hook_callback, can_use_tool, mcp_message, interrupt. Đếm = biết hết capabilities.

**Shortcut 3 — Gotcha: "async_ KHÔNG PHẢI async"**
Khi viết hook output, dùng `async_` (trailing underscore) và `continue_` — KHÔNG phải `async` và `continue` (Python keywords). SDK auto-convert qua [`_convert_hook_output_for_cli()`](../../src/claude_agent_sdk/_internal/query.py#L34-L50) (query.py:34-50). Sai tên field = silent failure, không có error. IDE autocomplete sẽ gợi ý sai.

**Shortcut 4 — Architecture Mental Model: "SDK = Translator"**
Python async world ↔ CLI subprocess JSON world. `Query` = translator. `Transport` = pipe. [`types.py`](../../src/claude_agent_sdk/types.py) = dictionary. Mọi thứ SDK làm = translate Python calls thành JSON stdin, translate JSON stdout thành Python objects.

**Shortcut 5 — Debug Strategy: "stderr → query.py → types.py"**
Khi debug: (1) check Transport stderr first (CLI errors, binary not found), (2) [`query.py`](../../src/claude_agent_sdk/_internal/query.py) _handle_control_request (protocol errors, wrong subtype), (3) [`message_parser.py`](../../src/claude_agent_sdk/_internal/message_parser.py)/[`types.py`](../../src/claude_agent_sdk/types.py) (parse errors, unknown message type). Follow the data flow.

### Exercises (3)

**Exercise 1: Create @tool SDK MCP tool (hands-on)**
Principle: Open/Closed — add tools without modifying core.
```python
from claude_agent_sdk import tool, create_sdk_mcp_server, query, ClaudeAgentOptions

@tool(name="get_time", description="Get current UTC time", input_schema={})
async def get_time(args):
    from datetime import datetime, timezone
    return {"content": [{"type": "text", "text": datetime.now(timezone.utc).isoformat()}]}

server = create_sdk_mcp_server([get_time])
options = ClaudeAgentOptions(
    mcp_servers={"time-server": server},
    permission_mode="acceptEdits",
)
async for msg in query(prompt="What time is it?", options=options):
    print(msg)
```
Verify: Response mentions current time. Tool appears in conversation.
Time: ~30 min.
Thought experiment: "Trace @tool → SdkMcpTool dataclass ([`__init__.py:100`](../../src/claude_agent_sdk/__init__.py#L100)) → [`create_sdk_mcp_server`](../../src/claude_agent_sdk/__init__.py) → mcp_servers option → Query._handle_sdk_mcp_request ([`query.py:394+`](../../src/claude_agent_sdk/_internal/query.py#L394)) → tool handler executes → result back to CLI. Which files touched? 3: [`__init__.py`](../../src/claude_agent_sdk/__init__.py), [`query.py`](../../src/claude_agent_sdk/_internal/query.py), [`types.py`](../../src/claude_agent_sdk/types.py)."

**Exercise 2: Write PreToolUse hook (hands-on)**
Principle: Observer pattern — intercept tool calls before execution.
```python
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import HookMatcher, HookCallback

async def log_tool(input_data, tool_use_id, context):
    print(f"Tool: {input_data.get('tool_name')}, Input: {input_data.get('tool_input')}")
    return {"continue_": True}  # NOTE: continue_ not continue!

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(callback=HookCallback(handler=log_tool))]},
    permission_mode="acceptEdits",
)
async for msg in query(prompt="List files in current directory", options=options):
    pass
```
Verify: Hook fires before each tool use, prints tool_name.
Gotcha test: Replace `continue_` with `continue` → SyntaxError (Python keyword).
Time: ~45 min.
Thought experiment: "CLI triggers PreToolUse → sends control_request with subtype=hook_callback → Query._handle_control_request ([`query.py:288`](../../src/claude_agent_sdk/_internal/query.py#L288)) → looks up hook_callbacks dict → calls log_tool() → _convert_hook_output_for_cli converts continue_ → continue → sends control_response back to CLI."

**Exercise 3: Error Hierarchy (thought experiment only)**
Principle: Exception hierarchy — catch specific before general.
Question: "If CLI binary is not found, which exception? Trace: query() → InternalClient → Transport.__init__ → _find_cli() → CLINotFoundError ([`subprocess_cli.py:88`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L88))."
Follow-up: "If CLI found but crashes during execution? → ProcessError ([`_errors.py:25`](../../src/claude_agent_sdk/_errors.py#L25))."
Follow-up: "If CLI sends malformed JSON? → CLIJSONDecodeError ([`_errors.py:42`](../../src/claude_agent_sdk/_errors.py#L42))."
Answer: In your app, catch `ClaudeSDKError` (base) to handle all, or specific subclasses for targeted handling.
Time: ~15 min.

### [COMPLETE] All AC met
- [x] 5 mental shortcuts (entry point, pattern recognition, gotcha, mental model, debug strategy)
- [x] 3 exercises (2 hands-on + thought experiment, 1 thought-only)
- [x] Each hands-on has thought experiment fallback
- [x] API signatures verified against [`types.py:1035`](../../src/claude_agent_sdk/types.py#L1035) (ClaudeAgentOptions) and [`types.py:491`](../../src/claude_agent_sdk/types.py#L491) (HookCallback)
