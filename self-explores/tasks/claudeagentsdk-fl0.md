---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-fl0
title: "P2: Vẽ Architecture Diagram (Mermaid graph TD)"
status: completed
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [research, diagram, architecture, mermaid, p2, visualization]
---

# Vẽ Architecture Diagram — Thiết kế chi tiết

## 1. Mục tiêu
Tạo sơ đồ kiến trúc Mermaid `graph TD` hiển thị 4+ tầng của Claude Agent SDK, với mũi tên luồng dữ liệu và thành phần được gắn nhãn, render được trên GitHub Markdown.

## 2. Phạm vi
**Trong phạm vi:**
- Sơ đồ kiến trúc chính với 5 tầng: API công khai, Nội bộ, Transport, Bên ngoài, Xuyên suốt (Cross-cutting)
- Mũi tên luồng dữ liệu hiện cách prompts chảy từ code user đến CLI và phản hồi chảy ngược lại
- Nhãn thành phần kèm tên module khớp file mã nguồn thực tế
- Sub-diagram chi tiết cho hệ thống con phức tạp nếu diagram chính quá chật
- Xác minh cú pháp Mermaid render đúng trên GitHub

**Ngoài phạm vi:**
- Sơ đồ tuần tự (sequence diagram) (chỉ sơ đồ kiến trúc/thành phần)
- Sơ đồ lớp UML (quá chi tiết cho tổng quan)
- Diagram tương tác hoặc có animation
- Format Draw.io hoặc ngoài Mermaid
- Diagram của hạ tầng test hoặc CI/CD pipeline
- Diagram nội bộ CLI binary (chỉ ranh giới SDK-CLI)

## 3. Đầu vào / Đầu ra
**Đầu vào:**
- `self-explores/context/code-architecture.md` (từ task claudeagentsdk-d0g) -- cây thư mục có chú thích, 4 luồng đã truy vết, cây lỗi, tổng quan hệ thống kiểu
- Phần kiến trúc CLAUDE.md -- cho mô tả tầng cấp cao

**Đầu ra:**
- `self-explores/tasks/claudeagentsdk-fl0-diagrams.md` -- File Markdown chứa:
  1. Sơ đồ kiến trúc chính (Mermaid `graph TD`)
  2. Tuỳ chọn: Sơ đồ chi tiết hệ thống Hook
  3. Tuỳ chọn: Sơ đồ chi tiết luồng MCP tool
  4. Chú giải giải thích mã màu và kiểu mũi tên
  5. Mô tả ngắn đi kèm mỗi diagram

## 4. Phụ thuộc (Dependencies)
- **Phụ thuộc task:**
  - `claudeagentsdk-d0g` (P1: kiến trúc code) -- BẮT BUỘC. Tài liệu kiến trúc cung cấp kiến thức cấu trúc cần thiết cho diagram chính xác. Task này không thể bắt đầu cho đến khi d0g hoàn thành.
- **Phụ thuộc công cụ:**
  - Read tool -- đọc code-architecture.md đầu vào
  - Write tool -- tạo file đầu ra
  - Không cần MCP tools (đây là task tổng hợp, không phải nghiên cứu)

## 5. Luồng xử lý (Flow)

### Bước 1: Thiết kế bố cục -- 5 tầng và kiểm kê thành phần (~10 phút)
Đọc `self-explores/context/code-architecture.md` từ task d0g. Trích xuất tất cả thành phần và tổ chức thành 5 tầng:

**Tầng 1 -- API công khai (trên cùng, hướng user):**
- Hàm `query()` (query.py)
- Lớp `ClaudeSDKClient` (client.py)
- Decorator `@tool` (__init__.py)
- `create_sdk_mcp_server()` (__init__.py)

**Tầng 2 -- Xử lý nội bộ:**
- `InternalClient` (_internal/client.py) -- chỉ dùng bởi query()
- `Query` (_internal/query.py) -- xử lý giao thức điều khiển, streaming message, dispatch hook
- `MessageParser` (_internal/message_parser.py) -- JSON -> đối tượng Message có kiểu

**Tầng 3 -- Transport:**
- `SubprocessCLITransport` (_internal/transport/subprocess_cli.py) -- vòng đời subprocess, pipe stdin/stdout/stderr, phát hiện CLI binary

**Tầng 4 -- Bên ngoài (dưới cùng, ngoài SDK):**
- Tiến trình Claude Code CLI (subprocess)
- MCP Server bên ngoài (tiến trình riêng)

**Tầng 5 -- Xuyên suốt (bảng bên):**
- Types (types.py) -- Message, ClaudeAgentOptions, content blocks
- Errors (_errors.py) -- Cây lỗi ClaudeSDKError
- Sessions (sessions.py) -- đọc session lịch sử

