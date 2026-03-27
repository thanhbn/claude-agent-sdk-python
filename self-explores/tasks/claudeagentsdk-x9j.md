---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-x9j
title: "Đẩy kết quả lên Notion & Trello"
status: open
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [notion, trello, publishing, mcp, phase-3]
---

# Đẩy lên Notion & Trello — Thiết kế chi tiết

## 1. Mục tiêu

Đẩy toàn bộ kết quả nghiên cứu lên Notion (dưới mục Experiments) và tạo Trello tracking cards cho các giai đoạn dự án, với kiểm tra tiên quyết bắt buộc về khả dụng MCP và fallback nhẹ nhàng sang chỉ-lưu-local nếu nền tảng bên ngoài không truy cập được.

## 2. Phạm vi

**Trong phạm vi:**
- Kiểm tra tiên quyết khả dụng MCP cho cả Notion và Trello trước khi làm bất cứ gì
- Tạo trang Notion với tóm tắt tổng quan, sơ đồ (dạng code blocks), use cases, và Feynman learning
- Tạo Trello cards cho Phase 1, Phase 2, và Phase 3 tracking
- Xác minh nội dung truy cập được sau khi đẩy
- Fallback nhẹ nhàng sang "chỉ local" với lý do ghi rõ nếu MCP không khả dụng

**Ngoài phạm vi:**
- Format trang Notion với blocks nâng cao (databases, toggles, v.v.) — nội dung thuần đủ
- Tạo Trello boards hoặc workspaces (dùng cái hiện có)
- Tự động đồng bộ giữa files local và Notion/Trello trong tương lai
- Đẩy mã nguồn thô hoặc file binary lớn
- Thiết lập MCP servers nếu chưa cấu hình

## 3. Đầu vào / Đầu ra

**Đầu vào:**
- `self-explores/context/claude-agent-sdk-overview.md` (tóm tắt tổng quan)
- `self-explores/context/claude-agent-sdk-cheatsheet.md` (tham chiếu nhanh)
- `self-explores/tasks/claudeagentsdk-554-diagrams.md` (sơ đồ tuần tự)
- `self-explores/tasks/claudeagentsdk-7mq-usecase.md` (sơ đồ use case)
- `self-explores/learnings/2026-03-21-claude-agent-sdk-feynman.md` (Feynman learning)

**Đầu ra:**
- URL trang Notion (dưới Experiments > claude-agent-sdk-python) — hoặc ghi chú "chỉ local"
- URL Trello cards (3 cards trên list Tracking) — hoặc ghi chú "chỉ local"
- `self-explores/tasks/claudeagentsdk-x9j-report.md` — báo cáo xác nhận local với links hoặc trạng thái fallback

## 4. Phụ thuộc (Dependencies)

- `claudeagentsdk-aca` (tóm tắt tổng quan + cheatsheet) — phụ thuộc nội dung
- `claudeagentsdk-1ig` (Feynman learning) — phụ thuộc nội dung
- Phụ thuộc công cụ: `mcp__notion__*` MCP tools, `mcp__trello__*` MCP tools
- Phụ thuộc môi trường: MCP servers đã cấu hình và truy cập được

## 5. Luồng xử lý (Flow)

### Bước 0: KIỂM TRA TIÊN QUYẾT BẮT BUỘC — Khả dụng MCP (~3 phút)

Bước này PHẢI hoàn thành trước khi tạo nội dung hoặc đẩy. Không được bỏ qua.

**Kiểm tra Notion MCP:**
1. Gọi `mcp__notion__notion-search` với truy vấn "Experiments"
2. Nếu thành công: Ghi nhận page ID của mục Experiments để dùng sau
3. Nếu thất bại (lỗi kết nối, lỗi xác thực, timeout): Đặt `notion_available = false`, ghi lại thông báo lỗi

**Kiểm tra Trello MCP:**
1. Gọi `mcp__trello__get_lists` (hoặc `mcp__trello__list_boards` trước nếu chưa có active board)
2. Nếu thành công: Ghi nhận list ID của "Tracking" (hoặc list tương đương) để dùng sau
3. Nếu thất bại: Đặt `trello_available = false`, ghi lại thông báo lỗi

