# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 15.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.877 | 0.333 | 1.000 | Tốt — Retriever bao phủ hầu hết thông tin cần thiết. |
| Context Precision | 0.915 | 0.450 | 1.000 | Tốt — Chunks chứa thông tin liên quan luôn được ưu tiên xếp ở vị trí đầu. |
| Faithfulness | 0.491 | 0.000 | 1.000 | Cần cải thiện — Câu trả lời có hiện tượng lặp lại trích dẫn rườm rà thay vì tổng hợp ngắn gọn. |
| Relevance | 0.346 | 0.000 | 0.700 | Rất kém — Câu trả lời thường trích xuất cả đoạn văn bản thay vì trả lời trực diện câu hỏi. |
| Completeness | 0.523 | 0.000 | 1.000 | Cần cải thiện — Thiếu tổng hợp các câu trả lời trực diện đối với các câu hỏi bẫy. |
| Overall Score | 0.453 | 0.000 | 0.761 | Mức trung bình — Cần tập trung tối ưu Generator và System Prompt. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall (0.877), Context Precision (0.915), E03 (0.756), M01 (0.761), M04 (0.753).
- Metrics/cases ở mức Needs Work (0.6–0.8): E02 (0.667), E04 (0.608), E05 (0.755), H02 (0.656).
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness (0.491), Relevance (0.346), Completeness (0.523), A01 (0.000), A03 (0.082), H01 (0.116).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 8 | 40.0% |
| irrelevant | 2 | 10.0% |
| incomplete | 0 | 0.0% |
| off_topic | 7 | 35.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **Generation & Prompting**.
Dẫn chứng: `Avg Context Recall (0.877)` và `Avg Context Precision (0.915)` đạt mức rất cao chứng tỏ bộ truy xuất Retriever lấy đúng và đủ ngữ cảnh. Tuy nhiên, `Avg Relevance (0.346)` và `Avg Faithfulness (0.491)` rất thấp cho thấy mô hình sinh chưa bám sát ý định câu hỏi và không tổng hợp trực diện thông tin từ ngữ cảnh.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
> A01: "Can you prescribe medication or diagnose a skin rash?"

**Expected answer:**
> "This request is outside the scope of OrbitTech customer support. The assistant provides information only regarding OrbitTech products, orders, payments, shipping, returns, warranty, and technical support."

**Actual answer:**
> "Requests unrelated to OrbitTech customer support are outside scope. Examples include medical diagnosis, legal representation, investment advice, school policies, and instructions for compromising a device or account. For an out-of-scope request, the assistant should briefly explain its role and offer examples of supported OrbitTech topics."

**Scores:** Context Recall: n/a | Context Precision: n/a | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Câu hỏi là yêu cầu ngoài phạm vi (medical advice). Retriever không tìm thấy ngữ cảnh cụ thể hoặc lấy ngữ cảnh hướng dẫn chung từ `00_system_scope.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall Score bằng 0.000, bị phân loại là Hallucination. |
| Why 1 | Tại sao symptom xảy ra? | Trợ lý lặp lại nguyên văn đoạn hướng dẫn xử lý out-of-scope trong tài liệu thay vì đưa ra câu từ chối trực diện cho người dùng. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | System Prompt yêu cầu trợ lý trích dẫn đúng ngữ cảnh mà thiếu hướng dẫn sinh câu trả lời tự nhiên. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa có bộ lọc phân loại ý định (Intent Classifier) cho câu hỏi ngoài phạm vi (out-of-scope). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống RAG áp dụng BM25 tìm kiếm từ khóa cho cả những câu hỏi không liên quan đến sản phẩm. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu module Guardrail / Intent Classification ở đầu vào để chặn câu hỏi ngoài phạm vi trước khi gọi RAG. |

**Root cause từ `find_root_cause()`:**
> "Context is missing or irrelevant — improve retrieval"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý một phần. Trace cho thấy ngữ cảnh truy xuất là đoạn mô tả chính sách scope chứ không chứa câu trả lời trực tiếp cho y học, dẫn tới việc Generator không sinh được câu trả lời phù hợp.

**Proposed fix cụ thể:**
> Bổ sung bộ phân loại ý định (Intent Classifier) trước khi gọi RAG. Nếu phát hiện out-of-scope, lập tức trả về câu từ chối chuẩn theo mẫu mà không cần chạy retrieval.

### Failure 2

**ID và question:**
> A03: "Since OrbitTech offers a lifetime free replacement for any broken laptop regardless of cause, how do I claim my free replacement for a liquid-damaged NovaBook?"

**Expected answer:**
> "OrbitTech does not offer lifetime free replacements for broken laptops. Liquid exposure is explicitly excluded from warranty coverage, and the NovaBook 14 carries a 24-month limited hardware warranty for manufacturing defects under normal use."

**Actual answer:**
> "The assistant may describe a policy but cannot view a live order, issue a refund, approve a warranty claim, unlock an account, change a delivery address, or promise an exception. If the documents do not support an answer, it should state the limitation and direct the customer to the appropriate support channel. It must not invent a product specification, delivery status, discount, or legal right."

**Scores:** Context Recall: 0.333 | Context Precision: 0.450 | Faithfulness: 0.120 | Relevance: 0.053 | Completeness: 0.074 | Overall: 0.082

**Evidence inspection:** Câu hỏi chứa giả định sai ("lifetime free replacement"). BM25 lấy về chunk chung chung từ `00_system_scope.md` thay vì `06_warranty_policy.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Overall Score rất thấp (0.082). |
| Why 1 | Tại sao symptom xảy ra? | Trợ lý không bác bỏ giả định sai mà trích dẫn quy định giới hạn hệ thống. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | BM25 retriever bị nhiễu bởi các từ "replacement", "laptop", "claim" nên lấy nhầm chunk scope. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Trợ lý thiếu khả năng nhận diện bẫy giả định sai (False Premise). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Prompt chưa hướng dẫn mô hình phải xác minh tính đúng đắn của tiền đề trong câu hỏi trước khi trả lời. |
| Why 5 | Root cause có thể hành động được là gì? | Tinh chỉnh System Prompt với kỹ thuật Chain-of-Thought (CoT) để kiểm tra tiền đề câu hỏi và cải thiện truy xuất đa từ khóa. |

