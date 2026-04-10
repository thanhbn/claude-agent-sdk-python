---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-7mq
title: "Vẽ Use Case Diagram"
status: completed
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [diagram, use-case, uml, draw-io, mermaid, phase-2]
---

# Vẽ Use Case Diagram — Thiết kế chi tiết

## 1. Mục tiêu

Tạo sơ đồ ca sử dụng (Use Case diagram) UML với các actor được phân loại đúng (primary, system, external) thể hiện tất cả khả năng chính của SDK, dùng draw.io MCP làm công cụ chính và Mermaid flowchart làm phương án dự phòng.

## 2. Phạm vi

**Trong phạm vi:**
- 3 actor với phân loại UML đúng (Developer = primary, Claude Code CLI = system, External MCP Server = external)
- 8+ ca sử dụng bao phủ toàn bộ bề mặt khả năng SDK
- Quan hệ actor-use case (association, include, extend)
- Nhóm ca sử dụng theo ranh giới hệ thống con (subsystem boundary)
- Cả hai cách tiếp cận draw.io và Mermaid dự phòng

**Ngoài phạm vi:**
- Sơ đồ hoạt động (activity diagram) hoặc sơ đồ trạng thái (state diagram) (task khác)
- Chi tiết triển khai nội bộ bên trong ca sử dụng
- Ca sử dụng ngoài SDK (VD: dùng CLI trực tiếp không qua SDK)
- Hook System như actor (nó là hệ thống con/cơ chế, không phải actor)
- Đặc tả chi tiết ca sử dụng/narrative (đã bao phủ trong task qw0)

## 3. Đầu vào / Đầu ra

**Đầu vào:**
- `self-explores/context/use-case-guide.md` (từ task qw0) — ca sử dụng đã phân tích kèm ánh xạ actor
- Bề mặt API công khai SDK từ `src/claude_agent_sdk/__init__.py` và [`client.py`](../../src/claude_agent_sdk/_internal/client.py)

**Đầu ra:**
- Chính: file diagram draw.io (nếu MCP khả dụng) tại `self-explores/tasks/claudeagentsdk-7mq-usecase.drawio` hoặc tương đương
- Dự phòng: `self-explores/tasks/claudeagentsdk-7mq-usecase.md` chứa Mermaid flowchart theo phong cách use case diagram
- Cả hai: text mô tả giải thích lý do phân loại actor

## 4. Phụ thuộc (Dependencies)

- `claudeagentsdk-qw0` (hướng dẫn use case) — PHẢI hoàn thành trước; cung cấp ca sử dụng đã phân tích và xác định actor
- Phụ thuộc công cụ: `mcp__drawio__open_drawio_mermaid` (chính), cú pháp Mermaid (dự phòng)

## 5. Luồng xử lý (Flow)

### Bước 1: Định nghĩa Actor với phân loại UML (~5 phút)

Thiết lập 3 actor với UML stereotype đúng:

1. **Developer** (Actor chính, hình người bên TRÁI)
   - Con người viết code Python dùng SDK
   - Khởi tạo tất cả ca sử dụng
   - Stereotype: `<<primary>>`

2. **Claude Code CLI** (Actor hệ thống, hình người bên PHẢI hoặc hộp)
   - Subprocess mà SDK bọc lại
   - Tham gia tất cả ca sử dụng như backend thực thi
   - Stereotype: `<<system>>`

3. **External MCP Server** (Actor bên ngoài, hình người bên PHẢI)
   - MCP server bên thứ ba kết nối qua subprocess
   - Chỉ tham gia ca sử dụng MCP bên ngoài
   - Stereotype: `<<external>>`

**Tại sao KHÔNG có Hook System như actor:** Trong UML, actor là thực thể bên ngoài ranh giới hệ thống tương tác với nó. Hook System nằm bên trong SDK — nó là cơ chế/hệ thống con, không phải actor. Hooks được kích hoạt bởi Developer (người định nghĩa) và CLI (gửi hook callbacks).

**Kiểm tra:** Danh sách actor tuân theo quy ước UML. Không hệ thống con nội bộ nào bị phân loại nhầm thành actor.

### Bước 2: Liệt kê và phân loại Ca sử dụng (~10 phút)

Trích xuất ca sử dụng từ kết quả qw0 và tổ chức theo hệ thống con:

**Hệ thống con Truy vấn cốt lõi:**
- UC1: Thực hiện truy vấn đơn giản (Developer -> CLI)
- UC2: Chạy hội thoại tương tác (Developer -> CLI)
- UC3: Stream message phản hồi (Developer -> CLI)

