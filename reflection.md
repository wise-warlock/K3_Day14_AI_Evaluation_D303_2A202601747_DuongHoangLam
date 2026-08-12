# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 15.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.890 | 0.409 | 1.000 | Rất tốt: BM25 Retriever phủ đầy đủ evidence cho 17/20 câu hỏi. |
| Context Precision | 0.935 | 0.500 | 1.000 | Rất cao: Các chunk liên quan xuất hiện ở thứ tự đầu danh sách (AP@K cao). |
| Faithfulness | 0.461 | 0.000 | 1.000 | Yếu: generator tạo ra câu trả lời bỏ sót mốc số liệu cốt lõi trong context. |
| Relevance | 0.341 | 0.000 | 0.846 | Rất yếu: generator chưa bám sát từ khóa chính của câu hỏi. |
| Completeness | 0.514 | 0.000 | 1.000 | Trung bình: thiếu thông tin chi tiết ở các câu hỏi phức hợp. |
| Overall Score | 0.438 | 0.000 | 0.843 | Mức Significant Issues (< 0.6) do khâu sinh câu trả lời (generation). |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 3 cases (E01, E05, M02).
- Metrics/cases ở mức Needs Work (0.6–0.8): 3 cases (M04, H02, A02).
- Metrics/cases ở mức Significant Issues (<0.6): 14 cases.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 6 | 35.3% |
| irrelevant | 4 | 23.5% |
| incomplete | 0 | 0.0% |
| off_topic | 7 | 41.2% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở khâu **Generation** (sinh câu trả lời), không phải Retrieval.
Dẫn chứng: Avg Context Recall đạt **0.890** và Avg Context Precision đạt **0.935** (retriever lấy đúng và đủ dữ liệu), nhưng Avg Answer Relevance chỉ đạt **0.341** và Avg Faithfulness chỉ đạt **0.461** do generator chọn và cắt gọt câu chưa bám sát từ khóa thắc mắc của sinh viên.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**
> E03 — What minimum attendance percentage is expected from students in courses that record attendance?

**Expected answer:**
> Students are expected to attend at least 80% of scheduled sessions in courses that record attendance.

**Actual answer:**
> A course syllabus may set a higher threshold when required by accreditation, laboratory safety, or clinical practice, but it may not set a lower threshold. Three consecutive unexplained absences trigger an academic-support alert; the alert itself is not a disciplinary finding. Instructor permission alone does not replace a prerequisite unless the programme director also records the waiver.

**Scores:** Context Recall: 1.000 | Context Precision: 0.833 | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?
> Retriever lấy đúng chunk chính chứa thông tin "attend at least 80% of scheduled sessions". Tuy nhiên generator cắt gọt bỏ qua câu đầu tiên (chứa 80%) và trích câu 2-3 phía sau.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời bỏ sót con số phần trăm tham dự 80% mà câu hỏi trực tiếp yêu cầu. |
| Why 1 | Tại sao symptom xảy ra? | Generator trích xuất các câu 2 và 3 trong chunk mà bỏ qua câu đầu tiên chứa con số 80%. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Trình chọn câu (Sentence selector) không chấm điểm độ trùng lặp từ khóa giữa từng câu và câu hỏi. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa có bước Query-Aware Sentence Selection trong Generator trước khi trả lời. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống thiếu một màng lọc Faithfulness / Fact Verification tự động đối chiếu các thực thể số (numerical entities). |
| Why 5 | Root cause có thể hành động được là gì? | Thêm bước Query-Sentence Overlap Selection hoặc sử dụng LLM Prompt ép buộc trích xuất đúng thực thể con số được hỏi. |

**Root cause từ `find_root_cause()`:**
> "Context is missing or irrelevant — improve retrieval" (hoặc "Multiple issues detected — review full pipeline")

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Không đồng ý với nhận định "missing retrieval" vì Context Recall ở case E03 đạt 1.000 (retriever đã lấy đúng chunk chứa câu 80%). Nguyên nhân thực sự hoàn toàn nằm ở Generator.

