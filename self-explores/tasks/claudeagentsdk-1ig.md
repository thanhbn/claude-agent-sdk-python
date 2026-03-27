---
date: 2026-03-21
type: task-worklog
task: claudeagentsdk-1ig
title: "Viết nội dung Feynman learning"
status: completed
detailed_at: 2026-03-21
detail_score: ready-for-dev
tags: [learning, feynman, explanation, mental-model, phase-3]
---

# Feynman Learning — Thiết kế chi tiết

## 1. Mục tiêu

Viết nội dung học theo phương pháp Feynman (~2000 từ) giúp lập trình viên Python biết async/await nhưng chưa từng dùng claude-agent-sdk có thể hiểu các khái niệm cốt lõi thông qua phép so sánh đơn giản, xác định lỗ hổng kiến thức, và sơ đồ mô hình tư duy.

## 2. Phạm vi

**Trong phạm vi:**
- Chỉ các khái niệm cốt lõi: `query()`, `ClaudeSDKClient`, hooks, MCP tools in-process
- 4 phép so sánh rõ ràng ánh xạ đến các khái niệm này
- Xác định lỗ hổng kiến thức và cách giải quyết
- Sơ đồ mô hình tư duy (Mermaid hoặc ASCII)
- Ghi rõ kiến thức tiên quyết cho đối tượng mục tiêu
- Giới hạn cứng: ~2000 từ

**Ngoài phạm vi:**
- Chủ đề nâng cao: plugins, điều phối agent, lưu trữ session, chế độ sandbox
- Tham chiếu API hoặc tài liệu tham số đầy đủ
- Hướng dẫn cài đặt/thiết lập (đã bao phủ trong cheatsheet)
- So sánh với SDK hoặc công cụ khác
- Ví dụ code dài hơn 10 dòng (đây là tài liệu khái niệm, không phải tutorial)

## 3. Đầu vào / Đầu ra

**Đầu vào:**
- `self-explores/context/claude-agent-sdk-overview.md` (từ task aca — tóm tắt tổng quan)
- `self-explores/context/claude-agent-sdk-cheatsheet.md` (từ task aca — tham chiếu nhanh)
- `self-explores/context/code-architecture.md` (từ task d0g — để lấp lỗ hổng kiến thức)
- Mã nguồn (chỉ cho Bước 3 lấp lỗ hổng): `src/claude_agent_sdk/_internal/query.py`, `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`

**Đầu ra:**
- `self-explores/learnings/2026-03-21-claude-agent-sdk-feynman.md` — file Markdown duy nhất, ~2000 từ

## 4. Phụ thuộc (Dependencies)

- `claudeagentsdk-aca` (tóm tắt tổng quan + cheatsheet) — PHẢI hoàn thành trước; cung cấp hiểu biết tổng hợp cần giải thích đơn giản
- Truy cập mã nguồn cho Bước 3 (lấp lỗ hổng) — không phụ thuộc task, chỉ cần truy cập file

## 5. Luồng xử lý (Flow)

### Bước 1: Feynman Bước 1 — Giải thích đơn giản bằng phép so sánh (~15 phút)

Viết bản thảo đầu tiên dùng 4 phép so sánh cốt lõi. Mỗi phép so sánh phải:
- Dễ liên hệ với lập trình viên Python (dùng ẩn dụ lập trình khi hữu ích)
- Ánh xạ chính xác đến hành vi SDK thực tế (không đơn giản hóa gây hiểu nhầm)
- Được giới thiệu theo mẫu "Hãy nghĩ X như là Y"

**Phép so sánh 1: query() = Gửi thư**
- Bạn viết thư (prompt), bỏ vào phong bì (options), thả vào hòm thư (hàm query)
- Bưu điện (subprocess CLI) chuyển đi, xử lý, và gửi lại phản hồi
- Bạn nhận phản hồi dưới dạng luồng từng trang (AsyncIterator[Message]) — bạn đọc khi chúng đến
- Khi trang cuối cùng đến, cuộc trao đổi kết thúc. Không có cách trả lời nếu không gửi thư mới
- Điểm mấu chốt: gửi-và-quên (fire-and-forget), không trạng thái, tự dọn dẹp

**Phép so sánh 2: ClaudeSDKClient = Cuộc gọi điện thoại**
- Bạn nhấc máy (async with ClaudeSDKClient)
- Kết nối được thiết lập (bắt tay initialize)
- Bạn nói (query), nghe (receive_response), nói lại (tiếp nối) — đó là cuộc hội thoại
- Bạn có thể ngắt giữa câu (client.interrupt())
- Khi bạn cúp máy (thoát async with), đường dây đóng và tài nguyên được giải phóng
- Điểm mấu chốt: có trạng thái, hai chiều, bạn kiểm soát vòng đời

