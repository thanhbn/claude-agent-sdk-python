---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-qw0
title: "P2: Nghiên cứu use cases & chiến lược sử dụng hiệu quả"
status: completed
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [research, use-cases, strategy, p2, examples, context7]
---

# Nghiên cứu use cases & chiến lược sử dụng hiệu quả — Thiết kế chi tiết

## 1. Mục tiêu
Phân tích tất cả 18 file ví dụ và ứng dụng demo Context7 để tạo hướng dẫn use case với 7+ danh mục và ma trận quyết định query() vs ClaudeSDKClient, giúp lập trình viên chọn đúng cách tiếp cận cho mọi tình huống.

## 2. Phạm vi
**Trong phạm vi:**
- Đọc và phân loại tất cả 18 file trong thư mục `examples/`
- Lấy Context7 `/anthropics/claude-agent-sdk-demos` (345 snippets) cho các mẫu bổ sung
- Tạo 7+ danh mục use case kèm mô tả
- Xây dựng ma trận quyết định query() vs ClaudeSDKClient với 5+ dòng tình huống
- Tài liệu hoá các mẫu phổ biến (patterns) và mẫu nên tránh (anti-patterns) kèm tham chiếu code snippet
- Xác định tính năng SDK nào mỗi ví dụ minh hoạ

**Ngoài phạm vi:**
- Chạy hoặc test bất kỳ code ví dụ nào
- Tạo file ví dụ mới
- Đo hiệu năng (benchmarking) các cách tiếp cận khác nhau
- So sánh với thư viện AI SDK khác
- Sửa đổi ví dụ hiện có

## 3. Đầu vào / Đầu ra
**Đầu vào:**
- Thư mục `examples/` (18 file):
  - `quick_start.py` -- cách dùng cơ bản
  - `streaming_mode.py`, `streaming_mode_ipython.py`, `streaming_mode_trio.py` -- các biến thể streaming
  - `hooks.py` -- cách dùng hệ thống hook
  - `mcp_calculator.py` -- MCP tools in-process
  - `agents.py`, `filesystem_agents.py` -- định nghĩa agent
  - `tool_permission_callback.py`, `tools_option.py` -- kiểm soát công cụ
  - `system_prompt.py` -- tuỳ chỉnh system prompt
  - `setting_sources.py` -- quản lý cài đặt
  - `max_budget_usd.py` -- kiểm soát ngân sách
  - `include_partial_messages.py` -- xử lý message từng phần
  - `stderr_callback_example.py` -- callback lỗi
  - `plugin_example.py` + `plugins/` -- hệ thống plugin
  - (các file còn lại sẽ được xác nhận trong Bước 1)
- Context7 MCP: `/anthropics/claude-agent-sdk-demos` (345 snippets)
- Kết quả task trước: `self-explores/context/docs-summary.md` (từ 3ma), `self-explores/context/code-architecture.md` (từ d0g)

**Đầu ra:**
- `self-explores/context/use-case-guide.md` -- Hướng dẫn toàn diện chứa:
  1. Danh mục use case (7+ danh mục)
  2. Phân tích theo danh mục (tính năng sử dụng, điểm vào, độ phức tạp)
  3. Ma trận quyết định query() vs ClaudeSDKClient (5+ dòng)
  4. Mẫu phổ biến kèm tham chiếu file ví dụ
  5. Mẫu nên tránh (anti-patterns) và lưu ý
  6. Mẫu nâng cao từ demo Context7

## 4. Phụ thuộc (Dependencies)
- **Phụ thuộc task:**
  - `claudeagentsdk-3ma` (P1: tổng quan docs) -- cần hiểu khái niệm về tính năng SDK
  - `claudeagentsdk-d0g` (P1: kiến trúc code) -- cần hiểu luồng nội bộ để giải thích tại sao chọn query() vs ClaudeSDKClient
- **Phụ thuộc công cụ:**
  - Read tool -- đọc tất cả 18 file ví dụ
  - Glob tool -- khám phá tất cả file trong examples/
  - Context7 MCP (`mcp__context7__query-docs`) -- mẫu ứng dụng demo
  - Write tool -- tạo file đầu ra
- **Thư mục:** `self-explores/context/` đã tồn tại từ task trước

## 5. Luồng xử lý (Flow)

### Bước 1: Đọc tất cả 18 file ví dụ (~20 phút)
Dùng Glob khám phá tất cả file trong `examples/`:
```
Glob pattern: examples/**/*.py
```

Đọc toàn bộ mỗi file. Với mỗi ví dụ, tài liệu hoá:
- **Tên file và mục đích** (1 câu)
- **Điểm vào sử dụng:** query() hoặc ClaudeSDKClient hoặc cả hai
- **Tính năng SDK minh hoạ:** (hooks, MCP, agents, streaming, permissions, budget, v.v.)
- **Mức độ phức tạp:** đơn giản (< 30 dòng), trung bình (30-80 dòng), nâng cao (80+ dòng)
- **Mẫu code chính:** Kỹ thuật chính ví dụ dạy

