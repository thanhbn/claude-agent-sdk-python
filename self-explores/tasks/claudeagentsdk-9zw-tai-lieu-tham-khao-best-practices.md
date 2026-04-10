---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-9zw
title: "[MAS-6] Tai lieu tham khao & Best Practices"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
tags: [mas, best-practices, lessons-learned, documentation, claude-agent-sdk-python]
---

# [MAS-6] Tai lieu tham khao & Best Practices — Detailed Design

## 1. Objective
Rut ra lessons learned tu MAS-1 den MAS-5. Tong hop thanh tai lieu tham khao co cau truc voi 5 categories: (1) Patterns that worked, (2) Anti-patterns, (3) Cost optimization, (4) Error handling, (5) Testing strategies. Moi best practice co code example cu the, when to use, when NOT to use. Bo sung vao learning guide section 7. AC: 5+ new best practices voi code examples. LAM SAU MAS-1 den MAS-5.

## 2. Scope
**In-scope:**
- Extract patterns/anti-patterns tu MAS-1 (SDK architecture), MAS-2 (@tool mastery), MAS-3 (advanced features), MAS-4 (simple MAS), MAS-5 (complex MAS)
- 5 categories: Patterns, Anti-patterns, Cost optimization, Error handling, Testing strategies
- Each best practice: pattern name, code example, when to use, when NOT to use
- Append to `self-explores/tasks/mas-learning-guide.md` Section 7
- Minimum 5 new best practices with verified code examples

**Out-of-scope:**
- Rewriting MAS-1 to MAS-5 content (only extracting lessons)
- Production deployment guides (this is learning-focused)
- Performance benchmarks (cost is tracked, not latency)
- SDK contribution guidelines

## 3. Input / Output
**Input:**
- MAS-1 worklog: SDK architecture overview (mas-learning-guide.md Section 1)
- MAS-2 worklog: @tool decorator patterns (mas-learning-guide.md Section 2-3)
- MAS-3 worklog: Hooks, permissions, agents, streaming (mas-learning-guide.md Section 4)
- MAS-4 worklog: Simple MAS orchestrator pattern (mas-learning-guide.md Section 5)
- MAS-5 worklog: Complex MAS full-layer (claudeagentsdk-lav.md + mas-learning-guide.md Section 6)
- Existing best practices in Section 7.3-7.4 of learning guide (lines 1374-1423)

**Output:**
- Updated Section 7 in `self-explores/tasks/mas-learning-guide.md` with 5+ new best practices
- Each practice follows the standard template (see Section 5)
- Best practices verified against actual SDK source code

## 4. Dependencies
- **Depends on:** MAS-1, MAS-2, MAS-3, MAS-4, MAS-5 — ALL must be completed first
- **Blocks:** nothing (this is the final task in the MAS series)

## 5. Flow xu ly

### Best Practice Template

Every best practice MUST follow this template:

```markdown
### BP-{N}: {Pattern Name}

**Category:** {Patterns | Anti-patterns | Cost Optimization | Error Handling | Testing}
**Source:** MAS-{N} ({brief context where this was discovered})
**When to use:** {1-2 sentences describing the scenario}
**When NOT to use:** {1-2 sentences describing when this pattern is wrong}

**Code example (DO):**
```python
# Good: descriptive comment
from claude_agent_sdk import ...
# working code
```

**Code example (DON'T):**
```python
# Bad: descriptive comment why this fails
# broken or suboptimal code
```

**Verification:** {How to test that this practice works}
```

### Step 1: Extract Lessons from MAS-1 to MAS-5 (~20 min)

Read all 5 MAS worklogs + learning guide sections. For each, identify:
- What worked well (pattern)
- What failed or caused confusion (anti-pattern)
- What cost more than expected (cost lesson)
- What error handling was needed (error lesson)
- What was hard to test (testing lesson)

**Expected extractions per task:**

| Task | Expected Patterns | Expected Anti-patterns |
|------|------------------|----------------------|
| MAS-1 (Architecture) | query() vs ClaudeSDKClient selection, types.py as spec | Mixing sync/async, ignoring Message union type |
| MAS-2 (@tool) | @tool closure pattern, error return with is_error | Blocking I/O in handlers, exception propagation |
| MAS-3 (Advanced) | Hook continue_ gotcha, HookMatcher with matcher=None | Using `continue` instead of `continue_`, missing hook timeout |
| MAS-4 (Simple MAS) | Tool-as-agent-dispatch, query() for sub-agents | Sharing ClaudeSDKClient between agents, no budget caps |
| MAS-5 (Complex MAS) | SharedState with anyio.Lock, layered safety, per-agent budgets | Lock-free shared state, single global budget |

