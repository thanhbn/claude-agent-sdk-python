---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-554
title: "Vẽ Sequence Diagrams"
status: completed
started_at: 2026-03-22 10:00
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [diagram, sequence, mermaid, architecture, phase-2]
---

# Vẽ Sequence Diagrams — Thiết kế chi tiết

## 1. Mục tiêu

Tạo 4 sơ đồ tuần tự (sequence diagram) Mermaid mô tả chính xác luồng dữ liệu runtime cho 4 mẫu tương tác cốt lõi của SDK: query() một lần, ClaudeSDKClient phiên tương tác, dispatch hook callback, và gọi MCP tool in-process.

## 2. Phạm vi

**Trong phạm vi:**
- Sơ đồ tuần tự cho luồng `query()` một lần (tạo transport, initialize, stream, teardown)
- Sơ đồ tuần tự cho phiên tương tác `ClaudeSDKClient` (kết nối, đa lượt, ngắt, ngắt kết nối)
- Sơ đồ tuần tự cho dispatch hook callback (CLI request, matcher routing, Python handler, response)
- Sơ đồ tuần tự cho gọi MCP tool in-process (CLI tool request, Query chặn, SDK MCP server, thực thi handler)
- Cú pháp Mermaid hợp lệ để render trên GitHub
- Tên participant khớp với tên class thực trong mã nguồn

**Ngoài phạm vi:**
- Sơ đồ lớp (class diagram) hoặc sơ đồ thành phần (component diagram) (task riêng)
- Giao tiếp MCP server bên ngoài (subprocess IPC, không phải in-process)
- Luồng điều phối agent (quá phức tạp cho một sơ đồ tuần tự)
- Luồng lỗi/retry (chỉ ghi chú trong diagram dạng alt block)
- Công cụ diagram tương tác (chỉ text Mermaid)

## 3. Đầu vào / Đầu ra

**Đầu vào:**
- `self-explores/context/code-architecture.md` (từ task d0g)
- File mã nguồn để xác minh:
  - [`query.py`](../../src/claude_agent_sdk/_internal/query.py) (điểm vào query)
  - [`client.py`](../../src/claude_agent_sdk/_internal/client.py) (ClaudeSDKClient)
  - [`query.py`](../../src/claude_agent_sdk/_internal/query.py) (Query xử lý giao thức điều khiển)
  - [`subprocess_cli.py`](../../src/claude_agent_sdk/_internal/transport/subprocess_cli.py) (SubprocessCLITransport)
  - [`message_parser.py`](../../src/claude_agent_sdk/_internal/message_parser.py) (phân tích message)

**Đầu ra:**
- `self-explores/tasks/claudeagentsdk-554-diagrams.md` — file duy nhất chứa tất cả 4 sơ đồ tuần tự Mermaid kèm text giải thích giữa mỗi sơ đồ

## 4. Phụ thuộc (Dependencies)

- `claudeagentsdk-d0g` (phân tích kiến trúc code) — PHẢI hoàn thành trước; cung cấp hiểu biết kiến trúc cần thiết cho diagram chính xác
- Phụ thuộc công cụ: Kiến thức cú pháp Mermaid, GitHub Markdown renderer để xác minh

## 5. Luồng xử lý (Flow)

### Bước 1: Sequence Diagram 1 — Luồng query() Một lần (~10 phút)

Vẽ toàn bộ vòng đời của một lệnh gọi `query()` từ khi user gọi đến khi yield message cuối cùng.

**Participants (trái sang phải):**
- `UserApp` — code gọi
- `query()` — hàm API công khai trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py)
- `InternalClient` — [`client.py`](../../src/claude_agent_sdk/_internal/client.py), điều phối vòng đời query
- `Query` — [`query.py`](../../src/claude_agent_sdk/_internal/query.py), xử lý giao thức điều khiển
- `Transport` — `SubprocessCLITransport`, quản lý subprocess CLI
- `CLI` — subprocess Claude Code CLI