Đọc theo nhóm 3-4 file để duy trì ngữ cảnh. Thứ tự ưu tiên:
1. `quick_start.py` trước (hiểu nền tảng)
2. Ví dụ API cốt lõi: `streaming_mode.py`, `system_prompt.py`, `max_budget_usd.py`
3. Ví dụ tính năng: `hooks.py`, `mcp_calculator.py`, `agents.py`, `tool_permission_callback.py`
4. Ví dụ nâng cao: `plugins/`, `filesystem_agents.py`, `include_partial_messages.py`
5. Các file còn lại

**Kiểm tra:** Tất cả file tìm được bởi Glob đã được đọc và tài liệu hoá. Số đếm khớp dự kiến 18.

### Bước 1b: Lấy ứng dụng demo Context7 (~5 phút)
Dùng `mcp__context7__query-docs` với library ID `/anthropics/claude-agent-sdk-demos`:
- Truy vấn 1: `"demo applications use cases hooks MCP tools agents"`
- Truy vấn 2: `"real world examples patterns production"` (nếu truy vấn 1 không đủ)

Trích xuất:
- Tên ứng dụng demo và mô tả
- Các mẫu không có trong thư mục `examples/` local
- Mẫu cấp production (xử lý lỗi, retry, logging)
- Ví dụ điều phối đa agent (multi-agent orchestration)

**Kiểm tra:** Tìm được ít nhất 3 mẫu hoặc use case bổ sung ngoài ví dụ local.

### Bước 2: Phân loại thành 7+ danh mục use case (~10 phút)
Dựa trên phân tích từ Bước 1 và 1b, tạo các danh mục. Danh mục dự kiến:

1. **Truy vấn một lần (One-shot Query)** -- Xử lý hàng loạt, tích hợp CI/CD, script
   - Ví dụ: `quick_start.py`, file dùng `query()`
   - Mẫu: Gửi-và-quên (fire-and-forget), không cần trạng thái

2. **Hội thoại tương tác** -- Giao diện chat, REPL, phiên debug
   - Ví dụ: file dùng `ClaudeSDKClient` với đa lượt (multi-turn)
   - Mẫu: Vòng đời session, câu hỏi tiếp nối

3. **Công cụ tuỳ chỉnh qua MCP** -- Mở rộng Claude với công cụ chuyên biệt
   - Ví dụ: `mcp_calculator.py`
   - Mẫu: Decorator `@tool`, thực thi in-process

4. **Kiểm soát dựa trên Hook** -- Cổng bảo mật, logging, lọc nội dung
   - Ví dụ: `hooks.py`
   - Mẫu: Callback PreToolUse/PostToolUse, quyết định cho phép/từ chối

5. **Điều phối Agent** -- Hệ thống đa agent, tác vụ uỷ quyền
   - Ví dụ: `agents.py`, `filesystem_agents.py`
   - Mẫu: Định nghĩa agent, cấu hình subagent

6. **Quản lý quyền & An toàn** -- Kiểm soát truy cập, hạn chế công cụ
   - Ví dụ: `tool_permission_callback.py`, `tools_option.py`
   - Mẫu: Callback can_use_tool, chế độ phân quyền

7. **Streaming & Thời gian thực** -- Giao diện trực tiếp, theo dõi tiến trình, kết quả từng phần
   - Ví dụ: `streaming_mode.py`, `include_partial_messages.py`
   - Mẫu: Lặp async, xử lý message từng phần

Danh mục bổ sung nếu ví dụ hỗ trợ:
8. **Kiểm soát ngân sách & Chi phí** -- Giới hạn sử dụng, theo dõi chi phí
9. **Hệ thống Plugin** -- Kiến trúc plugin mở rộng
10. **Cài đặt & Cấu hình** -- Quản lý cài đặt động

Với mỗi danh mục, tài liệu hoá: mô tả, file ví dụ, khuyến nghị điểm vào, độ phức tạp, tính năng SDK chính sử dụng.

**Kiểm tra:** Ít nhất 7 danh mục được định nghĩa, mỗi danh mục ánh xạ đến 1+ file ví dụ. Không file ví dụ nào không được phân loại.

### Bước 3: Tạo ma trận quyết định query() vs ClaudeSDKClient (~10 phút)
Xây dựng bảng với cột: Tình huống, Điểm vào khuyến nghị, Lý do, File ví dụ

Tối thiểu 5 dòng:

