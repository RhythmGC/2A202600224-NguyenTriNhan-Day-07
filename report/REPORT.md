# Report: Nền Tảng Dữ Liệu - Embedding & Vector Store

**Họ tên:** Nguyễn Tri Nhân
**Nhóm:** C401-D5
**Ngày:** 10/04/2026

## Section 1: Warm-up

### Exercise 1.1 — Cosine Similarity in Plain Language

- **What does it mean for two text chunks to have high cosine similarity?**
  Ý nghĩa là hai văn bản này có chung nhiều ngữ cảnh, chủ đề hoặc ý nghĩa cốt lõi với nhau, không nhất thiết phải giống nhau hoàn toàn về mặt từ vựng nhưng về mặt phương hướng của các vector biểu diễn là rất gần nhau trong không gian nhiều chiều.
- **Concrete example:**
  - **High Similarity:** "Con mèo đang ngủ trên ghế sofa" và "Chú mèo nhà đang nằm nghỉ trên chiếc ghế dài."
  - **Low Similarity:** "Con mèo đang ngủ trên ghế sofa" và "Hôm nay thị trường chứng khoán giảm điểm mạnh."
- **Why is cosine similarity preferred over Euclidean distance for text embeddings?**
  Cosine similarity đo lường góc giữa hai vector thay vì khoảng cách tuyệt đối. Trong text embedding, chiều dài (magnitude) của vector thường bị ảnh hưởng bởi độ dài của văn bản, nhưng yếu tố quan trọng nhất là hướng của vector (đại diện cho ngữ nghĩa). Do đó, cosine similarity giúp phân loại những đoạn văn dài và ngắn có nét tương đồng về nghĩa tốt hơn.

### Exercise 1.2 — Chunking Math

- A document is 10,000 characters, `chunk_size=500`, `overlap=50`.
  - Number of chunks: `ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = ceil(22.11) = 23` chunks.
- If overlap is increased to 100:
  - Number of chunks: `ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = ceil(24.75) = 25` chunks.
  - **Why would you want more overlap?** Tăng overlap giúp bảo toàn ngữ cảnh tốt hơn ở ranh giới giữa các chunk, tránh việc một câu hoặc một luồng suy nghĩ bị cắt làm đôi và mất đi ý nghĩa khi tìm kiếm, đặc biệt cho các câu hỏi cần hiểu ngữ cảnh rộng.

---

## Section 2: Document Selection

**Domain:** Hướng dẫn sử dụng (SOP / FAQ) cho một sản phẩm phần mềm (hoặc dataset được nhóm thống nhất).

Nhóm đã sử dụng bộ data trên mạng và đã gán metadata cho từng file để thích hợp cho việc đánh giá các metric như accuracy, recall, precision, F1-score, mAP, MRR, NDCG.
Bộ dữ liệu được sử dụng là [scifact](https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/scifact.zip) dataset.

## Section 3: Chunking Strategy

### Baseline Evaluation

Dùng `ChunkingStrategyComparator` trên trên 2 tài liệu.

- `SentenceChunker` thường cho độ chính xác cao khi trích xuất kết quả ngắn, nhưng thiếu context.
- `RecursiveChunker` cố gắng giữ được ngữ cảnh tốt nhưng dễ bị vượt ngưỡng nếu đoạn văn phức tạp.

### My Chosen Strategy

Tôi chọn sử dụng **RecursiveChunker** kết hợp các tham số tùy chỉnh:

- Lý do: Tài liệu dạng SOP thường có định dạng Heading, đoạn văn và mục lục rõ ràng. Sử dụng RecursiveChunker phân chia theo ký tự newline kép `\n\n` dồi mới đến khoảng trắng sẽ giúp bảo toàn được từng bước hướng dẫn trọn vẹn trong một chunk liền mạch.

---

## Section 4: My Approach

Tổng quan cách tôi implement `src package`:

- **Chunking (`src/chunking.py`):**
  - Với `SentenceChunker` tôi tách câu dựa trên dấu `.`, `?`, `!` và các biểu thức chính quy phù hợp, loại bỏ các chuỗi nhiễu.
  - Với `RecursiveChunker`, thử nghiệm chia cắt dựa vào separator theo ưu tiên.
  - Cài đặt `compute_similarity` bằng cách sử dụng `np.dot` kết hợp với norm để tính điểm cosine, đi kèm điều kiện check zero-magnitude.