**Hệ thống con Công cụ & Mở rộng:**
- UC4: Định nghĩa MCP tool tuỳ chỉnh (Developer)
- UC5: Kết nối MCP server bên ngoài (Developer -> External MCP Server)
- UC6: Điều phối agent với tools (Developer -> CLI -> External MCP Server)

**Hệ thống con Kiểm soát & An toàn:**
- UC7: Kiểm soát quyền tool qua hooks (Developer -> CLI)
- UC8: Quản lý chế độ phân quyền (Developer -> CLI)
- UC9: Đặt giới hạn ngân sách/token (Developer -> CLI)

**Hệ thống con Session & Cấu hình:**
- UC10: Tuỳ chỉnh system prompt (Developer -> CLI)
- UC11: Quản lý trạng thái session (Developer -> CLI)
- UC12: Cấu hình tham số model (Developer -> CLI)

Xác định quan hệ `<<include>>` và `<<extend>>`:
- UC2 `<<include>>` UC3 (tương tác luôn stream)
- UC6 `<<include>>` UC5 (điều phối agent cần MCP)
- UC7 `<<extend>>` UC1 (hooks tuỳ chọn mở rộng queries)
- UC9 `<<extend>>` UC1 (ngân sách tuỳ chọn mở rộng queries)

**Kiểm tra:** Ít nhất 8 ca sử dụng duy nhất. Mỗi ca có ít nhất một liên kết actor. Quan hệ include/extend đúng ngữ nghĩa.

### Bước 3: Vẽ Diagram với draw.io MCP (~10 phút)

**Cách tiếp cận chính — draw.io MCP:**

Dùng `mcp__drawio__open_drawio_mermaid` để tạo diagram. Vì draw.io MCP chấp nhận cú pháp Mermaid, chuẩn bị Mermaid graph đại diện use case diagram:

```
graph LR
    subgraph "claude-agent-sdk System Boundary"
        UC1[Thực hiện truy vấn đơn giản]
        UC2[Chạy hội thoại tương tác]
        ...
    end
    Developer((Developer)) --> UC1
    CLI((Claude Code CLI)) --> UC1
    ...
```

Tạo kiểu actor dạng vòng tròn `(( ))`, ca sử dụng dạng hình chữ nhật bo góc `[ ]`, ranh giới hệ thống dạng `subgraph`.

**Cách tiếp cận dự phòng — Mermaid trong Markdown:**

Nếu `mcp__drawio__open_drawio_mermaid` lỗi hoặc không khả dụng, tạo diagram tương tự dạng Mermaid flowchart trong file Markdown. Dùng:
- `graph LR` (bố cục trái-sang-phải)
- `subgraph` cho ranh giới hệ thống
- `(( ))` cho actor (hình tròn)
- `([  ])` cho ca sử dụng (hình sân vận động, gần nhất với hình bầu dục UML)
- `-.->` cho `<<extend>>`, `-->` cho association, `==>` cho `<<include>>`

**Kiểm tra:** Diagram render không lỗi. Tất cả 3 actor hiện diện. Tất cả ca sử dụng trong ranh giới hệ thống. Quan hệ có kiểu mũi tên đúng.

### Bước 4: Rà soát và thêm mô tả (~5 phút)

- Thêm chú giải (legend) giải thích actor stereotype và kiểu quan hệ
- Thêm mô tả ngắn (1 câu) cho mỗi ca sử dụng phía dưới diagram
- Kiểm tra đầy đủ: mọi ca sử dụng từ Bước 2 đều xuất hiện trong diagram
- Kiểm tra đúng đắn: không ca sử dụng nào có liên kết actor bất khả thi
- Thêm tiêu đề và ngày vào file đầu ra

**Kiểm tra:** Chú giải có mặt. Tất cả ca sử dụng có mô tả. Liên kết actor hợp lý.

## 6. Trường hợp biên & Xử lý lỗi

| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| draw.io MCP không khả dụng | MCP server không cấu hình hoặc kết nối lỗi | Không tạo được diagram draw.io | Chuyển ngay sang Mermaid flowchart trong file Markdown |
| Quá nhiều ca sử dụng | >15 ca sử dụng | Diagram lộn xộn không đọc được | Nhóm theo hệ thống con dùng khối `subgraph` lồng nhau; giới hạn 12 quan trọng nhất |
| Mermaid không có UC diagram native | Cú pháp Mermaid thiếu từ khoá `usecase` | Không tạo được UML use case diagram thật | Dùng `graph LR` với node có kiểu: `(( ))` cho actor, `([  ])` cho ca sử dụng |
| Hướng dẫn use case (qw0) chưa đầy đủ | Task qw0 chưa hoàn thành hoặc thiếu output | Thiếu ca sử dụng hoặc thông tin actor | Trích xuất ca sử dụng trực tiếp từ bề mặt API công khai SDK làm backup |
| draw.io MCP cho layout khó đọc | Auto-layout đặt phần tử kém | Diagram khó hiểu | Thêm gợi ý vị trí rõ ràng hoặc chuyển sang Mermaid nơi layout dễ đoán hơn |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)