**Ma trận quyết định:**
| Notion | Trello | Hành động |
|--------|--------|--------|
| OK | OK | Tiếp tục với cả hai nền tảng |
| OK | LỖI | Chỉ đẩy Notion, ghi nhận Trello không khả dụng |
| LỖI | OK | Chỉ đẩy Trello, ghi nhận Notion không khả dụng |
| LỖI | LỖI | Nhảy đến Bước 4 (fallback), ghi "chỉ local" |

**Kiểm tra:** Cả hai kiểm tra MCP hoàn tất. Quyết định đã ghi nhận. Không bắt đầu công việc trên nền tảng không khả dụng.

### Bước 1: Tạo trang Notion (~5 phút)

*Bỏ qua bước này nếu `notion_available = false`.*

1. Tìm hoặc điều hướng đến trang cha Experiments (dùng page ID từ Bước 0)
2. Tạo trang mới tiêu đề "Nghiên cứu claude-agent-sdk-python" dưới Experiments
3. Cấu trúc nội dung trang:

```
# Nghiên cứu claude-agent-sdk-python
Ngày: 2026-03-21
Trạng thái: Hoàn thành (Phase 1-3)

## Tóm tắt tổng quan
(Dán nội dung từ claude-agent-sdk-overview.md)

## Sơ đồ kiến trúc
(Dán Mermaid diagrams dạng code blocks — Notion không render Mermaid native)

### Sơ đồ tuần tự: Luồng query()
(dán diagram 1 dạng code block)

### Sơ đồ tuần tự: Luồng ClaudeSDKClient
(dán diagram 2 dạng code block)

(... lặp cho cả 4 sơ đồ tuần tự)

### Sơ đồ Use Case
(dán use case diagram dạng code block)

## Hướng dẫn Use Case
(Dán phần chính từ use-case-guide.md)

## Ghi chú Feynman Learning
(Dán nội dung từ feynman.md)

## Tham chiếu nhanh
(Dán nội dung cheatsheet hoặc link đến nó)
```

4. Nếu nội dung quá lớn cho một trang (giới hạn API Notion), chia thành trang con:
   - Trang chính: Tóm tắt tổng quan + links đến trang con
   - Trang con 1: Sơ đồ kiến trúc
   - Trang con 2: Use Cases + Feynman Learning
   - Trang con 3: Cheatsheet

**Kiểm tra:** Trang Notion đã tạo và truy cập được. Nội dung khớp file nguồn. Trang xuất hiện dưới mục Experiments.

### Bước 2: Tạo Trello Cards (~5 phút)

*Bỏ qua bước này nếu `trello_available = false`.*

1. Đảm bảo active board đã đặt (gọi `mcp__trello__set_active_board` nếu cần)
2. Tìm list "Tracking" (hoặc tạo nếu chưa có bằng `mcp__trello__add_list_to_board`)
3. Tạo 3 cards:

**Card 1: "claude-agent-sdk: Phase 1 - Nền tảng"**
- Mô tả: "Đọc docs chính thức, phân tích kiến trúc code, thiết lập tài nguyên học tập"
- Checklist items:
  - Task 2e7: Đọc thư viện tài liệu chính thức
  - Task d0g: Đọc code + vẽ kiến trúc
  - Task 3ma: Tìm tài nguyên học tập

**Card 2: "claude-agent-sdk: Phase 2 - Đào sâu"**
- Mô tả: "Phân tích use case, sơ đồ tuần tự, sơ đồ use case"
- Checklist items:
  - Task qw0: Phân tích use cases
  - Task fl0: Chọn lọc tài nguyên học tập
  - Task 554: Vẽ Sequence Diagrams
  - Task 7mq: Vẽ Use Case Diagram

**Card 3: "claude-agent-sdk: Phase 3 - Tổng hợp"**
- Mô tả: "Tóm tắt tổng quan, Feynman learning, xuất bản kết quả"
- Checklist items:
  - Task aca: Tổng hợp (summary + cheatsheet)
  - Task 1ig: Feynman learning
  - Task x9j: Đẩy lên Notion & Trello

