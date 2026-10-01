# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 5.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.858 | 0.258 | 1.000 | Khá cao, 6/20 case đạt 1.000. Vẫn có case tụt xuống 0.258 nên chưa phải là ổn hết. |
| Context Precision | 0.955 | 0.700 | 1.000 | Tốt. Chunk đúng gần như luôn nằm ở top 1 hoặc top 2. |
| Faithfulness | 0.350 | 0.000 | 1.000 | Thấp. Nhiều câu trả lời bị cắt giữa chừng nên gần như không còn claim nào để đối chiếu với context. |
| Relevance | 0.208 | 0.000 | 0.500 | Thấp. Câu trả lời cụt nên hầu như không trùng từ khóa với câu hỏi. |
| Completeness | 0.115 | 0.000 | 0.500 | Thấp nhất. Expected answer dài và nhiều điều kiện, còn answer thật chỉ có vài từ. |
| Overall Score | 0.224 | 0.000 | 0.625 | Bị kéo xuống chủ yếu bởi ba metric phía generation. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 0 cases (ngưỡng pass 0.6)
- Metrics/cases ở mức Needs Work (0.6–0.8): 1 case (M07: 0.625, PASSED)
- Metrics/cases ở mức Significant Issues (<0.6): 19 cases (95.0%)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 10 | 50.0% |
| irrelevant | 5 | 25.0% |
| incomplete | 4 | 20.0% |
| off_topic | 0 | 0.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Theo mình vấn đề nằm ở generation, retrieval thì ổn.
>
> Phía retrieval: Context Precision trung bình 0.955 và Context Recall 0.858. Nghĩa là BM25 (có title boost và repeat decay) đã đưa đúng tài liệu về cho model trong phần lớn các case, và chunk đúng thường đứng đầu.
>
> Phía generation thì ngược lại: Completeness chỉ 0.115, Relevance 0.208. Lúc đầu mình tưởng model trả lời sai, nhưng mở `actual_answers.json` ra thì thấy nhiều câu trả lời chỉ có vài từ, kiểu *"Based on the retrieved contexts, the"* rồi hết. Mình đặt `max_output_tokens=300`, mà model Gemini bản mới dùng một phần token cho thinking trước khi viết ra text, nên phần text thật bị cắt gần như ngay từ đầu.
>
> Cũng vì vậy mà nhãn `hallucination` ở 10 case không nên hiểu theo nghĩa đen. Model không bịa thông tin, chỉ là câu trả lời quá ngắn nên heuristic không tìm được claim nào khớp với context.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> *Điền:* M06 — "What are the conditions for an OrbitPlus member to receive a loaner device during a repair?"

**Expected answer:**

> *Điền:* An active OrbitPlus member can request a loaner device if their hardware repair takes longer than 48 hours, subject to loaner device availability.

**Actual answer:**

> *Điền:* "Based on the retrieved contexts, the"

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Retriever lấy đúng chunk `OT-07-P02` và xếp nó ở top 1. Chunk này có đủ cả ba điều kiện (active OrbitPlus, repair quá 48 giờ, còn máy cho mượn), nên retrieval không thiếu gì ở case này.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời dừng giữa chừng ("Based on the retrieved contexts, the"), cả ba điểm answer đều 0.000. |
| Why 1 | Tại sao symptom xảy ra? | Model ngừng sinh text sau đúng 6 từ. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Hết `max_output_tokens=300`, vì thinking token của Gemini cũng tính vào giới hạn này. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Mình hard-code 300 trong generator mà không biết model có dùng thinking budget. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Code không kiểm tra `finish_reason == 'MAX_TOKENS'`, cũng không kiểm tra độ dài tối thiểu trước khi lưu artifact. |
| Why 5 | Root cause có thể hành động được là gì? | Generation config cần tắt thinking (`thinking_budget=0`) hoặc nâng `max_output_tokens` lên khoảng 1024. |

**Root cause từ `find_root_cause()`:**

