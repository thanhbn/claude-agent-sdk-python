---
date: 2026-03-28
type: task-worklog
task: claudeagentsdk-ciw
title: "claude-agent-sdk-python — Deep Research (Tư duy Top 0.1%)"
status: open
detailed_at: 2026-03-28
detail_score: ready-for-dev
tags: [system-design, deep-research, design-principles, claude-agent-sdk-python]
---

# claude-agent-sdk-python — Deep Research — Detailed Design

## 1. Objective
Phân tích "tại sao họ thiết kế như vậy" cho 6 key decisions. 3 DEEP (full 4 points) + 3 BRIEF (principle + industry ref). Acknowledge TS SDK inheritance where applicable.

## 2. Scope
**In-scope:**
- 6 design decisions analyzed (3 deep, 3 brief)
- Each: nguyên lý gốc, tradeoff, history/inheritance, industry reference
- TS SDK inheritance acknowledged as valid finding

**Out-of-scope:**
- CLI internal design decisions
- Performance optimization decisions
- Future roadmap speculation

## 3. Input / Output
**Input:**
- Task 2 findings (Core Components, Leverage Points)
- Source code for tradeoff analysis
- Git log for historical context: `git log --oneline --since="2024-01-01" -- src/`
- TypeScript SDK reference (if available for comparison)

**Output:**
- Worklog with 3 deep + 3 brief decision analyses
- Format: Decision → Principle → Rationale → Industry Reference

## 4. Dependencies
- claudeagentsdk-3ht (Task 2) phải xong trước
- Chạy SONG SONG với Task 3 (Code Mapping)

## 5. Flow xử lý

### Step 1: Đọc Task 2 findings (~5 phút)
Read `self-explores/tasks/claudeagentsdk-3ht.md` → identify key design decisions.
**Verify:** 6 decisions listed and prioritized for deep vs brief