**Tương tác:**
1. UserApp gọi `query(prompt, options)`
2. query() tạo `InternalClient`
3. InternalClient tạo `SubprocessCLITransport`
4. Transport khởi động subprocess CLI với `--input-format stream-json`
5. Transport gửi initialize request qua stdin
6. CLI phản hồi với initialize result qua stdout
7. Query gửi user prompt dạng control request
8. CLI stream JSON messages (assistant text, tool use, v.v.)
9. Transport đọc stdout từng dòng, phân tích JSON
10. `MessageParser` chuyển đổi raw dict thành đối tượng `Message` có kiểu
11. Messages được yield cho UserApp qua `AsyncIterator[Message]`
12. Khi stream kết thúc, Transport kill subprocess
13. InternalClient dọn dẹp tất cả tài nguyên

Dùng `activate`/`deactivate` để hiện thời gian sống. Thêm `note` cho "JSON streaming qua stdin/stdout".

**Kiểm tra:** Render Mermaid trong previewer Markdown hoặc paste vào Mermaid live editor. Tất cả participants phải xuất hiện, mũi tên tuần tự, không lỗi cú pháp.

### Bước 2: Sequence Diagram 2 — Phiên tương tác ClaudeSDKClient (~10 phút)

Vẽ một phiên hội thoại đa lượt thể hiện tính chất hai chiều.

**Participants:**
- `UserApp`
- `ClaudeSDKClient` — API công khai trong [`client.py`](../../src/claude_agent_sdk/_internal/client.py)
- `Transport` — `SubprocessCLITransport`
- `Query` — [`query.py`](../../src/claude_agent_sdk/_internal/query.py)
- `CLI` — Claude Code CLI

**Tương tác:**
1. UserApp vào `async with ClaudeSDKClient(options) as client:`
2. Client tạo Transport, khởi động subprocess
3. Client gửi initialize request
4. CLI trả initialize result (capabilities, session info)
5. UserApp gọi `client.query("prompt đầu tiên")`
6. Query gửi prompt cho CLI, CLI xử lý, stream phản hồi
7. UserApp nhận phản hồi qua `receive_response()`
8. UserApp gọi `client.query("câu hỏi tiếp nối")` (đa lượt)
9. CLI tiếp tục hội thoại với ngữ cảnh
10. UserApp gọi `client.interrupt()` giữa chừng
11. Query gửi interrupt control request
12. CLI xác nhận interrupt, dừng sinh (generation)
13. Khối `async with` thoát, Client ngắt kết nối, Transport kill subprocess

Dùng khối `opt` cho "Tiếp nối đa lượt" và `break` cho luồng interrupt.

**Kiểm tra:** Kiểm tra rằng entry/exit của async context manager hiện rõ ràng. Xác minh luồng interrupt khớp triển khai thực tế trong [`client.py`](../../src/claude_agent_sdk/_internal/client.py).

### Bước 3: Sequence Diagram 3 — Dispatch Hook Callback (~15 phút)

Đây là diagram phức tạp nhất. Hiện cách CLI khởi tạo hook callback và SDK dispatch nó đến hàm Python do user định nghĩa.

**Participants:**
- `CLI` — khởi tạo hook request
- `Query` — [`query.py`](../../src/claude_agent_sdk/_internal/query.py), nhận và định tuyến hook callbacks
- `HookMatcher` — logic khớp mẫu trong Query
- `UserHookFn` — hàm async Python do user cung cấp

**Tương tác:**
1. CLI gặp điểm hook (VD: PreToolUse cho Bash)
2. CLI gửi hook callback request qua stdout (JSON với `request_id`, loại hook, thông tin tool)
3. Query nhận hook callback request từ Transport
4. Query xác định loại hook (PreToolUse, PostToolUse, Stop, v.v.)
5. Query duyệt các HookMatcher đã đăng ký cho loại hook này
6. HookMatcher kiểm tra tên tool / pattern có khớp không
7. Khối `alt`: Tìm thấy khớp vs. Không khớp
8. Nếu khớp: Query gọi UserHookFn với dữ liệu sự kiện hook
9. UserHookFn thực thi logic async (chấp nhận, từ chối, sửa đổi)
10. UserHookFn trả quyết định (VD: `{"decision": "approve"}` hoặc `{"decision": "block", "reason": "..."}`)
11. Query bọc quyết định trong control response với `request_id` khớp
12. Query gửi response lại CLI qua stdin
13. CLI xử lý quyết định: tiếp tục (approve) hoặc chặn tool use (reject)

Thêm `note` giải thích: "Tên trường: async_ và continue_ trong Python, chuyển thành async/continue trên wire". Thêm khối `alt` cho quyết định approve vs. block.

