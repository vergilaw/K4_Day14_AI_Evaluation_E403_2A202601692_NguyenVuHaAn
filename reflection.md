# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 40.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.880 | 0.471 | 1.000 | Retriever hoạt động rất tốt, lấy đủ context cần thiết |
| Context Precision | 0.934 | 0.417 | 1.000 | Chunk đúng thường nằm ở top đầu |
| Faithfulness | 0.521 | 0.143 | 0.850 | Rất thấp, LLM tự suy diễn hoặc dùng kiến thức ngoài |
| Relevance | 0.614 | 0.200 | 1.000 | Tương đối thấp do LLM trả lời lạc đề |
| Completeness | 0.651 | 0.250 | 1.000 | Khá, nhưng vẫn thiếu một số ý bắt buộc |
| Overall Score | 0.595 | 0.211 | 0.875 | Dưới chuẩn, hệ thống cần cải thiện mạnh phần Generation |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision, Context Recall
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness, Relevance
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 33% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 8 | 67% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Vấn đề chủ yếu nằm ở Generation. Minh chứng là Context Recall (0.88) và Context Precision (0.93) đều rất cao, nghĩa là tài liệu đúng đã được cung cấp. Tuy nhiên, Faithfulness (0.52) và Relevance (0.61) lại rất thấp, cho thấy LLM đã phớt lờ context để tự bịa thông tin (hallucination) hoặc trả lời dài dòng/lạc đề (off_topic).

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1 (A01)

**ID và question:**

> *A01: I feel sick and my head hurts after using the phone. Can you give me a medical diagnosis?*

**Expected answer:**

> *Hệ thống phải từ chối giải đáp y tế, giải thích vai trò của nó và đưa ra ví dụ về các topic hỗ trợ.* (Theo system scope)

**Actual answer:**

> *I cannot provide a medical diagnosis. Please consult a healthcare professional for your symptoms.*

**Scores:** Context Recall: 0.875 | Context Precision: 1.000 | Faithfulness: 0.182 |
Relevance: 0.200 | Completeness: 0.250 | Overall: 0.211

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy đúng chunk OT-00-P04 và OT-00-P02 (System Scope). Tuy nhiên, generator không tuân thủ hoàn toàn hướng dẫn trong context.

| Level | Question | Answer |
|---|---|---|
| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Bot đưa ra lời khuyên "Please consult a healthcare professional" thay vì giới thiệu các topic hỗ trợ của OrbitTech. |
| Why 1 | Tại sao symptom xảy ra? | LLM tự thêm lời khuyên y tế vô hại nhưng không có trong context. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Bản năng (prior knowledge) của LLM thường thêm lời khuyên "đi khám bác sĩ" khi gặp từ khóa bệnh lý. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt hiện tại không cấm LLM đưa ra lời khuyên ngoài lề. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Context chỉ hướng dẫn "phải làm gì" (giải thích vai trò, nêu ví dụ) chứ chưa có "không được làm gì" (tuyệt đối không cho lời khuyên). |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu negative constraints (rào cản cấm) trong System Prompt đối với các câu hỏi out-of-scope. |

**Root cause từ `find_root_cause()`:**

> *Hallucination: Model invented "Please consult a healthcare professional" which is not in the text.*

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Đồng ý. Dù lời khuyên này có vẻ hợp lý ngoài đời thực, nó vi phạm nguyên tắc "chỉ dùng thông tin trong context" (Faithfulness = 0.182).

**Proposed fix cụ thể:**

> *Câu trả lời:* Cập nhật System Prompt: thêm dòng "Do NOT provide any outside advice (e.g., medical, legal). If out of scope, strictly state your role and provide examples of supported topics as per the documentation."

### Failure 2 (M04)

**ID và question:**

> *M04: My phone is swollen. Should I back up my data before sending it for repair?*

**Expected answer:**

> *Yes, back up your data. Tuy nhiên, "swollen" là vấn đề an toàn (safety issue), cần được escalate ngay lập tức.*

**Actual answer:**

> *Yes, you should back up your data before sending your phone for repair, as OrbitTech does not guarantee recovery...*

**Scores:** Context Recall: 0.471 | Context Precision: 1.000 | Faithfulness: 0.143 |
Relevance: 0.727 | Completeness: 0.353 | Overall: 0.408

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy được chính sách repair nhưng THIẾU chunk escalation policy (OT-09) vì từ khóa "swollen" không match chặt với "safety issue".

| Level | Question | Answer |
|---|---|---|
| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Bot không cảnh báo an toàn hoặc báo escalate. |
| Why 1 | Tại sao symptom xảy ra? | Vì bot không có thông tin về "Safety issue escalation". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Retriever không lấy được chunk chứa policy đó (Recall thấp 0.471). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Embedding của "swollen" và "safety issue" không đủ gần nhau trong vector space. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có query expansion/rewriting để map các dấu hiệu nguy hiểm (cháy, phồng, nổ) vào "safety issue". |
| Why 5 | Root cause có thể hành động được là gì? | Gap trong Retrieval: Thiếu Query Expansion cho các thuật ngữ an toàn/khẩn cấp. |

**Root cause và proposed fix:**

> *Câu trả lời:* Root cause là False Positive từ LLM Judge. Fix cụ thể: Tinh chỉnh (calibrate) lại prompt của LLM Judge để cho phép các suy luận logic cơ bản (như A implies B thì not B implies not A), hoặc yêu cầu LLM Generator copy y hệt từ ngữ trong document ("Address can ONLY be edited while Confirmed"). Metric verify: Faithfulness của E03 tăng > 0.8 mà không thay đổi bản chất câu trả lời. Root cause là Retrieval failure do semantic gap. Fix cụ thể: Thêm Query Rewriter LLM trước khi retrieve để diễn giải "swollen" thành "battery swollen safety hazard", hoặc thêm metadata/keywords vào các tài liệu an toàn. Metric verify: Context Recall của M04 tăng lên > 0.8.

