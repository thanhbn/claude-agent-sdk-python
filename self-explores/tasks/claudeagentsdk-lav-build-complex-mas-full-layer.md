---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-lav
title: "[MAS-5] Build Complex MAS: Full-layer Architecture"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
tags: [mas, multi-agent, full-layer, reference-implementation, claude-agent-sdk-python]
---

# [MAS-5] Build Complex MAS: Full-layer Architecture — Detailed Design

## 1. Objective
Xây dựng reference implementation cho Multi-Agent System 6 tầng hoàn chỉnh (~300 lines). Sáu tầng: (1) Router — task decomposition + dispatch, (2) Agent Pool — 3 agents: researcher/writer/reviewer, (3) Tool Registry — shared MCP + per-agent tools, (4) Memory — SharedState with anyio.Lock, (5) Safety — PreToolUse hook chặn lệnh nguy hiểm, (6) Monitoring — cost tracking via ResultMessage. Output file: `examples/mas/complex_mas.py`. End-to-end chạy được, 3 agents dispatched, memory shared across agents, hooks active, tổng cost < $1.00.

## 2. Scope
**In-scope:**
- 6-layer architecture implemented as Python classes
- 3 specialist agents: researcher (read-only), writer (read+write), reviewer (read-only)
- Shared MCP server for cross-agent memory (store/retrieve/list)
- Per-agent tool restrictions via `allowed_tools`
- Safety layer: PreToolUse hook blocks dangerous Bash commands
- Monitoring layer: cost aggregation per agent via ResultMessage
- Router dispatches agents via MCP tools (tool handler calls `query()` internally)
- Parallel dispatch support via `anyio.create_task_group()`
- Complete `main()` with example user request

**Out-of-scope:**
- Persistent storage (disk/database) — memory is in-process only
- Dynamic agent creation at runtime (pool is fixed at init)
- Web UI or API endpoints
- Multi-turn orchestrator (single request → pipeline → result)
- Agent-to-agent direct communication (all goes through shared memory)

## 3. Input / Output
**Input:**
- MAS-4 output: working simple MAS (orchestrator pattern from learning guide Section 5)
- Learning guide Section 6 skeleton code (mas-learning-guide.md lines 812-1330)
- SDK types: `ClaudeAgentOptions`, `query()`, `ClaudeSDKClient`, `@tool`, `create_sdk_mcp_server`
- Hook types: `HookMatcher`, `PreToolUseHookInput`, `PostToolUseHookInput`, `HookInput`, `HookContext`, `HookJSONOutput`
- Permission types: `ToolPermissionContext`, `PermissionResultAllow`, `PermissionResultDeny`
- Message types: `AssistantMessage`, `ResultMessage`, `TextBlock`, `ToolUseBlock`, `RateLimitEvent`

**Output:**
- `examples/mas/complex_mas.py` (~300 lines, self-contained, runnable)
- Console output: agent dispatch logs, monitoring summary with per-agent costs
- Final result stored in SharedState and printed

## 4. Dependencies
- **Depends on:** MAS-4 (Simple MAS: Orchestrator Pattern) — must understand tool-as-agent-dispatch pattern
- **Blocks:** nothing

## 5. Flow xu ly

### Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        MASRouter (Layer 1)                       │
│  - state: SharedState                                           │
│  - monitor: MonitoringLayer                                     │
│  - safety: SafetyLayer                                          │
│  - tools: ToolRegistry                                          │
│  - agent_pool: AgentPool                                        │
│  + run(user_request: str) -> None                               │
│    Creates dispatch_agent, dispatch_parallel, store_final,      │
│    get_memory tools → creates orch MCP server →                 │
│    runs ClaudeSDKClient with orchestrator system prompt          │
├─────────────────────────────────────────────────────────────────┤
│                        AgentPool (Layer 2)                       │
│  - state: SharedState                                           │
│  - tools: ToolRegistry                                          │
│  - safety: SafetyLayer                                          │
│  - monitor: MonitoringLayer                                     │
│  - configs: dict[str, AgentConfig]                              │
│  + run_agent(agent_name: str, task: str) -> str                 │
│    Builds ClaudeAgentOptions per agent config →                 │
│    calls query() → streams AssistantMessage/ResultMessage →     │
│    records cost in monitor → stores result in SharedState       │
├─────────────────────────────────────────────────────────────────┤
│                      ToolRegistry (Layer 3)                      │
│  - state: SharedState                                           │
│  - servers: dict[str, SdkMcpServer]                             │
│  + _build_servers() -> None                                     │
│    Creates "shared" server: store_result, retrieve_result,      │
│    list_stored_keys                                             │
│    Creates "orchestrator" server: dispatch_agent,               │
│    get_agent_status                                             │
│  + get_servers() -> dict                                        │
│  + get_allowed_tools(role: str) -> list[str]                    │
├─────────────────────────────────────────────────────────────────┤
│                       SharedState (Layer 4)                      │
│  - task_results: dict[str, Any]                                 │
│  - conversation_history: list[dict]                             │
│  - agent_logs: list[dict]                                       │
│  - metadata: dict[str, Any]                                     │
│  - _lock: anyio.Lock                                            │
│  + store(key: str, value: Any) -> None     [async, locked]      │
│  + retrieve(key: str) -> Any | None        [async, locked]      │
│  + log(agent: str, action: str, details: str) [async, locked]  │
├─────────────────────────────────────────────────────────────────┤
│                       SafetyLayer (Layer 5)                      │
│  - state: SharedState                                           │
│  - monitor: MonitoringLayer                                     │
│  - blocked_commands: list[str]                                  │
│  + pre_tool_hook(input, tool_use_id, context) -> HookJSONOutput│
│  + post_tool_hook(input, tool_use_id, context) -> HookJSONOutput│
│  + permission_handler(tool_name, tool_input, context)           │
│    -> PermissionResultAllow | PermissionResultDeny              │
│  + get_hooks() -> dict                                          │
├─────────────────────────────────────────────────────────────────┤
│                    MonitoringLayer (Layer 6)                      │
│  - total_cost: float                                            │
│  - total_turns: int                                             │
│  - agent_costs: dict[str, float]                                │
│  - rate_limit_events: list[dict]                                │
│  - stderr_logs: list[str]                                       │
│  + stderr_callback(line: str) -> None                           │
│  + record_result(agent_name: str, result: ResultMessage) -> None│
│  + record_rate_limit(event: RateLimitEvent) -> None             │
│  + summary() -> str                                             │
└─────────────────────────────────────────────────────────────────┘

Dependency graph (init order):
  SharedState → MonitoringLayer → SafetyLayer(state, monitor) →
  ToolRegistry(state) → AgentPool(state, tools, safety, monitor) →
  MASRouter(owns all)
```

### Step 1: Implement SharedState (Layer 4) (~15 min)

SharedState is a `@dataclass` with four data fields and one `anyio.Lock`. All mutation methods acquire the lock.

```python
from dataclasses import dataclass, field
from typing import Any
import anyio
import time

@dataclass
class SharedState:
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
            entry = {"ts": time.time(), "agent": agent, "action": action, "details": details}
            self.agent_logs.append(entry)
            print(f"  [{agent}] {action}: {details[:80]}")
