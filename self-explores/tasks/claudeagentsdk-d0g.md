---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-d0g
title: "P1: Map cấu trúc thư mục & luồng code chính"
status: completed
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [research, architecture, code-flow, p1, discovery]
---

# Map cấu trúc thư mục & luồng code chính — Thiết kế chi tiết

## 1. Mục tiêu
Tạo cây thư mục có chú thích cho tất cả 15 file src/*.py và truy vết 4 luồng code chính (query, ClaudeSDKClient, hooks, MCP) với mô tả từng bước bao gồm tham chiếu file:class:method.

## 2. Phạm vi
**Trong phạm vi:**
- Lập bản đồ cây thư mục đầy đủ với chú thích cho mỗi file .py trong `src/`
- Truy vết luồng `query()`: query.py -> InternalClient -> Query -> SubprocessCLITransport
- Truy vết luồng `ClaudeSDKClient`: client.py -> connect -> query/receive_response cycle
- Truy vết luồng hooks: HookMatcher -> Query dispatch -> Python callbacks -> response
- Truy vết luồng MCP in-process: @tool -> create_sdk_mcp_server -> Server -> call_tool
- Lập bản đồ cây lỗi (error hierarchy) và tổng quan hệ thống kiểu (type system)
- Đối chiếu với Context7 `/anthropics/claude-agent-sdk-python` (51 snippets)

**Ngoài phạm vi:**
- Đọc file test (17 file trong tests/)
- Đọc file ví dụ (18 file trong examples/)
- Tạo sơ đồ (diagram) (đó là task claudeagentsdk-fl0)
- Sửa đổi bất kỳ mã nguồn nào
- Phân tích hoặc đo hiệu năng (benchmarking)
- Nội bộ CLI binary (chỉ tương tác của SDK với nó)

## 3. Đầu vào / Đầu ra
**Đầu vào:**
- 15 file .py trong `src/claude_agent_sdk/` và `src/claude_agent_sdk/_internal/`:
  - Công khai: `__init__.py`, [`query.py`](../../src/claude_agent_sdk/_internal/query.py), [`client.py`](../../src/claude_agent_sdk/_internal/client.py), [`types.py`](../../src/claude_agent_sdk/types.py)
  - Nội bộ: `_internal/__init__.py`, [`client.py`](../../src/claude_agent_sdk/_internal/client.py), [`query.py`](../../src/claude_agent_sdk/_internal/query.py), [`message_parser.py`](../../src/claude_agent_sdk/_internal/message_parser.py), [`sessions.py`](../../src/claude_agent_sdk/_internal/sessions.py), [`_errors.py`](../../src/claude_agent_sdk/_errors.py)
  - Transport: `_internal/transport/__init__.py`, [`subprocess_cli.py`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py)
  - (các file còn lại sẽ được phát hiện trong Bước 1)
- Context7 MCP: `/anthropics/claude-agent-sdk-python` (51 snippets)

**Đầu ra:**
- `self-explores/context/code-architecture.md` -- Tài liệu kiến trúc hợp nhất chứa:
  1. Cây thư mục có chú thích (mỗi file .py kèm mô tả 1 dòng)
  2. Luồng query() (5+ bước)
  3. Luồng ClaudeSDKClient (5+ bước)
  4. Luồng Hooks (5+ bước)
  5. Luồng MCP in-process (5+ bước)
  6. Cây lỗi (Error hierarchy)
  7. Tổng quan hệ thống kiểu (Type system)

## 4. Phụ thuộc (Dependencies)
- **Phụ thuộc task:** Không có (đây là task P1 khởi đầu, chạy song song với 3ma và 2e7)
- **Phụ thuộc công cụ:**
  - Read tool -- đọc tất cả 15 file mã nguồn
  - Glob tool -- khám phá tất cả file .py trong src/
  - Grep tool -- truy vết chuỗi gọi hàm và tìm định nghĩa method
  - Context7 MCP (`mcp__context7__resolve-library-id` + `mcp__context7__query-docs`) -- đối chiếu
  - Write tool -- tạo file đầu ra
- **Thư mục:** `self-explores/context/` có thể cần tạo nếu chưa tồn tại

## 5. Luồng xử lý (Flow)

### Bước 1: Lập bản đồ cây thư mục có chú thích (~10 phút)
Dùng Glob tìm tất cả file .py dưới `src/`:
```
Glob pattern: src/**/*.py
```

Sau đó đọc 20-30 dòng đầu mỗi file để nắm docstring và imports. Xây dựng cây có chú thích như:
```
src/claude_agent_sdk/
  __init__.py          -- Exports công khai, decorator @tool, create_sdk_mcp_server()
  query.py             -- query() async generator, điểm vào một lần (one-shot)
  client.py            -- ClaudeSDKClient, quản lý session có trạng thái
  types.py             -- Tất cả kiểu công khai: Message, ClaudeAgentOptions, hooks, v.v.
  _internal/
    __init__.py         -- Khởi tạo package nội bộ
    client.py           -- InternalClient, dùng bởi query() nội bộ
    query.py            -- Lớp Query, xử lý giao thức điều khiển (control protocol)
    message_parser.py   -- JSON dict -> đối tượng Message có kiểu
    sessions.py         -- Đọc dữ liệu session lịch sử
    _errors.py          -- Cây lỗi (ClaudeSDKError tree)
    transport/
      __init__.py       -- Khởi tạo package transport
      subprocess_cli.py -- SubprocessCLITransport, quản lý tiến trình CLI