4. Thêm labels nếu có (VD: "Nghiên cứu", "claude-agent-sdk")

**Kiểm tra:** 3 cards đã tạo trên list Tracking. Mỗi card có mô tả và checklist. Cards hiện trên board.

### Bước 3: Xác minh và báo cáo (~5 phút)

1. **Xác minh Notion:**
   - Gọi `mcp__notion__notion-search` tìm "Nghiên cứu claude-agent-sdk-python"
   - Xác nhận trang tồn tại và có nội dung
   - Ghi nhận URL trang

2. **Xác minh Trello:**
   - Gọi `mcp__trello__get_cards_by_list_id` cho list Tracking
   - Xác nhận 3 cards tồn tại với tiêu đề đúng
   - Ghi nhận URLs các cards

3. **Viết báo cáo local** (`claudeagentsdk-x9j-report.md`):

```markdown
# Báo cáo đẩy — Nghiên cứu claude-agent-sdk

## Ngày: 2026-03-21

## Notion
- Trạng thái: {THÀNH CÔNG | THẤT BẠI | BỎ QUA}
- URL trang: {url hoặc "N/A"}
- Nội dung: Tóm tắt tổng quan, 5 sơ đồ, use cases, Feynman learning
- Ghi chú: {các vấn đề nếu có}

## Trello
- Trạng thái: {THÀNH CÔNG | THẤT BẠI | BỎ QUA}
- Cards:
  - Phase 1: {url hoặc "N/A"}
  - Phase 2: {url hoặc "N/A"}
  - Phase 3: {url hoặc "N/A"}
- Ghi chú: {các vấn đề nếu có}

## Tổng kết
- Nền tảng đã đẩy: {số}/2
- File local vẫn là nguồn chính xác tại: self-explores/
```

**Kiểm tra:** File báo cáo tồn tại. Tất cả URLs hợp lệ (nếu nền tảng khả dụng). Báo cáo phản ánh chính xác những gì đã đẩy.

### Bước 4: Fallback — Chỉ Local (~2 phút)

*Thực hiện bước này CHỈ KHI cả Notion và Trello đều không khả dụng.*

1. Viết file báo cáo với trạng thái "CHỈ LOCAL" cho cả hai nền tảng
2. Bao gồm thông báo lỗi cụ thể từ Bước 0
3. Thêm ghi chú: "Tất cả nội dung nghiên cứu có sẵn tại local dưới self-explores/. Để đẩy sau, chạy lại task x9j khi MCP servers đã cấu hình."
4. Tuỳ chọn: tạo file HTML index đơn giản link tất cả nội dung local để duyệt dễ dàng

**Kiểm tra:** Báo cáo ghi rõ "chỉ local" với lý do. File nội dung local đều truy cập được.

## 6. Trường hợp biên & Xử lý lỗi

| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| Notion MCP chưa cấu hình | MCP server không trong môi trường | `mcp__notion__notion-search` thất bại | Đặt `notion_available = false`, bỏ qua bước Notion, ghi trong báo cáo |
| Trello chưa có active board | Chưa đặt board hoặc user không có boards | `mcp__trello__get_lists` thất bại | Thử `mcp__trello__list_boards` trước; nếu không có boards, đặt `trello_available = false` |
| Trang Notion "Experiments" không tồn tại | Trang cha không tìm thấy khi search | Không tạo được trang con dưới Experiments | Tìm trang cha gần nhất (VD: root workspace); tạo trang ở đó; ghi vị trí trong báo cáo |
| Mermaid diagrams không render trong Notion | Notion không hỗ trợ Mermaid rendering | Diagrams hiện dạng text code block thuần | Đây là dự kiến — dán dạng fenced code blocks với tag ngôn ngữ `mermaid` để tham khảo sau |
| Nội dung quá lớn cho một trang Notion | Vượt giới hạn API payload | Tạo trang thất bại | Chia thành nhiều trang con liên kết (trang chính + 3 trang con như mô tả trong Bước 1) |
| Trello list "Tracking" không tồn tại | List không có trên active board | Không thêm được cards | Tạo list bằng `mcp__trello__add_list_to_board`, sau đó thêm cards |
| Token xác thực MCP hết hạn | Token hết hạn giữa kiểm tra tiên quyết và đẩy thực tế | Đẩy thất bại giữa chừng | Bắt lỗi, ghi nhận những gì đã đẩy thành công, ghi thất bại một phần trong báo cáo |
| Cả hai MCP khả dụng nhưng một lỗi giữa chừng | Lỗi thoáng qua khi tạo nội dung | Đẩy một phần | Hoàn thành nền tảng hoạt động, ghi thất bại một phần cho nền tảng kia, báo cáo cả hai trạng thái |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)

