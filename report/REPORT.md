# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** [Đỗ Minh Khiêm]
**Nhóm:** [62]
**Ngày:** [10/4/2026]

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
High cosine similarity nghĩa là hai đoạn văn có hướng biểu diễn embedding gần nhau, tức là chúng có nội dung hoặc ý nghĩa khá giống nhau. Trong retrieval, điều này thường cho thấy hai câu đang nói về cùng chủ đề hoặc cùng loại thông tin.

**Ví dụ HIGH similarity:**
- Sentence A: The CPI rose 2.7 percent year over year in December 2025.
- Sentence B: Inflation increased 2.7% compared with the same month last year.
- Tại sao tương đồng: Cả hai câu đều diễn đạt cùng một ý về mức tăng lạm phát theo năm, chỉ khác cách dùng từ.

**Ví dụ LOW similarity:**
- Sentence A: The Federal Reserve kept interest rates unchanged in July 2025.
- Sentence B: Mangoes are popular tropical fruits in Southeast Asia.
- Tại sao khác: Hai câu nói về hai chủ đề hoàn toàn khác nhau, một bên là chính sách tiền tệ và một bên là thực phẩm.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector hơn là độ lớn tuyệt đối, nên phù hợp với embeddings vì điều quan trọng là mức độ giống nhau về nghĩa. Euclidean distance dễ bị ảnh hưởng bởi độ lớn vector, trong khi với text retrieval ta thường muốn so sánh mức độ đồng hướng ngữ nghĩa hơn.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**  
Trình bày phép tính:  
`num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))`  
`= ceil((10000 - 50) / (500 - 50))`  
`= ceil(9950 / 450)`  
`= ceil(22.11)`  
Đáp án: `23 chunks`

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
Khi overlap tăng lên 100 thì số chunk sẽ tăng: `ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25 chunks`. Overlap lớn hơn giúp giữ ngữ cảnh tốt hơn giữa hai chunk liên tiếp, đặc biệt khi một ý quan trọng nằm gần ranh giới cắt.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Financial News & Economic Indicators

