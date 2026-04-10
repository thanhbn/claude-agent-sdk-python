# Sơ đồ tuần tự — Claude Agent SDK Python

| # | Sơ đồ | Mô tả | Participants |
|---|---------|-------------|--------------|
| 1 | Luồng query() một lần | Vòng đời đầy đủ của truy vấn stateless một lần | UserApp, query(), InternalClient, Transport, Query, CLI |
| 2 | Phiên tương tác ClaudeSDKClient | Hội thoại hai chiều đa lượt kèm interrupt | UserApp, ClaudeSDKClient, Transport, Query, CLI |
| 3 | Dispatch Hook Callback | Hook callback do CLI khởi tạo, định tuyến đến Python handler | CLI, Transport, Query, UserHookFn |
| 4 | Gọi MCP Tool SDK In-Process | Thực thi MCP tool in-process qua decorator @tool | CLI, Transport, Query, MCPServer, ToolHandler |

---

## 1. Luồng query() một lần

Hiện toàn bộ vòng đời khi user gọi `query(prompt="...", options=...)`. SDK tạo `InternalClient`, thiết lập transport và `Query`, thực hiện bắt tay initialize, gửi prompt, stream phản hồi, và dọn dẹp toàn bộ.

```mermaid
sequenceDiagram
    title query() One-Shot Flow

    participant App as UserApp
    participant QF as query()
    participant IC as InternalClient
    participant T as SubprocessCLITransport
    participant Q as Query
    participant CLI as Claude CLI

    App->>QF: query(prompt, options)
    QF->>IC: InternalClient()
    QF->>IC: process_query(prompt, options)

    IC->>T: SubprocessCLITransport(prompt, options)
    IC->>T: connect()
    activate T
    T->>CLI: spawn subprocess<br/>--output-format stream-json<br/>--input-format stream-json
    activate CLI

    IC->>Q: Query(transport, is_streaming_mode=True)
    IC->>Q: start()
    Note over Q: Creates TaskGroup,<br/>starts _read_messages() task

    IC->>Q: initialize()
    Q->>T: write(control_request: initialize)
    T->>CLI: stdin: {"type":"control_request", "request":{"subtype":"initialize", ...}}
    CLI-->>T: stdout: {"type":"control_response", "response":{...capabilities...}}
    T-->>Q: control_response (initialize result)
    Q-->>IC: initialization complete

    Note over IC: String prompt path

    IC->>T: write(user_message JSON)
    T->>CLI: stdin: {"type":"user", "message":{"role":"user","content":"..."}}

    IC->>Q: wait_for_result_and_end_input()
    Note over Q: If hooks/MCP servers exist,<br/>waits for first result event

    loop Streaming Response
        CLI-->>T: stdout: JSON message (assistant, tool_use, etc.)
        T-->>Q: _read_messages() parses JSON
        Q-->>Q: routes to message stream
    end

    Q->>T: end_input()
    T->>CLI: close stdin

    CLI-->>T: stdout: {"type":"result", ...}
    T-->>Q: result message
    Q-->>Q: _first_result_event.set()

    loop Yield Messages
        IC->>Q: receive_messages()
        Q-->>IC: raw message dict
        IC->>IC: parse_message(data) → typed Message
        IC-->>QF: yield Message
        QF-->>App: yield Message
    end

    Note over Q: Stream ends with {"type":"end"}

    IC->>Q: close()
    Q->>Q: cancel TaskGroup
    Q->>T: close()
    T->>CLI: terminate()
    deactivate CLI
    deactivate T
```