**Kiểm tra:** Đối chiếu với [`query.py`](../../src/claude_agent_sdk/_internal/query.py) để xác nhận cơ chế dispatch hook. Xác minh rằng khớp `request_id` được hiển thị. Kiểm tra tính chất async của callback rõ ràng.

### Bước 4: Sequence Diagram 4 — Gọi MCP Tool In-Process (~10 phút)

Hiện cách SDK MCP server xử lý gọi tool hoàn toàn in-process không qua subprocess IPC.

**Participants:**
- `CLI` — Claude quyết định dùng tool
- `Query` — chặn gọi tool SDK MCP
- `MCPServer` — MCP server in-process tạo qua `create_sdk_mcp_server()`
- `ToolHandler` — hàm do user định nghĩa với decorator `@tool`

**Tương tác:**
1. Claude (qua CLI) quyết định gọi tool (VD: `get_weather`)
2. CLI gửi tool call request qua stdout (tên tool, arguments, request_id)
3. Query nhận tool call request
4. Query kiểm tra: đây có phải tool SDK MCP không? (vs. MCP bên ngoài)
5. Khối `alt`: SDK MCP tool vs. External MCP
6. Nếu SDK MCP: Query định tuyến đến instance MCPServer khớp
7. MCPServer tìm tool theo tên trong registry
8. MCPServer gọi `call_tool()` để thực thi ToolHandler được trang trí `@tool`
9. ToolHandler thực thi (có thể async), trả kết quả
10. MCPServer đóng gói kết quả thành MCP tool response
11. Query bọc kết quả trong control response với `request_id` khớp
12. Query gửi response cho CLI qua stdin
13. CLI nhận kết quả tool, tiếp tục sinh

Thêm `note` giải thích: "SDK MCP tools chạy trong tiến trình Python, không phải subprocess riêng".

**Kiểm tra:** Xác nhận logic chặn trong [`query.py`](../../src/claude_agent_sdk/_internal/query.py). Xác minh luồng `create_sdk_mcp_server()` và decorator `@tool` chính xác.

### Bước 5: Rà soát nhất quán và hoàn thiện (~10 phút)

- Xác minh tất cả 4 diagram dùng tên participant nhất quán (cùng class = cùng tên xuyên suốt)
- Xác minh cú pháp Mermaid: khai báo `sequenceDiagram` đúng, kiểu mũi tên đúng (`->>` cho async, `-->>` cho response)
- Đảm bảo mỗi diagram có tiêu đề dùng `title:` hoặc comment `%%`
- Thêm bảng tóm tắt đầu file đầu ra liệt kê tất cả 4 diagram kèm mô tả 1 dòng
- Test mỗi diagram render không lỗi (paste vào Mermaid live editor hoặc GitHub preview)

**Kiểm tra:** Tất cả 4 diagram render. Tên participant nhất quán. Mỗi diagram có 5+ tương tác. Không lỗi cú pháp Mermaid.

## 6. Trường hợp biên & Xử lý lỗi

| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| Lỗi cú pháp Mermaid | Gõ sai mũi tên hoặc participant | Diagram không render | Test từng diagram riêng trong Mermaid live editor trước khi hoàn thiện |
| Participant không khớp code | Tài liệu kiến trúc lỗi thời hoặc sai tên class | Diagram gây hiểu nhầm | Đối chiếu tên participant với tên class thực trong mã nguồn |
| Luồng hook quá phức tạp | Nhiều matcher, khối alt lồng nhau | Diagram không đọc được | Tách thành diagram chính "happy path" + sub-diagram "chi tiết alt" nếu cần |
| Luồng MCP nhầm lẫn external vs SDK | Trộn lẫn in-process và subprocess MCP | Gây nhầm lẫn về nơi thực thi | Dán nhãn rõ "in-process" trong notes; giữ MCP bên ngoài ngoài phạm vi |
| Chuỗi tương tác dài | >15 mũi tên trong một diagram | Render quá dài, khó theo dõi | Nhóm tương tác liên quan bằng khối `rect`; thêm `note` tóm tắt |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)