```

**How shared memory works:** All agents receive the same `SharedState` instance via the `ToolRegistry`. The `store_result` and `retrieve_result` MCP tools are closures that capture `self.state`. When Agent A calls `store_result(key="findings", value="...")`, the tool handler acquires `self._lock`, writes to `self.task_results["findings"]`, and releases. When Agent B later calls `retrieve_result(key="findings")`, it acquires the same lock and reads the value. The `anyio.Lock` ensures that concurrent agents (dispatched via `anyio.create_task_group()`) never see partial writes or corrupt state. Since `query()` is async and all tool handlers are async, the lock works correctly across interleaved coroutines.

**Verify:** Unit test — create SharedState, run 3 concurrent `store()` + `retrieve()` tasks with `anyio.create_task_group()`, assert all values are stored correctly without data loss.

### Step 2: Implement MonitoringLayer (Layer 6) (~10 min)

```python
@dataclass
class MonitoringLayer:
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
```

**Key insight:** `stderr_callback` is a sync function passed to `ClaudeAgentOptions(stderr=...)`. The CLI transport calls it with each stderr line. `record_result` is called in `AgentPool.run_agent()` after the agent finishes. MonitoringLayer does NOT need a lock because `record_result` is called sequentially per agent (even if agents run in parallel, each agent's `async for msg in query()` processes its own stream independently).

**Verify:** After MAS run, `monitor.summary()` shows per-agent costs that sum to `total_cost`, and `total_cost < 1.00`.

### Step 3: Implement SafetyLayer (Layer 5) (~15 min)

```python
class SafetyLayer:
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
```

**How hooks intercept dangerous commands:** The flow is: CLI subprocess is about to call a tool → CLI sends a `control_request` with `subtype: "hook_callback"` and `hookEventName: "PreToolUse"` → `Query._handle_control_request()` receives it → dispatches to registered hook handler → `SafetyLayer.pre_tool_hook()` checks `input.tool_name` and `input.tool_input` → if Bash command contains blocked pattern, returns `{"decision": "block", "reason": "..."}` → Query converts this to a `control_response` → CLI receives the block decision and SKIPS the tool call entirely. The agent sees the block reason and can adjust its approach.

For the `permission_handler`, the flow is different: `can_use_tool` callback is invoked BEFORE the hook system, acting as a first gate. If `PermissionResultDeny` is returned, the tool is never even offered to hooks. This provides defense-in-depth: permission_handler for broad policies, hooks for fine-grained inspection.

**Key gotcha:** Hook output uses `continue_` (trailing underscore) because `continue` is a Python keyword. The SDK's `_convert_hook_output_for_cli()` in [`query.py:34-50`](../../src/claude_agent_sdk/_internal/query.py#L34-L50) strips the underscore before sending to CLI.

**Verify:** Send a prompt that tries to run `rm -rf /` → hook blocks it → agent sees "Blocked dangerous command" in response.

### Step 4: Implement ToolRegistry (Layer 3) (~15 min)

```python
class ToolRegistry:
    def __init__(self, state: SharedState):
        self.state = state
        self.servers: dict[str, Any] = {}
        self._build_servers()

    def _build_servers(self):
        # Shared tools — all agents can store/retrieve
        @tool("store_result", "Store a result in shared memory", {"key": str, "value": str})
        async def store_result(args):
            await self.state.store(args["key"], args["value"])
            return {"content": [{"type": "text", "text": f"Stored: {args['key']}"}]}

        @tool("retrieve_result", "Retrieve a result from shared memory", {"key": str})
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

    def get_allowed_tools(self, role: str) -> list[str]:
        base = [
            "mcp__shared__store_result",
            "mcp__shared__retrieve_result",
            "mcp__shared__list_stored_keys",
        ]
        if role == "researcher":
            return base + ["Read", "Glob", "Grep", "Bash"]
        elif role == "writer":
            return base + ["Read", "Write", "Glob"]
        elif role == "reviewer":
            return base + ["Read", "Glob", "Grep"]
        return base
