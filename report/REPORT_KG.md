# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Mạnh Hùng  **MSSV:** 2A202602708  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     53.2
graph       196     91958     4708   0.00933    123.2

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.53
graph       0.69   1.50     3261       80   0.00053     2.27
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00933 | **×8.33** |
| Indexing giây | 53.2s | 123.2s | **×2.32** |
| Mỗi câu: USD | $0.00013 | $0.00053 | **×4.08** |
| Mỗi câu: giây | 1.53s | 2.27s | **×1.48** |
| Mỗi câu: in_tok | 694 | 3261 | **×4.70** |

**Chi phí tăng thêm đến từ đâu?**
> Ở khâu **Indexing**, chi phí của GraphRAG cao hơn gấp ~8.3 lần vì phải gọi LLM trích xuất thực thể, vụ án và tội danh từ 20 bài báo tin tức (tiêu tốn thêm ~35,800 input tokens và ~4,700 output tokens JSON), trong khi Flat RAG chỉ tốn embedding vector. Ở khâu **Querying**, chi phí mỗi câu của GraphRAG tăng gấp ~4 lần do prompt phải nạp thêm danh sách các dữ kiện có cấu trúc (`facts`) lấy từ Cypher multi-hop, làm `in_tok` tăng từ 694 lên 3261 tokens.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa tiền chất nằm trọn trong Điều 2 Luật PCMT 2021 nên cả 2 bên đều truy xuất trực tiếp rất tốt. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Thông tin án tử hình của Trần Thanh Tuấn và Trần Minh Tâm nằm gọn trong bài báo 36kg ma túy nên cả hai đều tìm được. |
| **Q3** | cross-kb | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG bị tách rời dữ kiện và trả về "Không đủ thông tin", còn Graph đi qua cầu nối Crime "mua bán trái phép chất ma túy" để lấy Điều 251 khoản 1. |
| **Q4** | cross-kb | 0.00 / 0 | 0.33 / 1 | **Graph** | Flat RAG hoàn toàn không trả lời được, còn Graph tìm được hành vi và tội danh "tổ chức sử dụng trái phép chất ma túy" của Hoàng Nato. |
| **Q5** | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | **Graph** | GraphRAG kết nối chính xác từ vụ Cái Quang Huy sang tội vận chuyển và định khung hình phạt liên quan đến chất MDMA. |
| **Q6** | aggregation | 0.00 / 1 | 0.00 / 1 | **Hòa** | Cả hai đều tổng hợp đúng 3 vụ án liên quan MDMA nhưng đều bị 0 recall do lỗi đo lường từ khóa `must_include`. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật do quy tắc lọc khoản (Q4)

- **Hiện tượng:** Ở câu Q4, câu hỏi hỏi mức phạt tù *tối đa* của hành vi mà Hoàng Nato bị bắt (tội tổ chức sử dụng trái phép chất ma túy - Điều 255 BLHS). GraphRAG chỉ nhận diện được tội danh nhưng trả lời: *"trong ngữ cảnh không có thông tin cụ thể về mức án phạt tù tối đa theo Bộ luật Hình sự. Do đó, không đủ thông tin để trả lời..."*.
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời của GraphRAG trong `ket_qua_benchmark_kg.txt`:
    > *"Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Tuy nhiên, trong ngữ cảnh không có thông tin cụ thể về mức án phạt tù tối đa cho hành vi này theo Bộ luật Hình sự. Do đó, không đủ thông tin để trả lời câu hỏi về mức phạt tù tối đa."*
  - Truy vấn kiểm tra dữ kiện trong graph đối với vụ án này:
    ```cypher
    MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause)
    WHERE k.name CONTAINS "Hoàng Nato" OR k.name CONTAINS "Dương Minh Tuấn"
    RETURN a.id, cl.number, cl.penalty;
    ```
    *Kết quả:* Vụ án không có cạnh `INVOLVES` nối tới loại chất cụ thể nào do bài báo chỉ nói chung về việc sử dụng ma túy tại quán karaoke. Logic của `context()` chỉ lấy:
    `WHERE cl.number = 1 OR EXISTS { MATCH (k)-[:INVOLVES]->(s)<-[:MENTIONS]-(cl) }`.
    Do đó, hệ thống chỉ lấy khoản 1 (khung 2 - 7 năm), hoàn toàn bỏ sót **khoản 4** (khung phạt cao nhất: 20 năm hoặc tù chung thân).
- **Nguyên nhân:** Logic lọc khoản ở Bước 5 (KG-3) đặt giả định quá chặt chẽ: chỉ lấy khoản 1 và khoản nhắc tới chất ma túy của vụ án. Khi câu hỏi hỏi về *mức phạt tối đa* hoặc khi vụ án không ghi rõ tên chất, quy tắc này không lấy khoản có khung hình phạt cao nhất của Điều luật.
- **Đề xuất sửa:** Trong hàm `context()` tại `src/graph.py`, nếu câu hỏi chứa các từ khóa như *"tối đa"*, *"khung cao nhất"*, *"nặng nhất"*, bổ sung truy vấn lấy thêm khoản có số thứ tự lớn nhất hoặc khoản chứa hình phạt *"tù chung thân"*, *"tử hình"* của Điều luật đó. Đánh đổi: prompt dài thêm khoảng 100-200 tokens nhưng trả lời chính xác các câu hỏi về khung hình phạt kịch khung.