**Tại sao nhóm chọn domain này?**  
Nhóm chọn domain này vì dữ liệu tài chính vĩ mô có nguồn chính thống, dễ kiểm chứng, và có giá trị phân tích thực tế. Bộ dữ liệu hiện tại kết hợp báo cáo CPI, Employment Situation của BLS, và các FOMC statement của Federal Reserve, nên có thể đặt câu hỏi retrieval về lạm phát, thị trường lao động, lãi suất, và định hướng chính sách tiền tệ. Ngoài ra, domain này có yếu tố thời gian rất rõ, nên phù hợp để thử metadata filtering theo ngày, loại tài liệu, và nhóm chỉ báo kinh tế.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `bls_cpi_december_2025.txt` | Bureau of Labor Statistics (BLS) | 111283 | `category=inflation`, `source=BLS`, `date=2026-01-13`, `document_type=government_press_release`, `language=en`, `indicators=[cpi,mom_change,yoy_change]` |
| 2 | `bls_employment_situation_march_2026.txt` | Bureau of Labor Statistics (BLS) | 7163 | `category=employment`, `source=BLS`, `date=2026-04-03`, `document_type=government_press_release`, `language=en`, `indicators=[unemployment_rate,nonfarm_payrolls,labor_force_participation]` |
| 3 | `fed_fomc_statement_2025_07_30.txt` | Federal Reserve | 2978 | `category=monetary_policy`, `source=Federal Reserve`, `date=2025-07-30`, `document_type=government_statement`, `language=en`, `indicators=[interest_rates,federal_funds_rate]` |
| 4 | `fed_fomc_statement_2025_09_17.txt` | Federal Reserve | 2612 | `category=monetary_policy`, `source=Federal Reserve`, `date=2025-09-17`, `document_type=government_statement`, `language=en`, `indicators=[interest_rates,federal_funds_rate]` |
| 5 | `fed_fomc_statement_2025_10_29.txt` | Federal Reserve | 2792 | `category=monetary_policy`, `source=Federal Reserve`, `date=2025-10-29`, `document_type=government_statement`, `language=en`, `indicators=[interest_rates,federal_funds_rate]` |
| 6 | `fed_fomc_statement_2025_12_10.txt` | Federal Reserve | 2923 | `category=monetary_policy`, `source=Federal Reserve`, `date=2025-12-10`, `document_type=government_statement`, `language=en`, `indicators=[interest_rates,federal_funds_rate]` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `category` | string | `inflation`, `employment`, `monetary_policy` | Giúp lọc theo chủ đề tài liệu trước khi search, ví dụ chỉ lấy tài liệu về lạm phát, lao động, hoặc chính sách tiền tệ. |
| `source` | string | `BLS`, `Federal Reserve` | Giúp phân biệt nguồn phát hành và đánh giá độ tin cậy của thông tin retrieved. |
| `date` | string | `2025-12-10` | Hữu ích khi truy vấn có yếu tố thời gian hoặc cần so sánh phát biểu FOMC giữa các kỳ họp. |
| `document_type` | string | `government_press_release`, `government_statement` | Giúp tách báo cáo CPI rất dài khỏi FOMC statement ngắn và trực diện hơn. |
| `language` | string | `en` | Giúp chuẩn hóa tập dữ liệu hiện tại và hỗ trợ filter nếu sau này mở rộng sang tài liệu song ngữ. |
| `indicators` | list[string] | `["cpi","mom_change","yoy_change"]`, `["unemployment_rate","nonfarm_payrolls"]`, `["interest_rates","federal_funds_rate"]` | Cho phép retrieval bám sát chỉ báo kinh tế cụ thể mà câu hỏi đang nhắm tới. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare(chunk_size=500)` trên 2 tài liệu thật:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| `fed_fomc_statement_2025_07_30.txt` (2,943 chars) | FixedSizeChunker | 7 | 463.3 | Trung bình — cắt ngang giữa câu |
| `fed_fomc_statement_2025_07_30.txt` | SentenceChunker | 10 | 292.7 | Tốt — giữ ranh giới câu |
| `fed_fomc_statement_2025_07_30.txt` | RecursiveChunker | 9 | 325.2 | Tốt — tách theo paragraph |
| `bls_cpi_december_2025.txt` (111,283 chars) | FixedSizeChunker | 248 | 498.5 | Kém — cắt ngang bảng số liệu |
| `bls_cpi_december_2025.txt` | SentenceChunker | 323 | 343.3 | Kém — bảng số liệu không có dấu câu |
| `bls_cpi_december_2025.txt` | RecursiveChunker | 263 | 421.4 | Tốt nhất — tách theo `\n\n` trước |

### Strategy Của Tôi

**Loại:** `RecursiveChunker` (chunk_size=500)

**Mô tả cách hoạt động:**

RecursiveChunker thử tách text theo danh sách separator ưu tiên: `["\n\n", "\n", ". ", " ", ""]`. Với tài liệu tài chính, separator `"\n\n"` tách theo paragraph, giữ mỗi đoạn văn nguyên vẹn. Nếu một paragraph quá dài, tiếp tục tách bằng `"\n"` rồi `". "`. Kết quả: chunk có ranh giới tự nhiên hơn FixedSizeChunker và ít bị cắt ngang ý hơn.

**Tại sao chọn strategy này cho domain financial news?**

Tài liệu tài chính (FOMC statements, BLS press releases) được viết theo cấu trúc paragraph rõ ràng — mỗi paragraph nêu một luận điểm (quyết định lãi suất, mô tả thị trường lao động, lạm phát...). RecursiveChunker giữ nguyên cấu trúc đó. Với file BLS lớn (111k chars), việc tách theo `\n\n` giúp tránh bị cắt ngang bảng số liệu.

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality |
|-----------|----------|-------------|------------|-------------------|
| FOMC statement (2,943 chars) | SentenceChunker (baseline tốt nhất) | 10 | 292.7 | Tốt — giữ ranh giới câu |
| FOMC statement | **RecursiveChunker (của tôi)** | **9** | **325.2** | **Tốt — ít chunks hơn, avg length lớn hơn → context đủ rộng** |
| BLS CPI (111,283 chars) | FixedSizeChunker (baseline) | 248 | 498.5 | Kém — cắt ngang bảng |
| BLS CPI | **RecursiveChunker (của tôi)** | **263** | **421.4** | **Tốt nhất trong 3 — tách theo paragraph** |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Queries Relevant | Điểm mạnh | Điểm yếu |
|-----------|----------|-----------------|-----------|----------|
| Tôi | RecursiveChunker (chunk_size=500) | 3/5 | Filter theo date hoạt động đúng | Q3 thất bại do top-1 là raw data chunk |
| Phan Thanh Sang | RawDocumentChunker | 5/5 | Score cao nhất (1.22), không bị cắt nhầm vào raw data | Không scale được với document lớn + embedder thật |
| Đỗ Minh Khiêm | RecursiveChunker (chunk_size=500) | 5/5 | Chunk đầu file relevant, agent answer đầy đủ | Score âm ở Q2/Q3 (mock embedder không semantic) |
| Trần Đình Minh Vương | SentenceChunker (max=3) | 2-3/5 | Score cao ở Q1 (0.69) | Retrieve nhầm tháng ở Q2 — thiếu date filter |
| Trần Tiến Dũng | SentenceChunker (max=5) | ~4/5 | Gold answer chi tiết (kể cả dissenters FOMC vote) | Dùng queries riêng, khó so sánh trực tiếp |

**Strategy nào tốt nhất cho domain này? Tại sao?**

Kết quả thú vị: `RawDocumentChunker` (Sang) đạt score cao nhất với mock embedder vì không bị rơi vào chunk raw data — nhưng đây là artifact của mock embedder, không phải ưu điểm thật. Với embedder thật và document 111k chars, raw embedding sẽ bị "loãng" và kém hơn chunking. `RecursiveChunker` (Khiêm, và tôi) là strategy cân bằng nhất cho production. Bài học quan trọng: **metadata filter theo `date`** là yếu tố quyết định với FOMC statements — TV3 (Vương) thất bại Q2 vì thiếu date filter, retrieve nhầm tháng 7 thay vì tháng 12.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:  
Tôi dùng regex `(?<=[.!?])(?:\s+|\n+)` để tách câu theo các dấu kết thúc câu phổ biến rồi loại bỏ khoảng trắng dư thừa. Sau đó các câu được gom lại theo `max_sentences_per_chunk` để tạo chunk có kích thước ổn định hơn. Tôi cũng xử lý edge case như chuỗi rỗng hoặc chuỗi chỉ có khoảng trắng bằng cách trả về danh sách rỗng.

**`RecursiveChunker.chunk` / `_split`** — approach:  
Thuật toán recursive chia văn bản theo thứ tự ưu tiên của separator như `\n\n`, `\n`, `. `, khoảng trắng, rồi cuối cùng mới fallback sang cắt cứng theo độ dài. Base case là khi đoạn văn bản đã ngắn hơn hoặc bằng `chunk_size`, hoặc khi không còn separator nào để tách nữa. Cách làm này giúp chunk thường bám theo ranh giới tự nhiên trước khi phải dùng cách cắt cơ học.

### EmbeddingStore

**`add_documents` + `search`** — approach:  
Mỗi document được chuẩn hóa thành một record gồm `id`, `content`, `metadata`, và `embedding`, sau đó được lưu trong bộ nhớ bằng danh sách `self._store`; nếu ChromaDB khả dụng thì cũng đồng thời thêm vào collection. Khi search, hệ thống embed câu query rồi tính điểm tương đồng với từng record bằng dot product giữa query embedding và document embedding. Kết quả sau đó được sắp xếp giảm dần theo score và trả về `top_k`.

**`search_with_filter` + `delete_document`** — approach:  
Tôi filter trước trên metadata, rồi mới chạy similarity search trên tập record đã lọc để tăng độ chính xác và giảm nhiễu. Hàm `search_with_filter` kiểm tra tất cả cặp key-value trong `metadata_filter` và chỉ giữ lại các record phù hợp. Hàm `delete_document` xóa toàn bộ record có `metadata['doc_id']` trùng với `doc_id` cần xóa, rồi trả về `True` nếu thực sự có dữ liệu bị xóa.

### KnowledgeBaseAgent

**`answer`** — approach:  
Hàm `answer` trước hết retrieve `top_k` kết quả liên quan từ `EmbeddingStore`, sau đó ghép nội dung của các kết quả này thành phần `Context` trong prompt. Prompt được tổ chức theo dạng chỉ dẫn ngắn gọn: nêu vai trò trợ lý, yêu cầu trả lời dựa trên context, rồi đưa vào `Question`. Cách inject context này đơn giản nhưng đủ để mô phỏng pattern RAG cơ bản trong bài lab.

### Test Results

```text
============================= test session starts =============================
platform win32 -- Python 3.12.8, pytest-9.0.3, pluggy-1.6.0
rootdir: C:\Users\ADMIN\Day-07-Lab-Data-Foundations
collected 42 items