```

**How tools dispatch agents (the key pattern):** In `MASRouter.run()`, the orchestrator's MCP tools include `dispatch_agent` which internally calls `self.agent_pool.run_agent(agent_name, task)`. The `run_agent` method calls `query()` — a completely separate subprocess — passing agent-specific options. So the flow is:

1. Orchestrator (ClaudeSDKClient) decides to call `mcp__orch__dispatch_agent`
2. CLI sends tool call to SDK → `Query` intercepts because it's an SDK MCP tool
3. Tool handler `dispatch_tool(args)` executes in-process
4. Inside the handler, `self.agent_pool.run_agent("researcher", "...")` is awaited
5. `run_agent` creates new `ClaudeAgentOptions` with researcher config + shared MCP servers
6. `query(prompt=task, options=options)` spawns a NEW CLI subprocess for the researcher
7. Researcher runs, uses tools, stores findings in SharedState via `mcp__shared__store_result`
8. Researcher finishes → `query()` yields `ResultMessage` → cost recorded in MonitoringLayer
9. Result string returned → dispatch_tool returns MCP result → orchestrator sees the output

**Critical:** The `shared` MCP server is the SAME Python object passed to both the orchestrator and all agents. Since `@tool` handlers are closures capturing `self.state`, they all read/write the same `SharedState` instance. This is how cross-agent communication works: researcher stores → writer retrieves.

**Verify:** After researcher runs, call `state.retrieve("researcher_last_result")` → returns non-None string.

### Step 5: Implement AgentPool (Layer 2) (~20 min)

```python
class AgentPool:
    def __init__(self, state, tools, safety, monitor):
        self.state = state
        self.tools = tools
        self.safety = safety
        self.monitor = monitor

    async def run_agent(self, agent_name: str, task: str) -> str:
        configs = {
            "researcher": {
                "system_prompt": "You are a senior research analyst. ...",
                "tools": self.tools.get_allowed_tools("researcher"),
                "max_turns": 8,
                "budget": 0.15,
            },
            "writer": {
                "system_prompt": "You are a technical writer. ...",
                "tools": self.tools.get_allowed_tools("writer"),
                "max_turns": 5,
                "budget": 0.10,
            },
            "reviewer": {
                "system_prompt": "You are a code reviewer. ...",
                "tools": self.tools.get_allowed_tools("reviewer"),
                "max_turns": 5,
                "budget": 0.10,
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
                await self.state.log(agent_name, "complete",
                    f"cost=${msg.total_cost_usd:.4f}, turns={msg.num_turns}")
                break

        output = "\n".join(results) if results else "No output produced."
        await self.state.store(f"{agent_name}_last_result", output)
        return output
```

**Design decisions:**
- Uses `query()` (not `ClaudeSDKClient`) for agents because each sub-agent is one-shot stateless — it receives a task, works, returns. No multi-turn needed.
- Each agent gets the SAME `mcp_servers` dict (shared tools) but DIFFERENT `allowed_tools` list. This means all agents technically have access to the shared MCP server, but the researcher also gets `Bash` while the reviewer does not get `Write`.
- Budget per agent is capped individually ($0.10-0.15). Combined with the orchestrator's $1.00 budget, the system cannot exceed $1.00 total.

**Verify:** After `run_agent("researcher", "...")`, check `monitor.agent_costs["researcher"]` > 0 and `state.task_results["researcher_last_result"]` is non-empty.

### Step 6: Implement MASRouter (Layer 1) (~20 min)

The Router creates all layers, then defines orchestrator-only MCP tools that close over `self.agent_pool`.

```python
class MASRouter:
    def __init__(self):
        self.state = SharedState()
        self.monitor = MonitoringLayer()
        self.safety = SafetyLayer(self.state, self.monitor)
        self.tools = ToolRegistry(self.state)
        self.agent_pool = AgentPool(self.state, self.tools, self.safety, self.monitor)

    async def run(self, user_request: str):
        # Define orchestrator dispatch tools (closures over self.agent_pool)
        @tool("dispatch_agent", "Dispatch a task to a specialist agent",
              {"agent_name": str, "task": str})
        async def dispatch_tool(args):
            result = await self.agent_pool.run_agent(args["agent_name"], args["task"])
            return {"content": [{"type": "text", "text": result}]}

        @tool("dispatch_parallel", "Dispatch tasks to multiple agents in parallel",
              {"tasks": list})
        async def dispatch_parallel_tool(args):
            results = {}
            async with anyio.create_task_group() as tg:
                for item in args["tasks"]:
                    async def _run(i=item):
                        r = await self.agent_pool.run_agent(i["agent"], i["task"])
                        results[i["agent"]] = r
                    tg.start_soon(_run)
            return {"content": [{"type": "text", "text": json.dumps(results, default=str)}]}

        # ... store_final, get_memory tools ...

        orch_server = create_sdk_mcp_server("orch", "1.0.0",
            tools=[dispatch_tool, dispatch_parallel_tool, store_final_tool, get_memory_tool])

        options = ClaudeAgentOptions(
            system_prompt="""You are the master orchestrator...""",
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
                # ... handle messages, print monitoring summary at end
```

**Key architectural insight:** The Router uses `ClaudeSDKClient` (stateful, multi-turn capable) while individual agents use `query()` (stateless, one-shot). This is deliberate — the orchestrator may need multiple turns to decompose a complex task, dispatch agents, read results, and synthesize. Sub-agents are simple workers.

**Parallel dispatch:** `dispatch_parallel` uses `anyio.create_task_group()` to run multiple `run_agent()` concurrently. Each `run_agent()` spawns its own CLI subprocess. Since `SharedState` uses `anyio.Lock`, concurrent writes to shared memory are safe. The orchestrator decides when to use parallel vs sequential dispatch.

**Verify:** Full end-to-end run: user request → orchestrator dispatches researcher → researcher stores findings → orchestrator dispatches writer with context → writer produces output → orchestrator dispatches reviewer → reviewer provides feedback → orchestrator aggregates and stores final.

### Step 7: Write main() and verify end-to-end (~10 min)

```python
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

**Verify:**
1. Run: `python examples/mas/complex_mas.py`
2. Console shows agent dispatch logs (`[researcher] start: ...`, `[safety] pre_tool: Bash`, etc.)
3. 3 agents are dispatched (researcher, writer, reviewer)
4. SharedState has keys: `researcher_last_result`, `writer_last_result`, `reviewer_last_result`
5. Monitoring summary: per-agent costs listed, total < $1.00
6. No hook block events (unless agent tries dangerous command)

## 6. Edge Cases & Error Handling
| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| Unknown agent name | `dispatch_agent(agent_name="hacker", ...)` | Returns "Unknown agent: hacker" | Orchestrator retries with valid name |
| Agent exceeds budget | `max_budget_usd=0.15` hit | ResultMessage with `subtype="error_max_budget_usd"` | Monitor records partial cost; partial result returned |
| Rate limit during agent run | API rate limit hit | `RateLimitEvent` yielded from `query()` | Monitor records event; SDK auto-retries (built-in) |
| Blocked command in hook | Researcher tries `rm -rf /` | Hook returns `{"decision": "block", ...}` | Agent sees block reason, adjusts approach |
| SharedState concurrent access | 2 agents store() simultaneously | `anyio.Lock` serializes writes | No data corruption |
| CLI binary not found | No `claude` in PATH or bundled | `CLINotFoundError` raised | Catch at MASRouter.run() level, print instructions |
| Agent produces no output | Model returns empty response | `"No output produced."` stored | Orchestrator sees this, may re-dispatch |
| Parallel dispatch partial failure | 1 of 3 agents fails | `anyio.create_task_group()` propagates first exception | Wrap individual agents in try/except inside task group |

## 7. Acceptance Criteria
- **AC-1:** End-to-end execution completes — user request in, final aggregated result out
- **AC-2:** 3 agents dispatched — researcher, writer, reviewer each appear in `monitor.agent_costs`
- **AC-3:** Memory shared — researcher stores findings, writer retrieves them (same SharedState instance)
- **AC-4:** Hooks active — `SafetyLayer.pre_tool_hook` fires for every tool call (visible in agent_logs)
- **AC-5:** Cost tracking — `monitor.summary()` shows per-agent costs, total < $1.00
- **Negative-1:** If an agent tries a blocked command, hook blocks it and agent continues with alternative approach
- **Negative-2:** If CLI not found, `CLINotFoundError` raised with clear message

## 8. Technical Notes
- **SDK version:** v0.1.48
- **anyio.Lock:** Must be created in async context or with `default_factory=anyio.Lock`. NOT `asyncio.Lock` — SDK uses anyio internally.
- **MCP tool naming convention:** `mcp__{server_name}__{tool_name}` — must match exactly in `allowed_tools`
- **Hook registration:** `HookMatcher(matcher=None, hooks=[handler], timeout=10.0)` — `matcher=None` matches ALL tools
- **`continue_` gotcha:** In hook output, use `continue_` not `continue`. SDK converts via `_convert_hook_output_for_cli()` at query.py:34-50
- **`async_` gotcha:** Similarly, `async_` not `async` in hook outputs
- **File location:** `examples/mas/complex_mas.py` — create `examples/mas/` directory if not exists
- **Imports:** All from `claude_agent_sdk` — see Section 3 Input for full import list
- **Target length:** ~300 lines. Prioritize clarity over compression. Comments on each layer boundary.

## 9. Risks
- **Subprocess cost explosion:** Each agent spawns a separate CLI subprocess. 3 agents + 1 orchestrator = 4 subprocesses. If orchestrator dispatches agents multiple times (retry logic), cost multiplies. Mitigation: strict `max_budget_usd` per agent + overall $1.00 cap on orchestrator.
- **SharedState not persistent:** If process crashes, all state is lost. This is by design for a reference implementation. Production would need Redis/DB.
- **Hook timing:** PreToolUse hooks add latency to every tool call. With safety checks on every tool, researcher's 8 turns could mean 20+ hook invocations. Mitigation: keep hook logic fast (string matching, no I/O).
- **anyio.Lock contention:** If 3 agents run in parallel and all try to store() simultaneously, they serialize on the lock. For this reference implementation, contention is negligible. Production would use finer-grained locking or lock-free structures.
- **API changes:** Hook callback signatures and MCP server APIs may change between SDK versions. Pin to v0.1.48 in comments.

## Worklog

(Not yet started — waiting for MAS-4 completion)
