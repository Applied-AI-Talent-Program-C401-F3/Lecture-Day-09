# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Lương Trung Kiên  
**Vai trò trong nhóm:** Trace & Docs Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---



---

## 1. Tôi phụ trách phần nào? (100–150 từ)

**Module/file tôi chịu trách nhiệm:**
- File chính: `eval_trace.py`
- Functions tôi implement: Toàn bộ pipeline đánh giá, bao gồm `run_test_questions()`, `run_grading_questions()`, `analyze_traces()` và `compare_single_vs_multi()`.

**Cách công việc của tôi kết nối với phần của thành viên khác:**
Phần việc của tôi đóng vai trò đánh giá và nghiệm thu toàn bộ hệ thống Multi-Agent mà nhóm đã xây dựng. Tôi cần lấy biểu đồ LangGraph hoàn thiện từ các thành viên khác (bao gồm supervisor và các workers) sau đó import thông qua hàm `run_graph`. Tôi kích hoạt hệ thống chạy qua các tập test và grading benchmark nhằm sinh ra files kết quả cuối cùng (`artifacts/grading_run.jsonl`) cũng như báo cáo tổng kết (`artifacts/eval_report.json`). Nếu không có mô-đun của tôi, team sẽ không thu thập được các metric quan trọng (kiểm định latency, confidence, supervisor tracing path) và không xuất được format bắt buộc để nộp bài.

**Bằng chứng (commit hash, file có comment tên bạn, v.v.):**
- File `eval_trace.py`: Chứa đầy đủ các vòng lặp execute graph và xuất file dữ liệu.
- Outputs sinh ra trong thư mục `artifacts/` (điển hình như `artifacts/grading_run.jsonl`) đều được tạo thành từ scripts của tôi.

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Định dạng lưu trữ trace tách biệt giữa quá trình Test và Grading. Tôi đã quyết định tổ chức việc lưu trace làm 2 chế độ: lưu các file JSON độc lập cho bước Test (sử dụng hàm helper `save_trace()` ghi vào `artifacts/traces/`) và lưu tập trung dưới định dạng JSONL theo dạng Append Mode cho bước Grading (ghi chung vào file `artifacts/grading_run.jsonl`).

**Lý do:** Đối với dữ liệu test during develop, việc lưu thành từng file JSON rời rạc với format indentation thụt lề chuẩn giúp biểu diễn rõ cấu trúc cây state phức tạp và logic path; các thành viên dễ dàng mở lên phân tích từng luồng chạy độc lập rất tiện dụng. Trái lại, khi chuyển sang Grading mode, hệ thống tự động chấm điểm cần có thể parse tuần tự và đọc tập trung kết quả trên từng dòng nhanh chóng. Bằng cách dùng JSONL tôi có thể gom toàn bộ list kết quả của 15+ câu lại vào một format duy nhất, giúp bảo đảm các record có thể tự động được tải (load stream) dễ dàng để tính điểm.

**Trade-off đã chấp nhận:** Đánh đổi lớn nhất là tính bảo trì mã nguồn (maintainability). Tôi phải duy trì hai mạch I/O ghi file trong cùng một script, khiến file bị chia đôi làm hai function hơi tương đồng mà khác đầu ra, có thể gây mất thời gian nếu phải thay đổi schema chung sau này.