- **Thành công 1:** Khi có code-architecture.md và xác minh mã nguồn, Khi tạo 4 sơ đồ tuần tự, Thì mỗi sơ đồ có participant chính xác khớp tên class thực, 5+ tương tác mỗi diagram, và render đúng trên GitHub Markdown
- **Thành công 2:** Khi hoàn thành diagrams, Khi lập trình viên đọc mà không xem mã nguồn, Thì họ có thể mô tả chính xác luồng dữ liệu cho query(), ClaudeSDKClient, hooks, và gọi MCP tool
- **Thất bại:** Khi có lỗi cú pháp Mermaid trong diagram, Khi test diagram, Thì lỗi được phát hiện và sửa trước khi file hoàn thiện

## 8. Ghi chú kỹ thuật

- Tham chiếu cú pháp sơ đồ tuần tự Mermaid: `sequenceDiagram`, `participant`, `->>` (mũi tên liền, gọi async), `-->>` (mũi tên nét đứt, phản hồi), `activate`/`deactivate`, `alt`/`else`/`end`, `opt`/`end`, `note right of`/`note over`
- GitHub render Mermaid nguyên bản trong file `.md` trong khối code ` ```mermaid `
- Quy ước mũi tên: `->>` cho calls/requests, `-->>` cho returns/responses, `--)` cho async fire-and-forget
- Giữ alias participant ngắn để dễ đọc (VD: `participant Q as Query`)
- Chiều rộng diagram khuyến nghị tối đa: 6 participants (quá số đó nên tách)
- Chuyển đổi tên trường Python cần ghi chú: `async_` -> `async`, `continue_` -> `continue` (wire format)

## 9. Rủi ro

- **Rủi ro:** Tài liệu kiến trúc từ task d0g có thể thiếu sót hoặc sai, lan truyền vào diagram. **Giảm thiểu:** Đối chiếu mỗi participant và tương tác với mã nguồn thực tế, không chỉ dựa vào tài liệu kiến trúc.
- **Rủi ro:** Khác biệt render Mermaid giữa GitHub và previewer local. **Giảm thiểu:** Test diagram cuối cùng trên GitHub trực tiếp (hoặc dùng hỗ trợ Mermaid trong GitHub issue/PR preview).
- **Rủi ro:** Luồng hook callback đủ phức tạp để một diagram trở nên không đọc được. **Giảm thiểu:** Dùng khối `alt`/`opt` Mermaid hợp lý; nếu vẫn quá phức tạp, tạo phiên bản "happy path" đơn giản và phiên bản chi tiết.

## Nhật ký công việc (Worklog)

### [10:00] Bắt đầu — Đọc mã nguồn
- Đọc 5 file mã nguồn: query.py, client.py, _internal/query.py, _internal/client.py, subprocess_cli.py
- Xác nhận luồng chính xác từ code thực tế (không chỉ dựa vào tài liệu kiến trúc)

### [10:15] Hoàn thành — 4 Sequence Diagrams
**Kết quả:**
- Tạo file `claudeagentsdk-554-diagrams.md` với 4 sơ đồ tuần tự Mermaid
- Mỗi diagram có >= 5 tương tác, tên participant khớp tên class trong code

**Diagrams đã vẽ:**
1. **Luồng query() Một lần** — 6 participants, vòng đời đầy đủ từ UserApp → InternalClient → Transport → Query → CLI
2. **Phiên tương tác ClaudeSDKClient** — đa lượt + luồng interrupt, 3 khối rect (Lượt 1, Lượt 2, Interrupt)
3. **Dispatch Hook Callback** — callback do CLI khởi tạo, pha đăng ký + kích hoạt, khối alt cho approve/block
4. **Gọi MCP Tool SDK In-Process** — JSONRPC routing, pha init + tool call, khối alt cho server found/not found

**Phát hiện quan trọng từ mã nguồn:**
- Transport LUÔN dùng `--input-format stream-json` (subprocess_cli.py:331)
- String prompt: user message gửi sau initialize, rồi gọi wait_for_result_and_end_input()
- Hook callbacks: callback_id sinh tuần tự ("hook_0", "hook_1"...), lưu trong dict
- MCP: JSONRPC routing thủ công vì Python MCP SDK thiếu Transport abstraction
- Chuyển đổi trường: async_ → async, continue_ → continue (query.py:34-50)

**File đã tạo:**
- self-explores/tasks/claudeagentsdk-554-diagrams.md
