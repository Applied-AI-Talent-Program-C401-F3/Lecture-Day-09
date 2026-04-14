# System Architecture — Lab Day 09

**Nhóm:** AI Orchestrators  
**Ngày:** 14-04-2026  
**Version:** 1.0

---

## 1. Tổng quan kiến trúc

Hệ thống được thiết kế theo mô hình **Supervisor-Worker**, trong đó một Supervisor trung tâm chịu trách nhiệm phân tích ý định của người dùng và điều phối các Worker chuyên biệt.

**Pattern đã chọn:** Supervisor-Worker  
**Lý do chọn pattern này (thay vì single agent):**
- **Tính chuyên môn hóa:** Mỗi worker (Retrieval, Policy, Synthesis) tập trung vào một nhiệm vụ duy nhất, giúp tăng độ chính xác của prompt.
- **Khả năng kiểm soát:** Supervisor có thể ngăn chặn các truy vấn rủi ro cao thông qua bước Human-in-the-Loop (HITL) trước khi thực hiện hành động.
- **Dễ dàng mở rộng:** Có thể thêm các worker mới (ví dụ: IT Support Tool) mà không làm ảnh hưởng đến logic của các worker hiện có.
- **Khả năng quan sát:** Trace chi tiết cho thấy rõ ràng tại sao một quyết định điều hướng được đưa ra.

---

## 2. Sơ đồ Pipeline

**Sơ đồ thực tế của nhóm:**

```
      [User Query]
            │
            ▼
    [Policy Orchestrator] ◄──── [Human Review (HITL)]
    (Keywords + Risk)               (if risk_high)
            │
    ┌───────┴───────┬──────────────┐
    ▼               ▼              ▼
[Knowledge]     [Policy]       [Synthesis]
[Retriever]     [Auditor]      [Agent]
 (ChromaDB)    (MCP Tools)    (LLM Answer)
    │               │              │
    └───────┬───────┴──────────────┘
            ▼
      [Final Answer]
```

---

## 3. Vai trò từng thành phần

### Supervisor (`graph.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Phân tích query, gán nhãn rủi ro và điều phối worker. |
| **Input** | Câu hỏi của người dùng (task). |
| **Output** | supervisor_route, route_reason, risk_high, needs_tool |
| **Routing logic** | Keyword-based matching (SLA, Refund, Access, P1). |
| **HITL condition** | Trigger khi `risk_high` (khẩn cấp, P1) hoặc mã lỗi không rõ (`err-`). |

### Retrieval Worker (`workers/retrieval.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Tìm kiếm thông tin semantic từ vector database. |
| **Embedding model** | OpenAI `text-embedding-3-small`. |
| **Top-k** | 3. |
| **Stateless?** | Yes. |

### Policy Tool Worker (`workers/policy_tool.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **Nhiệm vụ** | Kiểm tra các quy tắc nghiệp vụ và ngoại lệ chính sách. |
| **MCP tools gọi** | `search_kb`, `get_ticket_info`, `check_access_permission`. |
| **Exception cases xử lý** | Flash Sale, Digital Product, Activated Product, Temporal Policy (v3 vs v4). |

### Synthesis Worker (`workers/synthesis.py`)

| Thuộc tính | Mô tả |
|-----------|-------|
| **LLM model** | OpenAI `gpt-4o-mini` hoặc Gemini `gemini-1.5-flash`. |
| **Temperature** | 0.1 (đảm bảo tính grounded). |
| **Grounding strategy** | Strict context-only, trích dẫn nguồn [file_name]. |
| **Abstain condition** | Trả lời "Không đủ thông tin" nếu context trống hoặc không liên quan. |

### MCP Server (`mcp_server.py`)

| Tool | Input | Output |
|------|-------|--------|
| search_kb | query, top_k | chunks, sources |
| get_ticket_info | ticket_id | ticket details (Jira mock) |
| check_access_permission | access_level, requester_role | can_grant, approvers |
| create_ticket | priority, title, description | ticket_id, status |

---

## 4. Shared State Schema

| Field | Type | Mô tả | Ai đọc/ghi |
|-------|------|-------|-----------|
| task | str | Câu hỏi đầu vào | Supervisor đọc |
| supervisor_route | str | Worker được chọn | Supervisor ghi |
| route_reason | str | Lý do route | Supervisor ghi |
| retrieved_chunks | list | Evidence từ retrieval | Retrieval ghi, Synthesis đọc |
| policy_result | dict | Kết quả kiểm tra policy | Policy_tool ghi, Synthesis đọc |
| mcp_tools_used | list | Tool calls đã thực hiện | Policy_tool ghi |
| final_answer | str | Câu trả lời cuối | Synthesis ghi |
| confidence | float | Mức tin cậy | Synthesis ghi |
| risk_high | bool | Đánh dấu rủi ro cao | Supervisor ghi, Graph đọc |

---

## 5. Lý do chọn Supervisor-Worker so với Single Agent (Day 08)

| Tiêu chí | Single Agent (Day 08) | Supervisor-Worker (Day 09) |
|----------|----------------------|--------------------------|
| Debug khi sai | Khó — không rõ lỗi ở đâu | Dễ hơn — test từng worker độc lập |
| Thêm capability mới | Phải sửa toàn prompt | Thêm worker/MCP tool riêng |
| Routing visibility | Không có | Có route_reason trong trace |
| Safety & HITL | Không thể can thiệp giữa chừng | Có thể dừng ở node Human Review |

**Nhóm điền thêm quan sát từ thực tế lab:**
Việc tách biệt Synthesis giúp câu trả lời đồng nhất về văn phong và độ tin cậy, bất kể dữ liệu đến từ Retrieval hay Policy Tool.

---

## 6. Giới hạn và điểm cần cải tiến

1. **Routing Heuristics:** Hiện tại dùng keyword đơn giản, dễ bị đánh lừa bởi từ đồng nghĩa. Cần nâng cấp lên Small LLM Classifier.
2. **Context Passing:** Khi route qua nhiều worker, kích thước state có thể tăng nhanh. Cần cơ chế dọn dẹp (trimming) context.
3. **HITL Manual:** HITL hiện tại mới là placeholder auto-approve, cần giao diện thực tế để Admin can thiệp.