**Phép so sánh 3: Transport = Ống nước dưới sàn nhà**
- Giữa code Python của bạn và CLI, có một ống nước (SubprocessCLITransport)
- Bạn không bao giờ chạm trực tiếp vào ống — bạn nói chuyện với query() hoặc ClaudeSDKClient, và chúng lo phần ống nước
- Ống chuyển JSON messages hai chiều (stdin = bạn đến CLI, stdout = CLI đến bạn)
- Ống luôn streaming — ngay cả query() cũng dùng chế độ stream nội bộ
- Điểm mấu chốt: tầng trừu tượng, giao thức JSON, bạn không cần biết ống tồn tại

**Phép so sánh 4: Hooks = Lính gác tại trạm kiểm soát**
- Tưởng tượng CLI đang đi dọc hành lang (thực thi tác vụ)
- Tại một số cửa (điểm hook: PreToolUse, PostToolUse, Stop), có lính gác (hàm hook của bạn)
- Lính gác kiểm tra tình hình và quyết định: "đi tiếp" (approve), "dừng lại" (block), hoặc "đi đường khác" (modify)
- Lính gác là async — họ có thể suy nghĩ mà không chặn toàn bộ tòa nhà
- Bạn cài đặt lính gác bằng cách đăng ký hook matchers chỉ định cửa nào họ canh
- Điểm mấu chốt: điểm chặn để an toàn và kiểm soát, không chặn (non-blocking), khớp mẫu (pattern-matched)

**Phép so sánh thêm: MCP Tools = Hộp dụng cụ riêng**
- CLI đi kèm các công cụ sẵn có (Bash, Read, Write, v.v.)
- MCP tools in-process cho phép bạn thêm công cụ riêng vào hộp dùng decorator `@tool`
- Khi Claude quyết định dùng công cụ của bạn, yêu cầu ở nguyên trong tiến trình Python (không subprocess)
- Hãy nghĩ như hệ thống plugin nơi bạn dạy Claude kỹ năng mới
- Điểm mấu chốt: khả năng mở rộng, thực thi in-process, dựa trên decorator

Viết mỗi phép so sánh thành 2-3 đoạn. Thêm callout "Ánh xạ đến code" sau mỗi phép.

**Kiểm tra:** Mỗi phép so sánh hiểu được mà không cần biết SDK. Mỗi phép ánh xạ đến khái niệm SDK thực. Không phép nào gây hiểu nhầm về hành vi thực tế.

### Bước 2: Feynman Bước 2 — Xác định lỗ hổng kiến thức (~10 phút)

Sau khi viết giải thích đơn giản, xác định những gì vẫn còn mơ hồ hoặc chung chung. Liệt kê lỗ hổng:

**Lỗ hổng 1: Giao thức điều khiển (Control Protocol)**
- "JSON messages qua stdin/stdout" — nhưng CỤ THỂ thế nào? Format message ra sao? Làm sao khớp request với response?
- Cơ chế khớp `request_id` không hiển nhiên
- Bắt tay initialize là giao thức cụ thể, không phải chỉ "bắt đầu nói chuyện"

**Lỗ hổng 2: Mô hình streaming**
- "Luôn streaming nội bộ" — nghĩa là gì cho query() vốn trả về một phản hồi?
- SDK đệm/phân tích JSON từng phần từ stream thế nào?
- Chuyện gì xảy ra khi CLI gửi nhiều đối tượng JSON trên cùng một dòng stdout?

**Lỗ hổng 3: Cơ chế Hook Callback**
- CLI "tạm dừng" và chờ hook response thế nào?
- Mô hình luồng (threading model) ra sao? Hook có chặn message stream không?
- Chuyển đổi tên trường async_ và continue_ được xử lý thế nào?

**Lỗ hổng 4: Mô hình async anyio**
- Tại sao anyio thay vì asyncio thuần?
- Task groups là gì và SDK dùng chúng thế nào?
- Điều này có nghĩa SDK hoạt động với trio nữa không?

Viết mỗi lỗ hổng dạng câu hỏi mà người đọc tự nhiên sẽ hỏi sau khi đọc các phép so sánh.

**Kiểm tra:** Ít nhất 4 lỗ hổng được xác định. Mỗi lỗ hổng là điểm nhầm lẫn thực sự, không phải chi tiết vặt.

### Bước 3: Feynman Bước 3 — Quay lại mã nguồn để lấp lỗ hổng (~10 phút)

Với mỗi lỗ hổng, đọc mã nguồn liên quan và viết giải đáp rõ ràng:

**Giải quyết Lỗ hổng 1:** Đọc `_internal/query.py` — mô tả giao thức điều khiển:
- Mỗi request có `request_id` (UUID)
- Responses bao gồm `request_id` khớp
- Lớp `Query` duy trì dict các pending requests và giải quyết futures khi responses đến

