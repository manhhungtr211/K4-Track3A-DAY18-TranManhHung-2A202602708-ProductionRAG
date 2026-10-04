# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Trần Mạnh Hùng  
**Mã học viên:** 2A202602708  
**Khóa:** K4 - Track 3A  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8359 | 0.9643 | +0.1283 |
| Answer Relevancy | 0.7026 | 0.7837 | +0.0810 |
| Context Precision | 0.9103 | 0.8750 | -0.0353 |
| Context Recall | 0.8929 | 0.7917 | -0.1012 |

---

## Bottom-5 Failures

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9 ÷ 3 = 3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Trích xuất đoạn chính sách thâm niên nhưng tính toán nhầm theo quy chế cũ 2023 (5 năm cộng 1 ngày) hoặc chưa tổng hợp đầy đủ dải lương từ bảng phân bậc lương.
- **Worst metric:** `context_recall` & `faithfulness`
- **Error Tree:** Output chưa trọn vẹn → Context thiếu bảng lương kết hợp → Query là câu hỏi ghép (phép năm + dải lương) → Cần xử lý ở tầng Query Decomposition / Routing.
- **Root cause:** Câu hỏi đa khía cạnh (Multi-part question) đòi hỏi thông tin từ 2 tài liệu độc lập: Quy chế nghỉ phép 2024 và Quy chế ngạch bậc lương. Bộ tìm kiếm đơn luồng có thể chỉ ưu tiên tìm đoạn có số ngày thâm niên mà bỏ sót bảng ngạch lương Senior.
- **Suggested fix:** Áp dụng kỹ thuật Sub-query Decomposition (tách thành 2 truy vấn con: "Số ngày phép nhân viên 9 năm thâm niên 2024" và "Mức lương Senior") trước khi đưa qua Hybrid Search.

---

### #2
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** Trích xuất cả đoạn văn bản năm 2023 (12 ngày) và 2024 (15 ngày), có thể gây nhầm lẫn nếu không có metadata lọc theo phiên bản mới nhất.
- **Worst metric:** `context_precision`
- **Error Tree:** Output có nguy cơ xung đột → Context chứa cả 2 phiên bản văn bản cũ và mới → Search bắt từ khóa "phép năm" như nhau → Cần xử lý ở bước Metadata Filtering & Reranking.
- **Root cause:** Xung đột phiên bản (Version Conflict) giữa `Quy_che_nghi_phep_2023` và `Quy_che_nghi_phep_2024`. BM25 và Vector search thông thường đánh giá độ tương đồng từ khóa cao ở cả 2 tài liệu.
- **Suggested fix:** Tận dụng Metadata Enrichment (M5) gắn nhãn `effective_date: 2024` hoặc `status: active` để thực hiện Hard Filter loại bỏ tài liệu hết hiệu lực trước khi search.

---

### #3
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Mô hình trích xuất đúng điều khoản phạt 2%/tháng nhưng tính toán số học trực tiếp bằng LLM dễ có sai số nhỏ trong phép tính pro-rata.
- **Worst metric:** `faithfulness` (nếu LLM làm tròn số học khác ground truth)
- **Error Tree:** Context đúng điều khoản → LLM reasoning tính toán số ngày trễ (20 - 15 = 5 ngày) và áp công thức → Fix ở tầng Agentic Tool Calling (Code Interpreter / Calculator).
- **Root cause:** LLM thuần túy không mạnh về tính toán số học chính xác (arithmetic reasoning).
- **Suggested fix:** Tích hợp Tool-calling Python REPL / Calculator cho các câu hỏi liên quan đến tính toán lãi phạt, khấu trừ tài chính.

---