```

**Kiểm tra:** Mỗi file .py tìm được bởi Glob đều được liệt kê trong cây kèm mô tả 1 dòng. Tổng số khớp 15 (hoặc số thực tế nếu khác).

### Bước 2: Truy vết luồng query() (~10 phút)
Đọc các file theo thứ tự, theo dõi chuỗi gọi hàm:

1. [`query.py`](../../src/claude_agent_sdk/_internal/query.py) -- Tìm chữ ký hàm `query()`, nó tạo/gọi gì
2. [`client.py`](../../src/claude_agent_sdk/_internal/client.py) -- Tìm `InternalClient.process_query()`, cách nó tạo transport và Query
3. [`query.py`](../../src/claude_agent_sdk/_internal/query.py) -- Tìm lớp `Query`, bắt tay `initialize()`, vòng lặp streaming message
4. [`subprocess_cli.py`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py) -- Tìm `SubprocessCLITransport`, cách nó sinh tiến trình CLI, pipe stdin/stdout

Với mỗi bước, tài liệu hoá:
- Chữ ký method (class.method với tham số chính)
- Nó làm gì (1-2 câu)
- Nó gọi gì tiếp theo (bàn giao)
- Các biến đổi dữ liệu chính (VD: ClaudeAgentOptions -> CLI args)

Dùng Grep để tìm các lệnh gọi method cụ thể:
```
Grep: "process_query" trong src/
Grep: "SubprocessCLITransport" trong src/
Grep: "async def initialize" trong src/
```

**Kiểm tra:** Luồng có 5+ bước, mỗi bước có tham chiếu file:class:method. Đi từ `query()` công khai đến sinh subprocess và quay lại qua yield message.

### Bước 3: Truy vết luồng ClaudeSDKClient (~10 phút)
Đọc kỹ [`client.py`](../../src/claude_agent_sdk/_internal/client.py). Truy vết:

1. `ClaudeSDKClient.__init__()` -- Khởi tạo, lưu trữ options
2. `ClaudeSDKClient.__aenter__()` (hoặc `connect()`) -- Tạo transport, bắt tay khởi tạo
3. `ClaudeSDKClient.query()` -- Gửi prompt, nhận iterator phản hồi
4. `ClaudeSDKClient.receive_response()` -- Chu kỳ hội thoại đa lượt (multi-turn)
5. `ClaudeSDKClient.interrupt()` -- Ngắt quá trình sinh (generation) đang diễn ra
6. `ClaudeSDKClient.__aexit__()` -- Dọn dẹp, tắt transport

Cũng truy vết các tính năng có trạng thái:
- `set_permission_mode()` -- Thay đổi quyền lúc chạy
- `set_model()` -- Chuyển model lúc chạy
- `add_mcp_server()` / `remove_mcp_server()` -- Quản lý MCP server

Dùng Grep tìm các message giao thức điều khiển:
```
Grep: "request_id" trong src/
Grep: "control" trong src/_internal/query.py
```

**Kiểm tra:** Luồng có 5+ bước bao phủ vòng đời (init -> connect -> query -> receive -> cleanup). Các method có trạng thái được liệt kê.

### Bước 4: Truy vết luồng hooks (~10 phút)
Truy vết hệ thống hooks từ đầu đến cuối:

1. Tìm định nghĩa kiểu hook trong [`types.py`](../../src/claude_agent_sdk/types.py) -- PreToolUse, PostToolUse, Stop, v.v.
2. Tìm đăng ký hook trong [`client.py`](../../src/claude_agent_sdk/_internal/client.py) hoặc [`query.py`](../../src/claude_agent_sdk/_internal/query.py) -- Cách callback Python được đăng ký
3. Tìm `HookMatcher` hoặc logic dispatch hook trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py) -- Cách callback hook từ CLI đến và được khớp
4. Truy vết thực thi callback -- Cách hàm async Python được gọi với dữ liệu hook
5. Truy vết đường phản hồi -- Cách kết quả hook (cho phép/từ chối/sửa đổi) được gửi lại CLI

Đặc biệt chú ý:
- Ánh xạ tên trường `async_` / `continue_` (tránh từ khoá Python)
- Các kiểu sự kiện hook và payload của chúng
- Callback `can_use_tool` (hệ thống phân quyền công cụ)

Dùng Grep:
```
Grep: "hook" trong src/ (không phân biệt hoa thường)
Grep: "PreToolUse\|PostToolUse\|Stop" trong src/
Grep: "can_use_tool" trong src/
```

**Kiểm tra:** Luồng có 5+ bước từ định nghĩa hook đến phản hồi CLI. Tất cả kiểu sự kiện hook được liệt kê.

### Bước 5: Truy vết luồng MCP in-process (~10 phút)
Truy vết hệ thống SDK MCP server:

1. Tìm decorator `@tool` trong `__init__.py` -- Cách tool được định nghĩa
2. Tìm `create_sdk_mcp_server()` trong `__init__.py` -- Cách MCP server được xây dựng từ định nghĩa tool
3. Tìm tích hợp MCP server trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py) -- Cách Query chặn tool call dành cho SDK MCP server
4. Truy vết thực thi tool -- Cách tool call in-process được dispatch và kết quả trả về
5. Tìm các message giao thức MCP -- Cách tool call/result chảy qua giao thức điều khiển

Dùng Grep:
```
Grep: "sdk_mcp\|create_sdk_mcp" trong src/
Grep: "@tool\|tool_map" trong src/
Grep: "call_tool" trong src/
```

**Kiểm tra:** Luồng có 5+ bước từ định nghĩa @tool đến trả kết quả. Phân biệt rõ SDK MCP (in-process) và MCP server bên ngoài.

### Bước 6: Đối chiếu qua Context7 (~5 phút)
Lấy tài liệu Context7 bằng `mcp__context7__query-docs` với `/anthropics/claude-agent-sdk-python`:
- Truy vấn: "transport layer control protocol initialize handshake message streaming"
- So sánh: Các luồng đã truy vết có khớp với mô tả triển khai chính thức không?
- Ghi chú: Bất kỳ khác biệt hoặc chi tiết bổ sung không tìm thấy trong comment code local

**Kiểm tra:** Ít nhất 1 ghi chú đối chiếu được thêm vào tài liệu đầu ra.

### Bước 7: Lập bản đồ lỗi + kiểu (~5 phút)
Đọc [`_errors.py`](../../src/claude_agent_sdk/_errors.py) và [`types.py`](../../src/claude_agent_sdk/types.py):

**Lỗi:** Tài liệu hoá cây lỗi:
```
ClaudeSDKError
  CLIConnectionError
    CLINotFoundError
  ProcessError
  CLIJSONDecodeError
  MessageParseError
