---
date: 2026-03-29
type: task-worklog
task: claudeagentsdk-5w2
title: "[MAS-1] Nắm vững @tool decorator và SDK MCP Server"
status: open
detailed_at: 2026-03-29
detail_score: ready-for-dev
tags: [mas, mcp, tool-decorator, sdk-mcp-server, practice, hands-on]
depends_on: []
blocks: [claudeagentsdk-o9q, MAS-3]
estimated_hours: 2
output_file: examples/mas/tools_practice.py
---

# [MAS-1] Nắm vững @tool decorator và SDK MCP Server

## 1. Mục tiêu (Objective)

Thực hành tạo 4 custom MCP tools bằng `@tool` decorator, đăng ký vào `create_sdk_mcp_server()`, và chạy qua `ClaudeSDKClient` để xác nhận Claude gọi đúng tool, nhận đúng kết quả, và xử lý lỗi gracefully. Mục tiêu cuối cùng là nắm vững hoàn toàn pattern tạo tool trong SDK trước khi tiến đến các task nâng cao hơn.

## 2. Phạm vi (Scope)

**Trong phạm vi (In-scope):**
- Tạo 4 tools: Calculator (4 phép tính + error handling), File Reader, JSON Validator, HTTP Fetcher
- Mỗi tool dùng `@tool` decorator với `input_schema` đúng chuẩn
- Mỗi tool handle error bằng `is_error=True` trong response
- Đăng ký tất cả tools vào 1 SDK MCP server qua `create_sdk_mcp_server()`
- Chạy test end-to-end qua `ClaudeSDKClient` với `allowed_tools` đúng naming convention
- Output file duy nhất: `examples/mas/tools_practice.py`

**Ngoài phạm vi (Out-of-scope):**
- Tạo MCP server chạy ngoài process (external MCP server)
- Dùng TypedDict cho input_schema (chỉ dùng dict mapping đơn giản)
- Authentication/authorization cho tools
- Unit tests riêng (test bằng chạy thực tế với Claude)
- Performance benchmarking
- Tích hợp với hooks hoặc can_use_tool callback

## 3. Đầu vào / Đầu ra (Input / Output)

**Đầu vào:**
- SDK source code tại `src/claude_agent_sdk/` -- đặc biệt `__init__.py` chứa `@tool` decorator và `create_sdk_mcp_server()`
- Docs trong `CLAUDE.md` mô tả kiến trúc SDK
- CLI binary tại `/home/admin88/.nvm/versions/node/v24.13.0/bin/claude`

**Đầu ra:**
- `examples/mas/tools_practice.py` -- File Python chạy được, chứa:
  1. 4 tool definitions dùng `@tool` decorator
  2. 1 SDK MCP server chứa tất cả tools
  3. Hàm `main()` async kết nối `ClaudeSDKClient` và test từng tool
  4. Logging/print cho kết quả mỗi tool call
  5. Error handling demonstration

## 4. Phụ thuộc (Dependencies)

**Packages (Python):**
- `claude_agent_sdk` -- SDK chính (chạy từ source: `src/claude_agent_sdk/`)
- `anyio` -- async runtime (dependency của SDK)
- `aiohttp` hoặc `httpx` -- cho HTTP Fetcher tool (cần cài thêm nếu chưa có)
- `json` -- stdlib, cho JSON Validator tool
- `pathlib` -- stdlib, cho File Reader tool

**Biến môi trường:**
- `ANTHROPIC_API_KEY` -- bắt buộc, Claude API key
- `PATH` -- phải chứa đường dẫn đến `claude` CLI binary

**Điều kiện tiên quyết (Prerequisites):**
- Python 3.10+
- Claude Code CLI đã cài tại `/home/admin88/.nvm/versions/node/v24.13.0/bin/claude`
- `ANTHROPIC_API_KEY` đã set trong environment
- SDK source chạy được (có thể dùng `PYTHONPATH=src` hoặc `pip install -e .`)

## 5. Luồng xử lý (Flow)

### Bước 1: Tạo thư mục và file cơ bản (~5 phút)

Tạo `examples/mas/tools_practice.py` với imports và cấu trúc cơ bản:

```python
import asyncio
import json
from pathlib import Path

from claude_agent_sdk import (
    tool,
    create_sdk_mcp_server,
    ClaudeSDKClient,
    ClaudeAgentOptions,
    AssistantMessage,
    TextBlock,
    ToolUseBlock,
    ResultMessage,
)
```

**Kiểm tra:** File import thành công, không lỗi module.

### Bước 2: Tạo Tool 1 -- Calculator (~15 phút)

Calculator tool hỗ trợ 4 phép tính: cộng, trừ, nhân, chia. Đặc biệt handle chia cho 0.

```python
@tool(
    name="calculator",
    description="Thực hiện phép tính số học: cộng (add), trừ (subtract), nhân (multiply), chia (divide). Trả về kết quả hoặc lỗi nếu phép tính không hợp lệ.",
    input_schema={
        "type": "object",
        "properties": {
            "operation": {
                "type": "string",
                "enum": ["add", "subtract", "multiply", "divide"],
                "description": "Phép tính cần thực hiện"
            },
            "a": {
                "type": "number",
                "description": "Số thứ nhất"
            },
            "b": {
                "type": "number",
                "description": "Số thứ hai"
            }
        },
        "required": ["operation", "a", "b"]
    }
)
async def calculator(args: dict) -> dict:
    operation = args["operation"]
    a = args["a"]
    b = args["b"]

    try:
        if operation == "add":
            result = a + b
        elif operation == "subtract":
            result = a - b
        elif operation == "multiply":
            result = a * b
        elif operation == "divide":
            if b == 0:
                return {
                    "content": [{"type": "text", "text": "Error: Division by zero"}],
                    "is_error": True,
                }
            result = a / b
        else:
            return {
                "content": [{"type": "text", "text": f"Error: Unknown operation '{operation}'"}],
                "is_error": True,
            }

        return {"content": [{"type": "text", "text": f"Result: {result}"}]}

    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
```

**Kiểm tra:** `calculator` là instance `SdkMcpTool` với `.name == "calculator"`, `.handler` là async callable.

### Bước 3: Tạo Tool 2 -- File Reader (~15 phút)

Đọc nội dung file text từ đường dẫn cho trước. Handle file not found và permission errors.

```python
@tool(
    name="read_file",
    description="Đọc nội dung một file text từ đường dẫn. Trả về nội dung file hoặc lỗi nếu file không tồn tại hoặc không thể đọc.",
    input_schema={
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "Đường dẫn tuyệt đối hoặc tương đối đến file cần đọc"
            },
            "max_lines": {
                "type": "integer",
                "description": "Số dòng tối đa cần đọc (mặc định: 100)"
            }
        },
        "required": ["path"]
    }
)
async def read_file(args: dict) -> dict:
    file_path = Path(args["path"])
    max_lines = args.get("max_lines", 100)

    try:
        if not file_path.exists():
            return {
                "content": [{"type": "text", "text": f"Error: File not found: {file_path}"}],
                "is_error": True,
            }

        if not file_path.is_file():
            return {
                "content": [{"type": "text", "text": f"Error: Not a regular file: {file_path}"}],
                "is_error": True,
            }

        text = file_path.read_text(encoding="utf-8")
        lines = text.splitlines()

        if len(lines) > max_lines:
            truncated = "\n".join(lines[:max_lines])
            return {
                "content": [{
                    "type": "text",
                    "text": f"[Showing {max_lines}/{len(lines)} lines]\n{truncated}"
                }]
            }

        return {"content": [{"type": "text", "text": text}]}

    except PermissionError:
        return {
            "content": [{"type": "text", "text": f"Error: Permission denied: {file_path}"}],
            "is_error": True,
        }
    except UnicodeDecodeError:
        return {
            "content": [{"type": "text", "text": f"Error: File is not valid UTF-8 text: {file_path}"}],
            "is_error": True,
        }
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error reading file: {str(e)}"}],
            "is_error": True,
        }
```

**Kiểm tra:** `read_file` là instance `SdkMcpTool` với `.name == "read_file"`.

### Bước 4: Tạo Tool 3 -- JSON Validator (~15 phút)

Kiểm tra xem string có phải JSON hợp lệ không, trả về parsed result hoặc chi tiết lỗi syntax.