### #4
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Theo quy chế chi trả lương, thông tin lương được phân loại là dữ liệu Bí mật, cấm chia sẻ. Theo chính sách an toàn thông tin, dữ liệu Bí mật (cấp 3) phải mã hóa khi truyền và hạn chế truy cập.
- **Got:** Tìm được quy chế lương hoặc chính sách phân loại dữ liệu, nhưng có thể thiếu liên kết chéo giữa hai văn bản.
- **Worst metric:** `context_recall`
- **Error Tree:** Output đúng định tính nhưng thiếu ngữ cảnh chi tiết → Context chỉ lấy được 1 trong 2 file → Query cần multi-hop reasoning → Fix ở Cross-document HyQA & Retrieval.
- **Root cause:** Câu hỏi yêu cầu tổng hợp tri thức liên tài liệu (Cross-document synthesis: Quy chế lương $\leftrightarrow$ Chính sách phân loại dữ liệu IT).
- **Suggested fix:** Kỹ thuật HyQA (M5) sinh trước các câu hỏi bắc cầu như "Dữ liệu lương được bảo mật ở cấp độ nào trong chính sách ATTT?" để vector search kéo cả 2 chunks về top k.

---

### #5
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Cần xác nhận cấu hình kỹ thuật từ CNTT và đính kèm ít nhất 3 báo giá.
- **Got:** Trích xuất được hạn mức phê duyệt của Giám đốc phòng ban nhưng có thể thiếu điều kiện phụ về 3 báo giá hoặc xác nhận kỹ thuật từ IT.
- **Worst metric:** `context_recall`
- **Error Tree:** Output trả lời được thẩm quyền phê duyệt nhưng thiếu điều kiện đính kèm → Context bị cắt vụn do chunk size nhỏ → Fix ở Hierarchical Chunking (trả về Parent Chunk).
- **Root cause:** Điều khoản mua sắm chứa nhiều quy định con (ngưỡng tiền, thẩm quyền duyệt, quy định 3 báo giá cho đơn hàng > 10 triệu) trải dài qua nhiều đoạn.
- **Suggested fix:** Áp dụng triệt để Hierarchical Chunking (M1): khi child chunk về "Hạn mức 30 triệu" match, hệ thống trả về toàn bộ Parent Chunk chứa toàn bộ quy trình mua sắm để LLM có đầy đủ thông tin.

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
`"Nhân viên được nghỉ bao nhiêu ngày phép năm?"` (Case Xung đột phiên bản 2023 vs 2024)

**Error Tree walkthrough:**
1. **Output đúng?** $\rightarrow$ Chưa chắc chắn nếu hệ thống trích nhầm tài liệu cũ năm 2023 (12 ngày) thay vì 2024 (15 ngày).
2. **Context đúng?** $\rightarrow$ Nếu chỉ dùng Bi-Encoder (Dense thuần), cả 2 chunk đều đạt độ tương đồng rất cao (similarity > 0.85). Khi đó tài liệu 2023 có thể vô tình đứng trên 2024 do mật độ từ khóa.
3. **Query rewrite OK?** $\rightarrow$ Người dùng chỉ hỏi "bao nhiêu ngày phép năm", không chỉ định rõ năm.
4. **Fix ở bước:**
   - **M5 (Enrichment):** Gắn Contextual Prepend *"Trích từ Quy chế nghỉ phép năm 2024 (Phiên bản mới nhất có hiệu lực từ 01/01/2024)..."*.
   - **M3 (Cross-Encoder Reranker):** So khớp sâu cặp (Query, Doc) nhận diện tín hiệu văn bản mới nhất.
   - **Metadata Filtering:** Lọc cứng `is_active == True` trước khi đẩy vào Retrieval.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Triển khai **Metadata Pre-filtering** cho Qdrant để tự động ưu tiên tài liệu có `version` mới nhất hoặc lọc theo ngày hiệu lực.
- Bổ sung **Query Expansion / HyDE (Hypothetical Document Embeddings)** giúp các câu hỏi ngắn tự động mở rộng ngữ cảnh trước khi truy vấn BM25 và Dense Search.