**Verify:** At least 2 lessons extracted per task, totaling 10+ candidates. Select best 5-8 for final document.

### Step 2: Organize by Category (~10 min)

Group extracted lessons into 5 categories:

**Category 1: Patterns That Worked**
- Tool-as-agent-dispatch (MAS-4, MAS-5)
- SharedState with anyio.Lock for cross-agent memory (MAS-5)
- query() for one-shot sub-agents, ClaudeSDKClient for orchestrator (MAS-4, MAS-5)
- @tool closure capturing state reference (MAS-2, MAS-5)

**Category 2: Anti-patterns**
- Blocking I/O in @tool handlers (MAS-2)
- Using Python keywords in hook output (continue vs continue_) (MAS-3)
- Sharing ClaudeSDKClient across concurrent agents (MAS-4)
- Missing `is_error: True` in tool error returns (MAS-2)
- No budget cap per agent (cost explosion) (MAS-5)

**Category 3: Cost Optimization**
- Per-agent `max_budget_usd` caps (MAS-5)
- `max_turns` to limit runaway agents (MAS-4, MAS-5)
- Use `query()` for sub-agents (lighter than ClaudeSDKClient) (MAS-4)
- Researcher read-only tools (fewer write-triggered turns) (MAS-5)

**Category 4: Error Handling**
- Exception hierarchy: CLINotFoundError < CLIConnectionError < ClaudeSDKError (MAS-1)
- Tool error return pattern: `{"is_error": True}` (MAS-2)
- Hook block response for safety (MAS-5)
- ResultMessage.is_error check after every agent run (MAS-4, MAS-5)

**Category 5: Testing Strategies**
- Mock transport for unit tests (from SDK test suite)
- SharedState concurrent access test with anyio.create_task_group (MAS-5)
- Hook behavior test: verify block response format (MAS-5)
- End-to-end integration test with real CLI (MAS-5)

**Verify:** Each category has at least 1 best practice. No duplicates across categories.

### Step 3: Write Best Practices with Code Examples (~30 min)

Write each best practice following the template from above. Target: 5-8 best practices, each with DO and DON'T code examples.

**Candidate best practices (prioritized):**

**BP-1: Tool-as-Agent-Dispatch Pattern** (Category: Patterns)
```python
# DO: MCP tool handler that spawns a sub-agent via query()
@tool("dispatch_researcher", "Send research task to specialist", {"task": str})
async def dispatch_researcher(args):
    options = ClaudeAgentOptions(
        system_prompt="You are a research specialist...",
        max_turns=5,
        max_budget_usd=0.10,
    )
    results = []
    async for msg in query(prompt=args["task"], options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    results.append(block.text)
        elif isinstance(msg, ResultMessage):
            break
    return {"content": [{"type": "text", "text": "\n".join(results)}]}

# DON'T: Try to use ClaudeSDKClient inside a tool handler
@tool("bad_dispatch", "This will cause issues", {"task": str})
async def bad_dispatch(args):
    async with ClaudeSDKClient(options) as client:  # Nested client = nested subprocess
        await client.query(args["task"])             # Complex lifecycle management
        # ... hard to collect results cleanly
```
Verification: Run orchestrator, verify sub-agent executes and returns result string.

**BP-2: SharedState with anyio.Lock** (Category: Patterns)
```python
# DO: Thread-safe shared state
@dataclass
class SharedState:
    data: dict[str, Any] = field(default_factory=dict)
    _lock: anyio.Lock = field(default_factory=anyio.Lock)

    async def store(self, key: str, value: Any):
        async with self._lock:
            self.data[key] = value

# DON'T: Unprotected dict
class BadState:
    def __init__(self):
        self.data = {}  # No lock! Race condition with parallel agents!

    async def store(self, key, value):
        self.data[key] = value  # Concurrent writes can corrupt
```
Verification: Run 3 parallel store() operations via anyio.create_task_group(), verify all 3 keys present.