```python
@tool(
    name="validate_json",
    description="Kiểm tra xem một chuỗi có phải JSON hợp lệ không. Trả về JSON đã parse (pretty-printed) hoặc chi tiết lỗi cú pháp.",
    input_schema={
        "type": "object",
        "properties": {
            "json_string": {
                "type": "string",
                "description": "Chuỗi JSON cần kiểm tra"
            },
            "strict": {
                "type": "boolean",
                "description": "Nếu true, kiểm tra thêm duplicate keys (mặc định: false)"
            }
        },
        "required": ["json_string"]
    }
)
async def validate_json(args: dict) -> dict:
    json_string = args["json_string"]
    strict = args.get("strict", False)

    try:
        if strict:
            # Kiểm tra duplicate keys bằng custom decoder
            def check_duplicates(pairs):
                keys = [k for k, v in pairs]
                if len(keys) != len(set(keys)):
                    duplicates = [k for k in keys if keys.count(k) > 1]
                    raise ValueError(f"Duplicate keys found: {set(duplicates)}")
                return dict(pairs)

            parsed = json.loads(json_string, object_pairs_hook=check_duplicates)
        else:
            parsed = json.loads(json_string)

        pretty = json.dumps(parsed, indent=2, ensure_ascii=False)
        return {
            "content": [{
                "type": "text",
                "text": f"Valid JSON!\n\nParsed result:\n{pretty}"
            }]
        }

    except json.JSONDecodeError as e:
        return {
            "content": [{
                "type": "text",
                "text": f"Invalid JSON at line {e.lineno}, column {e.colno}: {e.msg}"
            }],
            "is_error": True,
        }
    except ValueError as e:
        return {
            "content": [{"type": "text", "text": f"JSON validation error: {str(e)}"}],
            "is_error": True,
        }
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
```

**Kiểm tra:** `validate_json` là instance `SdkMcpTool` với `.name == "validate_json"`.

### Bước 5: Tạo Tool 4 -- HTTP Fetcher (~20 phút)

Gửi HTTP GET request đến URL cho trước, trả về response body (text). Handle timeout, connection errors.

```python
@tool(
    name="http_fetch",
    description="Gửi HTTP GET request đến một URL và trả về response body dạng text. Hỗ trợ timeout và giới hạn kích thước response.",
    input_schema={
        "type": "object",
        "properties": {
            "url": {
                "type": "string",
                "description": "URL cần fetch (phải bắt đầu bằng http:// hoặc https://)"
            },
            "timeout_seconds": {
                "type": "integer",
                "description": "Timeout tính bằng giây (mặc định: 10)"
            },
            "max_length": {
                "type": "integer",
                "description": "Số ký tự tối đa trả về (mặc định: 5000)"
            }
        },
        "required": ["url"]
    }
)
async def http_fetch(args: dict) -> dict:
    import urllib.request
    import urllib.error
    from asyncio import to_thread

    url = args["url"]
    timeout = args.get("timeout_seconds", 10)
    max_length = args.get("max_length", 5000)

    # Validate URL scheme
    if not url.startswith(("http://", "https://")):
        return {
            "content": [{"type": "text", "text": "Error: URL must start with http:// or https://"}],
            "is_error": True,
        }

    try:
        # Dùng to_thread để không block event loop
        def _fetch():
            req = urllib.request.Request(url, headers={"User-Agent": "claude-sdk-tool/1.0"})
            with urllib.request.urlopen(req, timeout=timeout) as resp:
                body = resp.read().decode("utf-8", errors="replace")
                status = resp.status
                content_type = resp.headers.get("Content-Type", "unknown")
                return body, status, content_type

        body, status, content_type = await to_thread(_fetch)

        if len(body) > max_length:
            body = body[:max_length] + f"\n\n[Truncated: showing {max_length}/{len(body)} chars]"

        return {
            "content": [{
                "type": "text",
                "text": f"Status: {status}\nContent-Type: {content_type}\n\n{body}"
            }]
        }

    except urllib.error.HTTPError as e:
        return {
            "content": [{"type": "text", "text": f"HTTP Error {e.code}: {e.reason}"}],
            "is_error": True,
        }
    except urllib.error.URLError as e:
        return {
            "content": [{"type": "text", "text": f"Connection error: {str(e.reason)}"}],
            "is_error": True,
        }
    except TimeoutError:
        return {
            "content": [{"type": "text", "text": f"Error: Request timed out after {timeout}s"}],
            "is_error": True,
        }
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Error: {str(e)}"}],
            "is_error": True,
        }
```

