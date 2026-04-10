---
date: 2026-03-28
type: context
topic: design-principles
repo: claude-agent-sdk-python
sdk_version: 0.1.48
codebase: 15 files, 5331 LOC
---

# Design Principles: claude-agent-sdk-python

> Phân tích từ góc nhìn System Architect. Codebase: 15 files, 5331 LOC. Python 3.10+, async-only (anyio).
> Đây là WRAPPER SDK — giá trị nằm ở NHỮNG GÌ NÓ THÊM VÀO so với gọi CLI trực tiếp.

---

## 1. Architecture Overview

SDK wraps Claude Code CLI as subprocess, giao tiếp qua JSON streaming protocol trên stdin/stdout. Hai cánh cửa vào:

- **`query()`** (123 LOC) — one-shot, stateless. Tạo → dùng → huỷ. Đơn giản nhất.
- **`ClaudeSDKClient`** (498 LOC) — stateful, multi-turn, interrupt, hooks, runtime control. Full power.

Bốn tầng bên trong:
```
Public API (query.py, client.py)
  → InternalClient (145 LOC) — orchestrator cho query()
    → Query (678 LOC) — control protocol handler, THE BRAIN
      → SubprocessCLITransport (631 LOC) — CLI subprocess lifecycle
      → message_parser (251 LOC) — JSON → typed Python objects
```

Type system: [`types.py`](../../src/claude_agent_sdk/types.py) (1203 LOC) — file lớn nhất, định nghĩa MỌI THỨ: Message union, ClaudeAgentOptions (35+ fields), hooks, content blocks.

6 luồng chính: one-shot query, multi-turn session, SDK MCP tool call, hook callback dispatch, permission check, session history.

---

## 2. Core Components (Không thể thay thế)

Bốn thành phần mà xóa bất kỳ cái nào → SDK sụp đổ:

**Query** ([`query.py:53`](../../src/claude_agent_sdk/_internal/query.py#L53), 678 LOC) — Bộ não. Quản lý toàn bộ bidirectional communication: request_id matching, hook dispatch, MCP interception, initialize handshake. Xóa = SDK thành pipe thụ động.

**SubprocessCLITransport** ([`subprocess_cli.py:33`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L33), 631 LOC) — Đường ống. Binary discovery (bundled > PATH > known paths), subprocess lifecycle, JSON buffered parsing. Xóa = không spawn được CLI.

**[`types.py`](../../src/claude_agent_sdk/types.py)** (1203 LOC) — Từ điển. Message union ([line 950](../../src/claude_agent_sdk/types.py#L950)), ClaudeAgentOptions ([line 1035](../../src/claude_agent_sdk/types.py#L1035)), mọi hook type, content block. Mọi file khác import từ đây. Xóa = mất type safety hoàn toàn.

**[`message_parser.py`](../../src/claude_agent_sdk/_internal/message_parser.py)** (251 LOC) — Phiên dịch. JSON dict → typed Message objects. Forward-compatible (unknown types → None). Xóa = raw dicts thay vì typed objects.

---

## 3. Leverage Points (Nhỏ mà thay đổi toàn bộ)

Năm điểm tựa — mỗi cái chỉ vài dòng nhưng chi phối behavior lớn:

**`_convert_hook_output_for_cli()`** ([`query.py:34-50`](../../src/claude_agent_sdk/_internal/query.py#L34-L50), 16 LOC) — Mọi Python→CLI hook response đi qua đây. Maps `async_` → `async`, `continue_` → `continue`. Sai 1 mapping = silent failure cho TẤT CẢ hooks.

**`_is_streaming = True`** ([`subprocess_cli.py:44`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L44), 1 LOC) — Một boolean hardcoded. SDK LUÔN streaming internally. Thay thành False = mất bidirectional, mất hooks, mất MCP. Một dòng chi phối toàn bộ architecture.

**request_id counter** ([`query.py:98-102`](../../src/claude_agent_sdk/_internal/query.py#L98-L102), 5 LOC) — Multiplexed communication. Sai matching = responses lạc, deadlock.

**`_handle_control_request()` dispatch** ([`query.py:236`](../../src/claude_agent_sdk/_internal/query.py#L236), ~70 LOC) — Router cho MỌI THỨ CLI gửi SDK. Thêm 1 `elif subtype ==` = thêm 1 capability.

**`_find_cli()` fallback chain** ([`subprocess_cli.py:64-95`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L64-L95), 32 LOC) — bundled > shutil.which > 6 known paths. User KHÔNG CẦN configure. Tiny code, huge UX.

---

## 4. Design Principles & Rationale

### Deep: Always-Streaming (LSP Pattern)
SDK luôn dùng stream-json mode internally ([`subprocess_cli.py:44`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py#L44)). Tại sao? Vì streaming là MANDATORY cho value proposition của SDK — without it, bạn chỉ có `subprocess.run()`. Streaming enables: real-time messages, control protocol interleaving, hooks, MCP. Industry parallel: **Language Server Protocol** — same JSON-over-stdio streaming.

### Deep: request_id Multiplexing (JSON-RPC Pattern)
Single stdin/stdout channel, multiple concurrent request types. Counter + dict ([`query.py:98-102`](../../src/claude_agent_sdk/_internal/query.py#L98-L102)). Tại sao không separate channels? Vì one subprocess = two pipes (stdin/stdout) — thêm channel = thêm subprocess. Minimalist, proven by **JSON-RPC 2.0** (same id-based correlation).

### Deep: In-Process MCP (VSCode Extension Pattern)
@tool decorator → SDK intercepts mcp_message → executes locally. CLI "tưởng" external server. Tại sao? Zero-latency, shared Python state, single-process deployment. Tradeoff: tool crash = SDK crash. Industry parallel: **VSCode extensions** — in-process for speed, subprocess for isolation.

### Brief: Hook System = Observer Pattern
CLI notifies SDK at lifecycle points. Express.js middleware, Git hooks, pytest hooks.

### Brief: Error Hierarchy = Exception Chain
`ClaudeSDKError → CLIConnectionError → CLINotFoundError`. 56 LOC. Python `requests` library pattern.

### Brief: Two Entry Points = Facade Pattern
query() for simple, ClaudeSDKClient for full control. Redis get() vs pipeline(), Anthropic SDK create() vs stream().

---

## 5. Mental Shortcuts & Exercises

### Shortcuts
1. **Đọc [`types.py`](../../src/claude_agent_sdk/types.py) trước** — 1203 LOC, spec sống, mọi thứ khác reference nó
2. **Đếm request_id trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py) ** — mỗi usage = 1 kiểu message protocol
3. **async_ KHÔNG PHẢI async** — trailing underscore cho Python keywords, silent failure nếu sai
4. **SDK = Translator** — Python async ↔ CLI subprocess JSON. Query = translator, Transport = pipe
5. **Debug: stderr → [`query.py`](../../src/claude_agent_sdk/_internal/query.py) → [`types.py`](../../src/claude_agent_sdk/types.py)** — follow the data flow

### Exercises
1. **@tool custom tool** — create_sdk_mcp_server, query with mcp_servers option. Principle: Open/Closed.
2. **PreToolUse hook** — register HookMatcher with async callback. Gotcha: `continue_` not `continue`. Principle: Observer.
3. **Error hierarchy trace** — CLINotFoundError → ProcessError → CLIJSONDecodeError. Thought experiment. Principle: Exception chain.