**Bằng chứng từ trace/code:**
```python
# Từ file eval_trace.py - Ghi stream theo chuẩn JSONL trong run_grading_questions()
with open(output_file, "w", encoding="utf-8") as out:
    for i, q in enumerate(questions, 1):
        # ... generate record mapping ...
        out.write(json.dumps(record, ensure_ascii=False) + "\n")

# So với run_test_questions() - Ghi đơn lẻ config chuyên dùng debug JSON
trace_file = save_trace(result, f"artifacts/traces")
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** Trục trặc Error Serialization (báo lỗi xử lý chuỗi JSON) làm gián đoạn toàn bộ phiên ghi Grading Log.

**Symptom (pipeline làm gì sai?):** Khi chạy lệnh `python eval_trace.py --grading`, tiến trình gặp lỗi crash nghiêm trọng với thông báo exception: `TypeError: Object of type <ToolCall/Message> is not JSON serializable` và pipeline lập tức bị dừng. Điều này dẫn tới log không thể ghi lại được toàn vẹn dữ liệu cho các câu hỏi còn sót lại.

**Root cause (lỗi nằm ở đâu — indexing, routing, contract, worker logic?):** Bug xảy ra ở khâu trích xuất siêu dữ liệu tool call. Hàm `run_graph` trả về biến Dictionary chứa `result.get("mcp_tools_used")` dưới dạng mảng của các objects class nội tại do LangChain/MCP định nghĩa. Khi hàm helper `json.dumps()` cố chuyển hóa object dict này thành dạng text để in ra `grading_run.jsonl`, nó gặp kiểu class không hỗ trợ chuẩn JSON dẫn tới crash. 

**Cách sửa:** Tôi đã xử lý lỗi này tại chỗ (in-line override) bằng thao tác lọc qua list comprehension để trích xuất chuẩn phần lõi thông tin là string của tên function (`"tool"`). Nó giúp loại bỏ hoàn toàn các metadata rác không thể ép kiểu, đưa kết quả về list simple strings rất gọn.

**Bằng chứng trước/sau:**
> **Giải pháp đã cứu toàn bộ logic crash vào lúc ghi file — (Dòng 128 trong eval_trace.py):**
```python
"mcp_tools_used": [t.get("tool") for t in result.get("mcp_tools_used", [])],
```

---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**
Tôi đã hoàn thiện kịp thời một Evaluation Pipeline đáng tin cậy. Tôi đặc biệt hài lòng với việc bọc block `try-except` cẩn thận trong process vòng lặp chạy graph. Nhờ vậy, ngay cả khi worker trong LangGraph bị quăng một exception không thể ngờ, pipeline của tôi vẫn catch catch được, in "PIPELINE_ERROR", và cho phép thu thập trọn vẹn record đánh giá của các câu hỏi khác thay vì sụp đổ toàn mạng.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**
Do thời gian eo hẹp, mục benchmark so sánh Single vs Multi-agent (`compare_single_vs_multi`) tôi mới lên được template sơ bộ và điền placeholder cứng cho Baseline Day 08. Lẽ ra tôi nên tự động hóa móc nối tải log của Day 08 để xuất report xịn hơn.

**Nhóm phụ thuộc vào tôi ở đâu?** _(Phần nào của hệ thống bị block nếu tôi chưa xong?)_
Nếu tôi chưa hoàn thành phần `eval_trace.py` của mình, nhóm sẽ không có script và không thể generate ra được file evidence `grading_run.jsonl` và `eval_report.json` - các artifacts bắt buộc để qua môn và chấm điểm cuối lab.

**Phần tôi phụ thuộc vào thành viên khác:** _(Tôi cần gì từ ai để tiếp tục được?)_
Tôi cần các thành viên hoàn thành logic LangGraph (`graph.py`, `agent.py`) và trả về chuẩn metadata key contracts (`latency_ms`, `supervisor_route`, `mcp_tools_used`) thì eval log mới parse được dữ liệu chuẩn.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ thử xây dựng thêm module đánh giá chất lượng độ chính xác tự động (LLM-as-a-Judge) tương tự cơ chế Day 08. Từ trace hiện tại trong thư viện logs, ta mới chỉ ghi được "confidence score" sinh ra chủ quan từ LLM trả lời. Cần một LLM Judge khách quan so sánh chéo Answer tự sinh với Expected Ground Truth thì ta mới xác minh được năng lực Multi-Agent có tăng cường Accuracy tốt hơn Baseline như thế nào.

---