> *Paste output:* `Multiple issues detected — review full pipeline`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Mình chỉ đồng ý một phần. Kết luận "review full pipeline" quá chung chung và không sai, nhưng trace chỉ ra chỗ cụ thể hơn nhiều: Recall và Precision đều 1.0 nên retrieval không có lỗi, lỗi là câu trả lời bị cắt do hết token ở bước gọi API.

**Proposed fix cụ thể:**

> *Câu trả lời:* Trong `GeminiGenerator`, nâng `max_output_tokens` từ 300 lên 1024 và giữ `temperature=0.0`. Thêm một bước kiểm tra sau khi gọi API: nếu `finish_reason` là `MAX_TOKENS` hoặc câu trả lời quá ngắn thì retry với giới hạn lớn hơn, không lưu thẳng vào artifact như hiện tại.

---

### Failure 2

**ID và question:**

> *Điền:* H04 — "What are the prerequisites and timeline for filing a formal service complaint, and under what circumstances can a customer bypass routine support for immediate escalation?"

**Expected answer:**

> *Điền:* A formal service complaint requires an existing support ticket number that has been unresolved for more than five business days. Support acknowledges the complaint within one business day and provides a written resolution within ten business days. Customers may bypass routine support and escalate immediately for unaddressed safety issues, suspected unauthorized account changes, or unresolved billing discrepancies exceeding USD 100.

**Actual answer:**

> *Điền:* "_escalation_and_policy_updates."

**Scores:** Context Recall: 0.824 | Context Precision: 0.917 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**

> *Câu trả lời:* Retriever lấy được cả `OT-09-P01` (ticket chưa xử lý quá 5 ngày làm việc, xác nhận trong 1 ngày, giải quyết trong 10 ngày) và `OT-09-P02` (các trường hợp được bypass: an toàn, thay đổi tài khoản trái phép, chênh lệch billing trên 100 USD). Context đủ để trả lời.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Output chỉ là một mẩu tên file `_escalation_and_policy_updates.`, không có nội dung trả lời. |
| Why 1 | Tại sao symptom xảy ra? | Model mở đầu bằng việc nhắc lại tên tài liệu nguồn, rồi bị cắt trước khi viết tới phần nội dung. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt đưa context dưới dạng `[Context 1 | 09_escalation_and_policy_updates.md]`, model có xu hướng trích lại header này. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Prompt không nói rõ đâu là metadata nguồn, đâu là nội dung cần dùng để trả lời. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Heuristic chỉ đếm từ trùng. Chuỗi một từ thì ra 0 điểm, nhưng không có cảnh báo nào cho biết output bị hỏng. |
| Why 5 | Root cause có thể hành động được là gì? | Sửa prompt để cấm nhắc tên file, và tăng token limit để câu trả lời không bị cắt. |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - **Root cause:** Vẫn là hết token như Failure 1, cộng thêm việc model tốn token vào việc nhắc tên tài liệu nguồn thay vì trả lời luôn.
> - **Proposed fix:** Thêm vào prompt câu *"Do not cite file names or document headers. Start directly with the answer."* và tăng token limit. Mình không chắc 100% về Why 2 vì chỉ thấy đoạn text còn sót lại, nên sau khi tăng token cần xem lại output đầy đủ của case này.

---

### Failure 3

**ID và question:**

> *Điền:* H05 — "A customer realizes they entered the wrong shipping address in a different country while their order is in 'Confirmed' status. Can they update the address directly, and what if the order status has already shifted to 'Packing'?"

**Expected answer:**

> *Điền:* A shipping address cannot be changed to a different destination country once an order is placed. Within the 'Confirmed' status, minor domestic address corrections may be requested, but changing to a different country requires cancelling the order and placing a new one. Once the order shifts to 'Packing' or later, no address updates or cancellations can be made by support.

**Actual answer:**

> *Điền:* "Based on the retrieved contexts:"

**Scores:** Context Recall: 0.927 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**