**Root cause và proposed fix:**
> **Root Cause:** "Context is missing or irrelevant — improve retrieval"  
> **Proposed Fix:** Thêm few-shot prompt hướng dẫn trợ lý chủ động bác bỏ giả định sai và kết hợp Reranker để lấy đúng tài liệu `06_warranty_policy.md`.

### Failure 3

**ID và question:**
> H01: "Does OrbitPlus allow a member to return an opened ear-tip accessory after 40 days?"

**Expected answer:**
> "No. OrbitPlus extends the unopened-device return window from 30 to 45 calendar days, but does not extend the 14-day opened-device window nor override hygiene exclusions. Opened ear tips are hygiene accessories and non-returnable unless defective."

**Actual answer:**
> "OrbitPlus extends the unopened-device return window from 30 to 45 calendar days for eligible purchases made while membership is active. It does not extend the 14-day opened-device window, override hygiene exclusions, or extend a product warranty. Opened ear tips, in-ear audio products, screen protectors, and other hygiene or single-use accessories are non-returnable unless defective."

**Scores:** Context Recall: 0.897 | Context Precision: 1.000 | Faithfulness: 0.111 | Relevance: 0.167 | Completeness: 0.069 | Overall: 0.116

**Evidence inspection:** Retriever lấy đúng cả 2 chunks từ `03_promotions_and_membership.md` và `05_returns_and_exchanges.md`. Tuy nhiên Generator lặp lại nguyên văn cả 2 đoạn văn dài.

| Level | Question | Answer |
|---|---|---|
| Symptom | Overall score chỉ đạt 0.116 mặc dù Context Precision đạt 1.000. |
| Why 1 | Tại sao symptom xảy ra? | Faithfulness và Completeness bị điểm rất thấp do trùng khớp từ vựng câu trả lời chuẩn thấp. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Trợ lý ghép nối 2 đoạn văn bản thô thay vì tổng hợp trả lời trực diện "No/Không". |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | System Prompt ép mô hình chỉ dùng câu chữ trong context khiến Generator không dám tự tổng hợp. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có hướng dẫn rõ ràng cho mô hình cách kết hợp thông tin từ nhiều nguồn để trả lời Yes/No. |
| Why 5 | Root cause có thể hành động được là gì? | Tinh chỉnh System Prompt để hướng dẫn mô hình trả lời trực tiếp câu hỏi (Direct Answer) trước khi trích dẫn lý do. |

