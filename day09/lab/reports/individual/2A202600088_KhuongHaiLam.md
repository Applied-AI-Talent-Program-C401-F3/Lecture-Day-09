# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Khương Hải Lâm 
**Vai trò trong nhóm:** Supervisor Owner (Lead)  
**Ngày nộp:** 14-04-2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Trong dự án này, tôi chịu trách nhiệm chính về việc thiết kế và triển khai khung xương (skeleton) của toàn bộ hệ thống đa tác tử. Tôi trực tiếp quản lý file `graph.py`, nơi định nghĩa `AgentState` và cấu trúc luồng công việc (workflow). 

**Module/file tôi chịu trách nhiệm:**
- File chính: `day09/lab/graph.py`
- Functions tôi implement: `make_initial_state`, `build_graph`, `run_graph`, và quản lý cấu trúc `AgentState`.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Công việc của tôi là tạo khung kết nối tới các module khác. Tôi định nghĩa `AgentState` để Lưu Lê Gia Bảo có thể ghi kết quả routing, Thái Doãn Minh Hải và Đặng Tuấn Anh có thể đọc context và ghi kết quả xử lý, và Hoàng Quốc Hùng có thể lưu trữ các tool calls. Nếu tôi không định nghĩa cấu trúc state chuẩn, các worker sẽ không thể truyền dữ liệu cho nhau.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
Tôi đã thực hiện các thay đổi chính trong `graph.py` để tích hợp `load_dotenv()` và cấu trúc lại các node wrapper.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Tôi quyết định triển khai mô hình "State-based Orchestrator" sử dụng Python thuần thay vì sử dụng thư viện LangGraph ngay từ đầu.

**Lý do:**
- Lab cho chọn một trong hai phương thức, hoặc Python thuần hoặc sử dụng LangGraph
- Việc sử dụng Python thuần giúp nhóm kiểm soát hoàn toàn việc truyền state và dễ dàng debug các lỗi liên quan đến kiểu dữ liệu (`TypedDict`). Việc này cũng giúp giảm latency do không phải gánh thêm overhead từ framework bên thứ ba.

**Trade-off đã chấp nhận:**
Đánh đổi lớn nhất là khả năng scale lên trong tương lai. Nếu sau này hệ thống cần các tính năng như resume graph hoặc làm thêm agent loop phức tạp, việc viết bằng Python thuần sẽ tốn công hơn nhiều so với dùng LangGraph. Tuy nhiên, cho mục tiêu prototype nhanh, đây là lựa chọn tối ưu.

**Bằng chứng từ trace/code:**
```python
def build_graph():
    def run(state: AgentState) -> AgentState:
        # Step 1: Supervisor decides route
        state = supervisor_node(state)
        # ... logic if/else đơn giản để chuyển node
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Trùng lặp danh sách `workers_called` khi chạy graph.

**Symptom (pipeline làm gì sai?):**
Trong trace JSON, danh sách `workers_called` xuất hiện các tên worker bị lặp lại (ví dụ: `['retrieval_worker', 'retrieval_worker', ...]`), dẫn đến việc tính toán metrics ở Sprint 4 bị sai lệch.

**Root cause:**
Cả hàm wrapper trong `graph.py` (ví dụ: `retrieval_worker_node`) và hàm thực thi chính trong worker file (ví dụ: `workers/retrieval.py`) đều thực hiện lệnh `state["workers_called"].append()`.

**Cách sửa:**
Tôi đã refactor lại các node wrapper trong `graph.py`, xóa bỏ các dòng `append` thủ công và để cho các worker tự chịu trách nhiệm khai báo sự hiện diện của mình khi hàm `run()` của chúng được gọi.

**Bằng chứng trước/sau:**
- Trước: `Workers : ['retrieval_worker', 'retrieval_worker', 'synthesis_worker', 'synthesis_worker']`
- Sau: `Workers : ['retrieval_worker', 'synthesis_worker']`

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi đã thiết kế được một cấu trúc `AgentState` bao quát, giúp việc debug thông qua trace JSON trở nên rất dễ dàng vì mọi thông tin từ routing reason đến latency đều được lưu trữ tập trung.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Hiện tại chưa sử dụng LangGraph cho project này. Trong các hệ thống Production phức tạp, cần làm LangGraph

**Nhóm phụ thuộc vào tôi ở đâu?**
Mọi thành viên đều phụ thuộc vào file `graph.py` vì nó là điểm vào của chatbot. Nếu tôi thay đổi một key trong `AgentState`, code của tất cả các worker khác đều sẽ bị lỗi.

**Phần tôi phụ thuộc vào thành viên khác:**
Vì phần này chỉ là phần thiết kế và entry từ việc viết code của các worker khác, các thành viên khác phải cung cấp cho tôi về `eval_trace.py`, `mcp_server.py`,  và toàn bộ các subagent như policy, retrieval, synthesis.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ thử triển khai cơ chế "Retry logic" cho các worker. Theo quan sát từ trace của câu q12, thỉnh thoảng LLM call bị timeout hoặc lỗi API, nếu có cơ chế tự động gọi lại (retry) trong graph, hệ thống sẽ ổn định hơn nhiều.

Hoặc, áp dụng thêm agent loop  để hỏi thêm clarification từ người dùng nếu câu hỏi không rõ ràng.

---