**Proposed fix cụ thể:**
> Bổ sung Few-shot prompt cho Generator chỉ đạo rút trích chính xác câu chứa từ khóa/con số được hỏi trong câu hỏi tra cứu factual.

---

### Failure 2

**ID và question:**
> E04 — How many total verified hours are required for programmes with an internship requirement?

**Expected answer:**
> Programmes with an internship requirement require at least 240 verified hours.

**Actual answer:**
> Before starting, the student must have an approved placement agreement and workplace supervisor. Hours completed before approval do not normally count. The supervisor submits a completion evaluation, and the student submits the programme reflection within ten business days after the placement ends.

**Scores:** Context Recall: 1.000 | Context Precision: 0.887 | Faithfulness: 0.036 | Relevance: 0.111 | Completeness: 0.125 | Overall: 0.091

**Evidence inspection:**
> Retriever lấy đúng chunk từ `07_graduation_and_internship.md` chứa "240 verified hours", nhưng Generator trích các câu phía sau về thủ tục nộp báo cáo.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Câu trả lời thiếu con số 240 giờ thực tập theo yêu cầu. |
| Why 1 | Tại sao symptom xảy ra? | Generator trích các câu về quy trình phê duyệt thay vì câu chứa số giờ. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Chiến lược trích xuất câu chưa ưu tiên các câu chứa thực thể định lượng (number/hours). |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa cài đặt Intent Classifier nhận diện câu hỏi dạng "How many" (yêu cầu con số). |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống generator hiện tại xử lý chung mọi loại câu hỏi theo 1 template cố định. |
| Why 5 | Root cause có thể hành động được là gì? | Cài đặt Intent-Based Prompt Routing: khi gặp câu hỏi "How many", ưu tiên trích xuất câu chứa số liệu. |

**Root cause và proposed fix:**
> Root cause: Generator thiếu cơ chế ưu tiên câu chứa số liệu khi gặp câu hỏi định lượng "How many".
> Proposed fix: Thêm Intent Classifier cho dạng câu hỏi định lượng và dùng Few-shot Prompt ép buộc trả lời số giờ chính xác.

---

### Failure 3

**ID và question:**
> E02 — What is the undergraduate tuition rate per registered credit for the 2026–2027 academic year?

**Expected answer:**
> Undergraduate tuition for the 2026–2027 academic year is USD 420 per registered credit.

**Actual answer:**
> The student-services fee is USD 180 in Fall and Spring and USD 90 in Summer. Tuition and mandatory fees are due by the regular registration deadline listed in `01_academic_calendar.md`. A five-calendar-day grace period applies to the account balance, but it does not extend registration or scholarship deadlines.

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.061 | Relevance: 0.091 | Completeness: 0.182 | Overall: 0.111

**Evidence inspection:**
> Context Precision đạt 1.000 (chunk `03_tuition_payment_refund.md` chứa "USD 420 per registered credit" đứng ở ngay vị trí đầu tiên), nhưng Generator lại chọn thông tin về lệ phí 180 USD / 90 USD ở câu sau.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Trả lời sai mức học phí theo tín chỉ (bỏ sót USD 420, đưa nhầm phí student-services). |
| Why 1 | Tại sao symptom xảy ra? | Generator nhầm lẫn giữa học phí (tuition rate) và phí dịch vụ sinh viên (student-services fee). |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Generator chọn câu có chứa từ "fee" thay vì câu chứa từ "tuition". |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa phân biệt rõ khái niệm "tuition per credit" và "mandatory fees" trong prompt. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Thiếu quy tắc kiểm tra tính trùng khớp khái niệm giữa question entity và answer entity. |
| Why 5 | Root cause có thể hành động được là gì? | Cải thiện System Prompt phân biệt rõ các loại chi phí tài chính (Tuition vs Mandatory Fees). |

