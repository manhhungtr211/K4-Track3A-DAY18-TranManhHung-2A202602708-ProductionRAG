# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Trần Mạnh Hùng  
**Mã học viên:** 2A202602708  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)
Map từng concept trong lecture vào code bạn vừa viết trong lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| **Semantic Chunking** | M1 | `chunk_semantic()` | Tách văn bản theo ranh giới câu, dùng embedding model `all-MiniLM-L6-v2` để mã hóa từng câu và gom nhóm dựa trên cosine similarity (threshold 0.85). Giúp bảo toàn ngữ nghĩa của các luận điểm liền mạch, tránh cắt đứt ngữ cảnh giữa chừng như fixed-size chunking. |
| **Hierarchical Chunking** | M1 | `chunk_hierarchical()` | Tạo cấu trúc cha-con: Parent chunk (tối đa 2048 ký tự) lưu trữ toàn bộ bức tranh ngữ cảnh rộng, Child chunk (tối đa 256 ký tự, overlap 32 ký tự) dùng để tìm kiếm vector chính xác. Khi match child chunk, ta trả về parent chunk cho LLM để loại bỏ hiện tượng mất ngữ cảnh. |
| **Structure-Aware Chunking** | M1 | `chunk_structure_aware()` | Phân tích cấu trúc phân cấp đặc thù của văn bản quy phạm pháp luật / chính sách (Chương -> Điều -> Khoản -> Điểm) qua Regex và gán metadata breadcrumb. Cực kỳ hiệu quả cho việc lọc chính xác phiên bản tài liệu và điều khoản. |
| **BM25 + Vietnamese Segmentation** | M2 | `segment_vietnamese()` & `BM25Search` | Tiếng Việt có từ ghép đa âm tiết; thư viện `underthesea` nối từ bằng dấu gạch dưới (`nghỉ_phép`), do đó cần chuẩn hóa cả corpus và query về dạng token thống nhất để BM25 tính tần suất TF-IDF chính xác tuyệt đối với mã số văn bản và số liệu. |
| **Dense Search with Qdrant** | M2 | `DenseSearch` | Sử dụng mô hình đa ngôn ngữ mạnh mẽ `BAAI/bge-m3` (1024-dim) và vector database Qdrant (in-memory hoặc persistent). Tìm kiếm ngữ nghĩa vượt trội với các câu hỏi diễn đạt gián tiếp hoặc dùng từ đồng nghĩa. |
| **Reciprocal Rank Fusion (RRF)** | M2 | `reciprocal_rank_fusion()` | Áp dụng công thức RRF $RRF\_Score(d) = \sum \frac{1}{k + rank(d)}$ với $k=60$. Hợp nhất điểm thứ hạng không phụ thuộc vào phân phối điểm số (score normalization) của BM25 và Dense Search, giúp cân bằng hoàn hảo giữa Lexical precision và Semantic recall. |
| **Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Kiến trúc Cross-Encoder (`BAAI/bge-reranker-v2-m3`) nhận đồng thời cặp `(query, document)` và tính toán full self-attention giữa từng từ trong câu hỏi với từng từ trong văn bản. Giúp lọc 20 candidate xuống top 3-5 đoạn trích đắt giá nhất, loại bỏ tài liệu cũ hết hạn (như quy chế 2023 vs 2024). |
| **RAGAS 4 Core Metrics** | M4 | `evaluate_ragas()` | Đánh giá toàn diện 4 khía cạnh: **Faithfulness** (chống hallucination), **Answer Relevancy** (trả lời đúng trọng tâm), **Context Precision** (đoạn liên quan nằm ở top đầu), và **Context Recall** (trích xuất đầy đủ thông tin cần thiết từ ground truth). |
| **Failure Diagnostic Tree** | M4 | `failure_analysis()` | Xây dựng cây quyết định chẩn đoán lỗi: nếu Output sai, kiểm tra Context; nếu Context sai, kiểm tra Search/Rerank; nếu Context đúng mà Output sai, cải thiện Prompt/LLM reasoning. |
| **Contextual Prepend (M5)** | M5 | `contextual_prepend()` / `_enrich_single_call()` | Kỹ thuật của Anthropic: Bổ sung 1 câu tóm tắt vị trí tài liệu và chủ đề vào đầu mỗi chunk trước khi index, giúp vector embedding mang đầy đủ ngữ cảnh của toàn bộ tài liệu gốc. |
| **Hypothetical Questions (HyQA)** | M5 | `generate_hypothesis_questions()` | Sinh trước 2-3 câu hỏi người dùng có thể hỏi từ nội dung đoạn văn, giúp vector search khớp dễ dàng giữa câu truy vấn ngắn của người dùng và văn bản giải đáp dài. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật gặp phải (Exact error message):**
  1. `QdrantClient.search() got an unexpected keyword argument` hoặc `AttributeError: 'QdrantClient' object has no attribute 'search'` trên các phiên bản qdrant-client mới.
  2. `RateLimitError: 429 - qwen/qwen3.8-27b:free is temporarily rate-limited upstream (in_flight_budget_exhausted)`.
  3. `UnicodeEncodeError / cp949 codec can't encode character` khi in emoji hoặc tiếng Việt trên Windows terminal console.
  4. Lệch format tách từ tiếng Việt: BM25 không tìm thấy kết quả do câu hỏi người dùng gõ từ không có gạch dưới trong khi corpus có gạch dưới (`nghi_phep` vs `nghỉ phép`).

