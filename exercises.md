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
| Faithfulness | 0.6-0.8 có thể tạm chấp nhận với câu trả lời sáng tạo, small talk hoặc nội dung được gắn nhãn rõ là gợi ý, không phải sự thật. | Dưới 0.8 với câu trả lời về chính sách, thanh toán, bảo hành, quyền riêng tư; đặc biệt dưới 0.6 vì answer có nhiều claim không được context hỗ trợ. | Kiểm tra từng claim với context; siết prompt chỉ được dùng evidence, yêu cầu trích nguồn hoặc từ chối khi thiếu evidence. |
| Answer Relevance | 0.6-0.8 có thể chấp nhận cho câu hỏi mơ hồ khi answer vẫn giải quyết một cách hiểu hợp lý hoặc hỏi lại để làm rõ. | Dưới 0.6 với câu hỏi rõ ràng, hoặc answer lạc đề và không giúp hoàn thành intent của khách hàng. | Phân tích intent, cải thiện prompt/query routing và thêm test cho câu hỏi mơ hồ, nhiều ý. |
| Context Recall | 0.6-0.8 có thể tạm chấp nhận khi câu hỏi chỉ cần một phần evidence, answer là lời từ chối out-of-scope, hoặc phần thiếu không ảnh hưởng kết luận. | Dưới 0.6 cho câu hỏi factual/multi-document khi các điều kiện hay ngoại lệ bắt buộc không được retrieve. | Cải thiện query rewriting, chunking, embedding và `top_k`; bổ sung tài liệu/evidence bị thiếu rồi chạy lại retrieval eval. |
| Context Precision | 0.6-0.8 có thể chấp nhận nếu Recall cao, evidence đúng vẫn đứng sớm và generator chịu được một ít chunk dư. | Dưới 0.6 khi phần lớn top results là noise hoặc evidence đúng nằm quá thấp, làm tăng chi phí và nguy cơ trả lời sai. | Điều chỉnh ranking, metadata filter và `top_k`; thêm reranker và kiểm tra Precision@K theo từng nhóm câu hỏi. |
| Completeness | 0.6-0.8 có thể chấp nhận với câu trả lời chủ ý ngắn gọn, câu hỏi follow-up, hoặc khi phần thiếu chỉ là chi tiết tùy chọn. | Dưới 0.6 khi bỏ sót bước, điều kiện, ngoại lệ hoặc cảnh báo bắt buộc khiến người dùng có thể hành động sai. | Tách expected answer thành các ý bắt buộc; yêu cầu generator kiểm tra đủ ý và bổ sung regression cases cho các ý hay bị bỏ sót. |

**Chẩn đoán theo cặp metric:** Context Recall đo retriever có lấy đủ evidence cần thiết
hay không, còn Completeness đo answer có bao phủ đủ các ý trong expected answer hay không.
Vì vậy, Recall thấp đi cùng Completeness thấp thường cho thấy evidence đã bị bỏ sót
trước khi generator có cơ hội sử dụng, nên retriever là nghi phạm chính. Ngược lại, nếu
Recall và Precision đều tốt nhưng Faithfulness thấp, context cần thiết đã có và ít noise;
các claim không được context hỗ trợ nhiều khả năng do generator tự thêm, suy diễn quá mức
hoặc không tuân thủ grounding prompt.

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Chuẩn bị nhiều cặp answer A/B đã có nhãn chất lượng từ người đánh giá
> con người. Với mỗi cặp, giữ nguyên question, rubric và nội dung, chỉ thay thứ tự:
> Condition 1 hiển thị A trước B; Condition 2 hiển thị B trước A. Phân bổ ngẫu nhiên các
> cặp vào hai condition, ẩn tên model, dùng cùng judge và cấu hình ổn định; có thể lặp lại
> để ước lượng độ biến thiên. So sánh tỷ lệ answer được chọn trước và sau khi đảo vị trí.
> Nếu lựa chọn đổi theo vị trí một cách có ý nghĩa (ví dụ A thắng khi đứng đầu nhưng thua
> khi đứng sau), trong khi nhãn human không đổi, đó là bằng chứng position bias. Báo cáo
> thêm agreement với human ở từng condition để tránh nhầm bias với chất lượng thật.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Rubric phải chấm theo các claim/ý bắt buộc có evidence thay vì theo độ
> dài; nêu rõ câu trả lời ngắn nhưng đủ ý được điểm tối đa. Không cộng điểm cho việc nhắc
> lại câu hỏi, diễn giải trùng lặp hay thêm chi tiết không cần thiết, và trừ điểm cho claim
> không được hỗ trợ hoặc nội dung làm loãng câu trả lời. Tách điểm accuracy, completeness,
> relevance và conciseness với mô tả cụ thể cho từng mức để judge không dùng độ dài làm
> tín hiệu thay thế cho chất lượng.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels là mốc tham chiếu để đo judge có thực sự phản ánh tiêu chuẩn
> chất lượng và rủi ro của domain hay không. So sánh judge với một tập được nhiều người
> gán nhãn giúp phát hiện bias, xu hướng quá dễ/quá nghiêm và các loại lỗi judge thường bỏ
> qua; từ đó chỉnh rubric, prompt và threshold. Nếu không calibrate, CI/CD có thể cho qua
> lỗi nguy hiểm hoặc chặn release tốt chỉ vì score của judge có vẻ nhất quán nhưng sai lệch.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | >= 0.85 | Đây là guardrail cao nhất: claim không grounded có thể tạo thông tin sai về chính sách hoặc giao dịch. Bất kỳ case an toàn/chính sách nào dưới ngưỡng cũng block dù average đạt. |
| Answer Relevance | >= 0.80 | Answer phải giải quyết đúng intent; mức này vẫn cho phép một ít biến thiên diễn đạt nhưng chặn xu hướng lạc đề. |
| Completeness | >= 0.80 | Bảo đảm phần lớn điều kiện, bước và ngoại lệ bắt buộc được nêu; các ý critical bị thiếu phải block theo per-case rule. |

Deployment chỉ được qua khi cả ba aggregate metric đạt ngưỡng, không có regression đáng
kể so với baseline và không có critical case thất bại. Ngưỡng cần được hiệu chỉnh bằng
golden set và human labels; không hạ ngưỡng chỉ để làm pipeline xanh.

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Dùng **offline evaluation** trước merge/release và mỗi khi đổi model,
> prompt, retriever hoặc corpus; chạy trên golden dataset để so sánh có lặp lại với
> baseline và chặn regression. Dùng **online evaluation** sau deployment trên traffic thật
> để theo dõi drift, latency, cost, feedback và các intent mà golden set chưa bao phủ; nên
> lấy mẫu, ẩn danh dữ liệu và có alert/rollback. Dùng **human review** để tạo và calibrate
> nhãn chuẩn, xử lý case bất đồng hoặc score sát ngưỡng, và duyệt các tình huống high-stakes
> như thanh toán, privacy, security hay policy exception. Ba hình thức bổ sung cho nhau:
> offline là gate trước release, online phát hiện vấn đề thực tế, còn human review kiểm
> chứng các quyết định mà metric tự động chưa đủ tin cậy.

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