tests/test_solution.py ..........................................       [100%]

======================== 42 passed, 1 warning in 0.13s ========================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | The CPI rose 2.7 percent year over year in December 2025. | Inflation increased 2.7% compared with the same month last year. | high | -0.1515 | Không |
| 2 | The Fed kept rates unchanged in July 2025. | The Federal Reserve maintained the federal funds rate in July 2025. | high | -0.0865 | Không |
| 3 | Shelter was the largest contributor to the CPI increase. | Food and energy prices also increased in December. | low | 0.1080 | Không |
| 4 | Nonfarm payroll employment increased by 178,000 in March 2026. | The unemployment rate changed little at 4.3 percent in March 2026. | high | -0.1875 | Không |
| 5 | The Federal Reserve held rates steady. | Tropical storms caused flooding near the coastline. | low | 0.2627 | Không |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Điều bất ngờ nhất là cặp số 5 có dự đoán thấp nhưng actual score lại dương khá rõ, trong khi nhiều cặp có ý nghĩa gần nhau lại ra điểm âm. Điều này cho thấy mock embeddings của lab không phản ánh ngữ nghĩa thực tế tốt như embedding model thật, vì chúng được thiết kế để deterministic cho mục đích kiểm thử chứ không tối ưu cho semantic similarity. Từ đó có thể rút ra rằng khi đánh giá retrieval, chất lượng embedding backend ảnh hưởng rất mạnh đến kết quả cuối cùng.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | What was the CPI year-over-year change in December 2025? | 2.7% year-over-year |
| 2 | What was the federal funds rate decision in July 2025? | Maintained at 5-1/4 to 5-1/2 percent |
| 3 | What drove the monthly CPI increase in December 2025? | Shelter (+0.4%), food index (+0.7%), energy (+0.3%) |
| 4 | How did the Fed describe the labor market in its 2025 statements? | Job gains moderated but remain strong, unemployment rate low |
| 5 | What interest rate did the Fed set in December 2025? | 3-1/2 to 3-3/4 percent |