**Điểm quan trọng từ mã nguồn:**
- `query()` trong [`query.py:11`](../../src/claude_agent_sdk/_internal/query.py#L11) tạo `InternalClient` và uỷ quyền cho `process_query()`
- `InternalClient.process_query()` trong [`client.py:44`](../../src/claude_agent_sdk/_internal/client.py#L44) điều phối toàn bộ vòng đời
- Transport luôn dùng `--input-format stream-json` (dòng 331 trong subprocess_cli.py)
- Với string prompt, user message được ghi vào stdin sau initialize (client.py:126-133)
- `wait_for_result_and_end_input()` giữ stdin mở nếu hooks/MCP servers cần giao tiếp hai chiều

---

## 2. Phiên tương tác ClaudeSDKClient

Hiện một phiên hội thoại đa lượt dùng `ClaudeSDKClient` như async context manager, bao gồm message tiếp nối và khả năng interrupt.

```mermaid
sequenceDiagram
    title ClaudeSDKClient Interactive Session

    participant App as UserApp
    participant C as ClaudeSDKClient
    participant T as SubprocessCLITransport
    participant Q as Query
    participant CLI as Claude CLI

    App->>C: async with ClaudeSDKClient(options) as client:
    C->>C: __aenter__() → connect()

    C->>T: SubprocessCLITransport(prompt, options)
    C->>T: connect()
    activate T
    T->>CLI: spawn subprocess
    activate CLI

    C->>Q: Query(transport, is_streaming_mode=True,<br/>can_use_tool, hooks, sdk_mcp_servers)
    C->>Q: start()
    Note over Q: TaskGroup created,<br/>_read_messages() running

    C->>Q: initialize()
    Q->>T: write(initialize request)
    T->>CLI: stdin: control_request (initialize)
    CLI-->>T: stdout: control_response (capabilities)
    T-->>Q: initialize result
    Q-->>C: connected

    rect rgb(230, 245, 255)
        Note over App,CLI: Turn 1: First Query
        App->>C: client.query("first prompt")
        C->>T: write(user_message JSON)
        T->>CLI: stdin: {"type":"user", ...}

        CLI-->>T: stdout: streaming messages
        T-->>Q: parsed messages → stream

        App->>C: async for msg in client.receive_response()
        loop Until ResultMessage
            C->>Q: receive_messages()
            Q-->>C: message dict
            C->>C: parse_message() → Message
            C-->>App: yield Message
        end
        Note over App: ResultMessage received → iterator stops
    end

    rect rgb(230, 255, 230)
        Note over App,CLI: Turn 2: Follow-up (multi-turn)
        App->>C: client.query("follow-up question")
        C->>T: write(user_message JSON)
        T->>CLI: stdin: follow-up message
        Note over CLI: Continues with<br/>conversation context

        CLI-->>T: stdout: streaming response
        T-->>Q: parsed messages

        App->>C: async for msg in client.receive_response()
        loop Until ResultMessage
            C->>Q: receive_messages()
            Q-->>C: message dict
            C-->>App: yield Message
        end
    end

    rect rgb(255, 240, 230)
        Note over App,CLI: Interrupt Flow
        App->>C: client.query("long task...")
        C->>T: write(user_message)
        T->>CLI: stdin: user message

        CLI-->>T: stdout: partial response...
        App->>C: client.interrupt()
        C->>Q: interrupt()
        Q->>T: write(control_request: interrupt)
        T->>CLI: stdin: {"type":"control_request",<br/>"request":{"subtype":"interrupt"}}
        CLI-->>T: stdout: control_response (ack)
        T-->>Q: interrupt acknowledged
        CLI-->>T: stdout: result (interrupted)
    end

    App->>C: exit async with block
    C->>C: __aexit__() → disconnect()
    C->>Q: close()
    Q->>Q: cancel TaskGroup
    Q->>T: close()
    T->>CLI: terminate()
    deactivate CLI
    deactivate T
```

**Điểm quan trọng từ mã nguồn:**
- `ClaudeSDKClient.__aenter__()` gọi `connect()` không có prompt → dùng async generator rỗng (client.py:102-107)
- `client.query()` trong [`client.py:197`](../../src/claude_agent_sdk/_internal/client.py#L197) ghi user messages trực tiếp vào transport qua JSON
- `receive_response()` trong [`client.py:442`](../../src/claude_agent_sdk/_internal/client.py#L442) bọc `receive_messages()` và dừng sau `ResultMessage`
- `interrupt()` gửi control request với `subtype: "interrupt"` (query.py:536-538)
- `__aexit__()` luôn gọi `disconnect()` → `query.close()` → `transport.close()`

---

## 3. Dispatch Hook Callback

Hiện cách CLI khởi tạo hook callback (VD: PreToolUse) và cách lớp `Query` của SDK dispatch nó đến hàm async Python do user định nghĩa. Đây là luồng phức tạp nhất vì CLI là bên khởi tạo.

```mermaid
sequenceDiagram
    title Hook Callback Dispatch

    participant CLI as Claude CLI
    participant T as SubprocessCLITransport
    participant Q as Query
    participant Fn as UserHookFn

    Note over Q: During initialize(),<br/>hooks registered with callback_ids:<br/>hook_callbacks[id] = user_function

    rect rgb(245, 245, 255)
        Note over CLI,Fn: Registration Phase (during initialize)
        Q->>T: write(initialize request with hooks config)
        T->>CLI: stdin: {"hooks": {"PreToolUse": [{"matcher": {...},<br/>"hookCallbackIds": ["hook_0"]}]}}
        CLI-->>T: stdout: initialize response
        T-->>Q: initialized
    end

    rect rgb(255, 245, 235)
        Note over CLI,Fn: Hook Trigger Phase
        Note over CLI: CLI encounters hook point<br/>(e.g., PreToolUse for Bash tool)
        CLI->>T: stdout: {"type":"control_request",<br/>"request_id":"req_1",<br/>"request":{"subtype":"hook_callback",<br/>"callback_id":"hook_0",<br/>"input":{tool_name, tool_input},<br/>"tool_use_id":"toolu_xxx"}}
        T->>Q: _read_messages() receives control_request

        Q->>Q: identifies type == "control_request"
        Q->>Q: spawns _handle_control_request() in TaskGroup

        Q->>Q: subtype == "hook_callback"
        Q->>Q: lookup hook_callbacks["hook_0"]

        Q->>Fn: await callback(input, tool_use_id, context)
        activate Fn
        Note over Fn: User async function executes<br/>(approve, reject, or modify)
        Fn-->>Q: return {"decision": "approve"}<br/>or {"decision": "block", "reason": "..."}
        deactivate Fn

        Q->>Q: _convert_hook_output_for_cli()<br/>async_ → async, continue_ → continue

        Q->>T: write(control_response)
        T->>CLI: stdin: {"type":"control_response",<br/>"response":{"subtype":"success",<br/>"request_id":"req_1",<br/>"response":{"decision":"approve"}}}

        alt Decision: approve
            Note over CLI: CLI continues tool execution
        else Decision: block
            Note over CLI: CLI blocks tool use,<br/>reports reason to model
        end
    end
```

**Điểm quan trọng từ mã nguồn:**
- Đăng ký hook xảy ra trong `Query.initialize()` (query.py:119-163): mỗi hook nhận `callback_id` duy nhất ánh xạ đến hàm Python
- CLI gửi hook callbacks dạng `control_request` với `subtype: "hook_callback"` (query.py:288)
- `_handle_control_request()` tại query.py:236 dispatch dựa trên subtype
- Tên trường Python `async_` và `continue_` được chuyển thành `async`/`continue` cho wire format bởi `_convert_hook_output_for_cli()` (query.py:34-50)
- Khớp response dùng `request_id` để tương quan requests và responses

---

## 4. Gọi MCP Tool SDK In-Process

Hiện cách SDK MCP servers (định nghĩa qua decorator `@tool` và `create_sdk_mcp_server()`) xử lý gọi tool hoàn toàn in-process. Khác với MCP server bên ngoài chạy như subprocess, SDK MCP tools thực thi ngay trong tiến trình Python.

```mermaid
sequenceDiagram
    title SDK MCP In-Process Tool Call

    participant CLI as Claude CLI
    participant T as SubprocessCLITransport
    participant Q as Query
    participant MCP as MCPServer<br/>(in-process)
    participant TH as @tool Handler

    Note over Q,MCP: SDK MCP servers extracted from options<br/>during connect/process_query:<br/>sdk_mcp_servers[name] = McpServer instance

    rect rgb(240, 248, 255)
        Note over CLI,TH: MCP Initialization (during SDK init)
        CLI->>T: stdout: control_request (mcp_message:<br/>{"method":"initialize"})
        T->>Q: _read_messages() → control_request
        Q->>Q: _handle_control_request()
        Q->>Q: subtype == "mcp_message"
        Q->>Q: _handle_sdk_mcp_request(server_name, message)
        Q-->>T: write(control_response with MCP init result)
        T-->>CLI: stdin: {capabilities: {tools: {}},<br/>serverInfo: {name, version}}

        CLI->>T: stdout: mcp_message (tools/list)
        T->>Q: route to _handle_sdk_mcp_request
        Q->>MCP: handler = request_handlers[ListToolsRequest]
        MCP-->>Q: list of tool definitions
        Q-->>T: write(JSONRPC response with tools)
        T-->>CLI: stdin: {"result":{"tools":[...]}}
    end

    rect rgb(255, 248, 240)
        Note over CLI,TH: Tool Call Execution
        Note over CLI: Claude decides to call tool "get_weather"
        CLI->>T: stdout: {"type":"control_request",<br/>"request_id":"req_5",<br/>"request":{"subtype":"mcp_message",<br/>"server_name":"my-server",<br/>"message":{"method":"tools/call",<br/>"params":{"name":"get_weather",<br/>"arguments":{"city":"Paris"}}}}}

        T->>Q: _read_messages() → control_request
        Q->>Q: _handle_control_request()
        Q->>Q: subtype == "mcp_message"
        Q->>Q: _handle_sdk_mcp_request("my-server", message)

        Q->>Q: Check: server_name in sdk_mcp_servers?

        alt Server found
            Q->>MCP: handler = request_handlers[CallToolRequest]
            activate MCP
            MCP->>TH: call_tool("get_weather", {"city":"Paris"})
            activate TH
            Note over TH: @tool decorated function<br/>executes (may be async)
            TH-->>MCP: return result content
            deactivate TH
            MCP-->>Q: CallToolResult(content=[TextContent(...)])
            deactivate MCP

            Q->>Q: Convert result to JSONRPC response
            Q->>T: write(control_response)
            T->>CLI: stdin: {"type":"control_response",<br/>"response":{"subtype":"success",<br/>"request_id":"req_5",<br/>"response":{"mcp_response":<br/>{"result":{"content":[{"type":"text",<br/>"text":"Sunny, 22C"}]}}}}}

            Note over CLI: CLI receives tool result,<br/>continues generation
        else Server not found
            Q->>T: write(JSONRPC error: server not found)
            T->>CLI: stdin: error response
        end
    end
```

**Điểm quan trọng từ mã nguồn:**
- SDK MCP servers được trích xuất từ `options.mcp_servers` nơi `type == "sdk"` (client.py:143-147)
- `_handle_sdk_mcp_request()` tại query.py:394 định tuyến JSONRPC methods thủ công vì Python MCP SDK thiếu Transport abstraction
- Methods được hỗ trợ: `initialize`, `tools/list`, `tools/call`, `notifications/initialized` (query.py:431-514)
- Gọi tool đi qua `server.request_handlers[CallToolRequest]` để gọi handler được trang trí `@tool`
- Trường `instance` bị loại bỏ khỏi SDK server config trước khi truyền cho CLI (subprocess_cli.py:246-250)
- Tất cả giao tiếp đều in-process — không có subprocess IPC cho SDK MCP tools