**Giải quyết Lỗ hổng 2:** Đọc `_internal/transport/subprocess_cli.py` — mô tả streaming:
- Transport đọc stdout từng dòng
- Mỗi dòng là một đối tượng JSON hoàn chỉnh (JSON phân cách bằng newline, tức NDJSON)
- `MessageParser` chuyển raw dicts thành dataclasses có kiểu
- query() thu thập stream nội bộ; ClaudeSDKClient phơi bày stream ra ngoài

**Giải quyết Lỗ hổng 3:** Đọc `_internal/query.py` phần xử lý hook:
- CLI gửi hook callback request (chỉ là JSON message khác với request_id)
- Query nhận nó, gọi hàm async Python đã đăng ký
- Hàm Python trả quyết định
- Query gửi quyết định ngược lại dạng response
- CLI đang chờ response này trước khi tiếp tục — đồng bộ từ góc nhìn CLI

**Giải quyết Lỗ hổng 4:** Giải thích ngắn gọn anyio:
- anyio là lớp trừu tượng trên asyncio và trio
- SDK dùng `anyio.create_task_group()` cho các thao tác đồng thời
- Thực tế, hầu hết người dùng sẽ dùng asyncio — hỗ trợ trio là bonus
- Task groups đảm bảo tất cả tasks được dọn dẹp đúng cách

**Kiểm tra:** Mỗi lỗ hổng có giải đáp cụ thể. Giải đáp chính xác (đối chiếu mã nguồn). Ngôn ngữ vẫn đơn giản.

### Bước 4: Feynman Bước 4 — Đơn giản hóa và tạo sơ đồ mô hình tư duy (~10 phút)

Tạo một sơ đồ mô hình tư duy kết nối mọi thứ lại. Đây nên là hình ảnh "eureka" (khoảnh khắc hiểu ra).

**Sơ đồ mô hình tư duy (Mermaid):**
- Hiện 4 tầng: Code của bạn -> Điểm vào SDK -> Query Engine -> Transport -> CLI
- Hiện hooks dạng kênh phụ tại tầng Query Engine
- Hiện MCP tools dạng phần mở rộng tại tầng Query Engine
- Dùng mã màu hoặc nhãn cho "bạn viết phần này" vs "SDK xử lý phần này" vs "CLI xử lý phần này"

**Lượt đơn giản hóa:**
- Đọc lại toàn bộ tài liệu
- Loại bỏ mọi thuật ngữ chuyên môn chưa được định nghĩa
- Thay thế câu bị động bằng câu chủ động
- Đảm bảo mỗi đoạn trả lời "vậy sao?" — tại sao người đọc quan tâm?
- Thêm phần "Tiếp theo làm gì" chỉ đến cheatsheet cho code thực hành

**Kiểm tra:** Sơ đồ render được trong Mermaid. Tài liệu chảy logic từ phép so sánh -> lỗ hổng -> giải đáp -> mô hình tư duy. Tổng số từ ~2000 (+/- 200).

## 6. Trường hợp biên & Xử lý lỗi

| Trường hợp | Điều kiện kích hoạt | Hành vi mong đợi | Cách khôi phục |
|------|---------|----------|----------|
| Nội dung quá kỹ thuật | Dùng thuật ngữ chuyên môn chưa định nghĩa, hoặc giải thích chi tiết triển khai | Người đọc mục tiêu (test sinh viên năm 2) sẽ không hiểu | Áp dụng bộ lọc "giải thích như đang dạy lập trình viên mới"; thay thuật ngữ bằng tiếng thường |
| Nội dung quá dài | Vượt quá 2000 từ | Vi phạm ràng buộc format | Cắt: bỏ giải thích trùng lặp, gộp đoạn tương tự, chuyển chi tiết vào chú thích |
| Phép so sánh gây hiểu nhầm | Phép so sánh ánh xạ sai hành vi thực (VD: "gửi-và-quên" ngụ ý không có phản hồi) | Người đọc xây dựng mô hình tư duy sai | Xác minh mỗi phép so sánh theo hành vi code thực; thêm callout "phép so sánh bị sai ở đâu" |
| Mã nguồn thay đổi kể từ phân tích kiến trúc | SDK cập nhật giữa task d0g và task 1ig | Giải đáp lỗ hổng có thể sai | Luôn xác minh giải đáp theo mã nguồn hiện tại, không chỉ tài liệu kiến trúc |
| Người đọc không biết async | Nội dung giả định biết async | Người đọc bị lạc ngay từ đầu | Thêm ghi chú tiên quyết rõ ràng ở đầu: "Tiên quyết: Python async/await (cơ bản asyncio)" |