- **Thành công 1:** Khi ca sử dụng đã phân tích trong task qw0, Khi tạo diagram, Thì chứa 3+ actor phân loại đúng (primary/system/external), 8+ ca sử dụng, và quan hệ UML đúng (association, include, extend)
- **Thành công 2:** Khi hoàn thành diagram, Khi lập trình viên hoặc PM xem nó, Thì họ có thể xác định tất cả khả năng SDK chính và actor nào liên quan mà không cần đọc tài liệu
- **Thất bại:** Khi draw.io MCP không khả dụng, Khi dùng Mermaid flowchart dự phòng, Thì diagram Mermaid vẫn đại diện tất cả 3 actor, 8+ ca sử dụng, và render đúng trên GitHub

## 8. Ghi chú kỹ thuật

- Công cụ draw.io MCP: `mcp__drawio__open_drawio_mermaid` — chấp nhận cú pháp Mermaid và mở trong draw.io để chỉnh sửa trực quan
- Mermaid KHÔNG có kiểu `usecaseDiagram` native (khác PlantUML); phải mô phỏng bằng `graph` hoặc `flowchart`
- PlantUML có cú pháp `@startuml` use case native nhưng không render native trên GitHub — Mermaid được ưu tiên cho tương thích GitHub
- Cho Mermaid dự phòng, dùng quy ước node sau:
  - Actor: `ActorName((Tên Actor))` (ngoặc kép = hình tròn)
  - Ca sử dụng: `UC1([Tên Ca Sử Dụng])` (ngoặc + móc vuông = hình sân vận động/bầu dục)
  - Ranh giới hệ thống: `subgraph "Tên Hệ Thống"` ... `end`
- Quy ước mũi tên: `-->` liền = association, `-.->|<<extend>>|` nét đứt = extend, `-->|<<include>>|` liền có nhãn = include

## 9. Rủi ro

- **Rủi ro:** draw.io MCP có thể không được cấu hình trong môi trường hiện tại, lãng phí thời gian cho cách tiếp cận chính. **Giảm thiểu:** Test khả dụng MCP bằng lệnh gọi đơn giản trước khi đầu tư thời gian vào format draw.io. Giới hạn 2 phút cho test.
- **Rủi ro:** Mermaid flowchart trông quá khác so với UML use case diagram chuẩn, gây nhầm lẫn cho người quen UML. **Giảm thiểu:** Thêm chú giải rõ ràng và ghi chú "Render dạng flowchart do hạn chế Mermaid; tuân theo ngữ nghĩa UML use case."
- **Rủi ro:** Ca sử dụng từ task qw0 có thể không bao phủ hết khả năng SDK nếu phân tích qw0 chưa đầy đủ. **Giảm thiểu:** Đối chiếu ca sử dụng với exports API công khai SDK (`__init__.py`) để phát hiện thiếu sót.

## Nhật ký công việc (Worklog)

### [10:40] Bắt đầu
- Đọc use-case-guide.md + worklog task chi tiết

### [10:50] Hoàn thành — Use Case Diagram
**Kết quả:**
- Tạo `claudeagentsdk-7mq-usecase.md` với Mermaid use case diagram
- Mở draw.io MCP thành công — diagram render trong trình duyệt

**Diagram:**
- 3 actor đúng UML: Developer (primary), Claude Code CLI (system), External MCP Server (external)
- 12 ca sử dụng chia 4 hệ thống con:
  - Core Query (3): Truy vấn đơn giản, Hội thoại tương tác, Stream Messages
  - Tool & Extension (3): Custom MCP Tools, External MCP, Điều phối Agent
  - Control & Safety (4): Hooks, Chế độ phân quyền, Ngân sách, Quyền Tool
  - Session & Config (2): System Prompt/Model, Trạng thái Session
- 2 quan hệ include: UC2→UC3, UC6→UC5
- 3 quan hệ extend: UC7→UC1, UC9→UC1, UC10→UC2
- Mã màu: xanh lá (core), xanh dương (tools), đỏ (control), tím (config)
- Chú giải + bảng mô tả cho mỗi ca sử dụng

**File đã tạo:**
- self-explores/tasks/claudeagentsdk-7mq-usecase.md (Mermaid + mô tả)
- Diagram draw.io mở qua MCP
