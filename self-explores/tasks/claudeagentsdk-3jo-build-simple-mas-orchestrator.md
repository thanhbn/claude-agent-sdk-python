---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-3jo
title: "[MAS-4] Build Simple MAS: Orchestrator + 2 Specialist Agents"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
depends_on: [MAS-2, MAS-3]
blocks: [MAS-5]
tags: [mas, multi-agent, orchestrator, researcher, writer, claude-agent-sdk-python]
---

# [MAS-4] Build Simple MAS: Orchestrator + 2 Specialist Agents — Detailed Design

## 1. Objective
Build a working Multi-Agent System (MAS) where an **Orchestrator** agent coordinates two specialist agents — **Researcher** and **Writer** — to analyze the claude-agent-sdk-python codebase and produce a technical overview document. The Orchestrator uses MCP tools (`ask_researcher`, `ask_writer`, `save_result`) that internally call `query()` one-shot to spawn specialist agents. This is the first end-to-end MAS demonstrating the SDK's agent orchestration capabilities.

**Use case:** "Analyze the SDK's architecture and write a technical overview"
- Researcher: reads code with Read/Glob/Grep tools, returns structured findings
- Writer: takes research findings, writes polished markdown document
- Orchestrator: coordinates the workflow, manages budgets, saves output

**Budget constraints:**
- Orchestrator: max $0.30
- Researcher: max $0.15 per call
- Writer: max $0.10 per call
- Total system: < $0.60

## 2. Scope
**In-scope:**
- `examples/mas/simple_mas.py` — complete working MAS
- `output/sdk-overview.md` — generated output document
- Orchestrator with 3 MCP tools: ask_researcher, ask_writer, save_result
- Error handling: retry failed agent calls 1x before reporting failure
- Cost tracking: print per-agent and total cost
- Logging: print workflow steps as they happen

**Out-of-scope:**
- Complex MAS patterns (agent memory, shared state, parallel agents)
- Production features (persistent storage, rate limit handling, circuit breakers)
- Multiple researcher/writer instances in parallel
- Streaming output from specialist agents to orchestrator
- Interactive/multi-turn orchestrator (single request, full execution)

## 3. Input / Output
**Input:**
- User request string: "Analyze the architecture of this Python SDK project and write a technical overview document."
- SDK source code at `src/claude_agent_sdk/` (15 files, ~5300 LOC) — target of analysis
- MAS-3 knowledge: hooks, permissions, AgentDefinition, dynamic control
- MAS-2 knowledge: @tool decorator, create_sdk_mcp_server, MCP tool naming

**Output:**
- `examples/mas/simple_mas.py` — the MAS implementation (~200-300 LOC)
- `output/sdk-overview.md` — generated overview document (content varies per run)
- Console output: step-by-step execution log with costs

**Expected console output:**
```
[Orchestrator] Processing: Analyze the architecture of this Python SDK...
  >> Calling: mcp__agents__ask_researcher({"query": "..."})
  [Researcher] Analyzing... (cost: $0.08)
  >> Calling: mcp__agents__ask_writer({"topic": "...", "research_context": "..."})
  [Writer] Writing... (cost: $0.06)
  >> Calling: mcp__agents__save_result({"content": "...", "filename": "output/sdk-overview.md"})
  [Save] Written to output/sdk-overview.md
[Done] Total cost: $0.35, Turns: 8
```

## 4. Dependencies
- **MAS-2** (@tool + MCP server patterns) — cần biết cách tạo MCP tools cho orchestrator
- **MAS-3** (hooks + permissions + agents) — cần biết advanced features để layer safety/control
- **Blocks MAS-5** (Complex MAS) — builds on this pattern with more agents, shared state, parallel execution

## 5. Flow xu ly

### Step 1: Create directory structure (~2 phut)

```bash
mkdir -p examples/mas
mkdir -p output
```

**Verify:** Both directories exist

### Step 2: Design Specialist Agent Functions (~10 phut)

**Researcher agent** — Uses `query()` one-shot with read-only tools:

```python
async def call_researcher(research_query: str) -> tuple[str, float]:
    """
    Spawn researcher agent via query() one-shot.
    Returns (findings_text, cost_usd).
    """
    options = ClaudeAgentOptions(
        system_prompt=(
            "You are a code research specialist analyzing a Python SDK. "
            "Use Read, Glob, and Grep tools to investigate the codebase. "
            "Return structured findings with:\n"
            "- Summary (2-3 sentences)\n"
            "- Key Components (bulleted list with file:line)\n"
            "- Notable Patterns (design patterns observed)\n"
            "Be precise and cite file paths."
        ),
        tools=["Read", "Glob", "Grep"],
        allowed_tools=["Read", "Glob", "Grep"],
        max_turns=8,
        max_budget_usd=0.15,
    )

    results = []
    cost = 0.0

    async for msg in query(prompt=research_query, options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    results.append(block.text)
        elif isinstance(msg, ResultMessage):
            cost = msg.total_cost_usd or 0.0
            break

    return ("\n".join(results) if results else "No findings.", cost)
```

**Writer agent** — Uses `query()` one-shot with no tools (pure generation):

```python
async def call_writer(topic: str, research_context: str) -> tuple[str, float]:
    """
    Spawn writer agent via query() one-shot.
    Returns (document_text, cost_usd).
    """
    options = ClaudeAgentOptions(
        system_prompt=(
            "You are a technical writer. "
            "Write clear, well-structured markdown documents. "
            "Use headings, bullet points, and code examples. "
            "Base your writing ONLY on the research context provided — "
            "do not invent facts. Target: 500-1000 words."
        ),
        max_turns=3,
        max_budget_usd=0.10,
    )

    prompt = f"""Write a technical overview document about: {topic}

Research findings to base your writing on:
---
{research_context}
---

Output format: Markdown with ## headings. Include an architecture section,
key components section, and design patterns section."""

    results = []
    cost = 0.0

    async for msg in query(prompt=prompt, options=options):
        if isinstance(msg, AssistantMessage):
            for block in msg.content:
                if isinstance(block, TextBlock):
                    results.append(block.text)
        elif isinstance(msg, ResultMessage):
            cost = msg.total_cost_usd or 0.0
            break

    return ("\n".join(results) if results else "Could not generate content.", cost)
```

**Design decisions:**
- `query()` (not `ClaudeSDKClient`) for specialists — stateless, one-shot, simpler lifecycle
- Separate `max_budget_usd` per agent — cost isolation
- Researcher gets tools, Writer gets none — separation of concerns
- Return `tuple[str, float]` — content + cost for tracking

**Verify:** Both functions return (str, float), use query() internally, have budget caps

### Step 3: Design MCP Tools for Orchestrator (~10 phut)

Three MCP tools the orchestrator can call:

```python
@tool(
    "ask_researcher",
    "Ask the research agent to investigate a topic in the SDK codebase. "
    "Returns structured findings with file paths and patterns.",
    {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Research question to investigate",
            },
        },
        "required": ["query"],
    },
)
async def ask_researcher_tool(args):
    query_text = args["query"]
    print(f"  [Researcher] Investigating: {query_text[:80]}...")

    try:
        findings, cost = await call_researcher(query_text)
        print(f"  [Researcher] Done. Cost: ${cost:.4f}, Length: {len(findings)} chars")
        return {"content": [{"type": "text", "text": findings}]}
    except Exception as e:
        # Retry 1x on failure
        print(f"  [Researcher] Failed: {e}. Retrying...")
        try:
            findings, cost = await call_researcher(query_text)
            print(f"  [Researcher] Retry succeeded. Cost: ${cost:.4f}")
            return {"content": [{"type": "text", "text": findings}]}
        except Exception as retry_err:
            return {
                "content": [{"type": "text", "text": f"Research failed after retry: {retry_err}"}],
                "is_error": True,
            }


@tool(
    "ask_writer",
    "Ask the writer agent to create a document based on research findings. "
    "Returns polished markdown content.",
    {
        "type": "object",
        "properties": {
            "topic": {
                "type": "string",
                "description": "Topic/title for the document",
            },
            "research_context": {
                "type": "string",
                "description": "Research findings to base the writing on",
            },
        },
        "required": ["topic", "research_context"],
    },
)
async def ask_writer_tool(args):
    topic = args["topic"]
    context = args["research_context"]
    print(f"  [Writer] Writing about: {topic[:60]}...")

    try:
        document, cost = await call_writer(topic, context)
        print(f"  [Writer] Done. Cost: ${cost:.4f}, Length: {len(document)} chars")
        return {"content": [{"type": "text", "text": document}]}
    except Exception as e:
        # Retry 1x on failure
        print(f"  [Writer] Failed: {e}. Retrying...")
        try:
            document, cost = await call_writer(topic, context)
            print(f"  [Writer] Retry succeeded. Cost: ${cost:.4f}")
            return {"content": [{"type": "text", "text": document}]}
        except Exception as retry_err:
            return {
                "content": [{"type": "text", "text": f"Writing failed after retry: {retry_err}"}],
                "is_error": True,
            }


@tool(
    "save_result",
    "Save the final document content to a file on disk.",
    {
        "type": "object",
        "properties": {
            "content": {
                "type": "string",
                "description": "Document content to save",
            },
            "filename": {
                "type": "string",
                "description": "Output file path (e.g. output/sdk-overview.md)",
            },
        },
        "required": ["content", "filename"],
    },
)
async def save_result_tool(args):
    import pathlib
    filepath = pathlib.Path(args["filename"])
    filepath.parent.mkdir(parents=True, exist_ok=True)
    filepath.write_text(args["content"], encoding="utf-8")
    word_count = len(args["content"].split())
    print(f"  [Save] Written {word_count} words to {filepath}")
    return {"content": [{"type": "text", "text": f"Saved {word_count} words to {filepath}"}]}
```

**Design decisions:**
- Full JSON Schema for `input_schema` (production pattern, not shorthand dict)
- Retry 1x with `is_error=True` on permanent failure — orchestrator sees the error and can adapt
- `save_result` creates parent dirs automatically
- Print statements provide workflow visibility

**Verify:** 3 tools defined, each with error handling + retry, full JSON Schema input

### Step 4: Design Orchestrator (~15 phut)

```python
async def run_mas(user_request: str):
    """Main MAS orchestrator — ClaudeSDKClient with MCP tools."""

    # Build MCP server with agent tools
    agent_server = create_sdk_mcp_server(
        name="agents",
        version="1.0.0",
        tools=[ask_researcher_tool, ask_writer_tool, save_result_tool],
    )

    options = ClaudeAgentOptions(
        system_prompt="""You are an orchestrator managing a research team.

Available agents (call them via tools):
1. ask_researcher — Investigates the SDK codebase using Read/Glob/Grep tools
2. ask_writer — Writes polished documents from research findings
3. save_result — Saves final content to a file

WORKFLOW (follow this order):
1. Call ask_researcher with a focused research question about the SDK architecture
2. Call ask_researcher again if you need more detail on specific areas
3. Collect ALL research findings
4. Call ask_writer with the topic and ALL research context combined
5. Call save_result with the writer's output and filename "output/sdk-overview.md"

RULES:
- ALWAYS delegate to agents — do NOT write content yourself
- Pass the FULL research context to the writer (do not summarize)
- If an agent fails, try a different query angle
- Keep total budget under $0.60""",
        mcp_servers={"agents": agent_server},
        allowed_tools=[
            "mcp__agents__ask_researcher",
            "mcp__agents__ask_writer",
            "mcp__agents__save_result",
        ],
        max_turns=15,
        max_budget_usd=0.30,
    )

    print(f"[Orchestrator] Processing: {user_request[:80]}...\n")

    total_cost = 0.0

    async with ClaudeSDKClient(options=options) as client:
        async for msg in client.send_message(user_request):
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        # Only print orchestrator reasoning if non-trivial
                        text = block.text.strip()
                        if len(text) > 20:
                            print(f"[Orchestrator] {text[:200]}")
                    elif isinstance(block, ToolUseBlock):
                        print(f"  >> Calling: {block.name}({str(block.input)[:100]}...)")
            elif isinstance(msg, ResultMessage):
                total_cost = msg.total_cost_usd or 0.0
                print(f"\n{'='*60}")
                print(f"[Done] Orchestrator cost: ${total_cost:.4f}")
                print(f"       Turns: {msg.num_turns}")
                break

    # Verify output file exists
    import pathlib
    output_path = pathlib.Path("output/sdk-overview.md")
    if output_path.exists():
        word_count = len(output_path.read_text().split())
        print(f"       Output: {output_path} ({word_count} words)")
        print(f"       Total system cost: ~${total_cost:.4f} (orchestrator only; agent costs logged above)")
    else:
        print("       WARNING: Output file not created!")

    return total_cost
```