**Kiểm tra:** `http_fetch` là instance `SdkMcpTool` với `.name == "http_fetch"`. Dùng `urllib.request` (stdlib) thay vì package bên ngoài để giảm dependency.

### Bước 6: Đăng ký tools vào SDK MCP Server (~10 phút)

Tạo server chứa tất cả 4 tools và cấu hình `ClaudeAgentOptions`:

```python
# Tạo SDK MCP Server chứa tất cả tools
practice_server = create_sdk_mcp_server(
    name="practice_tools",
    version="1.0.0",
    tools=[calculator, read_file, validate_json, http_fetch],
)

# Cấu hình options cho ClaudeSDKClient
# Naming convention: mcp__{server_name}__{tool_name}
options = ClaudeAgentOptions(
    mcp_servers={"practice_tools": practice_server},
    allowed_tools=[
        "mcp__practice_tools__calculator",
        "mcp__practice_tools__read_file",
        "mcp__practice_tools__validate_json",
        "mcp__practice_tools__http_fetch",
    ],
    permission_mode="bypassPermissions",
    max_turns=10,
)
```

**Lưu ý quan trọng:**
- `mcp_servers` nhận dict với key là tên server, value là `McpSdkServerConfig` trả về từ `create_sdk_mcp_server()`
- `allowed_tools` dùng naming convention `mcp__{server_name}__{tool_name}` để Claude tự động gọi tool không cần hỏi permission
- `permission_mode="bypassPermissions"` cho phép Claude gọi tool mà không cần xác nhận từ user

**Kiểm tra:** `practice_server` có type `McpSdkServerConfig`. Options có đúng 4 allowed_tools.

### Bước 7: Viết hàm main() test end-to-end (~20 phút)

```python
async def main():
    """Test tất cả 4 tools bằng cách yêu cầu Claude sử dụng chúng."""

    print("=" * 60)
    print("MAS-1: Testing 4 custom MCP tools")
    print("=" * 60)

    client = ClaudeSDKClient(options=options)

    try:
        # Prompt yêu cầu Claude dùng tất cả 4 tools
        prompt = (
            "Please test ALL 4 tools available to you by doing the following:\n"
            "1. Use calculator to compute: (15 + 27) and (100 / 0)\n"
            "2. Use read_file to read the file at './pyproject.toml' (first 20 lines)\n"
            "3. Use validate_json to check this string: '{\"name\": \"test\", \"value\": 42}'\n"
            "4. Use validate_json to check this INVALID string: '{bad json}'\n"
            "5. Use http_fetch to fetch https://httpbin.org/get\n"
            "\nFor each tool call, report the result clearly."
        )

        await client.connect(prompt)

        tool_calls_seen = set()

        async for message in client.receive_messages():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(f"\n[Claude]: {block.text}")
                    elif isinstance(block, ToolUseBlock):
                        tool_calls_seen.add(block.name)
                        print(f"\n[Tool Call]: {block.name}({json.dumps(block.input, ensure_ascii=False)})")

            elif isinstance(message, ResultMessage):
                print(f"\n[Result]: cost=${message.cost_usd:.4f}, turns={message.num_turns}")
                break

        # Báo cáo kết quả
        print("\n" + "=" * 60)
        print("Tool calls detected:")
        for name in sorted(tool_calls_seen):
            print(f"  - {name}")

        expected_tools = {"calculator", "read_file", "validate_json", "http_fetch"}
        missing = expected_tools - tool_calls_seen
        if missing:
            print(f"\nWARNING: These tools were NOT called: {missing}")
        else:
            print("\nSUCCESS: All 4 tools were called!")
        print("=" * 60)

    finally:
        await client.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
```

**Kiểm tra:** Script chạy end-to-end, in ra tool calls và kết quả. Claude gọi ít nhất 4 tool calls (calculator 2 lần, validate_json 2 lần, read_file 1 lần, http_fetch 1 lần).

### Bước 8: Chạy thử và debug (~20 phút)

```bash
# Từ thư mục gốc project
cd /home/admin88/1_active_projects/claude-agent-sdk-python
PYTHONPATH=src python examples/mas/tools_practice.py
```