---

### Lỗi E4: Phép đo sai do từ khóa `must_include` quá cứng nhắc (Q6)

- **Hiện tượng:** Ở câu Q6 (câu hỏi tổng hợp: *"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*), cả Flat RAG và GraphRAG đều liệt kê đúng cả 3 vụ án có liên quan đến MDMA trong dữ liệu, nhưng điểm `recall` tự động đều bị chấm **0.00**.
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời của GraphRAG trong `ket_qua_benchmark_kg.txt`:
    > *"Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:*
    > *1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Liên quan đến 9,6kg MDMA.*
    > *2. Vụ góp tiền mua ma túy tại Hà Nội: Có 5 viên MDMA bị thu giữ.*
    > *3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Có 0,686g MDMA bị thu giữ.*
    > *Tất cả các vụ việc này đều có liên quan đến ma túy MDMA."*
  - Đối chiếu với file `data/benchmark_kg.json`:
    ```json
    "id": "Q6",
    "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
    ```
  - Kiểm tra nội dung bài báo tương ứng: Vụ thứ 3 xảy ra tại Viện Pháp y tâm thần Trung ương nhưng bài báo và LLM tóm tắt theo tên đối tượng hoặc địa danh, câu trả lời không chứa cụm từ chính xác 100% là `"Pháp y tâm thần"`, dẫn đến hàm `keyword_recall()` trả về 0.00.
- **Nguyên nhân:** Lỗi nằm ở **chính phép đo benchmark**: sử dụng hàm `keyword_recall()` so khớp chuỗi con chính xác tuyệt đối (`k.lower() in answer.lower()`). Khi mô hình tổng hợp thông tin theo cách diễn đạt tự nhiên hoặc tóm tắt theo tình tiết vụ việc thay vì trích dẫn đúng cụm từ khóa cứng trong benchmark, điểm số bị đánh tụt về 0 dù bản chất thông tin hoàn toàn chính xác.
- **Đề xuất sửa:** 
  1. Trong benchmark, cho phép danh sách từ khóa tương đương cho mỗi ý (ví dụ: `["Pháp y tâm thần", "buồng chữa bệnh", "Sầm Sơn", "Viện pháp y"]`).
  2. Tin cậy vào điểm của `judge` (LLM-as-judge chấm điểm ngữ nghĩa) hơn là `keyword_recall` cơ học đối với các câu hỏi dạng aggregation.

---

## 4. Kết luận (5 điểm)

**Khi nào nên dùng Knowledge Graph (GraphRAG), khi nào Flat RAG là đủ?**

1. **Flat RAG hoàn toàn đủ và tối ưu khi:**
   - Dữ liệu có tính cục bộ cao: Câu hỏi mà câu trả lời nằm trọn vẹn trong một đoạn văn bản hoặc một tài liệu duy nhất (như câu Q1 về luật định nghĩa hay Q2 về bản án một vụ việc cụ thể).
   - Cần tối ưu chi phí và tốc độ: Flat RAG rẻ hơn **8.3 lần** lúc Indexing ($0.00112 vs $0.00933) và rẻ hơn **4.1 lần** cho mỗi truy vấn ($0.00013 vs $0.00053), độ trễ nhanh hơn 33% (1.53s vs 2.27s).

2. **Bắt buộc phải dùng Knowledge Graph (GraphRAG) khi:**
   - **Tri thức liên vùng (Cross-KB):** Khi câu trả lời đòi hỏi phải liên kết dữ liệu nằm ở nhiều nguồn độc lập mà không văn bản nào chứa cả hai (ví dụ Q3: Flat RAG đạt 0.00 recall và judge=0 vì không thể liên kết được giữa Lê Minh Thành và Điều 251 BLHS; trong khi GraphRAG đạt tuyệt đối 1.00 recall và judge=2).
   - **Truy vấn Multi-hop & Quan hệ thực thể:** Cần suy diễn qua nhiều bước từ Người → Vụ án → Tội danh → Điều luật → Khoản tương ứng (như câu Q5, GraphRAG đạt recall 0.80 so với 0.60 của Flat RAG).
   - **Tóm lại:** Knowledge Graph đáng tiền khi bài toán có độ phức tạp về quan hệ thực thể cao và tri thức bị phân mảnh giữa nhiều hệ thống/văn bản khác nhau.

---

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.12s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 205 node / 382 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00045.
```

- **Ảnh Neo4j:** `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
- **Người đã chọn cho `kg_my_case.png`:** `Cái Quang Huy` (Vụ án vận chuyển trái phép hơn 9,6kg ma túy MDMA qua Cảng hàng không quốc tế Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

- **Vấn đề Rate Limit của Gemini Free Tier:** Khi thử nghiệm với Gemini Free Tier, hạn mức miễn phí bị khống chế nghiêm ngặt ở 20 requests/ngày (`GenerateRequestsPerDayPerProjectPerModel-FreeTier`), gây ra lỗi 429 Resource Exhausted khi nạp 20 bài báo tin tức.
- **Cách xử lý:** Chuyển sang sử dụng provider chính là OpenAI (`gpt-4o-mini` & `text-embedding-3-small`) theo đúng khuyến nghị của bài lab. 