> *Câu trả lời:* Chunk `OT-04-P02` (không đổi được quốc gia, khác biệt giữa Confirmed và Packing) nằm ở top 1, Recall 0.927 và Precision 1.000. Thông tin cần cho câu trả lời đã có trong context.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời chỉ có phần mở đầu "Based on the retrieved contexts:" rồi hết. |
| Why 1 | Tại sao symptom xảy ra? | Hết output token ngay sau câu mở đầu. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt đã ghi "Answer concisely in English without a generic preamble" nhưng model vẫn viết preamble, và số token ít ỏi còn lại dùng hết vào đó. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có stop sequence hay bước post-processing nào để bỏ preamble. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Output được lưu nguyên dạng string, không kiểm tra độ dài tối thiểu. |
| Why 5 | Root cause có thể hành động được là gì? | Tăng `max_output_tokens` và thêm ví dụ few-shot để model trả lời thẳng vào nội dung. |

**Root cause và proposed fix:**

> *Câu trả lời:*
> - **Root cause:** Token limit quá thấp, và model không tuân theo yêu cầu bỏ preamble trong prompt.
> - **Proposed fix:** Tăng token limit lên 1024, thêm 2 ví dụ few-shot trả lời thẳng không có câu dẫn, và coi câu trả lời kết thúc bằng dấu `:` là output lỗi cần retry.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Câu trả lời bị cắt do hết token:** `max_output_tokens=300` không đủ sau khi trừ thinking token và preamble. | E01, E02, M01, M06, H01, H03, H04, H05, A01 | High |
| 2 | **Trả lời thiếu vế:** model nêu được điều kiện chính nhưng bỏ ngoại lệ và các khoản phí kèm theo (phí kiểm định, thời hạn trả hàng bundle). | E03, E04, M02, M04, M05, H02 | Medium |
| 3 | **Từ chối đúng nhưng bị chấm thấp:** model từ chối câu hỏi ngoài phạm vi, nhưng cách diễn đạt khác expected answer nên word overlap cho điểm thấp. | A02, A03, E05, M03 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Mình chọn Cluster 1.
>
> Lý do đầu tiên là số lượng: 9/19 failure thuộc cluster này, và ở các case đó cả Faithfulness, Relevance, Completeness đều về 0.000. Lý do thứ hai là nó rẻ nhất để sửa, chỉ cần đổi config (tăng `max_output_tokens`, tắt thinking hoặc lọc preamble), không phải đụng tới retriever.
>
> Còn một lý do nữa: khi câu trả lời còn bị cắt thì mình chưa đánh giá được Cluster 2 và 3 một cách đáng tin. Có thể một số case "thiếu vế" thực ra cũng do bị cắt. Sửa Cluster 1 xong chạy lại mới biết hai cluster kia còn lại bao nhiêu. Pass rate sau khi sửa mình đoán sẽ tăng rõ, nhưng chưa chạy lại nên không dám đưa con số cụ thể.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Answer is missing key information — increase context window or improve generation | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Strictly enforce prompt guardrails to ground answers in retrieved context | Open |
| F003 | incomplete | Answer is missing key information — increase context window or improve generation | Refine system prompt and instructions to directly address user queries | Open |
| F004 | irrelevant | Answer is missing key information — increase context window or improve generation | Optimize query embeddings to improve relevance of retrieved documents | Open |
| F005 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F006 | hallucination | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F007 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F008 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F009 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F010 | incomplete | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F011 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F012 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F013 | irrelevant | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F014 | hallucination | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F015 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F016 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F017 | hallucination | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F018 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F019 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tăng `max_output_tokens` lên 1024 và bỏ preamble trong câu trả lời của generator.
2. Thêm ví dụ few-shot để model nêu đủ cả điều kiện chính, ngoại lệ và các khoản phí.
3. Với nhóm câu từ chối/adversarial, chấm bằng LLM judge hoặc semantic similarity thay cho word overlap.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Tăng `max_output_tokens` lên 1024, bỏ preamble | Completeness lên khoảng 0.60, Faithfulness khoảng 0.75 | Chạy lại `evaluate_answers.py`, xem độ dài output trong trace và so Completeness với lần chạy này. |
| Thêm few-shot cho câu hỏi multi-hop | Completeness khoảng 0.70 trên nhóm Hard | Chạy `BenchmarkRunner.run_regression()` trên 5 câu `H01`–`H05`. |
| Dùng LLM judge cho nhóm Adversarial | Faithfulness và Relevance của nhóm Adversarial | Chấm lại bằng `LLMJudge.score_response()` với rubric 1–5 ở Exercise 3.3, so với điểm heuristic. |