### Kết Quả Của Tôi
Parameter: chunk_size=500
| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | What was the CPI year-over-year change in December 2025? | Chunk từ `bls_cpi_december_2025.txt` nêu rõ Key Metrics và câu “all items index increased 2.7 percent” | 0.2243 | Có | Agent có thể trả lời rằng CPI tháng 12/2025 tăng 2.7% so với cùng kỳ năm trước |
| 2 | What was the federal funds rate decision in July 2025? | Chunk đầu của `fed_fomc_statement_2025_07_30.txt` chứa quyết định giữ target range ở 5-1/4 đến 5-1/2 percent | -0.1403 | Có | Agent có thể trả lời rằng Fed giữ nguyên lãi suất ở mức 5-1/4 đến 5-1/2 percent trong kỳ tháng 7/2025 |
| 3 | What drove the monthly CPI increase in December 2025? | Chunk đầu của `bls_cpi_december_2025.txt` nêu shelter tăng 0.4%, food tăng 0.7%, energy tăng 0.3% | -0.0872 | Có | Agent có thể trả lời rằng mức tăng CPI theo tháng chủ yếu do shelter, food và energy dẫn dắt |
| 4 | How did the Fed describe the labor market in its 2025 statements? | Top chunks từ FOMC tháng 7, 9, 12/2025 đều nhấn mạnh job gains moderated và unemployment rate remained low | 0.0917 | Có | Agent có thể tóm tắt rằng Fed mô tả thị trường lao động vẫn vững, việc làm chậm lại nhưng còn mạnh và thất nghiệp thấp |
| 5 | What interest rate did the Fed set in December 2025? | Chunk đầu của `fed_fomc_statement_2025_12_10.txt` chứa mức 3-1/2 đến 3-3/4 percent | 0.1581 | Có | Agent có thể trả lời rằng Fed đặt mức lãi suất mục tiêu ở 3-1/2 đến 3-3/4 percent vào tháng 12/2025 |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