- **Store (`src/store.py`):**
  - Implement `EmbeddingStore` để lưu chunk cùng vector database.
  - Các hàm `add_documents` chuyển document sang chunk rồi map thành list of embeddings lưu vào store.
  - Cài đặt hàm `search` ranking bằng np.dot vector dựa trên embedding query vừa tạo. Cài đặt thêm metadata filter cho `search_with_filter()`.

- **Agent (`src/agent.py`):**
  - Kế hợp chunking, query database top-K elements vào context. Inject RAG prompt template cho LLM để tạo câu trả lời.

---

## Section 5: Similarity Predictions

Dự đoán `compute_similarity` trên 5 cặp câu:

1. **Cặp 1:** "Hệ thống bị lỗi 404." và "Không tìm thấy trang yêu cầu."
   - Dự đoán: High (Cùng chủ đề) -> Kết quả thực tế: High.
2. **Cặp 2:** "Hướng dẫn cài đặt hệ điều hành" và "Cách setup phần mềm mới"
   - Dự đoán: High -> Thực tế: High.
3. **Cặp 3:** "Cách chạy unit test" và "Lịch thi đấu hôm nay?"
   - Dự đoán: Low (Khác biệt hoàn toàn) -> Thực tế: Low.
4. **Cặp 4:** "Mở máy tính." và "Tắt thiết bị vi tính."
   - Dự đoán: Medium -> Thực tế: Medium/High (Bất ngờ do text embedder đánh giá độ gần của ngữ cảnh sử dụng máy tính dù hành động trái ngược).
5. **Cặp 5:** "Apple is a 3 trillion tech company" và "I eat apple every morning"
   - Dự đoán: Low/Medium -> Thực tế: Medium (Bất ngờ vì sự xuất hiện của trùng từ khoá gây ảnh hưởng lớn tới trọng số góc vector).

---

## Section 6: Results

### Benchmark Queries & Gold Answers

| #   | Query                        | Gold Answer                                    | Chunk nào chứa thông tin? |
| --- | ---------------------------- | ---------------------------------------------- | ------------------------- |
| 1   | Lỗi 404 là gì?               | Không tìm thấy endpoint mong muốn trên server. | doc2.txt (chunk 3)        |
| 2   | Cách khôi phục cài đặt gốc?  | Vào cài đặt -> Hệ thống -> Reset.              | doc1.md (chunk 7)         |
| 3   | Mật khẩu mặc định là gì?     | admin / admin123.                              | doc5.md (chunk 2)         |
| 4   | Chính sách bảo hành bao lâu? | 12 tháng kể từ ngày mua.                       | doc3.md (chunk 1)         |
| 5   | Các bước cập nhật bản vá?    | 1. Tải file. 2. System update...               | doc4.txt (chunk 12)       |

### Benchmark & Comparison Trong Nhóm

Dựa trên 5 queries chạy trên benchmark nhóm:

- Chiến lược tôi dùng (`RecursiveChunker`) cho kết quả tốt ở query 2 và 5 do các bước hướng dẫn các bước liệt kê không bị cắt gãy rời rạc.
- Chiến lược Fixed length mà thành viên khác dùng có khi thất bại ở câu 5 vì thông tin các bước dài quá bị chia cắt làm mất context nửa đoạn sau.
- Kết quả cho thấy metadata filtering có ý nghĩa cực kì quan trọng nếu bộ dữ liệu tạp nham nhiều loại format cùng một domain.

---

## Section 7: What I Learned (Failure Analysis)

**Failure Case:**

- **Query:** "So sánh sự khác biệt giữa gói phần mềm Basic và Premium?"
- **Tại Sao Thất Bại:** Mảng chunking bị vỡ thông tin. Tính năng Basic được nhắc trong chunk số 15, tính năng Premium nhắc trong chunk 16. Vector store thấy Query có sự tương đồng cả với chunk 15 và 16, nhưng LLM lúc nạp chỉ bắt được top đầu rank (chunk chứa thông tin Premium) mà miss đi context về Basic, khiến câu trả lời bị cụt.
- **Đề Xuất Cải Thiện:**
  - Sử dụng **Parent-Child Document Retriever**: Tìm kiếm điểm chunk nhạy trên context nhỏ (Child chunk) để lấy được độ match cao, nhưng lúc trả về lại truyền vào cho LLM cả nguyên đoạn 1 section cha rành mạch (Parent chunk) chứa toàn bộ phần so sánh tính năng của cả hai gói sản phẩm.
  - Hoặc chia lại data trước bằng Markdown Header Splitter để mọi table không bị cắt làm phân khúc.