**Verify:**
- ClaudeSDKClient (stateful) for orchestrator — supports multi-turn tool calling
- System prompt enforces delegation workflow
- `allowed_tools` pre-approves all 3 agent tools (no permission prompts)
- Budget cap: $0.30 for orchestrator itself
- Output file verification at end

### Step 5: Write complete simple_mas.py (~15 phut)

Assemble all components into `examples/mas/simple_mas.py`:

```python
"""
Simple Multi-Agent System: Orchestrator + Researcher + Writer

Architecture:
  Orchestrator (ClaudeSDKClient, $0.30 budget)
    |
    +-- ask_researcher tool --> Researcher (query(), $0.15 budget)
    |                           Tools: Read, Glob, Grep
    |
    +-- ask_writer tool ------> Writer (query(), $0.10 budget)
    |                           Tools: none (pure generation)
    |
    +-- save_result tool -----> Local filesystem

Usage:
    python examples/mas/simple_mas.py

Output:
    output/sdk-overview.md
"""
import anyio
import json
from claude_agent_sdk import (
    tool, create_sdk_mcp_server,
    ClaudeSDKClient, ClaudeAgentOptions,
    query,
    AssistantMessage, ResultMessage, TextBlock, ToolUseBlock,
)

# ... (all functions from Steps 2-4 assembled here)

async def main():
    print("=" * 60)
    print("Simple MAS: Orchestrator + Researcher + Writer")
    print("=" * 60)
    print()

    try:
        cost = await run_mas(
            "Analyze the architecture of this Python SDK project "
            "(claude-agent-sdk-python in src/claude_agent_sdk/) and write "
            "a technical overview document. Focus on: entry points, internal "
            "layers, key design patterns, and extension points. "
            "Save the result as output/sdk-overview.md"
        )
        print(f"\nFinal orchestrator cost: ${cost:.4f}")
    except Exception as e:
        print(f"\n[ERROR] MAS failed: {e}")
        raise

if __name__ == "__main__":
    anyio.run(main)
```

**Verify:** File is self-contained, runs with `python examples/mas/simple_mas.py`

### Step 6: Test end-to-end (~15 phut)

```bash
# Step 6a: Syntax check
python -c "import ast; ast.parse(open('examples/mas/simple_mas.py').read())"

# Step 6b: Run the MAS
python examples/mas/simple_mas.py

# Step 6c: Verify output
ls -la output/sdk-overview.md
wc -w output/sdk-overview.md  # Should be 500-1000 words

# Step 6d: Check content quality (manual)
head -30 output/sdk-overview.md
```

**Success criteria:**
1. No Python syntax errors
2. Orchestrator calls researcher at least 1x
3. Orchestrator calls writer at least 1x
4. `output/sdk-overview.md` exists and has >200 words
5. Total cost < $0.60

## 6. Edge Cases & Error Handling

| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| CLI binary not found | No claude CLI installed | `CLINotFoundError` at first query() call | Print install instructions, exit gracefully |
| Researcher returns empty | No findings from tools | `"No findings."` string returned | Orchestrator should retry with different query |
| Writer returns empty | Bad/empty research context | `"Could not generate content."` string | Orchestrator should pass more context |
| Agent call exceeds budget | Research query too broad | `ResultMessage` with `subtype="error_max_budget_usd"` | Return partial results collected so far |
| Agent call timeout | CLI subprocess hangs | Transport timeout after default period | Retry 1x, then return error to orchestrator |
| save_result path permission denied | Writing to protected directory | `PermissionError` from `pathlib.write_text()` | Return `is_error=True`, orchestrator picks another path |
| Orchestrator budget exhausted | Too many agent calls | `ResultMessage` with budget error | Print partial progress, warn user |
| MCP tool name mismatch | Typo in `allowed_tools` | Tool call rejected by permission system | Verify names match `mcp__{server}__{tool}` pattern |
| Researcher uses disallowed tool | CLI gives tool not in `tools` list | Tool call blocked | Researcher only gets Read/Glob/Grep |
| Network error during query() | Connection drops | `CLIConnectionError` raised | Retry 1x, then propagate error |
| Output file already exists | Previous run's output | Overwritten by `write_text()` | Intentional — latest run wins |
| Concurrent MAS runs | Two processes running | File write race condition | Not supported — single run only |

## 7. Acceptance Criteria

### Happy Path (Given/When/Then)
- **AC-1 (End-to-end run):** Given CLI binary is installed and API key is valid, When `python examples/mas/simple_mas.py` runs, Then orchestrator calls researcher 1+ times AND calls writer 1+ times AND saves output file (verified by console log showing `[Researcher]`, `[Writer]`, `[Save]` entries)

- **AC-2 (Output file):** Given MAS run completes successfully, When checking `output/sdk-overview.md`, Then file exists AND contains >200 words AND contains markdown headings (`##`) AND references SDK file paths (verified by `wc -w` and `grep "##"` and `grep "src/"`)

- **AC-3 (Cost control):** Given budget constraints ($0.30 orch, $0.15 researcher, $0.10 writer), When MAS run completes, Then total cost < $0.60 (verified by `[Done]` console output showing cost and per-agent cost logs)

- **AC-4 (Error retry):** Given researcher agent fails on first attempt, When retry logic triggers, Then retry is attempted 1x AND either succeeds OR returns `is_error=True` to orchestrator (verified by `[Researcher] Failed: ... Retrying...` in console)

### Negative Path
- **AC-5 (CLI missing):** Given CLI is not installed, When MAS runs, Then `CLINotFoundError` is raised within 5 seconds (not hanging forever)

- **AC-6 (Budget exceeded):** Given orchestrator budget is set to $0.01 (artificially low), When MAS runs, Then `ResultMessage` is received with budget error AND partial progress is printed

### Integration
- **AC-7 (Self-contained):** Given `examples/mas/simple_mas.py` exists, When checked, Then file has zero imports from other example files (only `claude_agent_sdk` and stdlib imports)

## 8. Technical Notes

### Architecture Diagram
```
                    User Request
                         |
                         v
              ┌──────────────────┐
              │   ORCHESTRATOR   │
              │                  │
              │ ClaudeSDKClient  │
              │ Budget: $0.30    │
              │ Turns: max 15    │
              │                  │
              │ System prompt:   │
              │ "You manage a    │
              │  research team"  │
              └───────┬──────────┘
                      │
          MCP Server: "agents"
          ┌───────────┼───────────┐
          │           │           │
          v           v           v
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Researcher│ │  Writer   │ │   Save   │
  │          │ │           │ │          │
  │ query()  │ │ query()   │ │ pathlib  │
  │ $0.15    │ │ $0.10     │ │ write    │
  │          │ │           │ │          │
  │ Tools:   │ │ Tools:    │ │ (sync)   │
  │ Read     │ │ (none)    │ │          │
  │ Glob     │ │           │ │          │
  │ Grep     │ │           │ │          │
  └──────────┘ └───────────┘ └──────────┘
```

### Data Flow
```
1. User → Orchestrator: "Analyze SDK and write overview"
2. Orchestrator → ask_researcher: {"query": "What are the entry points and layers of claude-agent-sdk?"}
3. ask_researcher → query() → CLI subprocess → reads src/ files → returns findings
4. Orchestrator receives findings as tool result
5. Orchestrator → ask_writer: {"topic": "SDK Architecture Overview", "research_context": "<findings>"}
6. ask_writer → query() → CLI subprocess → generates document → returns markdown
7. Orchestrator receives document as tool result
8. Orchestrator → save_result: {"content": "<markdown>", "filename": "output/sdk-overview.md"}
9. save_result → pathlib.write_text() → file on disk
10. Orchestrator reports completion with cost
```