### Step 2: DEEP — Always-streaming stdin/stdout (~15 phút)
**Decision:** Both query() and ClaudeSDKClient use `--input-format stream-json` internally.
**Analysis:**
1. **Nguyên lý:** Uniform interface — single communication path regardless of API complexity
2. **Tradeoff:** vs batch mode (simpler but can't handle agents, large configs, streaming). Stream enables: real-time message delivery, control protocol interleaving, agent config via initialize request
3. **History:** Check `git log --all --oneline -- src/claude_agent_sdk/_internal/transport/` for evolution. Likely inherited from TS SDK.
4. **Industry ref:** LSP (Language Server Protocol) — same stdin/stdout JSON streaming. Docker CLI wrappers. MCP protocol itself.

Read: [`subprocess_cli.py`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py) — look for `stream-json` flag, command building.
**Verify:** All 4 points filled with code evidence

### Step 3: DEEP — Control Protocol request/response (~15 phút)
**Decision:** SDK uses request_id matching for bidirectional control over stdin/stdout.
**Analysis:**
1. **Nguyên lý:** Multiplexed communication — single channel, multiple concurrent request types
2. **Tradeoff:** vs separate channels (simpler but more processes). Single channel = one subprocess, less resource, but needs ID matching. vs REST API = no subprocess needed but loses streaming + CLI features
3. **History:** Check query.py for request_id usage patterns. Count distinct request types.
4. **Industry ref:** JSON-RPC 2.0 (id field), MCP protocol (request/response), HTTP/2 stream multiplexing

Read: [`query.py`](../../src/claude_agent_sdk/_internal/query.py) — grep for `request_id`, count message types handled.
**Verify:** All 4 points + code reference to request_id matching logic

### Step 4: DEEP — SDK MCP servers in-process (~15 phút)
**Decision:** @tool decorator creates MCP servers that run IN the SDK process, not as subprocess.
**Analysis:**
1. **Nguyên lý:** Inversion of Control — SDK intercepts tool calls from CLI, executes locally, returns results
2. **Tradeoff:** vs subprocess MCP (more isolated but slower IPC, more processes). In-process = zero-latency tool execution, shared Python state, but couples tool code to SDK process
3. **History:** Check [`__init__.py`](../../src/claude_agent_sdk/__init__.py) for @tool/create_sdk_mcp_server evolution. Check [`query.py`](../../src/claude_agent_sdk/_internal/query.py) for interception logic.
4. **Industry ref:** VSCode extensions (in-process), Webpack loaders (in-process transform), pytest plugins (in-process hooks)

Read: [`__init__.py`](../../src/claude_agent_sdk/__init__.py) — @tool decorator, [`query.py`](../../src/claude_agent_sdk/_internal/query.py) — MCP call handling.
**Verify:** All 4 points + clear in-process vs subprocess tradeoff documented

### Step 5: BRIEF — 3 remaining decisions (~25 phút)

**5a. Hook system (async callbacks):**
- Principle: Observer pattern — CLI notifies SDK at lifecycle points
- Industry ref: Express.js middleware, pytest hooks, Git hooks
- Note: Python async_ naming = language adaptation, not design choice

**5b. Error hierarchy (ClaudeSDKError → CLIConnectionError → CLINotFoundError):**
- Principle: Exception hierarchy — specific before general, categorized by cause
- Industry ref: Python requests library (ConnectionError → HTTPError), boto3 exceptions
- Read: [`_errors.py`](../../src/claude_agent_sdk/_errors.py) (56 LOC)

**5c. Two entry points (query() stateless vs ClaudeSDKClient stateful):**
- Principle: Facade pattern — simple API for simple use, full API for complex use
- Industry ref: Redis (redis.get vs pipeline), HTTP (requests.get vs Session), Anthropic SDK (client.messages.create vs streaming)
- Note: Likely inherited from TS SDK pattern

**Verify:** Each brief has principle + industry ref + 1 code reference

### Step 6: TS SDK Inheritance Check (~10 phút)
```bash
# Check if TypeScript SDK exists in nearby directories or references
grep -r "typescript\|TypeScript\|TS SDK" README.md CLAUDE.md src/ 2>/dev/null
git log --oneline | head -20  # Check for "port" or "migrate" in commit messages
```
For each decision, note: "Original design choice" vs "Inherited from TS SDK" vs "Python-specific adaptation"
**Verify:** Each decision tagged with origin

## 6. Edge Cases & Error Handling
| Case | Trigger | Expected | Recovery |
|------|---------|----------|----------|
| No git history | Squashed/imported | Skip historical context, increase industry refs | Add "(no commit history available)" |
| TS SDK not accessible | No reference repo | Note "likely inherited" without proof | Focus on industry parallels |
| Decision is trivial | Obvious choice | Brief instead of deep | Document as "industry standard, no alternative" |

## 7. Acceptance Criteria
- **Happy 1:** Given 6 decisions, When analysis complete, Then 3 have full 4-point analysis (deep)
- **Happy 2:** Given 3 brief decisions, When documented, Then each has principle name + 1 industry ref
- **Negative:** Given a decision inherited from TS SDK, When analyzed, Then tagged "inherited" with note on WHY TS SDK chose it

## 8. Technical Notes
- request_id pattern: grep -c "request_id" in [`query.py`](../../src/claude_agent_sdk/_internal/query.py) to count usages
- Message types in control protocol: initialize, user_message, interrupt, permission, hook_callback, mcp
- Error hierarchy in [`_errors.py`](../../src/claude_agent_sdk/_errors.py) is only 56 LOC — very concise, may not warrant deep analysis

## 9. Risks
- Over-attribution: assigning intent where none exists (SDK may just follow TS SDK conventions)
- Industry refs may feel forced for common patterns → mitigate by choosing MOST RELEVANT ref, not most impressive

## Phản biện (2026-03-28)

### Điểm chất lượng: 7/10 → fixed with 3+3 scope + inheritance tagging
- Clarified: 3 deep + 3 brief, TS SDK inheritance acknowledged

## Worklog

### [START] Auto-claimed from Task 3 completion

### DEEP 1: Always-Streaming Internal Design
**Decision:** `self._is_streaming = True` hardcoded ([`subprocess_cli.py:44`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L44)). Both query() and ClaudeSDKClient use `--input-format stream-json`.
**Nguyên lý:** Uniform Interface — single communication channel regardless of API surface.
**Tradeoff:** vs batch mode (simpler, fewer moving parts) → BUT streaming enables: (1) real-time message delivery, (2) control protocol interleaving on same channel, (3) large agent configs via initialize request, (4) hooks + MCP that require bidirectional communication. Streaming is MANDATORY for the SDK's value proposition — without it, you just have `subprocess.run()`.
**History:** `6ca3514 Initial Python SDK import` — streaming was there from day 1. Comment at [`subprocess_cli.py:42-43`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L42-L43): "Always use streaming mode internally (matching TypeScript SDK)". Inherited from TS SDK by design.
**Industry ref:** **LSP (Language Server Protocol)** — same JSON-over-stdin/stdout streaming pattern. Also: **MCP protocol** itself (JSON-RPC over stdio), **Docker CLI** (docker exec with streaming output). The pattern is proven for tool-wrapping scenarios where you need both structured requests AND streaming output.

### DEEP 2: Control Protocol via request_id Multiplexing
**Decision:** SDK uses `request_id` ([`query.py:98-102`](../../src/claude_agent_sdk/_internal/query.py#L98-L102)) to multiplex control requests/responses over single stdin/stdout channel.
**Nguyên lý:** Multiplexed Communication — one channel, many concurrent message types.
**Tradeoff:** vs separate channels (dedicated pipe per message type → simpler matching but more IPC). vs REST API (no subprocess needed but loses CLI features, streaming, local tools). vs gRPC (typed schema but heavy dependency, overkill for single subprocess). request_id approach: minimal overhead (counter + dict), works with existing stdin/stdout, handles concurrent hooks + permissions + MCP in same stream.
**History:** Core pattern unchanged since initial import. `pending_control_responses` (Event-based) and `pending_control_results` (dict-based) are the only state. Design is deliberately simple — no priority queues, no timeouts per request (except initialize_timeout).
**Industry ref:** **JSON-RPC 2.0** — uses `id` field for request/response correlation, exact same pattern. **HTTP/2 stream IDs** — multiplex requests over single connection. **MCP protocol** specification — built on JSON-RPC, uses same id matching.

### DEEP 3: SDK MCP Servers In-Process
**Decision:** @tool decorator creates MCP servers executed IN the SDK process. Query intercepts mcp_message at [`query.py:304`](../../src/claude_agent_sdk/_internal/query.py#L304), routes to in-process McpServer instead of subprocess.
**Nguyên lý:** Inversion of Control — CLI thinks it's talking to external MCP server, but SDK intercepts and executes locally.
**Tradeoff:** vs subprocess MCP servers (more isolated — tool crash doesn't kill SDK, but slower IPC + resource overhead + complexity). vs embedding tool logic in prompts (simpler but no structured tool calling). In-process: zero-latency execution, shared Python state (access to app's data), single process deployment. Downside: heavy tools block event loop, tool crash = SDK crash.
**History:** `242c719 Update MCP types to align with what Claude Code expects` — early commit shows MCP alignment was a priority. The in-process pattern emerged because Python MCP SDK lacks Transport abstraction (noted in code comment at [`query.py:394`](../../src/claude_agent_sdk/_internal/query.py#L394)), forcing manual JSONRPC routing.
**Industry ref:** **VSCode extensions** — run in-process within editor, similar interception pattern. **Webpack loaders** — transform code in-process during build. **pytest plugins** — hooks execute in same process. The pattern is standard for "plugin systems where latency matters more than isolation."

### BRIEF 4: Hook System (Observer Pattern)
**Principle:** Observer — CLI notifies SDK at lifecycle points (PreToolUse, PostToolUse, Stop). SDK dispatches to registered Python async callbacks.
**Industry ref:** Express.js middleware chain, Git hooks, pytest hooks, Webpack plugin `tap()`.
**Note:** Python keyword avoidance (async_ → async, continue_ → continue via _convert_hook_output_for_cli) is a Python-specific adaptation, not a design choice. Inherited pattern from TS SDK where field names are `async` and `continue` natively.

### BRIEF 5: Error Hierarchy
**Principle:** Exception Hierarchy — categorized by cause, specific before general.
`ClaudeSDKError → CLIConnectionError → CLINotFoundError` (inheritance chain). `ProcessError`, `CLIJSONDecodeError`, `MessageParseError` (siblings).
**Industry ref:** Python `requests` library (`ConnectionError → HTTPError`), `boto3` exceptions, `httpx` error hierarchy.
**Note:** Only 56 LOC ([`_errors.py`](../../src/claude_agent_sdk/_errors.py)). Clean, minimal, Pythonic. CLINotFoundError includes helpful install instructions in message — UX decision, not just engineering.

### BRIEF 6: Two Entry Points (Facade Pattern)
**Principle:** Facade — `query()` for simple one-shot use, `ClaudeSDKClient` for full control.
- query() = 123 LOC, delegates to InternalClient, tears down everything after
- ClaudeSDKClient = 498 LOC, async context manager, multi-turn, interrupt, runtime control
**Industry ref:** Redis (`redis.get()` vs `redis.pipeline()`), Anthropic SDK (`client.messages.create()` vs `client.messages.stream()`), `requests.get()` vs `requests.Session()`.
**Note:** Inherited from TS SDK pattern. Python adaptation adds async context manager (`async with ClaudeSDKClient() as client`).

### [COMPLETE] 3 deep + 3 brief = 6 decisions analyzed
- Deep: streaming (with LSP ref), request_id (with JSON-RPC ref), in-process MCP (with VSCode ref)
- Brief: hooks (Observer), errors (hierarchy), two entry points (Facade)
- TS SDK inheritance noted for: streaming (D1), hooks (B4), two entry points (B6)