**BP-3: Hook Output Keyword Gotcha** (Category: Anti-patterns)
```python
# DO: Use trailing underscore for Python keywords
return {"continue_": True}   # SDK converts to "continue" for CLI
return {"async_": True}       # SDK converts to "async" for CLI

# DON'T: Use Python keywords directly
return {"continue": True}    # SyntaxError! "continue" is a keyword
return {"async": True}        # SyntaxError! "async" is a keyword
```
Verification: Write hook that returns `continue_: True`, verify agent continues. Attempt `continue` in Python REPL, observe SyntaxError.

**BP-4: Per-Agent Budget Caps** (Category: Cost Optimization)
```python
# DO: Cap each agent individually
researcher_options = ClaudeAgentOptions(max_budget_usd=0.15, max_turns=8)
writer_options = ClaudeAgentOptions(max_budget_usd=0.10, max_turns=5)
orchestrator_options = ClaudeAgentOptions(max_budget_usd=1.00, max_turns=20)

# DON'T: Single global budget with no per-agent limits
options = ClaudeAgentOptions(max_budget_usd=5.00)  # One runaway agent eats all $5
```
Verification: Run MAS, check `monitor.agent_costs` — each agent under its cap, total under $1.00.

**BP-5: Tool Error Return Pattern** (Category: Error Handling)
```python
# DO: Return is_error flag
@tool("safe_tool", "Handles errors gracefully", {"path": str})
async def safe_tool(args):
    try:
        data = await read_async(args["path"])
        return {"content": [{"type": "text", "text": data}]}
    except FileNotFoundError:
        return {"content": [{"type": "text", "text": f"File not found: {args['path']}"}],
                "is_error": True}

# DON'T: Let exceptions propagate
@tool("unsafe_tool", "Crashes on error", {"path": str})
async def unsafe_tool(args):
    data = open(args["path"]).read()  # FileNotFoundError propagates!
    return {"content": [{"type": "text", "text": data}]}
```
Verification: Call tool with non-existent path, verify `is_error: True` in result (not an unhandled exception).

**BP-6: query() for Sub-agents, ClaudeSDKClient for Orchestrator** (Category: Patterns)
```python
# DO: query() for one-shot sub-agents
async def run_sub_agent(task: str) -> str:
    async for msg in query(prompt=task, options=agent_options):
        ...  # Collect results
    return result  # Clean, no cleanup needed

# DO: ClaudeSDKClient for orchestrator (may need multi-turn)
async with ClaudeSDKClient(options=orch_options) as client:
    await client.query(user_request)
    async for msg in client.receive_response():
        ...  # Multiple tool calls, orchestration decisions

# DON'T: ClaudeSDKClient for every sub-agent
async with ClaudeSDKClient(options) as agent1:  # Heavyweight
    async with ClaudeSDKClient(options) as agent2:  # More overhead
        ...  # Nested context managers, complex lifecycle
```
Verification: Measure setup time — query() starts faster than ClaudeSDKClient for one-shot tasks.

**BP-7: ResultMessage Checking** (Category: Error Handling)
```python
# DO: Always check ResultMessage
async for msg in query(prompt=task, options=options):
    if isinstance(msg, ResultMessage):
        if msg.is_error:
            print(f"Agent failed: {msg.subtype}")
        if msg.subtype == "error_max_budget_usd":
            print("Budget exceeded!")
        print(f"Cost: ${msg.total_cost_usd:.4f}, Turns: {msg.num_turns}")
        break

# DON'T: Ignore ResultMessage
async for msg in query(prompt=task, options=options):
    if isinstance(msg, AssistantMessage):
        print(msg)  # Only prints assistant messages
    # ResultMessage silently dropped — no error detection, no cost tracking!
```
Verification: Set `max_budget_usd=0.001`, verify ResultMessage with `subtype="error_max_budget_usd"` is caught.

**Verify:** Each best practice has: name, category, source task, when to use, when NOT to use, DO code, DON'T code, verification method.

### Step 4: Verify Best Practices Against SDK Source (~15 min)