Lập kế hoạch bố cục trực quan:
- Luồng trên-xuống-dưới cho đường request/response chính
- Bên trái: đường query()
- Bên phải: đường ClaudeSDKClient
- Cả hai hội tụ tại tầng Query/Transport
- Types xuyên suốt ở lề phải
- Mũi tên: liền cho luồng dữ liệu, nét đứt cho cấu hình/sử dụng kiểu

**Kiểm tra:** Tất cả thành phần từ code-architecture.md được đặt vào đúng một tầng. Không thành phần nào bị thiếu.

### Bước 2: Viết sơ đồ kiến trúc chính (~15 phút)
Viết code Mermaid `graph TD`. Nguyên tắc thiết kế:
- Dùng subgraph cho mỗi tầng kèm nhãn mô tả
- Mã màu node: xanh lá cho API công khai, xanh dương cho nội bộ, cam cho transport, xám cho bên ngoài
- Gắn nhãn mũi tên với kiểu dữ liệu (VD: "ClaudeAgentOptions", "JSON messages", "Message objects")
- Giữ tên node ngắn nhưng nhận biết được (VD: `QF[query&lpar;&rpar;]`, `CSC[ClaudeSDKClient]`)

**Kiểm tra:** Code Mermaid hợp lệ cú pháp (không có ngoặc chưa đóng, cú pháp mũi tên đúng). Tất cả 5 tầng được đại diện. Luồng dữ liệu hai chiều (request xuống, response lên).

### Bước 3: Viết diagram chi tiết nếu cần (~10 phút)
Nếu diagram chính có hơn ~25 node và trở nên khó đọc, tách thành sub-diagram tập trung:

**Sub-diagram A: Luồng hệ thống Hook**
Hiện: kiểu sự kiện PreToolUse, PostToolUse, Stop và chữ ký callback.

**Sub-diagram B: Luồng MCP Tool In-Process**
Hiện: Phân biệt SDK MCP (in-process) và MCP bên ngoài (subprocess).

**Sub-diagram C: Cây lỗi (Error Hierarchy)**
Sơ đồ cây đơn giản.

Chỉ tạo sub-diagram nào thêm sự rõ ràng vượt ngoài diagram chính. Nếu diagram chính đủ, bỏ qua bước này.

**Kiểm tra:** Mỗi sub-diagram có tiêu đề rõ và giải thích 1-2 câu. Cú pháp Mermaid hợp lệ.

### Bước 4: Xác minh render trên GitHub Markdown (~5 phút)
Rà soát toàn bộ code Mermaid cho các vấn đề cú pháp phổ biến:
- Dấu ngoặc trong nhãn node phải escape: `&lpar;` và `&rpar;` hoặc dùng ngoặc vuông
- Dấu ngoặc kép quanh nhãn có ký tự đặc biệt
- Tiêu đề subgraph trong dấu ngoặc kép
- Nhãn mũi tên dạng `|"nhãn"|`
- Không trùng node ID
- Định nghĩa style class sau định nghĩa graph

Thêm prose xung quanh file markdown:
- Giới thiệu ngắn giải thích diagram hiện gì
- Chú giải cho màu sắc và kiểu mũi tên
- Ghi chú cách đọc diagram (trên = code user, dưới = tiến trình CLI)

**Kiểm tra:** Mở file và kiểm tra trực quan các khối Mermaid cho lỗi cú pháp. Xác nhận tất cả tầng được gắn nhãn. Xác nhận mũi tên có nhãn hướng.

## 6. Trường hợp biên & Xử lý lỗi
| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| Diagram chính quá phức tạp | Hơn 25-30 node từ tài liệu kiến trúc | Diagram không đọc được khi render | Tách thành tổng quan chính (chỉ thành phần chính) + sub-diagram chi tiết cho mỗi hệ thống con |
| Lỗi cú pháp Mermaid | Ký tự đặc biệt trong nhãn, ngoặc chưa đóng | Diagram không render trên GitHub | Escape ký tự đặc biệt; dùng nhãn đơn giản không dấu ngoặc; test bằng Mermaid Live Editor |
| Nhãn chồng chéo khi render | Tên thành phần dài chật diagram | Trực quan kém | Rút ngắn tên (VD: "SubprocessCLITransport" -> "CLITransport"); dùng viết tắt kèm chú giải |
| code-architecture.md chưa có | Task d0g chưa hoàn thành | Thiếu dữ liệu đầu vào | Không thể tiến hành — task này phụ thuộc cứng vào d0g. Chờ hoặc dùng phần kiến trúc CLAUDE.md làm fallback tối thiểu |
| Kiến trúc có nhiều tầng hơn dự kiến | Code tiết lộ thêm tầng trừu tượng | Mô hình 5 tầng không đủ | Thêm tầng vào mô hình; diagram phải phản ánh thực tế, không phải cấu trúc dự định |
| Không tương thích phiên bản Mermaid | GitHub dùng Mermaid phiên bản cũ | Tính năng cú pháp nâng cao không hỗ trợ | Dùng cú pháp `graph TD` cơ bản; tránh flowchart-v2 hoặc tính năng thử nghiệm |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)
- **Thành công 1:** Khi có code-architecture.md từ task d0g, Khi tạo diagram chính, Thì nó render trên GitHub Markdown với 4+ tầng, mũi tên luồng dữ liệu giữa các tầng, và tất cả thành phần chính được gắn nhãn kèm tên file mã nguồn
- **Thành công 2:** Khi lập trình viên chưa quen codebase xem diagram, Khi họ đọc nó, Thì họ có thể xác định: hai điểm vào (query/ClaudeSDKClient), tầng xử lý nội bộ, ranh giới transport-CLI, và các mối quan tâm xuyên suốt — mà không đọc mã nguồn
- **Thất bại:** Khi diagram chính vượt 25 node, Khi khả năng đọc bị ảnh hưởng, Thì diagram được tách thành tổng quan cấp cao cộng 1-2 sub-diagram chi tiết, mỗi cái render độc lập