**Root cause và proposed fix:**
> Root cause: Generator nhầm lẫn giữa khái niệm Học phí (Tuition) và Phí dịch vụ (Services Fee).
> Proposed fix: Tinh chỉnh System Prompt định nghĩa rõ ràng các thuật ngữ tài chính để Generator chọn đúng câu trả lời.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generator chọn sai câu trong chunk, bỏ sót thông tin con số/định lượng (Missing entity in generation) | E02, E03, E04, M03, H01, A03 | High |
| 2 | Prompt chưa làm rõ quy tắc trả lời từng vế đối với câu hỏi so sánh/điều kiện (Incomplete multi-part answer) | M01, M04, M05, M06, M07, H02, H04 | High |
| 3 | Thiếu cơ chế phân loại Intent cho câu hỏi adversarial (Out-of-scope & Prompt injection guardrails) | H03, H05, A01, A02 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> Chọn **Cluster 1 (Missing entity in generation)** vì cluster này chịu trách nhiệm cho các thất bại nghiêm trọng nhất (điểm 0.000 – 0.111) ở các câu hỏi Easy tra cứu thực tế. Việc sửa Cluster 1 bằng cách nâng cấp Prompt/Generator sẽ ngay lập tức cải thiện điểm Faithfulness và Relevance cho toàn bộ benchmark.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Refine system prompt and query expansion to improve answer relevance | Open |
| F004 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F006 | off_topic | Multiple issues detected — review full pipeline | Add few-shot examples showing complete answers to improve completeness | Open |
| F007 | off_topic | Multiple issues detected — review full pipeline | Refine system prompt and query expansion to improve answer relevance | Open |
| F008 | off_topic | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F009 | off_topic | Multiple issues detected — review full pipeline | Implement hallucination checker to filter unsupported claims | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F011 | off_topic | Multiple issues detected — review full pipeline | Refine system prompt and query expansion to improve answer relevance | Open |
| F012 | irrelevant | Answer does not address the question — improve prompt clarity | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F013 | off_topic | Multiple issues detected — review full pipeline | Implement hallucination checker to filter unsupported claims | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | Add few-shot examples showing complete answers to improve completeness | Open |
| F015 | irrelevant | Answer does not address the question — improve prompt clarity | Refine system prompt and query expansion to improve answer relevance | Open |
| F016 | off_topic | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F017 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
```

**Ba improvement suggestions ưu tiên**

1. Tinh chỉnh System Prompt và thêm Query-Sentence Overlap Selector để đảm bảo Generator trích xuất chính xác câu chứa từ khóa/con số được hỏi.
2. Bổ sung Few-shot examples hướng dẫn Generator trả lời đầy đủ các điều kiện phức hợp trong câu hỏi Medium và Hard.
3. Tích hợp Hallucination Guardrail Layer để tự động chặn các câu trả lời thiếu căn cứ trước khi gửi cho người dùng.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Query-Sentence Overlap Selector | Faithfulness & Relevance | Chạy `python evaluate_answers.py`, đo xem Faithfulness có tăng từ 0.461 lên > 0.80 hay không. |
| Few-shot Prompting cho multi-part answers | Completeness | Chạy `pytest tests/`, đo điểm Completeness trung bình trên các câu Medium/Hard. |
| Hallucination Guardrail Layer | Overall Pass Rate | Đánh giá lại 20 câu trong `golden_dataset.json`, xác minh pass rate tăng từ 15% lên > 80%. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
> Chạy `run_regression()` tự động trong CI/CD pipeline bất cứ khi nào:
> 1. Thay đổi Code (khi tạo Pull Request hoặc Release mới).
> 2. Thay đổi Prompt / System Message.
> 3. Thay đổi tham số Retrieval (Top-k, Chunk size, Reranker).
> 4. Cập nhật tập dữ liệu tri thức (Corpus update).

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> *Câu trả lời:*
> Rất phù hợp. Trong lĩnh vực dịch vụ sinh viên, các quy định về học phí, hạn chót và điều kiện tốt nghiệp yêu cầu độ chính xác cao. Mức giảm 0.05 (tương đương 5%) đủ nhạy để phát hiện sớm các đợt suy giảm chất lượng mà không bị báo động giả do sai số ngẫu nhiên.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
> - **Block Deployment:** Lỗi **Faithfulness < 0.80** hoặc **Hallucination failure** (vi phạm quy định tài chính/học tập) và khi **Overall Pass Rate sụt giảm > 0.05**.
> - **Alert Only:** Metric **Context Precision nhẹ** hoặc **Latency/Word Count** biến động nhẹ chưa làm giảm điểm chất lượng tổng thể.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Eval (Golden Dataset)] → [Regression Check (run_regression)] → [Staging Smoke Test & Guardrails] → Deploy
```