- **Nguyên nhân gốc rễ & Cách debug:**
  1. **Qdrant API v1.x:** `qdrant-client` bản mới khuyến nghị dùng `query_points(collection_name=..., query=..., limit=...)`. Đã cập nhật `DenseSearch.search()` để tương thích cả `query_points` và fallback `search`.
  2. **API Rate Limits / Concurrency:** OpenRouter free tier có giới hạn concurrency. Đã triển khai cơ chế try-catch với extractive fallbacks (regex + heuristic NLP) trong `_enrich_single_call` và `m5_enrichment.py`, đồng thời gom 4 tác vụ enrichment (Summary, HyQA, Contextual, Metadata) vào **1 single LLM call duy nhất** để tiết kiệm 75% API quota và thời gian thực thi.
  3. **Console Encoding trên Windows:** Thêm cấu hình `sys.stdout.reconfigure(encoding="utf-8")` ở đầu các script entrypoint để hiển thị tiếng Việt và emoji mượt mà.
  4. **Chuẩn hóa Tokenizer tiếng Việt:** Áp dụng hàm `segment_vietnamese()` xử lý `underthesea.word_tokenize(format="text")` kết hợp thay thế dấu `_` thành khoảng trắng đồng nhất cho cả tài liệu và query tìm kiếm.

- **Kiến thức còn thiếu & Cách khắc phục:**
  - Nắm vững sự khác biệt giữa **Bi-Encoder** (nhúng độc lập, cosine distance nhanh) và **Cross-Encoder** (full cross-attention chậm hơn nhưng nhận biết chính xác quan hệ phụ thuộc ngữ nghĩa, đặc biệt là quan hệ điều kiện và phủ định).
  - Hiểu sâu bản chất toán học của các chỉ số trong **RAGAS** để không chỉ nhìn điểm trung bình mà có thể truy vết từng câu hỏi theo Decision Tree để tối ưu đúng khâu.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Trợ lý Pháp lý & Thủ tục Hành chính Doanh nghiệp (Enterprise Policy & Legal Assistant)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Naive RAG cơ bản với RecursiveCharacterTextSplitter (chunk size 500, overlap 50), lưu vào ChromaDB với OpenAI Embeddings và gán trực tiếp top 4 chunks vào prompt cho LLM.
- **Vấn đề / Bottlenecks đang gặp:**
  - **Xung đột phiên bản tài liệu:** Khi có văn bản sửa đổi bổ sung năm 2024 thay thế quy định năm 2022, Naive RAG thường xuyên trích xuất nhầm văn bản cũ do có nhiều từ khóa trùng lặp cao hơn.
  - **Mất ngữ cảnh bảng biểu và điều khoản con:** Các điều khoản có tính kế thừa (Ví dụ: "Khoản 2 Điều 15: Trường hợp quy định tại Điểm a Khoản 1...") bị tách rời làm LLM trả lời sai hoàn toàn.
  - **Hallucination và trích dẫn sai số liệu:** Độ trung thực (Faithfulness) chỉ đạt khoảng 60-65%.

#### 2. Kế hoạch cải tiến
1. **Chunking strategy:**
   - Áp dụng **Structure-Aware Chunking** cho các văn bản quy định, luật, quy chế nội bộ theo Chương/Điều/Khoản kết hợp metadata breadcrumb (`doc_name > chapter > article`).
   - Áp dụng **Hierarchical Chunking** (Parent 2048 chars, Child 256 chars) cho các tài liệu hướng dẫn quy trình dài.
2. **Search retrieval:**
   - Triển khai **Hybrid Search** kết hợp BM25 (xử lý chính xác số hiệu văn bản, mã thủ tục, từ viết tắt) và Dense Search dùng `BAAI/bge-m3` (hiểu ngữ nghĩa đa ngôn ngữ Việt-Anh).
   - Áp dụng **Reciprocal Rank Fusion (RRF)** với $k=60$ để merge kết quả top 30 candidates.
3. **Reranking:**
   - Tích hợp **Cross-Encoder Reranker** (`BAAI/bge-reranker-v2-m3` hoặc FlashRank) để chọn lọc 3-5 contexts chuẩn xác nhất cho LLM.
4. **Evaluation:**
   - Xây dựng benchmark test set 50 câu hỏi đặc thù doanh nghiệp (bao gồm 20% câu bẫy xung đột phiên bản văn bản).
   - Sử dụng **RAGAS 4 core metrics** tích hợp vào CI/CD pipeline để chặn hồi quy chất lượng khi cập nhật cơ sở tri thức.
5. **Enrichment:**
   - Áp dụng **Contextual Prepend** để gắn tên văn bản + năm ban hành + trạng thái hiệu lực vào đầu mỗi chunk.
   - Tự động trích xuất metadata ngày có hiệu lực / hết hiệu lực để lọc cứng (metadata filtering) trước khi search.

#### 3. Timeline triển khai
- **Tuần 1:**
  - Chuẩn hóa dữ liệu văn bản chính sách; triển khai Structure-Aware & Hierarchical Chunking.
  - Thiết lập Hybrid Search (BM25 + Qdrant BGE-M3) và đo lường độ trễ (latency).
- **Tuần 2:**
  - Tích hợp Cross-Encoder Reranker và bộ làm giàu Contextual Prepend.
  - Xây dựng bộ test set 50 câu hỏi, chạy tự động đánh giá RAGAS và tối ưu hóa Prompt Template chống hallucination.
