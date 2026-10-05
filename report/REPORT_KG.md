# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Dương Thị Ngân<br>
**MSSV:** 2A202602808<br>
**Ngày:** 05/10/2026

## 1. Chi phí

Benchmark dùng `gemini-3.5-flash-lite`, embedding `gemini-embedding-001`, `top_k=3`, `chunk_size=800`, tổng cộng 176 chunks. Graph đầy đủ có 200 nodes và 381 relationships.

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    151.2
graph       196     34619     5578   0.02433    265.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       83   0.00042     1.97
graph       1.00   2.00     5232      139   0.00192    62.64
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00000 | 0.02433 | N/A (Gemini embedding không trả usage token nên file ghi 0) |
| Indexing giây | 151.2 | 265.3 | 1.75× |
| Mỗi câu: USD | 0.00042 | 0.00192 | 4.57× |
| Mỗi câu: giây | 1.97 | 62.64 | 31.80× |
| Mỗi câu: input token | 696 | 5232 | 7.52× |

Chi phí tăng chủ yếu đến từ 20 lượt LLM trích xuất tin tức khi dựng graph và phần context nhiều bước dài hơn khi trả lời. Thời gian GraphRAG trung bình bị tăng mạnh do Q4 gặp giới hạn 15 request/phút của Gemini và phải chờ retry 360,91 giây; đây là độ trễ phía provider, không phải thời gian chạy Cypher.

## 2. Kết quả từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa nằm gọn trong một chunk luật nên Flat RAG đã đủ. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Hai tên bị cáo cùng nằm trong bài báo được truy xuất. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Graph nối Lê Minh Thành → vụ án → tội danh → Điều 251 → khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat chỉ biết hành vi; graph lấy được Điều 255 và mức cao nhất là chung thân. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Cầu `Crime` và `Substance` đưa vụ Cái Quang Huy tới khoản 4 Điều 250. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | Graph | Graph duyệt mọi `Case` nối với MDMA và nêu được các thực thể bắt buộc; Flat chỉ lấy ba chunk và không nêu tên vụ/người. |

Quy luật thấy rõ: câu single-hop hòa nhau, còn cả ba câu cross-KB đều cần dữ kiện ở tin và luật nên GraphRAG thắng. Với câu tổng hợp, graph cũng có lợi vì không bị giới hạn trong ba chunk gần nhất.

## 3. Phân tích lỗi

### Lỗi E1: Một số vụ án không nối sang luật

- **Hiện tượng:** graph đầy đủ vẫn có ba `Case` không có cạnh `CHARGED_WITH`; trong đó có một node về Cái Quang Huy dù bài nguồn nêu hành vi vận chuyển ma túy.
- **Bằng chứng:**

```cypher
MATCH (k:Case)
WHERE NOT (k)-[:CHARGED_WITH]->()
RETURN k.name, k.doc_id;
```

```text
Vụ vận chuyển ma túy qua sân bay Nội Bài của Cái Quang Huy | news-100260918080821054
Vụ vận chuyển hơn 800kg ma túy và súng đạn tại Preah Sihanouk | news-100260924145818945
Triệt phá chuyên án A3-626p | news-100261002184934505
```

- **Nguyên nhân:** KG-2 phụ thuộc JSON do LLM trích xuất. Một bài có thể sinh nhiều `Case`; nếu `charges` rỗng hoặc tên tội không qua được `link_entity`, node vẫn được tạo nhưng không có cầu `Crime`. Khóa `Case.name` do LLM đặt còn làm cùng một tài liệu sinh hai node vụ việc khác nhau.
- **Đề xuất sửa:** sau trích xuất, kiểm tra mọi case có `charges`; nếu bài có từ khóa tội danh chuẩn thì chạy bộ phân loại dự phòng hoặc đánh dấu `needs_review` thay vì lặng lẽ tạo node rời. Dùng khóa ổn định `doc_id + case_index` để tránh lệ thuộc tên LLM. Cách này thêm một bước kiểm tra và có thể tăng token nếu phải gọi LLM lại.

### Lỗi E4: Keyword recall và LLM judge mâu thuẫn ở Q6

- **Hiện tượng:** câu Q6 của Flat RAG có `recall=0.00` nhưng `judge=2`.
- **Bằng chứng:** file benchmark ghi:

```text
--- Q6 [aggregation] flat recall=0.00 judge=2
Dựa trên ngữ cảnh, cả 3 vụ việc đều có liên quan đến ma túy MDMA:
1. Vụ việc [1]: ... gần 4,3kg.
2. Vụ việc [2]: ... 5 viên MDMA ...
3. Vụ việc [3]: ... hơn 5,3kg.
```

Ba từ khóa bắt buộc của Q6 là `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`; câu Flat không chứa từ nào nên keyword recall bằng 0 là đúng theo công thức. Tuy nhiên câu vẫn mô tả đúng một phần nội dung của ba chunk, vì vậy LLM judge đã chấm quá rộng tay ở mức 2.

- **Nguyên nhân:** keyword recall chỉ kiểm tra chuỗi, không hiểu diễn đạt tương đương; ngược lại LLM judge hiểu nghĩa nhưng không bị ràng buộc chặt phải kiểm đủ từng thực thể trong đáp án chuẩn.
- **Đề xuất sửa:** dùng bộ điểm kết hợp: entity recall cho tên người/vụ, semantic similarity cho nội dung, và judge phải trả JSON kèm danh sách từng ý chuẩn đã/ chưa đạt. Cách này chính xác hơn nhưng cần thêm token và bước kiểm chứng.

## 4. Kết luận

Flat RAG phù hợp khi câu trả lời nằm trong một nguồn: Q1 và Q2 đều đạt recall `1.00`, judge `2`, trung bình chỉ `0.00042 USD` và `1.97 giây/câu`. Knowledge Graph đáng dùng khi câu hỏi phải nối tin tức với luật hoặc tổng hợp nhiều vụ: Q3–Q6 của GraphRAG đều recall `1.00`, judge `2`, trong khi Flat chỉ đạt lần lượt `0.33`, `0.33`, `0.40`, `0.00` về recall.

Đổi lại, GraphRAG tốn khoảng `4.57×` chi phí truy vấn, `7.52×` input token và có chi phí dựng graph một lần `0.02433 USD`. Vì vậy KG đáng tiền khi hệ thống có nhiều câu cross-KB lặp lại và cần truy vết căn cứ; với câu single-hop ít lượt hỏi thì Flat RAG đơn giản và rẻ hơn.

## 5. Tự kiểm

```text
$ python -m pytest tests/ -q
................................................                         [100%]
48 passed in 0.14s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM. Graph nhỏ vẫn còn trong Neo4j để xem.
```

## 6. Ảnh Neo4j

- `report/img/kg_count.png`: Q-A, đủ 7 loại node.
- `report/img/kg_cross_kb.png`: Q-B, thấy đường `Person → Case → Crime ← Article` và Results overview.
- `report/img/kg_my_case.png`: Q-D với **Cái Quang Huy** (không dùng Lê Minh Thành), thấy cả `Substance` và `Location`.