For each code example, verify:
1. Import paths exist in `src/claude_agent_sdk/__init__.py`
2. Type signatures match [`types.py`](../../src/claude_agent_sdk/types.py)
3. Hook callback signatures match [`query.py`](../../src/claude_agent_sdk/_internal/query.py)
4. `is_error` field exists in MCP tool result type
5. `ResultMessage` fields: `is_error`, `total_cost_usd`, `num_turns`, `subtype` exist

**Verification checklist:**
- [ ] `ClaudeAgentOptions` has `max_budget_usd`, `max_turns`, `system_prompt`, `allowed_tools`, `mcp_servers`, `hooks`, `can_use_tool`, `stderr` fields
- [ ] `query()` returns `AsyncIterator[Message]`
- [ ] `ResultMessage` has `total_cost_usd`, `num_turns`, `is_error`, `subtype`
- [ ] `@tool(name, description, schema)` decorator signature
- [ ] `create_sdk_mcp_server(name, version, tools)` function signature
- [ ] `HookMatcher(matcher=None, hooks=[...])` constructor
- [ ] Hook handler signature: `async def handler(input, tool_use_id, context) -> HookJSONOutput`

**Verify:** All code examples pass a manual import trace through the SDK source.

### Step 5: Append to Learning Guide Section 7 (~10 min)

Append new best practices to `self-explores/tasks/mas-learning-guide.md` Section 7, after the existing content (7.1-7.6). New subsection: `### 7.7 Best Practices from MAS-1 to MAS-5`.

Format:
```markdown
### 7.7 Best Practices from MAS-1 to MAS-5 (Lessons Learned)

> Rut ra tu qua trinh xay dung 5 MAS tasks. Moi practice da verify voi SDK v0.1.48.

#### BP-1: Tool-as-Agent-Dispatch Pattern
...
```

**Verify:** Read updated Section 7 — has 7.1-7.7, subsection 7.7 contains 5+ best practices.

## 6. Edge Cases & Error Handling
| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| MAS task not completed | MAS-3 still in progress | Cannot extract lessons from it | Wait or extract from available tasks, note gap |
| Duplicate best practice | Already in Section 7.3-7.4 | Skip or reference existing | Deduplicate, add "see also 7.3" |
| Code example outdated | SDK API changed since v0.1.48 | Example may not compile | Pin version in example comment |
| No anti-pattern found for a task | MAS-1 had no failures | Acceptable — not all tasks produce anti-patterns | Fill category from other tasks |
| Section 7 format conflict | Existing numbering broken | New section misaligned | Re-read section 7 before appending |

## 7. Acceptance Criteria
- **AC-1:** 5+ new best practices documented, each following the template (name, category, when to use, when NOT to use, code examples)
- **AC-2:** Each best practice has both DO and DON'T code examples
- **AC-3:** Best practices cover all 5 categories (patterns, anti-patterns, cost, error handling, testing)
- **AC-4:** Code examples verified against SDK v0.1.48 source (import paths, type signatures)
- **AC-5:** Content appended to learning guide Section 7 as subsection 7.7
- **Negative-1:** If a MAS task is incomplete, document gap and fill from available tasks (still meet 5+ minimum)

## 8. Technical Notes
- **SDK version:** v0.1.48 — pin in all code examples
- **Existing Section 7 content:** 7.1 (docs links), 7.2 (source refs), 7.3 (DO list), 7.4 (DON'T list + anti-patterns code), 7.5 (where to find), 7.6 (learning path)
- **Template enforcement:** Every BP MUST have the 6 fields: name, category, source, when to use, when NOT to use, code examples
- **Vietnamese + English:** Description in Vietnamese (matching learning guide style), code comments in English
- **Import verification:** All imports from `claude_agent_sdk` — no internal `_internal` imports in examples

## 9. Risks
- **Superficial lessons:** Risk of stating obvious patterns ("use try/except"). Mitigation: each BP must reference specific MAS task where the lesson was discovered, with concrete code showing the failure mode.
- **Code rot:** SDK evolves, examples break. Mitigation: pin to v0.1.48, add "tested with" comment.
- **Category imbalance:** Testing strategies may have fewer examples than Patterns. Mitigation: acceptable — quality over quantity. Minimum 1 per category.
- **Overlap with existing 7.3/7.4:** Some DO/DON'T items already exist. Mitigation: reference existing items, only add genuinely new insights from MAS build experience.

## Worklog

(Not yet started — waiting for MAS-1 to MAS-5 completion)
