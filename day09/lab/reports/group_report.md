# Báo Cáo Nhóm — Lab Day 09: Multi-Agent Orchestration

**Tên nhóm:** AI Orchestrators  
**Thành viên:**
| Tên | Vai trò | Email |
|-----|---------|-------|
| Khương Hải Lâm | Supervisor Owner (Lead) |  |
| Lưu Lê Gia Bảo | Supervisor Owner (Routing) | baoluu@example.com |
| Thái Doãn Minh Hải | Worker Owner (R&S) | |
| Đặng Tuấn Anh | Worker Owner (Policy) |  |
| Hoàng Quốc Hùng | MCP Owner |  |
| Lương Trung Kiên | Trace & Docs Owner |  |

**Ngày nộp:** 14-04-2026  
**Repo:** https://github.com/Applied-AI-Talent-Program-C401-F3/Lecture-Day-09.git
**Độ dài khuyến nghị:** 600–1000 từ

---

## 1. Kiến trúc nhóm đã xây dựng (150–200 từ)

**Hệ thống tổng quan:**
Nhóm đã xây dựng hệ thống đa tác tử theo mô hình **Supervisor-Worker**. Hệ thống bao gồm một **Policy Orchestrator** (do Khương Hải Lâm thiết kế) chịu trách nhiệm điều phối nhiệm vụ cho 3 Worker chuyên biệt: `retrieval_worker` (Thái Doãn Minh Hải phụ trách), `policy_tool_worker` (Đặng Tuấn Anh phụ trách) và `synthesis_worker`. Kiến trúc này giúp tách biệt việc tìm kiếm văn bản thô với việc áp dụng các quy tắc logic kinh doanh.

**Routing logic cốt lõi:**
Supervisor sử dụng **Keyword-based Routing** (do Lưu Lê Gia Bảo triển khai) kết hợp với phân tích rủi ro. Các từ khóa như "hoàn tiền", "quyền truy cập" sẽ ưu tiên route vào `policy_tool_worker`. Các mã lỗi lạ hoặc yêu cầu khẩn cấp sẽ trigger trạng thái `risk_high`. Điểm đặc biệt là hệ thống hỗ trợ **Dynamic Context Loading**: nếu Policy Worker nhận thấy thiếu dữ liệu, nó có thể tự gọi thêm retrieval qua MCP (do Hoàng Quốc Hùng phát triển) thay vì chờ Supervisor.

**MCP tools đã tích hợp:**
- `search_kb`: Công cụ tìm kiếm kiến thức chuẩn hóa cho tất cả các worker.
- `check_access_permission`: Công cụ kiểm tra ma trận phê duyệt (Line Manager, IT Admin, IT Security).
- `get_ticket_info`: Tra cứu trạng thái ticket Jira thời gian thực.

Ví dụ trace: `[policy_tool_worker] called MCP check_access_permission for Level 3`.

---

## 2. Quyết định kỹ thuật quan trọng nhất (200–250 từ)

**Quyết định:** Sử dụng logic "Temporal Scoping" (Phân vùng thời gian) cứng trong Policy Worker thay vì để LLM tự suy luận phiên bản chính sách.

**Bối cảnh vấn đề:**
Trong câu `gq02`, khách hàng đặt đơn ngày 31/01/2026 (trước khi chính sách v4 có hiệu lực vào 01/02/2026). Nếu để LLM tự đọc, nó thường bị nhầm lẫn và áp dụng quy tắc v4 cho đơn hàng cũ, dẫn đến sai sót về mặt pháp lý (Hallucination).

**Các phương án đã cân nhắc:**

| Phương án | Ưu điểm | Nhược điểm |
|-----------|---------|-----------|
| LLM-based Versioning | Linh hoạt, không cần sửa code khi có bản v5, v6 | Dễ bị "rò rỉ" thông tin giữa các phiên bản, không ổn định |
| Rule-based Temporal Check | Chính xác tuyệt đối, đảm bảo tính tuân thủ | Cần cập nhật code khi có thay đổi mốc thời gian |

**Phương án đã chọn và lý do:**
Nhóm chọn **Rule-based Temporal Check**. Trong doanh nghiệp, việc áp dụng sai phiên bản chính sách là lỗi nghiêm trọng. Đặng Tuấn Anh đã mã hóa mốc 01/02/2026 vào worker để tự động đánh dấu các đơn hàng cũ là "Ineligible for V4 Analysis" và yêu cầu hệ thống phải "Abstain" (từ chối trả lời) nếu thiếu tài liệu V3.

**Bằng chứng từ trace/code:**
Trace `gq02`: "Theo thông tin từ tài liệu, đơn hàng này thuộc về chính sách hoàn tiền phiên bản 3... không đủ thông tin để xác định...". Kết quả này khớp hoàn toàn với grading criteria.

---

## 3. Kết quả grading questions (150–200 từ)

**Tổng điểm raw ước tính:** 83 / 96 (Phân tích bởi Lương Trung Kiên)