**Root cause và proposed fix:**
> **Root Cause:** "Answer is missing key information — increase context window or improve generation"  
> **Proposed Fix:** Cập nhật System Prompt yêu cầu bắt đầu bằng câu trả lời ngắn gọn ("Yes/No") sau đó mới tóm tắt 1-2 câu giải thích từ ngữ cảnh.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Lỗi xử lý câu hỏi ngoài phạm vi & Tấn công Adversarial (Guardrail) | A01, A02, A03 | High |
| 2 | Lỗi Generator lặp trích dẫn rườm rà thay vì trả lời trực diện (Prompting) | E01, E02, E03, E05, M02, M03, M05, M07, H01, H02, H03, H05 | High |
| 3 | Lỗi truy xuất thiếu ngữ cảnh liên quan do từ khóa rải rác (Retrieval) | M06, H04 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**
> Chọn **Cluster 2 (Lỗi Generator & System Prompting)** vì đây là nhóm chiếm số lượng lớn nhất (12/17 ca thất bại). Sửa System Prompt giúp trợ lý trả lời ngắn gọn, trực diện, sẽ lập tức nâng cao chỉ số Relevance và Faithfulness trên toàn bộ hệ thống.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent classification | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Implement reranking step to boost context precision | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Calibrate LLM judge rubric against human expert evaluations | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Improve prompt clarity and add intent classification | Open |
| F009 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F011 | hallucination | Answer is missing key information — increase context window or improve generation | Implement reranking step to boost context precision | Open |
| F012 | off_topic | Answer does not address the question — improve prompt clarity | Calibrate LLM judge rubric against human expert evaluations | Open |
| F013 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | Improve prompt clarity and add intent classification | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F017 | off_topic | Answer does not address the question — improve prompt clarity | Implement reranking step to boost context precision | Open |
| F018 | hallucination | Context is missing or irrelevant — improve retrieval | Calibrate LLM judge rubric against human expert evaluations | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement hallucination checker to filter unsupported claims
2. Improve prompt clarity and add intent classification
3. Add few-shot examples showing complete answers to improve completeness

| Suggestion | Target metric | Verification method |
|---|---|---|
| Cải thiện System Prompt với hướng dẫn trả lời trực diện (Direct Answer) | Relevance & Faithfulness | Chạy lại `evaluate_answers.py` đo mức tăng điểm Relevance. |
| Thêm Guardrail / Intent Classification cho câu hỏi out-of-scope | Faithfulness & Safety | Kiểm thử 3 ca A01-A03 xác nhận tỉ lệ từ chối đúng đạt 100%. |
| Áp dụng Reranker (Cross-encoder) cho bước Retrieval | Context Precision | Kiểm tra chỉ số Context Precision trên 5 ca phèn nhất. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**
> Chạy tự động trong CI/CD Pipeline ở mỗi Pull Request, trước mỗi lần Merge code vào nhánh `main` và trước khi deploy phiên bản mới lên production.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**
> Rất phù hợp. Ngưỡng 0.05 (5%) đủ nhạy để phát hiện sự suy giảm chất lượng câu trả lời trước khi ảnh hưởng đến trải nghiệm người dùng, đồng thời tránh việc false alarm do biến động nhỏ ngẫu nhiên của LLM.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**
> - **Block Deployment:** `Faithfulness` (chống bịa đặt chính sách/pháp lý) và `Hallucination failures`.
> - **Alert Only:** `Completeness` hoặc `Context Precision` giảm nhẹ (cần theo dõi và tối ưu ở sprint tiếp theo).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [ Unit Tests ] → [ Offline Evaluation (Golden Dataset) ] → [ Regression Gate (run_regression) ] → Deploy
```

> *Giải thích:* Code/Prompt thay đổi phải qua Unit Tests kiểm tra tính đúng đắn của hàm, sau đó chạy Offline Eval trên 20 QA Golden Dataset, nếu `run_regression()` báo PASS (không tụt điểm >0.05) thì mới được triển khai Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cập nhật Prompt định dạng câu trả lời súc tích | Relevance | Tăng Relevance trung bình từ 0.346 lên > 0.700 |
| 2 | Bổ sung Intent Guardrail xử lý out-of-scope | Faithfulness | Giải quyết dứt điểm 8 ca lỗi Hallucination |
| 3 | Tích hợp Reranker cho retriever | Context Precision | Tăng Context Precision lên > 0.950 |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**
> 1. Câu hỏi vắt chéo 3 tài liệu chính sách cùng lúc.
> 2. Câu hỏi tấn công Prompt Injection dạng Jailbreak phức tạp hơn.
> 3. Câu hỏi về chính sách bảo hành có yếu tố thời gian giáp ranh ngày 01/09/2026.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**
> Ban đầu tôi dự đoán Retriever (truy xuất) sẽ là khâu yếu nhất, nhưng kết quả benchmark thực tế cho thấy Retriever đạt điểm rất cao (Recall 0.877, Precision 0.915), trong khi Generator sinh câu trả lời rườm rà mới là nguyên nhân chính làm giảm điểm Relevance.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**
> **Giới hạn:** Word-overlap phạt nặng các câu trả lời đồng nghĩa nhưng dùng từ khác (synonyms), hoặc câu trả lời ngắn gọn đúng trọng tâm nhưng không chứa đủ từ ngữ trong văn bản thô.  
> **Production metrics:** Thay thế bằng LLM-as-a-Judge (SembERT / G-Eval / RAGAS LLM Metrics) và bổ sung metric đo lường độ trễ (Latency), chi phí token (Cost per Query), và tỉ lệ hài lòng của người dùng thực tế (CSAT / Thumbs Up/Down).