Nếu gặp lỗi:
- `CLINotFoundError` -> kiểm tra `cli_path` trong options hoặc set `CLAUDE_CLI_PATH`
- `ANTHROPIC_API_KEY` chưa set -> export biến môi trường
- Tool không được gọi -> kiểm tra `allowed_tools` naming convention
- `is_error=True` response -> kiểm tra Claude có xử lý lỗi đúng không (nên retry hoặc báo lỗi)

**Kiểm tra cuối:** Script hoàn thành không crash. Console log hiển thị tất cả 4 tool names trong `tool_calls_seen`.

## 6. Edge Cases & Error Handling

| Tình huống | Tool | Hành vi mong đợi |
|---|---|---|
| Chia cho 0 | calculator | Trả về `is_error=True` với message "Division by zero" |
| Operation không hợp lệ (VD: "modulo") | calculator | Trả về `is_error=True` với message "Unknown operation" |
| Input không phải số | calculator | Exception handler bắt TypeError, trả về `is_error=True` |
| File không tồn tại | read_file | Trả về `is_error=True` với "File not found" |
| File quá lớn | read_file | Truncate theo `max_lines`, thông báo "[Showing X/Y lines]" |
| File binary (không phải UTF-8) | read_file | Trả về `is_error=True` với "not valid UTF-8 text" |
| Permission denied | read_file | Trả về `is_error=True` với "Permission denied" |
| JSON hợp lệ | validate_json | Trả về pretty-printed JSON |
| JSON không hợp lệ | validate_json | Trả về `is_error=True` với line/column lỗi |
| Duplicate keys (strict mode) | validate_json | Trả về `is_error=True` liệt kê keys trùng |
| URL không hợp lệ (thiếu scheme) | http_fetch | Trả về `is_error=True` trước khi gọi request |
| HTTP 404/500 | http_fetch | Trả về `is_error=True` với status code và reason |
| Connection timeout | http_fetch | Trả về `is_error=True` với timeout message |
| Response quá lớn | http_fetch | Truncate theo `max_length`, thông báo kích thước gốc |
| DNS resolution failure | http_fetch | Trả về `is_error=True` qua URLError handler |
| Server name trùng key trong mcp_servers | setup | Key dict phải khớp với `name` trong `create_sdk_mcp_server()` |
| Claude không gọi tool nào | main | WARNING in console, user debug prompt |

## 7. Acceptance Criteria (Given/When/Then)

### AC1: Calculator tool hoạt động đúng
- **Given:** SDK MCP server đã đăng ký calculator tool
- **When:** Claude gọi calculator với `{"operation": "add", "a": 15, "b": 27}`
- **Then:** Tool trả về `{"content": [{"type": "text", "text": "Result: 42"}]}`

### AC2: Calculator xử lý lỗi chia cho 0
- **Given:** SDK MCP server đã đăng ký calculator tool
- **When:** Claude gọi calculator với `{"operation": "divide", "a": 100, "b": 0}`
- **Then:** Tool trả về `{"content": [...], "is_error": true}` với message "Division by zero"

### AC3: File Reader đọc file thành công
- **Given:** File `pyproject.toml` tồn tại trong thư mục hiện tại
- **When:** Claude gọi read_file với `{"path": "./pyproject.toml", "max_lines": 20}`
- **Then:** Tool trả về nội dung 20 dòng đầu của file

### AC4: JSON Validator phát hiện JSON không hợp lệ
- **Given:** SDK MCP server đã đăng ký validate_json tool
- **When:** Claude gọi validate_json với `{"json_string": "{bad json}"}`
- **Then:** Tool trả về `is_error=True` với thông tin line/column lỗi cú pháp

### AC5: HTTP Fetcher lấy dữ liệu thành công
- **Given:** Internet khả dụng, SDK MCP server đã đăng ký http_fetch tool
- **When:** Claude gọi http_fetch với `{"url": "https://httpbin.org/get"}`
- **Then:** Tool trả về status 200 và body chứa JSON response

### AC6: Tất cả 4 tools được Claude gọi trong 1 session
- **Given:** Prompt yêu cầu Claude dùng tất cả 4 tools
- **When:** Script `tools_practice.py` chạy hoàn tất
- **Then:** `tool_calls_seen` chứa đủ 4 tên tool: `calculator`, `read_file`, `validate_json`, `http_fetch`