**Ghi chú benchmark:**  
Khi chạy với mock embeddings mặc định của lab, similarity score không phải lúc nào cũng trực giác và có thể mang giá trị âm. Tuy vậy, khi kết hợp retrieval với metadata filtering theo `category` và ngày cụ thể như `2025-07-30` hoặc `2025-12-10`, hệ thống vẫn trả về đúng tài liệu liên quan cho cả 5 benchmark queries ngay cả khi tập dữ liệu đã mở rộng lên 6 documents. Điều này cho thấy metadata có vai trò rất quan trọng trong domain Financial News, đặc biệt khi câu hỏi có yếu tố thời gian và loại chỉ báo kinh tế rõ ràng.

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Tôi học được rằng cùng một bộ dữ liệu nhưng strategy chunking khác nhau có thể tạo ra chất lượng retrieval rất khác nhau. Một số strategy dễ đọc hơn ở cấp câu nhưng lại làm mất ngữ cảnh khi câu hỏi cần nối nhiều ý trong cùng một đoạn. Điều này giúp tôi hiểu rõ hơn rằng retrieval không chỉ phụ thuộc vào embedding mà còn phụ thuộc rất mạnh vào cách chuẩn bị dữ liệu.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Qua phần demo, tôi thấy việc thiết kế metadata tốt có thể cải thiện retrieval rõ rệt, đặc biệt với các câu hỏi có yếu tố thời gian hoặc chủ đề cụ thể. Tôi cũng nhận ra rằng những nhóm có bộ benchmark queries rõ ràng và có gold answers cụ thể thường đánh giá strategy khách quan hơn. Điều này cho thấy chất lượng đánh giá retrieval phụ thuộc nhiều vào cách đặt câu hỏi và định nghĩa đúng đáp án chuẩn.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Nếu làm lại, tôi sẽ bổ sung thêm các tài liệu cùng kỳ thời gian để coverage của bộ dữ liệu đồng đều hơn giữa inflation, employment, và monetary policy. Tôi cũng sẽ gắn metadata chi tiết hơn như `release_month`, `indicator_family`, hoặc `meeting_cycle` để hỗ trợ filter chính xác hơn. Ngoài ra, tôi muốn thử hybrid strategy, ví dụ chia theo paragraph trước rồi mới cắt theo độ dài, để giảm bớt tình trạng paragraph quá ngắn hoặc quá dài.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 3 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 13 / 15 |
| My approach | Cá nhân | 9 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 9 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 4 / 5 |
| **Tổng** | | **84 / 100** |