> *Giải thích:*
> Khi có sự thay đổi, trước tiên chạy Offline Eval trên Golden Dataset. Nếu thỏa mãn các chỉ số, chuyển sang so sánh `run_regression()` với phiên bản baseline. Sau đó kiểm thử tự động trên Staging trước khi quyết định Deploy lên Production.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Cải thiện Generator Prompt với Few-shot examples & Overlap Selector | Faithfulness & Relevance | Nâng Faithfulness từ 0.461 lên > 0.85; tăng Pass Rate từ 15% lên > 75%. |
| 2 | Tích hợp Cross-Encoder Reranker | Context Precision | Tăng Context Precision từ 0.935 lên 0.980. |
| 3 | Bổ sung Guardrail Filter ngăn ngừa Prompt Injection | Safety / Adversarial Pass Rate | Chặn 100% các vụ tấn công prompt injection và out-of-scope. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
> 1. **Case hỏi kết hợp mốc lịch học và số tiền học phí cùng lúc** (ví dụ: Hạn nộp tiền học phí Spring 2027 và số tiền phạt nộp chậm nếu trễ hạn).
> 2. **Case hỏi quy trình khiếu nại điểm kết hợp với điều kiện giữ học bổng** (để kiểm tra khả năng suy luận liên văn bản giữa 04_scholarships và 08_student_support_and_appeals).
> 3. **Case tấn công Jailbreak tinh vi hơn** (sử dụng ngôn ngữ mã hóa hoặc nhập vai nhân viên PĐT để yêu cầu thay đổi điểm).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu tôi dự đoán Retriever (BM25) sẽ là mắt xích yếu nhất do tìm kiếm từ khóa thuần túy. Tuy nhiên, kết quả thực tế cho thấy Retriever đạt chỉ số rất ấn tượng (Context Recall 0.890, Context Precision 0.935), trong khi Generator lại là nguyên nhân chính khiến hệ thống thất bại (Pass rate chỉ đạt 15.0%) do trích gọt câu chưa bám sát từ khóa thắc mắc.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
> **Giới hạn của Word-overlap heuristics:**
> 1. Không nhận diện được từ đồng nghĩa (synonyms), diễn đạt lại (paraphrasing) hoặc ngữ cảnh học thuật.
> 2. Nhạy cảm với stop-words hoặc thay đổi cấu trúc ngữ pháp làm giảm điểm vô lý dù ý nghĩa hoàn toàn đúng.
> 3. Không đo lường được tính logic, văn phong tư vấn và mức độ an toàn của câu trả lời.
>
> **Nếu đưa vào Production, tôi sẽ:**
> 1. Thay thế bằng **RAGAS Framework thực thụ** hoặc **DeepEval** dùng LLM-as-a-Judge (GPT-4o / Claude 3.5 Sonnet) với Rubric domain-specific.
> 2. Bổ sung các metric nâng cao: **Semantic Similarity** (Vector Cosine Similarity), **G-Eval Correctness**, và **Real-time Latency / Cost metrics**.