**Câu pipeline xử lý tốt nhất:**
- ID: **gq10** (Flash Sale) — Lý do: Worker nhận diện chính xác ngoại lệ Flash Sale override điều kiện "lỗi nhà sản xuất", đưa ra kết luận "KHÔNG được hoàn tiền" kèm trích dẫn Điều 3 chính sách v4 rõ ràng.

**Câu pipeline fail hoặc partial:**
- ID: **gq09** (Level 2 Emergency) — Fail ở phần điều kiện phê duyệt. Hệ thống nêu cần "Tech Lead phê duyệt bằng lời", trong khi đáp án đúng là "Line Manager + IT Admin on-call".
- ID: **gq01** (Notification) — Partial: Thiếu kênh "PagerDuty" trong danh sách thông báo.
- **Root cause:** Context retrieval lấy được thông tin về Tech Lead từ một chunk phụ hoặc LLM bị "nhiễu" kiến thức cũ. Phần PagerDuty bị mất do synthesis worker cố gắng trả lời quá súc tích.

**Câu gq07 (abstain):** Pipeline xử lý rất tốt, trả về đúng câu "Không đủ thông tin trong tài liệu nội bộ" thay vì bịa ra mức phạt tài chính.

**Câu gq09 (multi-hop khó nhất):** Trace ghi nhận được chuỗi gọi: `retrieval_worker` -> `policy_tool_worker` (gọi MCP `check_access_permission`) -> `synthesis_worker`. Tuy nhiên logic phê duyệt vẫn bị sai do xung đột thông tin trong context.

---

## 4. So sánh Day 08 vs Day 09 — Điều nhóm quan sát được (150–200 từ)

**Metric thay đổi rõ nhất (có số liệu):**
- **Latency:** Tăng từ ~2.5s (Day 08) lên **6.7s** (Day 09).
- **Accuracy (Multi-hop):** Tăng từ ~60% lên **85%**. Day 09 xử lý các câu phức tạp như gq03 (Level 3 access) tốt hơn hẳn nhờ bước Auditor riêng biệt.

**Điều nhóm bất ngờ nhất khi chuyển từ single sang multi-agent:**
Khả năng "tự kiểm soát lỗi" (Self-Correction). Khi Supervisor gán nhãn `risk_high`, Synthesis Worker tự động trở nên khắt khe hơn trong việc trích dẫn nguồn, giúp triệt tiêu hoàn toàn các câu trả lời mang tính phỏng đoán ở Day 08.

**Trường hợp multi-agent KHÔNG giúp ích hoặc làm chậm hệ thống:**
Với các câu hỏi thông tin đơn lẻ như `gq04` (số % store credit), việc đi qua Supervisor và Policy Worker là dư thừa, làm tăng thời gian chờ đợi của user gấp 3 lần mà kết quả không thay đổi so với RAG đơn giản.

---

## 5. Phân công và đánh giá nhóm (100–150 từ)

**Phân công thực tế:**

| Thành viên | Phần đã làm | Sprint |
|------------|-------------|--------|
| Khương Hải Lâm | Quản lý AgentState và điều phối luồng Graph | 1 |
| Lưu Lê Gia Bảo | Logic định tuyến Supervisor & HITL | 1 |
| Thái Doãn Minh Hải | Xây dựng Retrieval & Synthesis (Strict Grounding) | 2 |
| Đặng Tuấn Anh | Triển khai Policy Auditor & Temporal Scoping | 2 |
| Hoàng Quốc Hùng | Phát triển MCP Server và tích hợp Tools | 3 |
| Lương Trung Kiên | Phân tích Trace Metrics & Viết Báo cáo | 4 |

**Điều nhóm làm tốt:** Phối hợp định nghĩa "Worker Contract" từ sớm giúp các module tích hợp vào Graph mà không bị lỗi kiểu dữ liệu.

**Điều nhóm làm chưa tốt:** Việc tinh chỉnh Prompt cho Synthesis chưa đủ bao quát để liệt kê đầy đủ 100% các chi tiết nhỏ (như kênh PagerDuty trong gq01).

**Nếu làm lại, nhóm sẽ thay đổi gì trong cách tổ chức?**
Nhóm sẽ dành thêm thời gian cho bước "Context Cleaning" trước khi đưa vào Synthesis để loại bỏ các thông tin gây nhiễu (như vai trò Tech Lead không liên quan trong gq09).

---

## 6. Nếu có thêm 1 ngày, nhóm sẽ làm gì? (50–100 từ)

Nhóm sẽ triển khai **Reranker** cho Retrieval Worker và nâng cấp Supervisor lên **LLM Classifier** siêu nhỏ (như gpt-4o-mini với 1-shot) để thay thế bộ từ khóa, giúp xử lý tốt hơn các câu hỏi có thuật ngữ biến biến đổi mà vẫn giữ latency ở mức chấp nhận được.

---
*File này lưu tại: `reports/group_report.md`*