| Tình huống | Khuyến nghị | Lý do | Ví dụ |
|----------|-------------|--------|---------|
| Một prompt, không tiếp nối | `query()` | Không trạng thái, đơn giản hơn, tự dọn dẹp | `quick_start.py` |
| Hội thoại đa lượt | `ClaudeSDKClient` | Duy trì trạng thái session, hỗ trợ tiếp nối | (ví dụ client) |
| Tích hợp pipeline CI/CD | `query()` | Không cần session, gửi-và-quên | -- |
| Công cụ MCP tuỳ chỉnh | Cả hai (ưu tiên Client cho phức tạp) | Cả hai hỗ trợ MCP; Client cho phép thay đổi tool động | `mcp_calculator.py` |
| Cổng bảo mật dựa trên Hook | `ClaudeSDKClient` | Hooks cần ngữ cảnh session cho quyết định có trạng thái | `hooks.py` |
| Giao diện streaming thời gian thực | `ClaudeSDKClient` | Message từng phần, hỗ trợ ngắt (interrupt) | `streaming_mode.py` |
| Điều phối agent | `ClaudeSDKClient` | Vòng đời phức tạp, trạng thái đa agent | `agents.py` |
| Xử lý hàng loạt giới hạn ngân sách | `query()` | Giới hạn ngân sách đơn giản, không cần trạng thái | `max_budget_usd.py` |

Với mỗi dòng, cũng ghi chú:
- Tính năng SDK cần thiết
- Độ phức tạp (đơn giản/trung bình/nâng cao)
- Lỗi thường gặp cần tránh

**Kiểm tra:** Ma trận có 5+ dòng. Mỗi dòng có khuyến nghị rõ ràng kèm lý do. Ít nhất 3 dòng khuyến nghị query() và 3 dòng khuyến nghị ClaudeSDKClient.

### Bước 4: Tài liệu hoá mẫu, mẫu nên tránh, và code snippet (~5 phút)
**Mẫu phổ biến (nên làm):**
- Sử dụng async context manager đúng cách cho ClaudeSDKClient
- Xử lý lỗi với cây ClaudeSDKError
- Mẫu lặp streaming message
- Cấu trúc hook callback (chữ ký hàm async)
- Định nghĩa MCP tool với decorator @tool

**Mẫu nên tránh (không nên làm):**
- Không await cleanup (quên `async with` hoặc `__aexit__`)
- Chặn (blocking) trong async callbacks (dùng I/O đồng bộ trong hàm hook)
- Bỏ qua message từng phần trong streaming (bỏ lỡ cập nhật trạng thái)
- Hardcode đường dẫn CLI thay vì dùng auto-discovery của SDK
- Không xử lý CLINotFoundError khi khởi động

Với mỗi mẫu/mẫu nên tránh, tham chiếu file ví dụ cụ thể minh hoạ (hoặc sẽ minh hoạ) nó.

**Kiểm tra:** Ít nhất 3 mẫu và 3 mẫu nên tránh được tài liệu hoá, mỗi cái kèm tham chiếu file ví dụ.

## 6. Trường hợp biên & Xử lý lỗi
| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| Một số ví dụ không chạy được | Ví dụ cần CLI binary hoặc API key không có sẵn | Không kiểm tra hành vi được | Vẫn phân loại dựa trên đọc code; ghi chú "chưa xác minh lúc chạy" |
| Repo demo có mẫu không có trong ví dụ local | Demo Context7 bao phủ tình huống nâng cao | Cần thêm danh mục | Bao gồm dạng phần "Mẫu nâng cao (từ demo)"; đánh dấu rõ nguồn |
| Số file ví dụ khác 18 | Glob tìm nhiều hơn hoặc ít hơn file | Ánh xạ danh mục có thể không đầy đủ | Dùng kết quả Glob thực tế; cập nhật số đếm trong đầu ra |
| Ví dụ dùng API đã deprecated | Ví dụ dùng tên cũ ClaudeCodeOptions | Phân loại gây hiểu nhầm | Ghi chú deprecation; tài liệu hoá tương đương hiện tại |
| Kết quả task trước chưa có | Task 3ma/d0g chưa hoàn thành | Thiếu ngữ cảnh cho lý do ma trận quyết định | Tiếp tục với hiểu biết hiện tại từ ghi chú kiến trúc CLAUDE.md; cập nhật hướng dẫn sau nếu cần |
| Demo Context7 không khả dụng | MCP server hoặc library ID lỗi | Không có mẫu bổ sung từ demo | Tập trung hoàn toàn vào ví dụ local; ghi chú "mẫu demo không khả dụng" |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)
- **Thành công 1:** Khi tất cả 18 ví dụ và demo Context7 đã phân tích, Khi tạo hướng dẫn, Thì có 7+ danh mục với mỗi danh mục ánh xạ đến ít nhất 1 file ví dụ và chứa mô tả, khuyến nghị điểm vào, và mức độ phức tạp
- **Thành công 2:** Khi ma trận quyết định đã tạo, Khi lập trình viên đọc nó, Thì họ có thể xác định nên dùng query() hay ClaudeSDKClient cho 5+ tình huống phổ biến, kèm lý do rõ ràng cho mỗi khuyến nghị
- **Thành công 3:** Khi mẫu và mẫu nên tránh đã tài liệu hoá, Khi lập trình viên xem xét, Thì mỗi mẫu có tham chiếu file ví dụ cụ thể và mỗi anti-pattern có giải thích vấn đề gì xảy ra
- **Thất bại:** Khi một số ví dụ không chạy được vì thiếu CLI, Khi phân loại, Thì ví dụ vẫn được phân loại dựa trên đọc code kèm ghi chú "hành vi chưa xác minh lúc chạy"