```

**Kiểu:** Tài liệu hoá các nhóm kiểu chính:
- Kiểu union Message (các biến thể tồn tại)
- Khối nội dung (content blocks): TextBlock, ToolUseBlock, ToolResultBlock, v.v.
- Cấu hình: ClaudeAgentOptions (tóm tắt trường)
- Kiểu hook: biến thể HookEvent, chữ ký HookCallback
- Kiểu MCP: định nghĩa tool, cấu hình server

**Kiểm tra:** Cây lỗi đầy đủ (khớp mô tả CLAUDE.md). Phần kiểu liệt kê 4+ nhóm.

## 6. Trường hợp biên & Xử lý lỗi
| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| _internal/query.py quá phức tạp | File lớn với nhiều handler giao thức điều khiển | Khó truy vết tất cả đường dẫn trong 10 phút | Tập trung vào hành vi hướng người dùng: initialize, streaming message, dispatch hook. Bỏ qua đường dẫn retry/error nội bộ |
| Số file khác 15 | Nhiều hơn hoặc ít hơn file .py so với dự kiến | Tổng chú thích cây thư mục sai | Dùng kết quả Glob làm chuẩn; cập nhật số đếm trong đầu ra tương ứng |
| Dữ liệu Context7 mâu thuẫn với code | Tài liệu platform mô tả hành vi khác với mã nguồn | Có thể gây nhầm lẫn về cái nào đúng | Tin tưởng mã nguồn local; ghi chú khác biệt với "Code nói X, Context7 nói Y" |
| Import vòng tròn hoặc init phức tạp | __init__.py re-export làm truy vết khó | Khó xác định ranh giới module thực tế | Theo dõi chuỗi import thực trong mỗi file; tài liệu re-export riêng |
| Transport có nhiều triển khai | Có nhiều hơn SubprocessCLITransport | Cần truy vết thêm transport | Tài liệu hoá tất cả lớp transport tìm được; tập trung chi tiết vào SubprocessCLITransport là chính |
| Context7 MCP không khả dụng | Lỗi mạng hoặc server | Không đối chiếu được | Bỏ qua Bước 6; thêm ghi chú rằng không thể đối chiếu |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)
- **Thành công 1:** Khi đọc xong tất cả 15 file mã nguồn, Khi tạo tài liệu kiến trúc, Thì mỗi trong 4 luồng (query, client, hooks, MCP) có 5+ bước với tên file:class:method chính xác
- **Thành công 2:** Khi cây thư mục được tạo, Thì mỗi file .py trong src/ được liệt kê kèm mô tả 1 dòng và tổng số khớp kết quả Glob
- **Thành công 3:** Khi cây lỗi được tài liệu hoá, Thì nó khớp với cây lỗi mô tả trong CLAUDE.md (ClaudeSDKError -> CLIConnectionError -> CLINotFoundError, cộng ProcessError, CLIJSONDecodeError, MessageParseError)
- **Thất bại:** Khi Context7 trả thông tin mâu thuẫn, Khi tài liệu hoá một luồng, Thì code local được coi là chính thống kèm ghi chú khác biệt rõ ràng

## 8. Ghi chú kỹ thuật
- SDK dùng anyio cho async -- có thể thấy mẫu `anyio.create_task_group()` trong query.py
- Giao tiếp transport là JSON streaming qua stdin/stdout với cờ CLI `--input-format stream-json`
- Giao thức điều khiển dùng `request_id` để khớp request/response -- tìm trong _internal/query.py
- Tránh từ khoá Python: `async_` ánh xạ thành `async` trên wire, `continue_` ánh xạ thành `continue`
- Decorator `@tool` được định nghĩa trong `__init__.py`, không phải module riêng
- [`sessions.py`](../../src/claude_agent_sdk/_internal/sessions.py) đọc từ `~/.claude/projects/` -- dùng cho dữ liệu session lịch sử, không phải quản lý session đang hoạt động
- Context7 chỉ có 51 snippets cho Python SDK source -- hạn chế nhưng hữu ích cho xác minh tổng quát

## 9. Rủi ro
- **Rủi ro:** 60 phút có thể chật cho việc đọc 15 file và truy vết 4 luồng kỹ lưỡng. **Giảm thiểu:** Ưu tiên 4 luồng truy vết (Bước 2-5) hơn tính đầy đủ; cây thư mục (Bước 1) và kiểu (Bước 7) có thể ngắn gọn hơn.
- **Rủi ro:** _internal/query.py có lẽ là file phức tạp nhất với giao thức điều khiển, hooks, MCP, và streaming message đan xen nhau. **Giảm thiểu:** Đọc nó nhiều lần, mỗi lần tập trung một luồng truy vết, chú ý đường dẫn code liên quan.
- **Rủi ro:** Một số mẫu nội bộ có thể không có tài liệu và khó hiểu chỉ từ code. **Giảm thiểu:** Dùng đối chiếu Context7 và ghi chú kiến trúc CLAUDE.md để lấp khoảng trống.

## Nhật ký công việc (Worklog)

### [21:36] Bước 1-7 — Đọc tất cả 15 file mã nguồn + truy vết luồng
**Kết quả:**
- Đã đọc: __init__.py, query.py, client.py, _errors.py, types.py (headers)
- Đã đọc: _internal/client.py (đầy đủ 146 dòng), _internal/query.py (350 dòng), _internal/message_parser.py, _internal/sessions.py, _internal/session_mutations.py
- Đã đọc: _internal/transport/subprocess_cli.py (100 dòng + cấu trúc)

**Các luồng đã truy vết:**
1. query() → InternalClient → SubprocessCLITransport → Query → initialize → stream → parse → yield (8 bước)
2. ClaudeSDKClient → connect → query → receive_response → interrupt → cleanup (9 bước)
3. Hook callback: CLI control_request → Query định tuyến → hook_callbacks dict → hàm async người dùng → chuyển đổi tên trường → response (7 bước)
4. MCP: CLI control_request → Query chặn → _handle_sdk_mcp_request → Server.call_tool → handler @tool → response (7 bước)

**Phát hiện quan trọng:**
- Query._read_messages() là bộ định tuyến trung tâm: control_response → pending dict, control_request → handler, regular → stream
- Khớp request_id qua anyio.Event + pending results dict
- Hook callbacks được đăng ký trong initialize() từ HookMatcher → callback_id mapping
- SDK MCP và external MCP được phân biệt bởi config type == "sdk"

**File đã tạo:** `self-explores/context/code-architecture.md`

### [21:40] Kiểm tra tiêu chí chấp nhận
- [x] 4 luồng mỗi luồng có 5+ bước — ĐẠT (8, 9, 7, 7 bước)
- [x] Mỗi file .py được liệt kê kèm mô tả — ĐẠT (15 file)
- [x] Cây lỗi được tài liệu hoá — ĐẠT (khớp CLAUDE.md)
- [x] Tổng quan hệ thống kiểu — ĐẠT (7 nhóm)