## 8. Ghi chú kỹ thuật
- **Phương ngữ Mermaid:** Dùng `graph TD` (trên-xuống). KHÔNG dùng C4 diagrams (hỗ trợ GitHub hạn chế) hoặc từ khoá `flowchart` (dùng `graph` cho tương thích tối đa)
- **Mã màu qua classDef:**
  - Xanh lá (`#d4edda`) -- Tầng API công khai
  - Xanh dương (`#cce5ff`) -- Tầng xử lý nội bộ
  - Vàng/Cam (`#fff3cd`) -- Tầng Transport
  - Xám (`#e2e3e5`) -- Tiến trình bên ngoài
  - Đỏ/Hồng (`#f8d7da`) -- Mối quan tâm xuyên suốt
- **Quy ước node ID:** Dùng viết tắt in hoa ngắn (QF, CSC, IC, Q, MP, SCT, CLI)
- **Kiểu mũi tên:** `-->` cho luồng dữ liệu, `-.->` cho quan hệ tuỳ chọn/cấu hình, `==>` cho lưu lượng hai chiều nặng
- **Render Mermaid trên GitHub:** Bọc trong khối triple-backtick với identifier ngôn ngữ `mermaid`
- **Ký tự đặc biệt:** Escape `()` trong nhãn bằng `&lpar;` `&rpar;` hoặc tránh chúng (dùng `["hàm query"]` thay vì `["query()"]`)

## 9. Rủi ro
- **Rủi ro:** Render Mermaid khác nhau giữa GitHub, VS Code preview, và renderer Markdown khác. **Giảm thiểu:** Chỉ dùng cú pháp `graph TD` cơ bản nhất; tránh tính năng nâng cao. Test bằng cách đối chiếu với spec Mermaid.
- **Rủi ro:** Kiến trúc có thể có nhiều sắc thái hơn một diagram đơn lẻ thể hiện được. **Giảm thiểu:** Diagram chính hiện cấu trúc "happy path"; sub-diagram xử lý độ phức tạp của hệ thống hooks và MCP.
- **Rủi ro:** Diagram có thể lỗi thời khi SDK phát triển. **Giảm thiểu:** Ghi phiên bản SDK (v0.1.48) và ngày trong header file diagram; ghi chú rằng diagram phản ánh code tại ngày hoàn thành task.

## Nhật ký công việc (Worklog)

### [10:20] Bắt đầu
- Đọc code-architecture.md + mã nguồn đã có trong context từ task trước

### [10:35] Hoàn thành — 6 diagrams
**Kết quả:**
- Tạo file `claudeagentsdk-fl0-diagrams.md` với 6 sơ đồ Mermaid

**Diagrams đã vẽ:**
1. **Kiến trúc chính** — graph TD, 5 tầng (Public/Internal/Transport/External/Cross-cutting), mã màu, 13 node, mũi tên luồng dữ liệu có nhãn
2. **Định tuyến message giao thức điều khiển** — Logic định tuyến Query._read_messages(): control_response/control_request/regular → 3 handler
3. **Kiến trúc hệ thống Hook** — Đăng ký (initialize) → Kích hoạt (CLI) → Dispatch (Query) → Luồng phản hồi
4. **SDK MCP vs External MCP** — So sánh song song: in-process vs subprocess
5. **Cây lỗi (Error Hierarchy)** — Cây: ClaudeSDKError → 5 lớp con
6. **Kiểm kê thành phần** — Bảng tổng hợp 11 thành phần với file, dòng, tầng, vai trò

**File đã tạo:**
- self-explores/tasks/claudeagentsdk-fl0-diagrams.md