## 8. Ghi chú kỹ thuật
- Thư mục `examples/` có thể chứa thư mục con (VD: `plugins/`) -- dùng Glob đệ quy
- SDK đổi tên từ "Claude Code SDK" sang "Claude Agent SDK" -- một số ví dụ có thể dùng tên cũ
- `streaming_mode_trio.py` dùng Trio thay vì asyncio -- ghi chú đây là ví dụ async runtime thay thế
- `streaming_mode_ipython.py` dành cho ngữ cảnh IPython/Jupyter -- ghi chú môi trường thực thi khác
- Thư viện demo Context7: `/anthropics/claude-agent-sdk-demos` (345 snippets)
- Ma trận quyết định nên cân nhắc cả ràng buộc kỹ thuật và trải nghiệm lập trình viên (DX)
- Anti-patterns nên tham chiếu các kiểu lỗi từ [`_errors.py`](../../src/claude_agent_sdk/_errors.py) khi áp dụng

## 9. Rủi ro
- **Rủi ro:** 50 phút có thể chật cho đọc 18 file + demo + tạo hướng dẫn toàn diện. **Giảm thiểu:** Đọc nhóm ví dụ; không quá 2 phút cho mỗi ví dụ đơn giản. Phân bổ 20 phút đọc, 30 phút phân tích và viết.
- **Rủi ro:** Ví dụ có thể quá đơn giản và không minh hoạ độ phức tạp thực tế. **Giảm thiểu:** Demo Context7 bổ sung mẫu production; cũng suy luận tình huống thực tế từ bề mặt API.
- **Rủi ro:** Ma trận quyết định có thể quá đơn giản cho trường hợp biên. **Giảm thiểu:** Thêm cột "Ghi chú" cho sắc thái; thêm phần "khi câu trả lời không rõ ràng."

## Nhật ký công việc (Worklog)

### [00:10] Bước 1-4 — Đọc ví dụ + demo Context7 + phân loại + ma trận
**Kết quả:**
- Đã đọc 16 file .py (không phải 18 — 2 là file config plugin, không phải .py)
- 7 file dùng query(), 9 file dùng ClaudeSDKClient, không file nào dùng cả hai
- Demo Context7: 4 mẫu TypeScript (WebSocket chat, file validation hook, email MCP server, AI classification)

**Danh mục đã xác định: 10**
1. Truy vấn một lần (One-Shot Query) (7 ví dụ)
2. Hội thoại tương tác (Interactive Conversation) (3 ví dụ)
3. Công cụ tuỳ chỉnh qua MCP (1 ví dụ + demo)
4. Kiểm soát dựa trên Hook (1 ví dụ, 5 mẫu)
5. Quyền & An toàn (Permission & Safety) (1 ví dụ)
6. Điều phối Agent (Agent Orchestration) (2 ví dụ)
7. Streaming & Thời gian thực (2 ví dụ + demo)
8. Kiểm soát ngân sách & Chi phí (1 ví dụ)
9. Cấu hình & Cài đặt (3 ví dụ)
10. Hệ thống Plugin (1 ví dụ)

**Ma trận quyết định: 12 dòng** với tình huống, khuyến nghị, lý do, file ví dụ

**Mẫu: 5 phổ biến + 5 nên tránh** kèm code snippets

**File đã tạo:** `self-explores/context/use-case-guide.md`

### [00:15] Kiểm tra tiêu chí chấp nhận
- [x] 10 danh mục (mục tiêu: 7+) — ĐẠT
- [x] Ma trận quyết định 12 dòng (mục tiêu: 5+) — ĐẠT
- [x] Khuyến nghị query() vs ClaudeSDKClient rõ ràng — ĐẠT
- [x] 5 mẫu + 5 anti-patterns kèm tham chiếu ví dụ — ĐẠT
- [x] Mẫu demo Context7 đã bao gồm — ĐẠT
