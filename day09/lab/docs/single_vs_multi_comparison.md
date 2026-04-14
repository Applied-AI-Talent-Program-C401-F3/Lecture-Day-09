# Single Agent vs Multi-Agent Comparison — Lab Day 09

**Nhóm:** AI Orchestrators  
**Ngày:** 14-04-2026

---

## 1. Metrics Comparison

| Metric | Day 08 (Single Agent) | Day 09 (Multi-Agent) | Delta | Ghi chú |
|--------|----------------------|---------------------|-------|---------|
| Avg confidence | 0.65 | 0.50 | -0.15 | Multi-agent khắt khe hơn khi trích dẫn nguồn. |
| Avg latency (ms) | 2,500 | 6,716 | +4,216 | Tăng do overhead điều phối và nhiều LLM calls. |
| Abstain rate (%) | 10% | 5% | -5% | Tốt hơn nhờ Policy Tool tự lấy thêm context qua MCP. |
| Multi-hop accuracy | 60% | 85% | +25% | Cải thiện rõ rệt nhờ tách biệt Auditor và Retriever. |
| Routing visibility | ✗ Không có | ✓ Có route_reason | N/A | Dễ dàng biết AI đang "nghĩ" gì. |
| Debug time (estimate) | 20 phút | 5 phút | -15 | Chỉ cần nhìn trace là biết worker nào lỗi. |

---

## 2. Phân tích theo loại câu hỏi

### 2.1 Câu hỏi đơn giản (single-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | 90% | 92% |
| Latency | 2.1s | 5.5s |
| Observation | Nhanh nhưng đôi khi bịa nguồn. | Chậm hơn nhưng trích dẫn cực kỳ chính xác. |

**Kết luận:** Multi-agent không cải thiện nhiều về accuracy cho câu hỏi dễ nhưng làm tăng latency đáng kể. Không nên dùng cho các task quá đơn giản.

### 2.2 Câu hỏi multi-hop (cross-document)

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Accuracy | 55% | 88% |
| Routing visible? | ✗ | ✓ |
| Observation | Thường bỏ sót một vế của câu hỏi. | Xử lý tuần tự qua nhiều worker nên bao quát đủ ý. |

**Kết luận:** Multi-agent là "vũ khí hạng nặng" cho các câu hỏi phức tạp cần kết hợp cả quy trình (SOP) và dữ liệu kỹ thuật (SLA).

### 2.3 Câu hỏi cần abstain

| Nhận xét | Day 08 | Day 09 |
|---------|--------|--------|
| Abstain rate | 10% | 15% |
| Hallucination cases | 2/15 | 0/15 |
| Observation | Cố gắng trả lời dù không có context. | Thà nói "Không biết" còn hơn trả lời sai nhờ HITL. |

**Kết luận:** Multi-agent an toàn hơn cho vận hành doanh nghiệp.

---

## 3. Debuggability Analysis

### Day 08 — Debug workflow
```
Khi answer sai → phải đọc toàn bộ RAG pipeline code → tìm lỗi ở indexing/retrieval/generation
Không có trace → không biết bắt đầu từ đâu
Thời gian ước tính: 20-30 phút
```

### Day 09 — Debug workflow
```
Khi answer sai → đọc trace → xem supervisor_route + route_reason
  → Nếu route sai → sửa supervisor routing logic
  → Nếu retrieval sai → test retrieval_worker độc lập
  → Nếu synthesis sai → test synthesis_worker độc lập
Thời gian ước tính: 5 phút
```

**Câu cụ thể nhóm đã debug:** Câu q09 về mã lỗi `ERR-403-AUTH`. Ban đầu system cố trả lời linh tinh, sau khi thêm logic HITL cho keyword `err-`, system đã chuyển sang trạng thái an toàn.

---

## 4. Extensibility Analysis

| Scenario | Day 08 | Day 09 |
|---------|--------|--------|
| Thêm 1 tool/API mới | Phải sửa toàn prompt | Thêm MCP tool + route rule |
| Thêm 1 domain mới | Phải retrain/re-prompt | Thêm 1 worker mới |
| Thay đổi retrieval strategy | Sửa trực tiếp trong pipeline | Sửa retrieval_worker độc lập |
| A/B test một phần | Khó — phải clone toàn pipeline | Dễ — swap worker |

**Nhận xét:** Day 09 cho phép phát triển song song (mỗi người làm 1 worker) mà không lo dẫm chân nhau.

---

## 5. Cost & Latency Trade-off

| Scenario | Day 08 calls | Day 09 calls |
|---------|-------------|-------------|
| Simple query | 1 LLM call | 2 LLM calls (Supervisor + Synthesis) |
| Complex query | 1 LLM call | 3 LLM calls (Supervisor + Worker + Synthesis) |
| MCP tool call | N/A | 0 LLM calls (Rule-based) |

**Nhận xét về cost-benefit:** Chi phí tăng gấp 2-3 lần, nhưng đổi lại là sự an toàn và chính xác cho các nghiệp vụ rủi ro cao. Đây là sự đánh đổi xứng đáng cho môi trường Enterprise.

---

## 6. Kết luận

**Multi-agent tốt hơn single agent ở điểm nào?**
1. Độ tin cậy và khả năng kiểm soát rủi ro thông qua HITL.
2. Khả năng giải quyết các câu hỏi logic phức tạp (Policy Auditor).

**Multi-agent kém hơn hoặc không khác biệt ở điểm nào?**
1. Latency chậm hơn rõ rệt (do tính tuần tự của graph).
2. Tốn kém tài nguyên LLM hơn cho mỗi lượt truy vấn.

**Khi nào KHÔNG nên dùng multi-agent?**
Khi hệ thống chỉ cần tra cứu thông tin FAQ đơn giản, không có các quy tắc ràng buộc logic hoặc không yêu cầu tính bảo mật cao.

**Nếu tiếp tục phát triển hệ thống này, nhóm sẽ thêm gì?**
Triển khai **Asynchronous Node Execution** để chạy Retrieval và Policy Auditor song song, giúp giảm latency xuống mức tương đương Day 08.
