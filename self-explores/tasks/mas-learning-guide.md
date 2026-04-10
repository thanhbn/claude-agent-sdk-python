---
date: 2026-03-29
type: task-worklog
title: "MAS Learning Guide - Toàn diện với claude-agent-sdk-python"
status: in_progress
tags: [mas, multi-agent, sdk, learning, guide]
---

# Hướng dẫn Toàn diện: Xây dựng Multi-Agent System với claude-agent-sdk-python

> Tài liệu này được biên soạn từ việc đọc TOÀN BỘ source code, examples, tests, và documentation của SDK.
> Mọi code snippet đều đã verify với source code thực tế.

---

## Mục lục

1. [Tổng quan SDK Architecture](#1-tổng-quan-sdk-architecture)
2. [Mastering @tool Decorator](#2-mastering-tool-decorator)
3. [Wrap Function/Serverless thành Tools](#3-wrap-functionserverless-thành-tools)
4. [Các tính năng nâng cao](#4-các-tính-năng-nâng-cao)
5. [Simple MAS: Orchestrator Pattern](#5-simple-mas-orchestrator-pattern)
6. [Complex MAS: Full-layer Architecture](#6-complex-mas-full-layer-architecture)
7. [Tài liệu & Best Practices](#7-tài-liệu--best-practices)

---

## 1. Tổng quan SDK Architecture

### 1.1 Hai entry point chính

```
┌─────────────────────────────────────────────────────┐
│                   PUBLIC API                         │
│                                                     │
│   query()              ClaudeSDKClient              │
│   (One-shot,           (Bidirectional,              │
│    stateless)           stateful sessions)           │
│                                                     │
├─────────────────────────────────────────────────────┤
│                 INTERNAL LAYERS                      │
│                                                     │
│   InternalClient → Query → SubprocessCLITransport   │
│                    ↕                                │
│              MCP Servers (in-process)                │
│              Hook Callbacks                          │
│              Permission Callbacks                    │
│              Message Parser                          │
└─────────────────────────────────────────────────────┘
```

**Khi nào dùng `query()`:** Batch processing, automation, câu hỏi đơn giản, không cần multi-turn.

**Khi nào dùng `ClaudeSDKClient`:** Multi-turn conversations, interactive sessions, cần interrupt, cần dynamic control (đổi model, permission giữa chừng), MAS.

### 1.2 Message Types (quan trọng phải nhớ)

```python
from claude_agent_sdk import (
    query, ClaudeSDKClient,
    # Message types
    UserMessage, AssistantMessage, SystemMessage, ResultMessage,
    # Content blocks
    TextBlock, ToolUseBlock, ToolResultBlock, ThinkingBlock,
    # MCP & Tools
    tool, create_sdk_mcp_server,
    # Types
    ClaudeAgentOptions, AgentDefinition, HookMatcher,
)
```

---

## 2. Mastering @tool Decorator

### 2.1 Cú pháp cơ bản

```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool(
    name="tool_name",           # Tên tool (unique trong server)
    description="What it does", # Mô tả cho Claude hiểu khi nào dùng
    input_schema={              # Schema cho input (dict hoặc type)
        "param1": str,
        "param2": int,
    },
)
async def my_tool(args):
    """Handler nhận args dict, trả về MCP result dict."""
    result = do_something(args["param1"], args["param2"])
    return {
        "content": [{"type": "text", "text": str(result)}]
    }
```

### 2.2 Input Schema patterns

**Pattern 1: Dict đơn giản (khuyên dùng cho bắt đầu)**
```python
@tool("greet", "Greet a user", {"name": str})
async def greet(args):
    return {"content": [{"type": "text", "text": f"Hello {args['name']}!"}]}
```

**Pattern 2: Dict phức tạp với nested types**
```python
@tool("search", "Search documents", {
    "query": str,
    "max_results": int,
    "filters": dict,  # nested object
})
async def search(args):
    query = args["query"]
    max_results = args.get("max_results", 10)
    filters = args.get("filters", {})
    # ... process
    return {"content": [{"type": "text", "text": json.dumps(results)}]}
```

**Pattern 3: Full JSON Schema (cho production)**
```python
@tool("analyze", "Analyze data", {
    "type": "object",
    "properties": {
        "data_path": {"type": "string", "description": "Path to data file"},
        "method": {"type": "string", "enum": ["mean", "median", "mode"]},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
    },
    "required": ["data_path", "method"],
})
async def analyze(args):
    # args already validated against schema
    ...
```

### 2.3 Error handling trong tools

```python
@tool("divide", "Divide two numbers", {"a": float, "b": float})
async def divide(args):
    if args["b"] == 0:
        return {
            "content": [{"type": "text", "text": "Error: Division by zero"}],
            "is_error": True,  # Claude biết đây là lỗi, sẽ xử lý khác
        }
    result = args["a"] / args["b"]
    return {"content": [{"type": "text", "text": str(result)}]}
```

### 2.4 Tool Annotations (metadata cho Claude)

```python
from claude_agent_sdk.types import ToolAnnotations

@tool(
    "delete_file",
    "Delete a file from disk",
    {"path": str},
    annotations=ToolAnnotations(
        title="File Deletion Tool",
        readOnlyHint=False,           # Tool có side effects
        destructiveHint=True,         # Tool phá hủy dữ liệu
        idempotentHint=True,          # Gọi nhiều lần kết quả giống nhau
        openWorldHint=False,          # Không truy cập external services
    ),
)
async def delete_file(args):
    ...
```

### 2.5 Tạo MCP Server và đăng ký vào SDK

```python
import anyio
from claude_agent_sdk import (
    tool, create_sdk_mcp_server, ClaudeSDKClient, ClaudeAgentOptions,
    AssistantMessage, TextBlock, ToolUseBlock, ResultMessage,
)

# --- Định nghĩa tools ---
@tool("add", "Add two numbers", {"a": float, "b": float})
async def add(args):
    return {"content": [{"type": "text", "text": str(args["a"] + args["b"])}]}

@tool("multiply", "Multiply two numbers", {"a": float, "b": float})
async def multiply(args):
    return {"content": [{"type": "text", "text": str(args["a"] * args["b"])}]}

# --- Tạo server ---
calc_server = create_sdk_mcp_server(
    name="calculator",
    version="1.0.0",
    tools=[add, multiply],
)

# --- Dùng trong client ---
async def main():
    options = ClaudeAgentOptions(
        mcp_servers={"calc": calc_server},
        # Pre-approve tools để Claude không cần hỏi permission
        allowed_tools=["mcp__calc__add", "mcp__calc__multiply"],
        # Hoặc dùng pattern: "mcp__calc__*" (nếu CLI hỗ trợ)
    )

    async with ClaudeSDKClient(options=options) as client:
        await client.query("What is 15 + 27, then multiply the result by 3?")
        async for msg in client.receive_response():
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(block.text)
                    elif isinstance(block, ToolUseBlock):
                        print(f"  [Tool: {block.name}({block.input})]")

anyio.run(main)
```

**QUAN TRỌNG:** Format tên tool khi dùng MCP: `mcp__{server_name}__{tool_name}`

---

## 3. Wrap Function/Serverless thành Tools

### 3.1 Pattern: Wrap existing sync function

```python
import asyncio
from claude_agent_sdk import tool

# Function gốc (sync, legacy code)
def calculate_tax(income: float, rate: float) -> float:
    return income * rate / 100

# Wrap thành MCP tool
@tool("calculate_tax", "Calculate tax amount", {"income": float, "rate": float})
async def tax_tool(args):
    # Chạy sync function trong thread pool
    result = await asyncio.to_thread(
        calculate_tax, args["income"], args["rate"]
    )
    return {"content": [{"type": "text", "text": f"Tax: ${result:.2f}"}]}
```

### 3.2 Pattern: Wrap async function

```python
# Function gốc (async)
async def fetch_weather(city: str) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.weather.com/{city}")
        return resp.json()

# Wrap thành MCP tool
@tool("get_weather", "Get current weather for a city", {"city": str})
async def weather_tool(args):
    try:
        data = await fetch_weather(args["city"])
        return {"content": [{"type": "text", "text": json.dumps(data)}]}
    except Exception as e:
        return {"content": [{"type": "text", "text": f"Error: {e}"}], "is_error": True}
```

### 3.3 Pattern: Wrap FastAPI endpoint logic

```python
# FastAPI endpoint logic (existing)
async def create_user(name: str, email: str) -> dict:
    user = await db.users.create(name=name, email=email)
    return {"id": user.id, "name": user.name, "email": user.email}

# Wrap thành MCP tool
@tool("create_user", "Create a new user in the system", {
    "name": str, "email": str,
})
async def create_user_tool(args):
    try:
        result = await create_user(args["name"], args["email"])
        return {"content": [{"type": "text", "text": json.dumps(result)}]}
    except Exception as e:
        return {"content": [{"type": "text", "text": f"Failed: {e}"}], "is_error": True}
```

### 3.4 Pattern: Generic wrapper factory

```python
from claude_agent_sdk import tool, SdkMcpTool
from typing import Callable, Any
import inspect

def wrap_function(
    name: str,
    description: str,
    func: Callable,
    schema: dict | None = None,
) -> SdkMcpTool:
    """Factory tự động wrap bất kỳ function nào thành MCP tool."""

    # Auto-generate schema từ type hints nếu không cung cấp
    if schema is None:
        hints = func.__annotations__
        schema = {k: v for k, v in hints.items() if k != "return"}

    @tool(name, description, schema)
    async def wrapped(args):
        try:
            if inspect.iscoroutinefunction(func):
                result = await func(**args)
            else:
                import asyncio
                result = await asyncio.to_thread(func, **args)
            return {"content": [{"type": "text", "text": str(result)}]}
        except Exception as e:
            return {"content": [{"type": "text", "text": f"Error: {e}"}], "is_error": True}

    return wrapped

# Sử dụng:
def my_legacy_func(x: int, y: int) -> int:
    return x ** y

power_tool = wrap_function("power", "Calculate x^y", my_legacy_func)
```

### 3.5 Pattern: Wrap serverless/Lambda handler

```python
# AWS Lambda handler (existing)
def lambda_handler(event, context):
    action = event["action"]
    if action == "process":
        return {"statusCode": 200, "body": "processed"}
    return {"statusCode": 400, "body": "unknown action"}

# Wrap thành MCP tool
@tool("invoke_lambda", "Invoke serverless function", {
    "action": str,
    "payload": dict,
})
async def lambda_tool(args):
    import asyncio
    # Simulate lambda invocation
    event = {"action": args["action"], **args.get("payload", {})}
    result = await asyncio.to_thread(lambda_handler, event, None)
    return {
        "content": [{"type": "text", "text": json.dumps(result)}],
        "is_error": result.get("statusCode", 200) >= 400,
    }
```

---

## 4. Các tính năng nâng cao

### 4.1 Hooks System (10 events)

```python
from claude_agent_sdk import ClaudeAgentOptions, HookMatcher
from claude_agent_sdk.types import (
    PreToolUseHookInput, PostToolUseHookInput, StopHookInput,
    HookInput, HookContext, HookJSONOutput,
)

# Hook chặn command nguy hiểm
async def safety_hook(
    input: HookInput,
    tool_use_id: str | None,
    context: HookContext,
) -> HookJSONOutput:
    if isinstance(input, PreToolUseHookInput):
        cmd = input.get("tool_input", {}).get("command", "")
        if "rm -rf" in cmd or "sudo" in cmd:
            return {
                "decision": "block",
                "reason": "Dangerous command blocked",
            }
        # Allow by default
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow",
            }
        }
    return {"continue_": True}

# Hook thêm context khi tool xong
async def logging_hook(
    input: HookInput,
    tool_use_id: str | None,
    context: HookContext,
) -> HookJSONOutput:
    if isinstance(input, PostToolUseHookInput):
        tool_name = input.get("tool_name", "")
        print(f"[LOG] Tool {tool_name} completed")
    return {"continue_": True}

# Đăng ký hooks
options = ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Bash", hooks=[safety_hook], timeout=30.0),
        ],
        "PostToolUse": [
            HookMatcher(matcher=None, hooks=[logging_hook]),  # matcher=None = all tools
        ],
        "Stop": [
            HookMatcher(hooks=[lambda i, t, c: {"continue_": True}]),
        ],
    }
)
```

**10 Hook Events:**
| Event | Khi nào | Dùng để |
|-------|---------|---------|
| `PreToolUse` | Trước khi dùng tool | Block, modify input, add context |
| `PostToolUse` | Sau khi tool xong | Log, validate output, add feedback |
| `PostToolUseFailure` | Tool bị lỗi | Error handling, retry logic |
| `UserPromptSubmit` | User gửi prompt | Validate, enrich prompt |
| `Stop` | Claude muốn dừng | Force continue, custom stop logic |
| `SubagentStop` | Sub-agent xong | Collect results, orchestrate |
| `SubagentStart` | Sub-agent bắt đầu | Track, limit parallelism |
| `PreCompact` | Trước khi compact context | Add important context |
| `Notification` | Claude gửi notification | Forward to Slack/email |
| `PermissionRequest` | Cần permission | Custom permission logic |

### 4.2 Permission Callbacks (can_use_tool)

```python
from claude_agent_sdk.types import (
    ToolPermissionContext, PermissionResultAllow, PermissionResultDeny,
)

async def my_permission_handler(
    tool_name: str,
    tool_input: dict,
    context: ToolPermissionContext,
) -> PermissionResultAllow | PermissionResultDeny:
    # Read-only tools: always allow
    if tool_name in ["Read", "Glob", "Grep", "Bash"]:
        if tool_name == "Bash":
            cmd = tool_input.get("command", "")
            if any(danger in cmd for danger in ["rm", "sudo", "chmod"]):
                return PermissionResultDeny(
                    message="Dangerous command blocked",
                    interrupt=False,  # True = stop entire conversation
                )
        return PermissionResultAllow()

    # Write tools: redirect to safe directory
    if tool_name in ["Write", "Edit"]:
        path = tool_input.get("file_path", "")
        if path.startswith("/etc/") or path.startswith("/usr/"):
            return PermissionResultDeny(message=f"Cannot write to {path}")
        return PermissionResultAllow()

    # Allow with modified input
    return PermissionResultAllow(
        updated_input=tool_input,  # Optional: modify the input
    )

options = ClaudeAgentOptions(
    can_use_tool=my_permission_handler,
)
```

### 4.3 Agent Definitions

```python
from claude_agent_sdk import ClaudeAgentOptions, AgentDefinition

options = ClaudeAgentOptions(
    agents={
        "code-reviewer": AgentDefinition(
            description="Reviews code for bugs, security, and performance",
            prompt="You are a senior code reviewer. Focus on: security vulnerabilities, performance bottlenecks, and code smells.",
            tools=["Read", "Glob", "Grep"],  # Read-only tools
            model="sonnet",
        ),
        "doc-writer": AgentDefinition(
            description="Writes comprehensive documentation",
            prompt="You are a technical writer. Write clear, concise documentation with examples.",
            tools=["Read", "Write", "Glob"],
            model="haiku",
        ),
        "test-engineer": AgentDefinition(
            description="Writes and runs tests",
            prompt="You are a test engineer. Write thorough tests covering edge cases.",
            tools=["Read", "Write", "Bash", "Glob", "Grep"],
            model="sonnet",
        ),
    },
)
```

### 4.4 Dynamic Control (runtime changes)

```python
async with ClaudeSDKClient(options=options) as client:
    await client.query("Start analyzing the codebase")

    # Đổi model giữa chừng
    await client.set_model("claude-opus-4-6")

    # Đổi permission mode
    await client.set_permission_mode("bypassPermissions")

    # Kiểm tra MCP servers
    status = await client.get_mcp_status()
    for server in status.get("mcpServers", []):
        print(f"{server['name']}: {server['status']}")

    # Interrupt task đang chạy
    await client.interrupt()

    # Reconnect failed MCP server
    await client.reconnect_mcp_server("my-server")

    # Toggle MCP server on/off
    await client.toggle_mcp_server("expensive-server", enabled=False)
```

### 4.5 Structured Output (JSON schema)

```python
options = ClaudeAgentOptions(
    output_format={
        "type": "object",
        "properties": {
            "summary": {"type": "string"},
            "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
            "score": {"type": "number", "minimum": 0, "maximum": 1},
            "key_points": {"type": "array", "items": {"type": "string"}},
        },
        "required": ["summary", "sentiment", "score"],
    },
    max_turns=1,
)

# ResultMessage.structured_output sẽ chứa parsed JSON
```

### 4.6 Thinking Configuration

```python
# Adaptive (Claude tự quyết định)
options = ClaudeAgentOptions(thinking={"type": "adaptive"})

# Enabled với budget
options = ClaudeAgentOptions(thinking={"type": "enabled", "budget_tokens": 10000})

# Disabled
options = ClaudeAgentOptions(thinking={"type": "disabled"})

# Effort level (thay thế thinking config)
options = ClaudeAgentOptions(effort="high")  # "low" | "medium" | "high" | "max"
```

### 4.7 Budget Control

```python
options = ClaudeAgentOptions(
    max_budget_usd=0.50,  # Maximum $0.50
    max_turns=10,         # Maximum 10 turns
)

# Check budget in ResultMessage
async for msg in query(prompt="...", options=options):
    if isinstance(msg, ResultMessage):
        if msg.subtype == "error_max_budget_usd":
            print("Budget exceeded!")
        print(f"Cost: ${msg.total_cost_usd}")
```

---

## 5. Simple MAS: Orchestrator Pattern

### 5.1 Architecture

```
┌─────────────────────────────────────────┐
│           ORCHESTRATOR AGENT             │
│  (ClaudeSDKClient with custom tools)    │
│                                         │
│  Tools:                                 │
│   - ask_researcher(query) ──────────┐   │
│   - ask_writer(topic, context) ─────┤   │
│   - save_result(content, path) ─────┤   │
│                                     │   │
├─────────────────────────────────────┤   │
│                                     ▼   │
│  ┌──────────────┐  ┌──────────────┐     │
│  │  RESEARCHER  │  │    WRITER    │     │
│  │  (query())   │  │  (query())   │     │
│  │              │  │              │     │
│  │ Read-only    │  │ Read + Write │     │
│  │ tools        │  │ tools        │     │
│  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────┘
```

### 5.2 Complete Working Code

```python
"""
Simple Multi-Agent System: Orchestrator + Researcher + Writer
"""
import anyio
import json
from claude_agent_sdk import (
    tool, create_sdk_mcp_server,
    ClaudeSDKClient, ClaudeAgentOptions,
    query,
    AssistantMessage, ResultMessage, TextBlock, ToolUseBlock,
)


# ============================================================
# AGENT 1: RESEARCHER — Tìm kiếm và phân tích thông tin
# ============================================================
async def call_researcher(research_query: str) -> str:
    """Gọi researcher agent bằng query() — stateless, one-shot."""
    options = ClaudeAgentOptions(
        system_prompt=(
            "You are a research specialist. "
            "Analyze the query thoroughly and provide structured findings. "
            "Format: Summary, Key Points (bullets), Sources/References."
        ),
        tools=["Read", "Glob", "Grep", "Bash"],  # Read-only + Bash for search
        allowed_tools=["Read", "Glob", "Grep", "Bash"],
        max_turns=5,
        max_budget_usd=0.10,
    )

    results = []
    async for msg in query(prompt=research_query, options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    results.append(block.text)
        elif isinstance(msg, ResultMessage):
            break

    return "\n".join(results) if results else "No findings."


# ============================================================
# AGENT 2: WRITER — Viết nội dung dựa trên research
# ============================================================
async def call_writer(topic: str, research_context: str) -> str:
    """Gọi writer agent bằng query() — stateless, one-shot."""
    options = ClaudeAgentOptions(
        system_prompt=(
            "You are a technical writer. "
            "Write clear, well-structured content based on the research provided. "
            "Use markdown formatting. Be concise but thorough."
        ),
        max_turns=3,
        max_budget_usd=0.10,
    )

    prompt = f"""Write about: {topic}

Research context:
{research_context}

Write a comprehensive but concise article."""

    results = []
    async for msg in query(prompt=prompt, options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    results.append(block.text)
        elif isinstance(msg, ResultMessage):
            break

    return "\n".join(results) if results else "Could not generate content."


# ============================================================
# ORCHESTRATOR TOOLS — Tools mà orchestrator dùng
# ============================================================
@tool(
    "ask_researcher",
    "Ask the research agent to investigate a topic. Returns structured findings.",
    {"query": str},
)
async def ask_researcher_tool(args):
    result = await call_researcher(args["query"])
    return {"content": [{"type": "text", "text": result}]}


@tool(
    "ask_writer",
    "Ask the writer agent to create content based on research findings.",
    {"topic": str, "research_context": str},
)
async def ask_writer_tool(args):
    result = await call_writer(args["topic"], args["research_context"])
    return {"content": [{"type": "text", "text": result}]}


@tool(
    "save_result",
    "Save final content to a file.",
    {"content": str, "filename": str},
)
async def save_result_tool(args):
    import pathlib
    path = pathlib.Path(args["filename"])
    path.write_text(args["content"])
    return {"content": [{"type": "text", "text": f"Saved to {path}"}]}


# ============================================================
# ORCHESTRATOR — Điều phối toàn bộ
# ============================================================
async def run_mas(user_request: str):
    """Main MAS orchestrator."""

    # Tạo MCP server chứa orchestrator tools
    orchestrator_server = create_sdk_mcp_server(
        name="agents",
        version="1.0.0",
        tools=[ask_researcher_tool, ask_writer_tool, save_result_tool],
    )

    options = ClaudeAgentOptions(
        system_prompt="""You are an orchestrator managing a team of specialist agents.

Available agents:
1. RESEARCHER (ask_researcher) — Investigates topics, finds information
2. WRITER (ask_writer) — Writes polished content from research

Workflow:
1. Break down the user's request into research questions
2. Send each question to the researcher
3. Collect findings and send to the writer with clear instructions
4. Save the final result

Always use the agents — do NOT write content yourself.""",
        mcp_servers={"agents": orchestrator_server},
        allowed_tools=[
            "mcp__agents__ask_researcher",
            "mcp__agents__ask_writer",
            "mcp__agents__save_result",
        ],
        max_turns=15,
        max_budget_usd=0.50,
    )

    print(f"[Orchestrator] Processing: {user_request}\n")

    async with ClaudeSDKClient(options=options) as client:
        await client.query(user_request)
        async for msg in client.receive_response():
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(f"[Orchestrator] {block.text}")
                    elif isinstance(block, ToolUseBlock):
                        print(f"  >> Calling: {block.name}({json.dumps(block.input)[:100]}...)")
            elif isinstance(msg, ResultMessage):
                print(f"\n[Done] Cost: ${msg.total_cost_usd:.4f}, Turns: {msg.num_turns}")


# ============================================================
# MAIN
# ============================================================
async def main():
    await run_mas(
        "Research the architecture of this Python SDK project and write "
        "a technical overview document. Save it as output/sdk-overview.md"
    )

if __name__ == "__main__":
    anyio.run(main)
```

---

## 6. Complex MAS: Full-layer Architecture

### 6.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    LAYER 6: MONITORING                        │
│   stderr callback, cost tracking, rate limit detection       │
├──────────────────────────────────────────────────────────────┤
│                    LAYER 5: SAFETY / PERMISSIONS              │
│   hooks (PreToolUse, PostToolUse), can_use_tool callback     │
├──────────────────────────────────────────────────────────────┤
│                    LAYER 4: MEMORY / STATE                    │
│   Shared state dict, conversation history, task results      │
├──────────────────────────────────────────────────────────────┤
│                    LAYER 3: TOOL REGISTRY                     │
│   SDK MCP Servers (per-agent tools + shared tools)           │
├──────────────────────────────────────────────────────────────┤
│                    LAYER 2: AGENT POOL                        │
│   AgentDefinition configs, query()/ClaudeSDKClient instances │
├──────────────────────────────────────────────────────────────┤
│                    LAYER 1: ROUTER / DISPATCHER               │
│   Task decomposition, agent selection, result aggregation    │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Complete Working Code

```python
"""
Complex Multi-Agent System with Full-Layer Architecture
claude-agent-sdk-python

Layers:
  1. Router/Dispatcher   — Task decomposition & agent selection
  2. Agent Pool          — Configured agents with specific capabilities
  3. Tool Registry       — MCP servers with shared + agent-specific tools
  4. Memory/State        — Cross-agent shared state
  5. Safety/Permissions  — Hooks + permission callbacks
  6. Monitoring          — Cost, rate limits, logging
"""
import anyio
import json
import time
from dataclasses import dataclass, field
from typing import Any

from claude_agent_sdk import (
    tool, create_sdk_mcp_server,
    ClaudeSDKClient, ClaudeAgentOptions,
    query,
    AssistantMessage, ResultMessage, SystemMessage,
    TextBlock, ToolUseBlock, ToolResultBlock,
    HookMatcher, AgentDefinition,
)
from claude_agent_sdk.types import (
    PreToolUseHookInput, PostToolUseHookInput,
    HookInput, HookContext, HookJSONOutput,
    ToolPermissionContext, PermissionResultAllow, PermissionResultDeny,
    RateLimitEvent,
)


# ====================================================================
# LAYER 4: MEMORY / STATE
# ====================================================================
@dataclass
class SharedState:
    """Cross-agent shared memory. Thread-safe via anyio."""
    task_results: dict[str, Any] = field(default_factory=dict)
    conversation_history: list[dict] = field(default_factory=list)
    agent_logs: list[dict] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    _lock: anyio.Lock = field(default_factory=anyio.Lock)

    async def store(self, key: str, value: Any):
        async with self._lock:
            self.task_results[key] = value

    async def retrieve(self, key: str) -> Any:
        async with self._lock:
            return self.task_results.get(key)

    async def log(self, agent: str, action: str, details: str):
        async with self._lock:
            entry = {
                "ts": time.time(),
                "agent": agent,
                "action": action,
                "details": details,
            }
            self.agent_logs.append(entry)
            print(f"  [{agent}] {action}: {details[:80]}")


# ====================================================================
# LAYER 6: MONITORING
# ====================================================================
@dataclass
class MonitoringLayer:
    """Track costs, rate limits, and performance."""
    total_cost: float = 0.0
    total_turns: int = 0
    agent_costs: dict[str, float] = field(default_factory=dict)
    rate_limit_events: list[dict] = field(default_factory=list)
    stderr_logs: list[str] = field(default_factory=list)

    def stderr_callback(self, line: str):
        self.stderr_logs.append(line)

    def record_result(self, agent_name: str, result: ResultMessage):
        cost = result.total_cost_usd or 0.0
        self.total_cost += cost
        self.total_turns += result.num_turns
        self.agent_costs[agent_name] = self.agent_costs.get(agent_name, 0) + cost

    def record_rate_limit(self, event: RateLimitEvent):
        self.rate_limit_events.append({
            "status": event.rate_limit_info.status,
            "resets_at": event.rate_limit_info.resets_at,
        })

    def summary(self) -> str:
        lines = [
            f"Total cost: ${self.total_cost:.4f}",
            f"Total turns: {self.total_turns}",
            f"Rate limit events: {len(self.rate_limit_events)}",
        ]
        for agent, cost in self.agent_costs.items():
            lines.append(f"  {agent}: ${cost:.4f}")
        return "\n".join(lines)


# ====================================================================
# LAYER 5: SAFETY / PERMISSIONS
# ====================================================================
class SafetyLayer:
    """Hooks and permission callbacks for security."""

    def __init__(self, state: SharedState, monitor: MonitoringLayer):
        self.state = state
        self.monitor = monitor
        self.blocked_commands = ["rm -rf /", "sudo rm", "chmod 777", ":(){ :|:& };:"]

    async def pre_tool_hook(
        self, input: HookInput, tool_use_id: str | None, context: HookContext,
    ) -> HookJSONOutput:
        if isinstance(input, PreToolUseHookInput):
            tool_name = input.get("tool_name", "")
            tool_input = input.get("tool_input", {})
            await self.state.log("safety", "pre_tool", f"{tool_name}")

            # Block dangerous Bash commands
            if tool_name == "Bash":
                cmd = tool_input.get("command", "")
                for blocked in self.blocked_commands:
                    if blocked in cmd:
                        return {
                            "decision": "block",
                            "reason": f"Blocked dangerous command: {blocked}",
                        }

            return {
                "hookSpecificOutput": {
                    "hookEventName": "PreToolUse",
                    "permissionDecision": "allow",
                }
            }
        return {"continue_": True}

    async def post_tool_hook(
        self, input: HookInput, tool_use_id: str | None, context: HookContext,
    ) -> HookJSONOutput:
        if isinstance(input, PostToolUseHookInput):
            tool_name = input.get("tool_name", "")
            await self.state.log("safety", "post_tool", f"{tool_name} completed")
        return {"continue_": True}

    async def permission_handler(
        self, tool_name: str, tool_input: dict, context: ToolPermissionContext,
    ) -> PermissionResultAllow | PermissionResultDeny:
        # Allow all read-only tools
        if tool_name in ["Read", "Glob", "Grep"]:
            return PermissionResultAllow()

        # Allow MCP tools (they have their own safety)
        if tool_name.startswith("mcp__"):
            return PermissionResultAllow()

        # Allow Bash with restrictions
        if tool_name == "Bash":
            cmd = tool_input.get("command", "")
            if any(d in cmd for d in self.blocked_commands):
                return PermissionResultDeny(message="Dangerous command")
            return PermissionResultAllow()

        # Allow Write/Edit in project directory only
        if tool_name in ["Write", "Edit"]:
            path = tool_input.get("file_path", "")
            if not path.startswith("/home/"):
                return PermissionResultDeny(message=f"Cannot write outside /home/")
            return PermissionResultAllow()

        return PermissionResultAllow()

    def get_hooks(self) -> dict:
        return {
            "PreToolUse": [
                HookMatcher(matcher=None, hooks=[self.pre_tool_hook], timeout=10.0),
            ],
            "PostToolUse": [
                HookMatcher(matcher=None, hooks=[self.post_tool_hook]),
            ],
        }


# ====================================================================
# LAYER 3: TOOL REGISTRY
# ====================================================================
class ToolRegistry:
    """Manages MCP servers for different agent capabilities."""

    def __init__(self, state: SharedState):
        self.state = state
        self.servers: dict[str, Any] = {}
        self._build_servers()

    def _build_servers(self):
        # --- Shared tools (available to all agents) ---
        @tool("store_result", "Store a result in shared memory", {
            "key": str, "value": str,
        })
        async def store_result(args):
            await self.state.store(args["key"], args["value"])
            return {"content": [{"type": "text", "text": f"Stored: {args['key']}"}]}

        @tool("retrieve_result", "Retrieve a result from shared memory", {
            "key": str,
        })
        async def retrieve_result(args):
            value = await self.state.retrieve(args["key"])
            if value is None:
                return {"content": [{"type": "text", "text": "Key not found"}], "is_error": True}
            return {"content": [{"type": "text", "text": str(value)}]}

        @tool("list_stored_keys", "List all keys in shared memory", {})
        async def list_keys(args):
            keys = list(self.state.task_results.keys())
            return {"content": [{"type": "text", "text": json.dumps(keys)}]}

        self.servers["shared"] = create_sdk_mcp_server(
            "shared", "1.0.0",
            tools=[store_result, retrieve_result, list_keys],
        )

        # --- Orchestrator tools (agent dispatch) ---
        @tool("dispatch_agent", "Dispatch a task to a specialist agent", {
            "agent_name": str,
            "task": str,
            "context": str,
        })
        async def dispatch_agent(args):
            agent_name = args["agent_name"]
            task = args["task"]
            context = args.get("context", "")

            await self.state.log("orchestrator", "dispatch", f"{agent_name}: {task[:50]}")

            # This will be replaced by AgentPool at runtime
            return {"content": [{"type": "text", "text": f"Dispatched to {agent_name}"}]}

        @tool("get_agent_status", "Get status of all agents and their results", {})
        async def get_agent_status(args):
            logs = self.state.agent_logs[-10:]  # Last 10 entries
            results = {k: str(v)[:100] for k, v in self.state.task_results.items()}
            status = {"recent_logs": logs, "stored_results": results}
            return {"content": [{"type": "text", "text": json.dumps(status, default=str)}]}

        self.servers["orchestrator"] = create_sdk_mcp_server(
            "orchestrator", "1.0.0",
            tools=[dispatch_agent, get_agent_status],
        )

    def get_servers(self) -> dict:
        return self.servers

    def get_allowed_tools(self, role: str) -> list[str]:
        """Return allowed tool names based on role."""
        base = [
            "mcp__shared__store_result",
            "mcp__shared__retrieve_result",
            "mcp__shared__list_stored_keys",
        ]
        if role == "orchestrator":
            return base + [
                "mcp__orchestrator__dispatch_agent",
                "mcp__orchestrator__get_agent_status",
            ]
        elif role == "researcher":
            return base + ["Read", "Glob", "Grep", "Bash"]
        elif role == "writer":
            return base + ["Read", "Write", "Glob"]
        elif role == "reviewer":
            return base + ["Read", "Glob", "Grep"]
        return base


# ====================================================================
# LAYER 2: AGENT POOL
# ====================================================================
class AgentPool:
    """Manages agent configurations and execution."""

    def __init__(
        self,
        state: SharedState,
        tool_registry: ToolRegistry,
        safety: SafetyLayer,
        monitor: MonitoringLayer,
    ):
        self.state = state
        self.tools = tool_registry
        self.safety = safety
        self.monitor = monitor

    async def run_agent(self, agent_name: str, task: str) -> str:
        """Execute an agent with appropriate configuration."""

        # Agent-specific configs
        configs = {
            "researcher": {
                "system_prompt": (
                    "You are a senior research analyst. "
                    "Find, analyze, and synthesize information. "
                    "Use tools to search the codebase. "
                    "Store important findings in shared memory using store_result. "
                    "Be thorough and cite specific files/lines."
                ),
                "tools": self.tools.get_allowed_tools("researcher"),
                "max_turns": 8,
                "budget": 0.15,
                "model": None,  # inherit default
            },
            "writer": {
                "system_prompt": (
                    "You are a technical writer. "
                    "Create clear, well-structured documentation. "
                    "Use retrieve_result to get research findings from shared memory. "
                    "Write in markdown format."
                ),
                "tools": self.tools.get_allowed_tools("writer"),
                "max_turns": 5,
                "budget": 0.10,
                "model": None,
            },
            "reviewer": {
                "system_prompt": (
                    "You are a code reviewer. "
                    "Review code for bugs, security issues, and best practices. "
                    "Use retrieve_result to get context from shared memory. "
                    "Provide actionable feedback."
                ),
                "tools": self.tools.get_allowed_tools("reviewer"),
                "max_turns": 5,
                "budget": 0.10,
                "model": None,
            },
        }

        config = configs.get(agent_name)
        if not config:
            return f"Unknown agent: {agent_name}"

        await self.state.log(agent_name, "start", task[:80])

        options = ClaudeAgentOptions(
            system_prompt=config["system_prompt"],
            allowed_tools=config["tools"],
            mcp_servers=self.tools.get_servers(),
            max_turns=config["max_turns"],
            max_budget_usd=config["budget"],
            hooks=self.safety.get_hooks(),
            can_use_tool=self.safety.permission_handler,
            stderr=self.monitor.stderr_callback,
        )

        results = []
        async for msg in query(prompt=task, options=options):
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        results.append(block.text)
            elif isinstance(msg, RateLimitEvent):
                self.monitor.record_rate_limit(msg)
            elif isinstance(msg, ResultMessage):
                self.monitor.record_result(agent_name, msg)
                await self.state.log(
                    agent_name, "complete",
                    f"cost=${msg.total_cost_usd:.4f}, turns={msg.num_turns}",
                )
                break

        output = "\n".join(results) if results else "No output produced."
        await self.state.store(f"{agent_name}_last_result", output)
        return output


# ====================================================================
# LAYER 1: ROUTER / DISPATCHER
# ====================================================================
class MASRouter:
    """
    Top-level orchestrator.
    Decomposes tasks, selects agents, aggregates results.
    """

    def __init__(self):
        self.state = SharedState()
        self.monitor = MonitoringLayer()
        self.safety = SafetyLayer(self.state, self.monitor)
        self.tools = ToolRegistry(self.state)
        self.agent_pool = AgentPool(
            self.state, self.tools, self.safety, self.monitor,
        )

    async def run(self, user_request: str):
        """Run the full MAS pipeline."""
        print(f"\n{'='*60}")
        print(f"MAS Router: Processing request")
        print(f"{'='*60}\n")

        # Build orchestrator tools that can actually dispatch
        @tool("dispatch_agent", "Dispatch a task to a specialist agent", {
            "agent_name": str,
            "task": str,
        })
        async def dispatch_tool(args):
            result = await self.agent_pool.run_agent(
                args["agent_name"], args["task"],
            )
            return {"content": [{"type": "text", "text": result}]}

        @tool("dispatch_parallel", "Dispatch tasks to multiple agents in parallel", {
            "tasks": list,  # [{"agent": str, "task": str}, ...]
        })
        async def dispatch_parallel_tool(args):
            async def run_one(item):
                return await self.agent_pool.run_agent(item["agent"], item["task"])

            results = {}
            async with anyio.create_task_group() as tg:
                for item in args["tasks"]:
                    async def _run(i=item):
                        r = await run_one(i)
                        results[i["agent"]] = r
                    tg.start_soon(_run)

            return {"content": [{"type": "text", "text": json.dumps(results, default=str)}]}

        @tool("store_final", "Store the final aggregated result", {
            "key": str, "content": str,
        })
        async def store_final_tool(args):
            await self.state.store(args["key"], args["content"])
            return {"content": [{"type": "text", "text": f"Final result stored: {args['key']}"}]}

        @tool("get_memory", "Get stored results from agent memory", {
            "key": str,
        })
        async def get_memory_tool(args):
            value = await self.state.retrieve(args["key"])
            if value is None:
                return {"content": [{"type": "text", "text": "Not found"}], "is_error": True}
            return {"content": [{"type": "text", "text": str(value)}]}

        # Create orchestrator server
        orch_server = create_sdk_mcp_server(
            "orch", "1.0.0",
            tools=[dispatch_tool, dispatch_parallel_tool, store_final_tool, get_memory_tool],
        )

        options = ClaudeAgentOptions(
            system_prompt="""You are the master orchestrator of a Multi-Agent System.

Available agents (use dispatch_agent):
- "researcher": Searches code, analyzes patterns, stores findings
- "writer": Creates documentation from research
- "reviewer": Reviews code for issues

Available tools:
- dispatch_agent: Send task to one agent
- dispatch_parallel: Send tasks to multiple agents simultaneously
- store_final: Save final aggregated result
- get_memory: Retrieve previous agent results

WORKFLOW:
1. Analyze the user's request
2. Break it into sub-tasks for appropriate agents
3. Dispatch tasks (parallel when possible)
4. Review and aggregate results
5. Store final output

Be strategic about parallelization — research tasks can often run in parallel.""",
            mcp_servers={"orch": orch_server},
            allowed_tools=[
                "mcp__orch__dispatch_agent",
                "mcp__orch__dispatch_parallel",
                "mcp__orch__store_final",
                "mcp__orch__get_memory",
            ],
            max_turns=20,
            max_budget_usd=1.00,
            hooks=self.safety.get_hooks(),
            stderr=self.monitor.stderr_callback,
        )

        async with ClaudeSDKClient(options=options) as client:
            await client.query(user_request)
            async for msg in client.receive_response():
                if isinstance(msg, AssistantMessage):
                    for block in msg.content:
                        if isinstance(block, TextBlock):
                            print(f"\n[Orchestrator] {block.text}")
                        elif isinstance(block, ToolUseBlock):
                            print(f"  >> {block.name}({json.dumps(block.input)[:100]}...)")
                elif isinstance(msg, ResultMessage):
                    print(f"\n{'='*60}")
                    print(f"MONITORING SUMMARY:")
                    print(self.monitor.summary())
                    print(f"{'='*60}")


# ====================================================================
# MAIN
# ====================================================================
async def main():
    mas = MASRouter()
    await mas.run(
        "Analyze the architecture of this SDK project. "
        "Have the researcher examine the source code structure, "
        "then have the writer create a technical overview, "
        "and finally have the reviewer check the writer's output for accuracy."
    )

if __name__ == "__main__":
    anyio.run(main)
```

### 6.3 Giải thích từng Layer

| Layer | Responsibility | SDK Feature Used |
|-------|---------------|------------------|
| **L1 Router** | Decompose tasks, select agents | ClaudeSDKClient + custom MCP tools |
| **L2 Agent Pool** | Configure & run agents | query() + ClaudeAgentOptions |
| **L3 Tool Registry** | Register MCP servers per role | @tool + create_sdk_mcp_server() |
| **L4 Memory/State** | Shared state across agents | Custom Python dataclass + anyio.Lock |
| **L5 Safety** | Block dangerous actions | hooks (PreToolUse) + can_use_tool |
| **L6 Monitoring** | Track cost, rate limits | stderr callback + ResultMessage |

---

## 7. Tài liệu & Best Practices

### 7.1 Official Documentation Links

| Resource | URL | Nội dung |
|----------|-----|----------|
| **SDK Docs** | https://docs.anthropic.com/en/docs/claude-code/sdk | Official guide |
| **GitHub Repo** | https://github.com/anthropics/claude-agent-sdk-python | Source + README |
| **Claude Code Docs** | https://docs.anthropic.com/en/docs/claude-code | Parent product docs |
| **MCP Specification** | https://modelcontextprotocol.io | MCP protocol spec |
| **Anthropic Cookbook** | https://github.com/anthropics/anthropic-cookbook | Examples & patterns |
| **API Reference** | https://docs.anthropic.com/en/api | Claude API docs |

### 7.2 Source Code References (trong repo này)

| File | Học gì |
|------|--------|
| `src/claude_agent_sdk/__init__.py` | Public API, @tool, create_sdk_mcp_server |
| [`types.py`](../../src/claude_agent_sdk/types.py) | ALL types, options, hooks, messages |
| [`client.py`](../../src/claude_agent_sdk/_internal/client.py) | ClaudeSDKClient full API |
| [`query.py`](../../src/claude_agent_sdk/_internal/query.py) | query() function |
| `examples/mcp_calculator.py` | @tool + MCP server pattern |
| `examples/hooks.py` | Hook patterns |
| `examples/agents.py` | AgentDefinition usage |
| `examples/tool_permission_callback.py` | Permission callback |
| `examples/streaming_mode.py` | 10 streaming patterns |
| `e2e-tests/test_sdk_mcp_tools.py` | Real MCP tool testing |
| `e2e-tests/test_hooks.py` | Real hook testing |

### 7.3 Best Practices (từ codebase analysis)

**DO:**
1. **Luôn dùng `allowed_tools`** khi dùng MCP tools — format: `mcp__{server}__{tool}`
2. **Dùng `max_budget_usd`** và `max_turns` cho mỗi agent — tránh runaway costs
3. **Handle `ResultMessage`** để check `is_error`, `total_cost_usd`, `stop_reason`
4. **Dùng `anyio` thay vì `asyncio` trực tiếp** — SDK dùng anyio internally
5. **Error handling trong @tool**: return `{"is_error": True}` thay vì raise exception
6. **Dùng context manager** (`async with ClaudeSDKClient`) — auto cleanup
7. **Pre-approve tools** trong `allowed_tools` — tránh permission prompts
8. **Dùng `create_sdk_mcp_server`** cho in-process tools — nhanh hơn subprocess

**DON'T:**
1. **Không share ClaudeSDKClient instances** giữa multiple coroutines — not thread-safe
2. **Không dùng `query()` cho multi-turn** — dùng ClaudeSDKClient instead
3. **Không ignore `RateLimitEvent`** — implement backoff
4. **Không hardcode model names** — dùng options.model, cho phép override
5. **Không bỏ qua `ResultMessage.is_error`** — luôn check
6. **Không tạo quá nhiều concurrent agents** — respect rate limits
7. **Không dùng blocking I/O** trong @tool handlers — always async

### 7.4 Anti-patterns cần tránh

```python
# BAD: Blocking I/O trong tool handler
@tool("bad", "Bad pattern", {"path": str})
async def bad_tool(args):
    data = open(args["path"]).read()  # BLOCKING!
    return {"content": [{"type": "text", "text": data}]}

# GOOD: Async I/O
@tool("good", "Good pattern", {"path": str})
async def good_tool(args):
    import aiofiles
    async with aiofiles.open(args["path"]) as f:
        data = await f.read()
    return {"content": [{"type": "text", "text": data}]}

# BAD: Exception propagation
@tool("bad2", "Bad error handling", {"x": int})
async def bad_tool2(args):
    return {"content": [{"type": "text", "text": str(1/args["x"])}]}  # ZeroDivisionError!

# GOOD: Explicit error handling
@tool("good2", "Good error handling", {"x": int})
async def good_tool2(args):
    if args["x"] == 0:
        return {"content": [{"type": "text", "text": "Cannot divide by zero"}], "is_error": True}
    return {"content": [{"type": "text", "text": str(1/args["x"])}]}
```

### 7.5 Nơi lấy Best Practices

1. **SDK Examples** (`examples/` folder) — Official patterns từ Anthropic
2. **E2E Tests** (`e2e-tests/` folder) — Real-world usage patterns
3. **README.md** — Quick reference
4. **CHANGELOG.md** — Breaking changes, new features
5. **Type definitions** ([`types.py`](../../src/claude_agent_sdk/types.py)) — Canonical reference cho tất cả options
6. **MCP Specification** (modelcontextprotocol.io) — Protocol understanding
7. **Anthropic Cookbook** — Community patterns & recipes

### 7.6 Learning Path gợi ý

```
Week 1: Foundation
  Day 1-2: Đọc README.md + chạy examples/quick_start.py
  Day 3-4: Thực hành @tool + examples/mcp_calculator.py
  Day 5:   Đọc types.py, hiểu ClaudeAgentOptions

Week 2: Intermediate
  Day 1-2: Hooks (examples/hooks.py) + Permissions (examples/tool_permission_callback.py)
  Day 3-4: Streaming (examples/streaming_mode.py) + Multi-turn
  Day 5:   AgentDefinition (examples/agents.py) + Dynamic control

Week 3: Advanced
  Day 1-2: Build Simple MAS (Section 5 above)
  Day 3-4: Add Memory + Safety layers
  Day 5:   Build Complex MAS (Section 6 above)

Week 4: Production
  Day 1-2: Testing patterns (from e2e-tests/)
  Day 3-4: Monitoring + Error handling + Rate limit management
  Day 5:   Code review + Optimization
```

---

## Appendix A: Quick Reference Card

```
# Install
pip install claude-agent-sdk

# One-shot query
async for msg in query(prompt="Hello", options=options):
    if isinstance(msg, AssistantMessage): ...

# Interactive client
async with ClaudeSDKClient(options=options) as client:
    await client.query("prompt")
    async for msg in client.receive_response(): ...

# Custom tool
@tool("name", "description", {"param": str})
async def handler(args): return {"content": [...]}

# MCP server
server = create_sdk_mcp_server("name", "1.0.0", tools=[handler])
options = ClaudeAgentOptions(
    mcp_servers={"name": server},
    allowed_tools=["mcp__name__tool"],
)

# Hooks
HookMatcher(matcher="Bash", hooks=[callback], timeout=30)

# Agents
AgentDefinition(description="...", prompt="...", tools=[...], model="sonnet")

# Key message types
UserMessage, AssistantMessage, SystemMessage, ResultMessage
TextBlock, ToolUseBlock, ToolResultBlock, ThinkingBlock
```

---

*Tài liệu này được tạo từ phân tích toàn bộ source code, 14 examples, 17 test files, và documentation của claude-agent-sdk-python v0.1.48.*