### AC7: Error responses được handle gracefully
- **Given:** Claude nhận `is_error=True` response từ tool
- **When:** Claude tiếp tục conversation sau lỗi
- **Then:** Claude không crash, giải thích lỗi cho user, và có thể retry hoặc báo cáo

## 8. Technical Notes

### Pattern chuẩn của @tool decorator

`@tool` decorator nhận 3 tham số chính:
1. `name: str` -- tên duy nhất, Claude dùng tên này để gọi tool
2. `description: str` -- mô tả cho Claude hiểu khi nào dùng tool
3. `input_schema: dict` -- có thể là:
   - **Simple dict mapping:** `{"a": float, "b": float}` -- SDK tự convert sang JSON Schema
   - **Full JSON Schema:** `{"type": "object", "properties": {...}, "required": [...]}` -- dùng trực tiếp

Decorator trả về `SdkMcpTool` instance (KHÔNG phải function). Variable `calculator` sau decorator là `SdkMcpTool`, không phải callable thông thường.

### Luồng tool call trong SDK

```
1. Claude quyết định gọi tool -> CLI gửi tool_use message
2. Query._handle_mcp_tool_call() nhận tool call
3. Query tìm SDK MCP server chứa tool
4. Gọi server.call_tool(name, arguments)
5. Server gọi handler function (async def calculator(args))
6. Handler trả về {"content": [...], "is_error": bool}
7. SDK convert response thành TextContent/ImageContent
8. Kết quả gửi ngược lại CLI -> Claude nhận tool result
```

### Naming convention cho allowed_tools

Format: `mcp__{server_key}__{tool_name}` trong đó:
- `server_key` = key trong dict `mcp_servers` (KHÔNG phải `name` param của `create_sdk_mcp_server()`)
- `tool_name` = `name` param của `@tool` decorator

Ví dụ: nếu `mcp_servers={"my_server": server}` và tool có `name="calc"`, thì `allowed_tools=["mcp__my_server__calc"]`.

**Quan trọng:** `server_key` trong dict `mcp_servers` thường nên trùng với `name` trong `create_sdk_mcp_server()` để tránh nhầm lẫn, nhưng kỹ thuật thì chúng là 2 giá trị khác nhau.

### is_error=True behavior

Khi tool trả về `is_error=True`:
- Claude nhận tool result với flag lỗi
- Claude thường sẽ giải thích lỗi cho user hoặc thử cách khác
- Claude KHÔNG crash hay dừng conversation
- Đây là cách handle lỗi chính thức trong MCP protocol

### asyncio.to_thread() cho I/O blocking

HTTP Fetcher dùng `asyncio.to_thread()` để chạy `urllib.request.urlopen()` trong thread pool, tránh block event loop. Đây là pattern chuẩn khi cần gọi sync I/O từ async context.

## 9. Rủi ro (Risks)

| Rủi ro | Xác suất | Tác động | Giảm thiểu |
|---|---|---|---|
| CLI binary không tìm thấy | Trung bình | Chặn hoàn toàn | Set `cli_path` trong `ClaudeAgentOptions` hoặc biến `CLAUDE_CLI_PATH` |
| `ANTHROPIC_API_KEY` hết hạn hoặc invalid | Thấp | Chặn hoàn toàn | Kiểm tra key trước khi chạy bằng `claude --version` |
| Claude không gọi đủ 4 tools | Trung bình | Thiếu coverage | Prompt rõ ràng, yêu cầu tuần tự; tăng `max_turns` nếu cần |
| httpbin.org không khả dụng | Thấp | 1 tool test fail | Dùng URL fallback như `https://api.github.com/zen` |
| Rate limiting từ Anthropic API | Trung bình | Chậm hoặc fail | Thêm retry logic hoặc chạy lại sau |
| `allowed_tools` naming sai | Cao (dễ sai) | Tool không được gọi | Double check format `mcp__{key}__{name}`, print allowed_tools trước khi chạy |
| SDK chưa cài (import error) | Trung bình | Chặn hoàn toàn | Chạy `PYTHONPATH=src` hoặc `pip install -e .` trước |
| File pyproject.toml không có ở cwd | Thấp | 1 tool test fail | Chạy script từ project root, hoặc dùng absolute path |
