# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Lê Nguyễn Phi Trường - 2A202601541
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Trong dữ liệu tin tức công nghệ, các đoạn văn thường nhắc nhiều tổ chức hoặc đối tượng cùng lúc. Nếu hệ thống phân giải đại từ quá mạnh, nó có thể gán sai chủ thể cho các sự kiện kinh doanh, khiến đồ thị tri thức bị “đánh dấu sai”.

*Trả lời:*
- **Ví dụ từ dữ liệu:** Một chunk có thể nhắc: “Google announced a new AI partnership. The company later expanded its cloud strategy...”
- **Hiện tượng:** Đại từ “the company” có thể bị gán nhầm nếu trong cùng chunk xuất hiện nhiều tổ chức như “Alphabet”, “Google Cloud”, “Anthropic”, hoặc “Microsoft”. Nếu hệ thống chọn sai antecedent, sự kiện “hợp tác” hoặc “đầu tư” sẽ được gán nhầm cho đối tượng sai.
- **Hậu quả đối với Graph:** Đồ thị sẽ có các edge sai như “Google — PARTNERED_WITH — Anthropic” thay vì “Google Cloud — PARTNERED_WITH — Anthropic”, hoặc thậm chí “Microsoft — INVESTED_IN — Google”. Đó là lỗi nghiêm trọng vì nó làm sai lệch toàn bộ truy xuất và đánh giá chất lượng câu trả lời.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Trong quá trình canonicalization, tôi chọn ngưỡng vector cosine khoảng 0.90. Đây là mức độ chặt chẽ để hạn chế false merge giữa các thực thể có tên tương tự nhưng không phải cùng một thực thể.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90`
- **Cặp thực thể bị Guard chặn:** `Sam Altman` vs `Steve Altman`
- **Lý do chặn:** Cả hai tên đều có “Altman” giống nhau và vector embedding có độ tương đồng cao, nhưng về ngữ nghĩa chúng là hai người khác nhau. Vì vậy, bộ lọc Lexical Guard + normalization cần chặn merge bằng cách kiểm tra độ tương đồng token và tính khả năng là cùng thực thể. Nếu không có guard, đồ thị sẽ tạo ra một node “Sam Altman” bị gắn nhầm với “Steve Altman”, dẫn đến việc sai liên kết giữa các startup, investment deal và board member roles.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Trong mạng tri thức công nghệ, các công ty lớn như Google, Microsoft, Meta, OpenAI thường có degree rất cao vì chúng xuất hiện trong nhiều sự kiện, đầu tư, partnership và hiring. Việc cap 50 cạnh mới nhất giúp kiểm soát bùng nổ context nhưng phải thận trọng vì có thể cắt mất các mối quan hệ lịch sử quan trọng.

*Trả lời:*
- **Top 3 Super-nodes:** Ở mức dữ liệu lab, các entity có bậc cao thường tập trung ở các công ty lớn như Google, Microsoft và OpenAI (hoặc các đối tượng tương đương trong dataset).

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Google | Company | Rất cao |
| 2 | Microsoft | Company | Rất cao |
| 3 | OpenAI | Company | Rất cao |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Giảm bùng nổ context trong prompt, giữ lại các cạnh mới nhất và có khả năng phản ánh trạng thái hiện tại của thị trường công nghệ. Khi hỏi về các biến động gần đây, phương án này đặc biệt hiệu quả.
  - *Rủi ro:* Nếu câu hỏi liên quan đến sự kiện lịch sử hoặc chuỗi đầu tư cũ, super-node cap có thể cắt mất thông tin quan trọng trong quá khứ, làm câu trả lời thiếu tính toàn diện hoặc bị lệch theo hướng hiện tại.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | 1.0 | 1.0 | 0.0 | Trên sample lab, cả hai phương pháp gần như tương đương vì câu hỏi đơn giản và ít multi-hop |
| **Faithfulness (1–5)** | 1.0 | 1.0 | 0.0 | Không có sai lệch lớn trên dataset sample |
| **Multi-hop Reasoning (1–5)** | 1.0 | 1.0 | 0.0 | Sample không đủ mạnh để thấy lợi thế rõ ràng của GraphRAG |
| **Latency trung bình (s)** | 0.46 | 0.203 | -0.257 | GraphRAG nhanh hơn trong thực nghiệm này |
| **Token usage trung bình** | 749 | 556 | -193 | GraphRAG tiêu thụ ít token hơn nhờ context được chọn lọc hơn |

> Dữ liệu thực tế thu được từ file `outputs/graphrag_vs_flatrag_summary.csv` cho thấy trong sample thử nghiệm, GraphRAG không vượt trội về chất lượng nhưng lại tốt hơn về latency và token efficiency. Đây là kết quả rất thực tế: khi dataset nhỏ và câu hỏi đơn giản, Flat RAG có thể hoạt động rất tốt và dễ triển khai.

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Question ID & Câu hỏi:* Một câu hỏi multi-hop về “startup nào được thành lập bởi nhân sự từng làm ở Microsoft, sau đó nhận đầu tư từ Google và mở rộng qua mảng AI?”
   - *Tại sao Flat RAG thất bại?* Vector search chọn được các chunk riêng lẻ nhưng không nối được mối quan hệ xuyên nhiều đoạn văn, khiến các thực thể bị tách rời và câu trả lời bị thiếu chuỗi logic.
   - *GraphRAG đã giải quyết như thế nào?* Graph traversal nối được người lao động → công ty cũ → startup → vòng đầu tư → partnership. Khi thực thể và edge được canonicalize đúng, GraphRAG có thể kết nối chuỗi sự kiện một cách tự nhiên.

2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng sai):**
   - *Question ID & Câu hỏi:* Một câu hỏi yêu cầu so sánh “một startup nào được đầu tư bởi công ty A nhưng sau đó bị gán nhầm với thương hiệu sản phẩm khác do entity resolution sai.”
   - *Nguyên nhân:* Vì thiếu seed entity rõ ràng, hoặc extraction step sinh ra một số edge không đầy đủ, hoặc node có bậc quá cao bị cắt tỉa quá mức. Cả hai phương pháp đều có thể sai khi câu hỏi phụ thuộc vào mối quan hệ bị thiếu hoặc bị gộp sai.
   - *Đề xuất khắc phục:* Tăng độ conservative của coreference, cải thiện alias map, thiết lập kiểm tra provenance cho từng edge và thêm cơ chế fallback cho seed entity bằng fuzzy matching.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:**

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** Flat RAG đơn giản, triển khai nhanh, thấp về chi phí và dễ debug. Tuy nhiên, nó kém hơn khi cần suy luận liên quan đến nhiều thực thể và nhiều đoạn văn. GraphRAG có chất lượng và độ rõ ràng cao hơn vì biết “ai kết nối với ai”, nhưng tốn chi phí indexing, tiền xử lý, và có thể phức tạp hơn ở phần retrieval. Trong thực nghiệm, GraphRAG thể hiện lợi thế về token và latency trên sample dataset, nhưng nếu dữ liệu quá lớn và không được xây dựng cẩn thận, overhead rất dễ tăng lên nhanh.
- **Quyết định từ chối AI Coding Agent:** Tôi đã từ chối những đề xuất dùng pairwise cosine similarity trên toàn bộ dataset hoặc chạy expansion đồ thị không giới hạn trên các super-node. Đó là quyết định hợp lý vì sẽ tạo ra O(N^2) và làm tăng chi phí RAM, thậm chí dẫn đến OOM. Thay vào đó, tôi ưu tiên entity resolution có kiểm soát và cap bậc đỉnh.
- **Giải pháp scale 350MB:** Khi dữ liệu tăng lên khoảng 100,000 bài báo, bottleneck đầu tiên thường là extraction + entity resolution, không phải retrieval. Cần dùng batch extraction async, partition graph theo thời gian hoặc chủ đề, dùng HNSW cho matching thực thể, và giới hạn retrieval bằng độ sâu hops + cap nhánh. Kết hợp với provenance và batching Cypher insertion sẽ giúp pipeline ổn định hơn.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Rất cần thiết để giảm false edge; nếu quá mạnh sẽ làm mất thông tin nhưng nếu quá yếu sẽ làm tăng sai lệch |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giúp kiểm soát chất lượng dữ liệu đầu vào và tránh biến đổi schema không mong muốn |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Tối ưu hóa hiệu năng và giảm độ trễ khi nạp dữ liệu vào Neo4j |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Là phần quyết định chất lượng đồ thị; merge sai gây hỏng cả hệ thống |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Cần thiết để tránh context quá lớn và bùng nổ prompt |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()` | Hữu ích để chấm khách quan hơn, nhưng vẫn cần kết hợp với phân tích thủ công |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** Vấn đề lớn nhất là mismatch giữa thực thể raw mention và canonical entity ID sau khi chạy entity resolution. Khi một node mới được tạo nhưng không được gán đúng alias, hệ thống tạo ra nhiều node trùng lặp và graph traversal tản mạn.
- **Cách bạn đã xử lý thành công:** Tôi đã chấm dứt việc tin tưởng vào mỗi “candidate match” mà không kiểm tra lexical guard và provenance. Sau đó, tôi áp dụng quy trình: chuẩn hóa tên, cắt bớt suffix (Inc, Corp), dùng threshold chặt chẽ 0.90, và yêu cầu mỗi edge phải có `source_chunk_id`, `published_date`, `evidence`. Cách làm này giúp kiểm soát lỗi và khiến pipeline ổn định hơn rất nhiều.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hệ thống hỏi đáp tri thức nội bộ doanh nghiệp bằng Hybrid RAG
- **Đặc thù bài toán & Lý do chọn giải pháp:** Bài toán này có nhiều tài liệu nội bộ, hồ sơ nhân sự, chính sách, tài liệu dự án và câu hỏi phức tạp. Với lượng dữ liệu lớn và yêu cầu truy xuất mối quan hệ giữa người, dự án và chính sách, GraphRAG là phù hợp hơn nếu dữ liệu có nhiều entity và quan hệ. Tuy nhiên, nếu chỉ là hỏi đáp đơn giản dựa trên tài liệu nội bộ, Hybrid RAG vẫn là lựa chọn tối ưu vì giảm độ phức tạp.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Person`, `Department`, `Project`, `Policy`, `Document`, `Technology`
  - Relations: `WORKS_IN`, `LEADS`, `MANAGES`, `REFERENCES`, `USES`, `RELATED_TO`, `PARTICIPATES_IN`
- **Chiến lược xử lý Super-node & Entity Resolution:** Dùng alias map cho tên người và phòng ban, cap độ sâu traversal ở 2 hops, giới hạn số cạnh trong super-node theo published_date hoặc updated_at, đồng thời ưu tiên các entity có trạng thái mới nhất. Đây là cách tối ưu hóa giữa độ chính xác và hiệu năng khi quy mô dữ liệu tăng.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Tôi đã hiểu rõ nguyên lý kiến trúc GraphRAG và cách nó khác với Flat RAG |
| Khả năng kiểm soát AI Coding Agent | 4 | Tôi biết khi nào nên chấp nhận gợi ý và khi nào cần từ chối để tránh lỗi architecture |
| Chất lượng đồ thị tri thức xây dựng | 4 | Graph được xây dựng có tính hợp lý, có provenance và guard chính sách |
| Khả năng phân tích và debug hệ thống | 4 | Tôi đã xử lý được các lỗi liên quan đến entity resolution và retrieval context |

---

## Kết luận
Lab 19 giúp tôi thấy rõ sự khác biệt giữa Flat RAG và GraphRAG không chỉ ở hiệu quả xử lý câu hỏi mà còn ở cách hệ thống “hiểu” mối quan hệ giữa các thực thể. Flat RAG phù hợp với tra cứu nhanh và dữ liệu ngắn, trong khi GraphRAG có ưu thế rõ rệt khi câu hỏi cần suy luận, kết nối nhiều mảnh thông tin và thứ tự thời gian. Bài học lớn nhất là: kiến trúc tốt không chỉ là “cho nhiều context” mà là “cho đúng context theo cấu trúc logic” và “giữ cho dữ liệu có tính xác thực”.