## 7. Tiêu chí chấp nhận (Acceptance Criteria)

- **Thành công 1:** Khi nội dung Feynman learning viết xong, Khi lập trình viên Python async chưa từng dùng claude-agent-sdk đọc nó, Thì họ có thể giải thích query(), ClaudeSDKClient, hooks, và MCP tools làm gì mà không cần xem tài liệu, và có thể viết script hoạt động dùng query()
- **Thành công 2:** Khi nội dung hoàn thành, Thì chứa 4 phép so sánh rõ ràng cho các khái niệm cốt lõi (query, client, hooks, MCP), mỗi phép có callout "ánh xạ đến code", và bao gồm sơ đồ mô hình tư duy render được trong Mermaid
- **Thất bại:** Khi người đọc không biết Python async/await, Khi họ bắt đầu đọc, Thì phần tiên quyết ở đầu ghi rõ "Tiên quyết: Python async/await" để họ biết cần học trước

## 8. Ghi chú kỹ thuật

- Phương pháp Feynman: (1) Chọn khái niệm, (2) Giải thích đơn giản, (3) Xác định lỗ hổng, (4) Đơn giản hóa tiếp. Task này thực hiện cả 4 bước.
- Mục tiêu số từ: ~2000 từ. Kiểm tra bằng `wc -w`. Phạm vi chấp nhận: 1800-2200.
- Đối tượng mục tiêu chính xác: "Lập trình viên Python có thể viết `async def main(): await asyncio.gather(...)` nhưng chưa nghe nói đến claude-agent-sdk"
- Sơ đồ mô hình tư duy: dùng Mermaid `graph TD` (trên-xuống) cho cái nhìn kiến trúc phân tầng
- Phép so sánh nên trung lập văn hóa (tránh ẩn dụ thể thao, tham chiếu cụ thể quốc gia)
- "Test sinh viên năm 2": sau khi viết mỗi phần, hỏi "sinh viên năm 2 biết Python có hiểu được không?" Nếu không, đơn giản hóa.
- Vị trí file: thư mục `self-explores/learnings/` (có thể cần tạo)

## 9. Rủi ro

- **Rủi ro:** Phép so sánh đơn giản hóa quá mức đến sai, khiến người đọc xây dựng mô hình tư duy sai. **Giảm thiểu:** Mỗi phép so sánh có ghi chú "phép so sánh bị sai ở đâu". Đối chiếu mỗi phép với mã nguồn, không chỉ tài liệu kiến trúc.
- **Rủi ro:** Phương pháp Feynman yêu cầu hiểu biết thực sự; nếu người viết không nắm vững SDK, lỗ hổng sẽ hời hợt. **Giảm thiểu:** Bước 3 yêu cầu đọc mã nguồn thực cho mỗi lỗ hổng, không chỉ tóm tắt kiến trúc. Nếu lỗ hổng không giải quyết được từ code, ghi rõ.
- **Rủi ro:** 2000 từ có thể không đủ để bao phủ 4 khái niệm với phép so sánh + lỗ hổng + giải đáp + sơ đồ. **Giảm thiểu:** Tập trung vào mô hình tư duy, không phải chi tiết đầy đủ. Mỗi khái niệm ~400 từ (phép so sánh + lỗ hổng + giải đáp), còn ~400 từ cho giới thiệu, sơ đồ, và kết luận.
- **Rủi ro:** Sơ đồ mô hình tư duy có thể đơn giản hóa quá mức kiến trúc phân tầng. **Giảm thiểu:** Sơ đồ bổ sung cho text, không thay thế. Gắn nhãn "mô hình tư duy đơn giản hóa" và link đến sơ đồ tuần tự chi tiết từ task 554.

## Nhật ký công việc (Worklog)

### [11:15] Bắt đầu + Hoàn thành
**Kết quả:**
- Viết nội dung Feynman learning: 1880 từ (trong phạm vi 1800-2200)
- 4 bước Feynman đầy đủ:
  1. Giải thích đơn giản với 5 phép so sánh: query=Thư, Client=Điện thoại, Transport=Ống nước, Hooks=Lính gác, MCP=Hộp dụng cụ
  2. Xác định 4 lỗ hổng: giao thức điều khiển, mô hình streaming, cơ chế tạm dừng hook, anyio
  3. Giải quyết lỗ hổng từ kiến thức mã nguồn
  4. Sơ đồ mô hình tư duy (Mermaid graph TD) + lượt đơn giản hóa
- Ví dụ code cho mỗi phép so sánh (3-10 dòng)
- Phần "Tiếp theo làm gì" link đến cheatsheet + video workshop

**File đã tạo:**
- self-explores/learnings/2026-03-21-claude-agent-sdk-feynman.md (1880 từ)