### Key SDK Patterns Used
| Pattern | Where | Why |
|---------|-------|-----|
| `query()` one-shot | Researcher, Writer | Stateless, no multi-turn needed, clean lifecycle |
| `ClaudeSDKClient` context manager | Orchestrator | Multi-turn tool calling, stateful conversation |
| `@tool` + `create_sdk_mcp_server` | Orchestrator tools | In-process MCP tools, zero-latency dispatch |
| `allowed_tools` | Orchestrator options | Pre-approve tools to skip permission prompts |
| `max_budget_usd` per agent | All agents | Cost isolation — one agent can't burn another's budget |
| `tools` list on query() | Researcher | Restrict to read-only tools |
| `is_error` in tool result | Error handling | Tell Claude the tool failed so it can adapt |

### MCP Tool Naming
```python
# Server name: "agents"
# Tool names: ask_researcher, ask_writer, save_result
# In allowed_tools and Claude's view:
"mcp__agents__ask_researcher"   # Format: mcp__{server}__{tool}
"mcp__agents__ask_writer"
"mcp__agents__save_result"
```

### Cost Tracking Architecture
```
Orchestrator cost:  msg.total_cost_usd (from ResultMessage)
                    Includes: orchestrator LLM calls only
                    Does NOT include: agent subprocess costs

Agent costs:        Tracked per-call in call_researcher() / call_writer()
                    Returned as second element of tuple
                    Printed to console but NOT aggregated to orchestrator

True total:         Must be calculated manually:
                    total = orchestrator_cost + sum(researcher_costs) + sum(writer_costs)
```

**Important:** The orchestrator's `ResultMessage.total_cost_usd` only reflects the orchestrator's own LLM usage. Agent costs from `query()` calls inside MCP tools are separate subprocess invocations with their own cost tracking. The console logs show per-agent costs for manual verification.

### Error Retry Pattern
```python
# Pattern used in ask_researcher_tool and ask_writer_tool:
try:
    result, cost = await call_agent(args)
    return {"content": [{"type": "text", "text": result}]}
except Exception as e:
    # Retry 1x
    try:
        result, cost = await call_agent(args)
        return {"content": [{"type": "text", "text": result}]}
    except Exception as retry_err:
        # Report permanent failure to orchestrator
        return {
            "content": [{"type": "text", "text": f"Failed: {retry_err}"}],
            "is_error": True,  # Claude sees this as tool error, can adapt
        }
```

## 9. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Total cost exceeds $0.60 | Medium | Unexpected billing | Hard budget caps on each agent + orchestrator; terminate on budget error |
| Researcher returns low-quality findings | Medium | Writer produces generic doc | Orchestrator prompt instructs: call researcher multiple times for different aspects |
| Writer ignores research context | Low | Output doesn't reflect actual SDK | Writer system prompt: "Base writing ONLY on research provided" |
| Orchestrator writes content itself | Medium | Bypasses agent architecture | System prompt: "ALWAYS delegate, do NOT write yourself" + no Write tool |
| CLI subprocess leak | Low | Zombie processes | query() cleans up transport on exit; ClaudeSDKClient.__aexit__ handles cleanup |
| Race condition on output file | Low | Corrupted output | Single-threaded orchestrator; no parallel writes |
| Researcher tool calls take too long | Medium | Orchestrator times out | max_turns=8 on researcher; budget cap forces termination |
| MCP server registration fails | Low | Orchestrator has no tools | Catch error in create_sdk_mcp_server, report immediately |
| Output path traversal attack | Low | Write to unexpected location | save_result hardcodes output/ prefix or validates path |
| API key rate limiting | Medium | Agent calls fail intermittently | Retry 1x handles transient rate limits; RateLimitEvent in stream |

## Worklog

### [PENDING] Awaiting MAS-2 and MAS-3 completion