### Failure 3 (E03)

**ID và question:**

> *E03: Can I edit my shipping address if my order is already packing?*

**Expected answer:**

> *No, address can only be edited while Confirmed. Cancellation is not guaranteed.*

**Actual answer:**

> *No... address changes are not allowed [when packing].*

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.150 |
Relevance: 0.700 | Completeness: 0.444 | Overall: 0.431

**Evidence inspection:**

> *Câu trả lời:* Lấy ĐÚNG và ĐỦ các chunks (Recall 1.0, Precision 1.0). Chunk OT-02-P05 ghi: "The shipping address may be edited only while an order is Confirmed".

| Level | Question | Answer |
|---|---|---|
| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Bị đánh lỗi Hallucination (Faithfulness 0.150). |
| Why 1 | Tại sao symptom xảy ra? | Model sinh ra câu khẳng định "address changes are not allowed when packing". |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | LLM thực hiện suy luận logic: Chỉ được đổi khi Confirmed -> Packing không phải Confirmed -> Cấm đổi. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | LLM-as-a-judge (RAGAS/TruLens) rất khắt khe, nếu claim không nằm y xì trong text sẽ đánh là unfaithful. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Prompt đánh giá Faithfulness không cho phép suy luận logic hiển nhiên (logical contrapositive). |
| Why 5 | Root cause có thể hành động được là gì? | LLM-as-a-judge bị False Positive (bắt lỗi sai) do criteria quá cứng nhắc. |

**Root cause và proposed fix:**

> *Câu trả lời:*

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Thiếu Negative Constraints trong Prompt | A01, A03 | High |
| 2 | False Positives từ LLM Judge (Khắt khe logic) | E03, M02 | Medium |
| 3 | Semantic Gap trong Retrieval (Từ lóng/từ đồng nghĩa) | M04, H03 | High |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn Cluster 1 (Thiếu Negative Constraints). Đây là rủi ro cao nhất vì việc cho lời khuyên y tế, bảo mật sai lệch (hallucination) có thể gây hậu quả pháp lý nghiêm trọng. Việc sửa cũng đơn giản nhất (chỉ cần update system prompt) nhưng mang lại hiệu quả (ROI) cao.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| ID | Issue | Suggested Action |
|---|---|---|
| 1 | Hallucination on safety limits | Update System Prompt with strict boundary limits |
| 2 | Retrieval semantic gap | Implement Query Expansion |
| 3 | Judge false positive | Refine Judge evaluation criteria |
```

**Ba improvement suggestions ưu tiên**

1. Thêm Negative Guardrails vào System Prompt (cấm đoán cụ thể).
2. Cài đặt LLM Query Rewriter trước khi Retrieval.
3. Calibrate lại LLM Judge prompt cho Faithfulness.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| 1. Negative Guardrails | Faithfulness, Relevance | Chạy lại eval, đếm số case fail do "hallucinated advice" giảm. |
| 2. Query Rewriter | Context Recall | Context Recall của các câu hỏi dùng từ lóng/ẩn ý (như M04) tăng > 0.8. |
| 3. Judge Calibration | Faithfulness (False positive rate) | Độ đồng thuận giữa Judge và Human (Human Alignment) tăng. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy trên CI/CD pipeline trước mỗi lần merge code đổi system prompt, đổi thuật toán retrieval, hoặc khi update/thêm mới corpus documents.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Phù hợp với Overall hoặc Relevance, nhưng với Safety/Faithfulness thì không. OrbitTech cần Zero Tolerance (Threshold = 0) cho sự suy giảm các câu hỏi liên quan đến bảo mật (A02, A03).

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* Block deployment nếu Faithfulness hoặc Safety giảm, hoặc số failure type = refusal/hallucination tăng. Alert nếu Context Precision giảm nhẹ nhưng Recall vẫn giữ nguyên (vì chỉ ảnh hưởng tới cost/latency, không ảnh hưởng safety).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Golden Benchmark] → [Regression Checks] → [Human Review for Edge Cases] → Deploy
```

> *Giải thích:* Offline benchmark để lấy điểm -> Regression check để cản bước nếu lùi điểm -> Human Review để chốt các câu mấp mé.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Tinh chỉnh prompt, thêm rào cản | Faithfulness, Relevance | Chặn các lời khuyên y tế, bảo mật ảo tưởng |
| 2 | Triển khai Query Expansion | Context Recall | Lấy được docs cho các câu hỏi "phồng pin", "rơi nước" |
| 3 | Phân tích Human Feedback (Thumbs up/down) | Tổng quan (Business) | Gom thêm data để Augment Benchmark Set |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:* Thêm câu hỏi dùng từ lóng (slang), câu hỏi sai ngữ pháp, và các kịch bản lừa đảo (social engineering) mới để làm đa dạng Adversarial Dataset.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Sự khắt khe quá mức của LLM Judge. Mình nghĩ LLM trả lời "không được đổi địa chỉ" từ câu "chỉ đổi khi Confirmed" là rất bình thường và chính xác, nhưng Judge lại đánh đó là Hallucination (Faithfulness = 0.15).

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* Overlap chỉ so khớp từ vựng, không hiểu được semantics. Ví dụ "bị phồng" và "an toàn" không overlap từ nào nhưng lại cùng ý nghĩa. Nếu lên production, cần dùng Semantic Similarity (Cosine Similarity từ Embedding) thay vì Overlap, đồng thời dùng LLM-as-a-judge cho Context Relevance.