- **Thành công 1:** Khi cả Notion và Trello MCP đều khả dụng, Khi thực hiện đẩy, Thì ít nhất 1 nền tảng có nội dung nghiên cứu với links hoạt động được báo cáo trong file báo cáo local
- **Thành công 2:** Khi đẩy Notion thành công, Thì trang Notion chứa: tóm tắt tổng quan, ít nhất 2 sơ đồ dạng code blocks, nội dung use case, và nội dung Feynman learning — tất cả dưới mục Experiments
- **Thất bại:** Khi cả Notion và Trello MCP đều thất bại, Khi fallback được kích hoạt, Thì báo cáo local ghi "chỉ local — MCP không khả dụng" với thông báo lỗi cụ thể cho mỗi nền tảng, và tất cả file nội dung local vẫn nguyên vẹn và truy cập được

## 8. Ghi chú kỹ thuật

- Notion MCP tools dùng: `mcp__notion__notion-search` (tìm trang), `mcp__notion__notion-create-pages` (tạo trang với nội dung), `mcp__notion__notion-fetch` (xác minh trang)
- Trello MCP tools dùng: `mcp__trello__list_boards`, `mcp__trello__set_active_board`, `mcp__trello__get_lists`, `mcp__trello__add_list_to_board`, `mcp__trello__add_card_to_list`, `mcp__trello__create_checklist`, `mcp__trello__add_checklist_item`, `mcp__trello__get_cards_by_list_id`
- Notion KHÔNG render Mermaid diagrams native — chúng sẽ hiện dạng code blocks. Đây là chấp nhận được; người đọc có thể dán vào Mermaid renderer
- Format nội dung trang Notion: dùng cú pháp giống Markdown trong tham số content (Notion API chuyển thành blocks)
- Trello card descriptions hỗ trợ Markdown
- Timeout MCP: cho phép tối đa 30 giây mỗi lần gọi MCP; nếu chậm hơn, coi như thất bại
- File báo cáo local đóng vai trò audit trail bất kể đẩy thành công hay thất bại

## 9. Rủi ro

- **Rủi ro:** MCP servers có thể chưa cấu hình trong môi trường hiện tại, khiến toàn bộ task thành no-op. **Giảm thiểu:** Kiểm tra tiên quyết Bước 0 là bắt buộc và nhanh (< 3 phút). Nếu cả hai thất bại, task hoàn thành nhanh với báo cáo "chỉ local" — không lãng phí thời gian.
- **Rủi ro:** Nội dung Notion có thể mất format khi chuyển từ Markdown. **Giảm thiểu:** Giữ format đơn giản (headings, danh sách, code blocks). Tránh tables hoặc Markdown phức tạp mà Notion có thể không parse đúng.
- **Rủi ro:** Cấu trúc Trello board có thể không khớp dự kiến (không có list "Tracking", tổ chức board khác). **Giảm thiểu:** Liệt kê boards trước, chọn board phù hợp nhất, tạo list "Tracking" nếu cần.
- **Rủi ro:** Nội dung đẩy lên Notion/Trello trở nên lỗi thời khi files local cập nhật. **Giảm thiểu:** Ghi trong header trang Notion: "Nguồn chính xác: files local tại self-explores/. Lần đẩy cuối: 2026-03-21." Đây là đẩy một lần, không phải cơ chế đồng bộ.

## Nhật ký công việc (Worklog)

*(Chưa bắt đầu)*
