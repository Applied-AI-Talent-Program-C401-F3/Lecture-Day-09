# Routing Decisions Log — Lab Day 09

**Nhóm:** AI Orchestrators  
**Ngày:** 14-04-2026

---

## Routing Decision #

**Task đầu vào:**
> SLA xử lý ticket P1 là bao lâu?

**Worker được chọn:** `retrieval_worker`  
**Route reason (từ trace):** `task contains P1/SLA/Ticket keyword -> retrieval preferred`  
**MCP tools được gọi:** None (gọi trực tiếp qua node wrapper)  
**Workers called sequence:** `['retrieval_worker', 'synthesis_worker']`

**Kết quả thực tế:**
- final_answer (ngắn): "Phản hồi ban đầu 15 phút, xử lý và khắc phục 4 giờ [sla_p1_2026.txt]."
- confidence: 0.57
- Correct routing? Yes

**Nhận xét:** Supervisor nhận diện đúng nhu cầu tra cứu thông tin vận hành (SLA) và chuyển hướng tới worker tìm kiếm dữ liệu thô.

---

## Routing Decision #2

**Task đầu vào:**
> Khách hàng Flash Sale yêu cầu hoàn tiền vì sản phẩm lỗi — được không?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `task contains policy/access keyword`  
**MCP tools được gọi:** `search_kb` (bên trong worker để lấy context)  
**Workers called sequence:** `['policy_tool_worker', 'retrieval_worker', 'synthesis_worker']`

**Kết quả thực tế:**
- final_answer (ngắn): "Không, đơn hàng Flash Sale không được hoàn tiền theo Điều 3 chính sách v4."
- confidence: 0.56
- Correct routing? Yes

**Nhận xét:** Supervisor nhận diện intent "hoàn tiền" là một quyết định chính sách, không chỉ đơn thuần là tìm kiếm text mà cần áp dụng logic loại trừ.

---

## Routing Decision #3

**Task đầu vào:**
> Cần cấp quyền Level 3 để khắc phục P1 khẩn cấp. Quy trình là gì?

**Worker được chọn:** `policy_tool_worker`  
**Route reason (từ trace):** `task contains policy/access keyword | risk_high flagged`  
**MCP tools được gọi:** `check_access_permission`, `search_kb`  
**Workers called sequence:** `['policy_tool_worker', 'retrieval_worker', 'synthesis_worker']`

**Kết quả thực tế:**
- final_answer (ngắn): "Quy trình cấp quyền Level 3 yêu cầu phê duyệt từ Line Manager, IT Admin và IT Security."
- confidence: 0.53
- Correct routing? Yes

**Nhận xét:** Đây là case phức tạp nhất vì chứa cả keyword chính sách và rủi ro cao. Routing đã chính xác khi chọn Auditor để kiểm tra ma trận phê duyệt.

---

## Routing Decision #4 (tuỳ chọn — bonus)

**Task đầu vào:**
> ERR-403-AUTH là lỗi gì và cách xử lý?

**Worker được chọn:** `human_review` (HITL)  
**Route reason:** `unknown error code + risk_high → human review`

**Nhận xét: Đây là trường hợp routing khó nhất trong lab. Tại sao?**
Vì mã lỗi `ERR-403-AUTH` không hề tồn tại trong bộ Knowledge Base đã nạp. Thay vì để LLM tự bịa câu trả lời, supervisor nhận diện mã lỗi lạ (keyword `err-`) và rủi ro cao để đẩy sang người duyệt. Điều này ngăn chặn hoàn toàn rủi ro hallucination trong vận hành kỹ thuật.

---

## Tổng kết

### Routing Distribution

| Worker | Số câu được route | % tổng |
|--------|------------------|--------|
| retrieval_worker | 11 | 52% |
| policy_tool_worker | 9 | 43% |
| human_review | 1 | 5% |

### Routing Accuracy

- Câu route đúng: 15 / 15 (dựa trên 15 câu test chuẩn)
- Câu route sai (đã sửa bằng cách nào?): 0 (sau khi tinh chỉnh keyword "khẩn cấp")
- Câu trigger HITL: 1

### Lesson Learned về Routing

1. **Hierarchy Matters:** Cần ưu tiên các worker có tính logic cao (Auditor) trước khi chuyển sang retrieval thông thường để tránh trả lời bề nổi.
2. **Standardization:** Việc chuyển câu hỏi về lowercase trước khi so khớp giúp tăng độ ổn định của routing lên đáng kể.

### Route Reason Quality

Các `route_reason` hiện tại đã đủ tốt để debug nhanh (ví dụ: biết ngay là do dính keyword "P1"). Tuy nhiên, nếu có thêm thông tin về **keyword nào cụ thể** đã kích hoạt route đó thì sẽ tốt hơn nữa cho việc fine-tuning.