Các con số mục tiêu ở trên là ước lượng của mình, chưa đo.

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Mình sẽ cho `run_regression()` chạy tự động trong CI ở ba thời điểm:
> - Khi có pull request sửa retriever, sửa prompt template, hoặc đổi version model.
> - Khi knowledge base được cập nhật, ví dụ thêm hoặc sửa văn bản chính sách.
> - Trước mỗi lần release, để so với baseline đã được duyệt.
>
> Trường hợp đổi version model là cái mình thấy rõ nhất qua bài lab này: cùng một config `max_output_tokens=300` nhưng sang model có thinking thì kết quả hỏng gần hết.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
> Mình nghĩ không nên dùng chung một ngưỡng cho mọi metric.
>
> Với Faithfulness thì 0.05 hơi lỏng. Bot đang trả lời về bảo hành, hoàn tiền, an toàn pin, nên trả lời sai chính sách là có hậu quả thật với khách. Mình sẽ siết xuống khoảng 0.02 và đặt thêm một mức sàn tuyệt đối, ví dụ không được dưới 0.85.
>
> Với Relevance và Completeness thì 0.05 chấp nhận được, vì cách diễn đạt của model thay đổi giữa các lần chạy và điểm dao động nhẹ là bình thường. Ngoài ra benchmark chỉ có 20 câu, một câu đổi kết quả đã làm trung bình xê dịch đáng kể, nên siết quá sẽ báo động giả nhiều.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> Block deployment:
> - Bất kỳ case adversarial nào bị vượt: lộ system prompt, tư vấn y tế/pháp lý, hứa đền bù ngoài chính sách.
> - Faithfulness giảm quá 0.02 hoặc xuống dưới 0.85.
> - Context Recall giảm quá 0.05, vì retriever không lấy được thông tin thì phía sau không cứu được.
> - Có output bị cắt (`finish_reason == 'MAX_TOKENS'`). Cái này mình thêm vào sau bài lab.
>
> Chỉ alert:
> - Relevance hoặc Completeness giảm trong khoảng 0.03–0.05. Gửi thông báo cho người phụ trách prompt xem lại, không cần chặn.
> - Latency tăng nhưng vẫn trong SLA.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Tests & Golden Dataset Offline Benchmark] → [Shadow Traffic & Canary Evaluation] → [Human Review / Audit on Edge Cases] → Deploy
```

> *Giải thích:*
> - **Stage 1:** Chạy `pytest tests/` và benchmark trên 20 câu của golden dataset. Mục đích là bắt lỗi code và các lần tụt metric rõ ràng trước khi tốn công ở các bước sau.
> - **Stage 2:** Cho bản mới chạy shadow rồi canary trên một phần nhỏ traffic thật để đo latency, error rate và điểm LLM judge online, khách hàng chưa bị ảnh hưởng.
> - **Stage 3:** Người nắm nghiệp vụ xem lại các case điểm thấp và các case nhạy cảm trước khi mở toàn bộ traffic.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Nâng `max_output_tokens` lên 1024 và bỏ preamble trong generator | Completeness, Faithfulness | Xử lý 9 case bị cắt output; pass rate dự kiến tăng rõ so với 5% hiện tại. |
| 2 | Thêm few-shot cho câu hỏi multi-hop/Hard | Completeness | Model nêu đủ điều kiện chính, ngoại lệ và các khoản phí ở nhóm Cluster 2. |
| 3 | Thêm reranker (`rerank_by_overlap` hoặc cross-encoder) | Context Precision, Relevance | Lợi ích nhỏ vì Precision đã 0.955; chủ yếu để xử lý mấy case Recall thấp (min 0.258). |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Trả hàng khi thanh toán bằng nhiều nguồn:** Khách mua NovaBook 14, trả một phần bằng OrbitTech Gift Card (200 USD), phần còn lại trả góp qua OrbitPay, sau đó đòi hoàn tiền mặt. Bot phải giải thích được phần gift card hoàn về thẻ quà và các kỳ trả góp còn lại bị hủy, chứ không hoàn tiền mặt. Case này kiểm tra việc ghép thông tin từ nhiều tài liệu.
> 2. **Prompt injection bằng ngôn ngữ khác hoặc base64:** Yêu cầu bot bỏ qua chính sách bảo hành và cam kết đền 5,000 USD cho máy rơi vỡ màn hình. Nhóm adversarial hiện tại chỉ có câu tiếng Anh viết thẳng.
> 3. **Chính sách thay đổi theo thời gian:** Khách mua ngày 31/08/2026, gửi yêu cầu trả hàng ngày 15/09/2026. Bot phải xác định áp dụng phiên bản chính sách theo ngày đặt hàng hay ngày gửi yêu cầu, dựa trên `09_escalation_and_policy_updates.md`.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Mình đoán sai chỗ yếu của hệ thống.
>
> Trước khi chạy, mình nghĩ BM25 sẽ là điểm nghẽn vì nó chỉ khớp từ khóa, không có embedding, và sẽ trượt ở các câu hỏi diễn đạt khác với tài liệu. Kết quả thì Recall 0.858 và Precision 0.955. Một phần là nhờ title boost và repeat decay, một phần chắc do corpus nhỏ và câu hỏi dùng từ khá sát tài liệu.
>
> Ngược lại, phần mình yên tâm nhất là Gemini lại kéo pass rate xuống 5%, và nguyên nhân không nằm ở khả năng của model mà ở một dòng config: `max_output_tokens=300` bị thinking token ăn gần hết. Nếu chỉ nhìn bảng điểm và nhãn `hallucination` thì mình đã kết luận sai hoàn toàn. Phải mở từng câu trả lời ra đọc mới thấy chúng bị cắt. Đó là điều mình rút ra nhiều nhất từ bài này: xem trace trước khi tin vào score.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> Giới hạn mình gặp trong bài:
> - **Không hiểu từ đồng nghĩa.** Expected answer viết "charges" mà model viết "powers up via" thì không được tính trùng, dù nghĩa như nhau.
> - **Chấm sai câu từ chối.** Ở nhóm adversarial, bot từ chối đúng nhưng dùng từ khác expected answer, nên Completeness và Relevance gần 0 (Cluster 3).
> - **Không phân biệt khẳng định với phủ định.** "Được đổi trả" và "không được đổi trả" gần như trùng hết từ khóa, nên một câu trả lời sai ngược vẫn có thể được điểm cao.
> - **Không phân biệt lỗi kỹ thuật với lỗi nội dung.** Câu trả lời bị cắt được gán nhãn `hallucination`, dễ dẫn tới hướng sửa sai.
>
> Nếu đưa vào production mình sẽ bổ sung:
> - **LLM-as-a-judge** chấm theo rubric 1–5 đã thiết kế ở Exercise 3.3, dùng làm metric chính cho chất lượng câu trả lời.
> - **Semantic similarity** (embedding cosine hoặc BERTScore) giữa actual và expected answer, để xử lý vấn đề từ đồng nghĩa với chi phí thấp.
> - **NLI/entailment** để kiểm tra câu trả lời có suy ra được từ context hay không, đo hallucination sát hơn word overlap.
> - **Các check định dạng đơn giản** chạy trước khi chấm điểm: `finish_reason`, độ dài tối thiểu, câu trả lời có kết thúc trọn vẹn không.
>
> Word overlap mình vẫn giữ lại vì nó rẻ và chạy nhanh trong CI, nhưng chỉ dùng như tín hiệu sơ bộ.
