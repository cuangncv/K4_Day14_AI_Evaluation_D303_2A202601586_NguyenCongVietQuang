# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Answer đúng nhưng diễn đạt lại bằng từ khác context (paraphrase), bị heuristic overlap phạt oan | Answer nêu số ngày, số tiền hoặc điều kiện bảo hành không có trong corpus, khiến khách hành động sai theo lời bot | Critical: siết prompt "chỉ dùng context", bắt trích `source_doc`, block deploy |
| Answer Relevance | Question mơ hồ hoặc rộng; answer đúng ý nhưng dùng từ vựng khác question | Answer trả lời sang chủ đề khác (hỏi đổi trả, đáp bảo hành), tức là sai intent | Kiểm tra intent routing và query rewriting; làm rõ nhiệm vụ trong prompt |
| Context Recall | Case adversarial hoặc out of scope: đúng ra không tồn tại evidence để lấy, recall thấp là hợp lý | Case Easy hoặc Medium mà recall thấp, tức evidence có sẵn trong corpus nhưng retriever bỏ sót | Tăng `top-k`, sửa chunking, thêm synonym và alias cho từ khoá domain |
| Context Precision | Chunk đúng vẫn nằm trong nhóm đầu, chỉ lẫn vài chunk thừa phía sau; answer không bị ảnh hưởng | Chunk đúng bị đẩy xuống cuối, generator đọc phải noise trước nên trả lời sai hoặc thiếu | Rerank kết quả, giảm `top-k`, cải thiện scoring function |
| Completeness | Expected answer dài dòng, answer súc tích nhưng đã đủ ý chính | Thiếu điều kiện, ngoại lệ hoặc mốc thời gian (nói "24 tháng" mà bỏ "tính từ confirmed delivery") | Xem Context Recall trước: nếu recall thấp thì sửa retriever, nếu recall cao thì bắt prompt liệt kê đủ conditions và exceptions |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Lấy N cặp answer (A, B) cho cùng một question. Condition 1 đưa
> judge theo thứ tự A rồi B. Condition 2 dùng đúng cặp đó nhưng đảo thành B rồi
> A. Giữ nguyên judge model, rubric và temperature ở cả hai condition, biến duy
> nhất được đổi là vị trí.
>
> Đo hai con số. Thứ nhất là tỉ lệ thắng của vị trí đứng đầu trong mỗi condition,
> không có bias thì phải xấp xỉ 50 phần trăm. Thứ hai là tỉ lệ cặp mà judge đổi
> kết luận khi đảo thứ tự. Nếu vị trí đầu thắng lệch hẳn ở cả hai condition, hoặc
> tỉ lệ đổi kết luận cao, judge đang chấm theo vị trí chứ không theo chất lượng.
> Khắc phục bằng cách randomize thứ tự, hoặc chấm cả hai chiều rồi lấy trung bình.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Chấm theo checklist claim bắt buộc, đánh dấu có hoặc không cho
> từng ý, thay vì chấm cảm tính "mức độ chi tiết". Như vậy answer dài không tự
> động ăn điểm, vì điểm gắn với số ý đúng chứ không gắn với độ dài.
>
> Ba biện pháp cụ thể. Một là ghi thẳng vào rubric rằng thông tin thừa, không
> được hỏi thì bị trừ điểm. Hai là tách thành nhiều dimension (Correctness,
> Completeness, Relevance) thay vì một điểm tổng, để độ dài không kéo theo mọi
> dimension. Ba là kèm ví dụ mẫu một answer ngắn được 5 điểm để neo chuẩn cho
> judge.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Judge cũng chỉ là một model. Nó có bias riêng (position,
> verbosity, self preference) và điểm sẽ trôi khi đổi model hoặc đổi version. Nếu
> không có human label đối chiếu, ta không biết "4 điểm" của judge tương ứng với
> chất lượng thật nào, nên con số đẹp mà vô nghĩa.
>
> Cách làm là gán nhãn tay một tập nhỏ khoảng 30 tới 50 case, đo mức đồng thuận
> giữa judge và người bằng Cohen's kappa hoặc tỉ lệ khớp, soi các case lệch để
> sửa lại rubric, rồi lặp đến khi mức đồng thuận đủ cao. Chỉ khi đó mới được dùng
> điểm judge làm quality gate, vì lúc đó ta đã biết sai số của thước đo là bao
> nhiêu.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Nghiêm nhất trong ba. Bot bịa chính sách bảo hành hoặc hoàn tiền khiến khách hành động sai và shop phải chịu trách nhiệm. Đây là thiệt hại trực tiếp, không sửa được bằng lời xin lỗi |
| Answer Relevance | 0.70 | Lạc đề làm khách phải hỏi lại, tốn trải nghiệm nhưng không gây cam kết sai. Để thấp hơn vì heuristic đếm từ phạt oan answer paraphrase |
| Completeness | 0.70 | Thiếu điều kiện hoặc ngoại lệ là rủi ro thật, nhưng answer súc tích đúng ý vẫn bị metric trừ điểm, nên không đặt ngang Faithfulness |

Ngoài ba ngưỡng trung bình trên, thêm hai luật chặn. Một là không case adversarial
nào được fail, tính cả A01, A02 và A03. Hai là nếu regression vượt 0.05 so với
baseline thì block, kể cả khi điểm tuyệt đối vẫn đạt ngưỡng.

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Offline evaluation chạy trên golden dataset mỗi lần đổi prompt,
> đổi model hoặc chỉnh retriever, thực hiện trước khi merge. Vì nó rẻ, nhanh và
> lặp lại được nên hợp làm CI gate. Nhược điểm là chỉ đo được 20 case cố định,
> không phản ánh traffic thật.
>
> Online evaluation chạy liên tục trên request thật sau khi deploy, theo dõi tỉ lệ
> escalation, câu hỏi lặp lại và phản hồi tiêu cực của khách. Nó bắt được những gì
> golden dataset không lường trước như câu hỏi mới, sản phẩm mới, chính sách vừa
> đổi. Bù lại không có đáp án chuẩn nên chỉ đo được gián tiếp.
>
> Human review dùng cho case rủi ro cao như từ chối bảo hành, hoàn tiền, dữ liệu
> cá nhân, dùng cho mẫu ngẫu nhiên định kỳ, và dùng để calibrate LLM judge như đã
> nói ở Ex 1.2 Câu 3. Cách này đắt và chậm nên không thể chạy mọi request, nhưng
> đây là nguồn chuẩn duy nhất để biết hai loại trên có đang đo đúng hay không.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
