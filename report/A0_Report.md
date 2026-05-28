# A0 Report — BM25 Session Baseline on LongMemEval-S Cleaned

**Project:** Evidence-Grounded Adaptive Reflective Memory Manager
**Experiment ID:** `A0_bm25_session_lme_s_cleaned`
**Status:** DONE
**Dataset:** LongMemEval-S cleaned
**Baseline:** BM25 session-level retrieval
**Report version:** v1

---

## 1. Executive Summary

A0 đã hoàn thành mục tiêu chính: chạy full benchmark trên **LongMemEval-S cleaned** với baseline **BM25 session-level retrieval**. Pipeline đã chạy end-to-end từ retrieval, generation, LLM-as-judge evaluation, logging kết quả, tổng hợp metric, đến xuất các file artifact phục vụ phân tích lỗi.

Kết quả tổng quan:

| Metric | Value |
|---|---:|
| Examples evaluated | 500 |
| Overall accuracy | 0.7360 |
| Task-averaged accuracy | 0.7622 |
| Abstention accuracy | 0.7667 |
| Correct examples | 368 / 500 |
| Failed examples | 132 / 500 |
| Avg prompt tokens/query | 16,789.9440 |
| Avg completion tokens/query | 213.8840 |
| Latency/query | 8.7857 s |

Kết luận quan trọng nhất từ A0:

> BM25 session-level retrieval là một baseline khá mạnh cho các câu hỏi factual hoặc update rõ ràng, nhưng yếu rõ rệt ở **multi-session reasoning** và **temporal reasoning**. Hai nhóm này tạo ra 94 / 132 failures, tương đương khoảng 71.2% tổng số lỗi.

Vì vậy, A0 không chỉ tạo baseline, mà còn chỉ ra hướng phát triển tiếp theo:

```text
A0.5 Oracle + Diagnostic Benchmark
→ A1 Turn/chunk-level evidence retrieval
→ A2 Evidence-table reader
```

---

## 2. Experiment Objective

Mục tiêu của A0 là tạo baseline đầu tiên, có thể tái lập, cho hướng nghiên cứu **Evidence-Grounded Adaptive Reflective Memory Manager**.

A0 cần trả lời các câu hỏi sau:

1. BM25 session-level retrieval đạt hiệu quả end-to-end như thế nào trên LongMemEval-S cleaned?
2. Những nhóm câu hỏi nào baseline xử lý tốt?
3. Những nhóm câu hỏi nào baseline xử lý kém?
4. Retrieval hiện tại có retrieve được evidence session cần thiết không?
5. Chi phí token và latency của baseline là bao nhiêu?
6. Kết quả A0 gợi ý next action nào cho nghiên cứu memory mechanism?

A0 không nhằm chứng minh proposed method tốt hơn baseline. A0 là bước nền để chẩn đoán bottleneck.

---

## 3. Experiment Setup

### 3.1. Dataset

| Field | Value |
|---|---|
| Dataset | `longmemeval_s_cleaned.json` |
| Split | LongMemEval-S cleaned |
| Number of examples | 500 |
| Answerable examples for retrieval metrics | 470 |
| Abstention examples | 30 |

### 3.2. Retrieval Configuration

| Field | Value |
|---|---|
| Retriever | BM25 |
| Retrieval granularity | Session-level |
| Top-k context for reader | 5 sessions |
| Retrieval log | `A0_bm25_session_lme_s_cleaned_retrieval_log.jsonl` |

### 3.3. Reader and Judge Configuration

| Component | Value |
|---|---|
| Reader model | `cx/gpt-5.2` |
| Judge model | `cx/gpt-5.2` |
| Evaluation protocol | LLM-as-judge |
| Hypothesis file | `A0_bm25_session_lme_s_cleaned_hypotheses.jsonl` |
| Evaluation log | `A0_bm25_session_lme_s_cleaned_eval_log.jsonl` |

### 3.4. Output Artifacts

A0 đã sinh các artifact chính sau:

```text
results/
  A0_bm25_session_lme_s_cleaned_retrieval_log.jsonl
  A0_bm25_session_lme_s_cleaned_hypotheses.jsonl
  A0_bm25_session_lme_s_cleaned_eval_log.jsonl
  A0_bm25_session_lme_s_cleaned_summary.json
  A0_bm25_session_lme_s_cleaned_failed_cases.csv
  A0_bm25_session_lme_s_cleaned_cost_latency.csv
```

Các file này đủ để phục vụ:

- phân tích accuracy tổng;
- phân tích accuracy theo nhóm câu hỏi;
- phân tích retrieval recall;
- phân tích failed cases;
- phân tích token cost;
- phân tích latency;
- chuẩn bị A0.5 oracle/diagnostic benchmark.

---

## 4. Main Results

### 4.1. Overall QA Results

| Metric | Value |
|---|---:|
| Examples evaluated | 500 |
| Correct examples | 368 |
| Failed examples | 132 |
| Overall accuracy | 0.7360 |
| Task-averaged accuracy | 0.7622 |
| Abstention accuracy | 0.7667 |

Diễn giải:

- A0 đạt accuracy tổng **73.60%**.
- Đây là một baseline đủ mạnh để làm điểm xuất phát.
- Tuy nhiên, overall accuracy che giấu khác biệt rất lớn giữa các nhóm câu hỏi.
- Vì vậy, kết quả quan trọng nhất là **accuracy by eval type**, không chỉ là overall score.

---

## 5. Accuracy by Eval Type

| Eval type | n | Accuracy | Failures |
|---|---:|---:|---:|
| single-session-user | 64 | 0.9219 | 5 |
| knowledge-update | 72 | 0.9028 | 7 |
| single-session-assistant | 56 | 0.8393 | 9 |
| abstention | 30 | 0.7667 | 7 |
| temporal-reasoning | 127 | 0.6929 | 39 |
| single-session-preference | 30 | 0.6667 | 10 |
| multi-session | 121 | 0.5455 | 55 |

### 5.1. Strong Groups

Các nhóm mà BM25 session baseline xử lý tốt:

| Group | Accuracy | Interpretation |
|---|---:|---|
| single-session-user | 0.9219 | Fact từ user-side evidence thường được retrieve và đọc tốt. |
| knowledge-update | 0.9028 | Baseline xử lý khá tốt các update rõ ràng ở session-level. |
| single-session-assistant | 0.8393 | Fact từ assistant-side evidence cũng tương đối ổn. |

Điều này cho thấy baseline không yếu toàn diện. Với factual questions hoặc update questions có evidence rõ, session-level retrieval vẫn hoạt động tốt.

### 5.2. Weak Groups

Các nhóm yếu nhất:

| Group | Accuracy | Failures | Interpretation |
|---|---:|---:|---|
| multi-session | 0.5455 | 55 | Cần gom, lọc, tổng hợp nhiều evidence từ nhiều session. |
| temporal-reasoning | 0.6929 | 39 | Cần xử lý thời gian, event status, before/after, planned vs actual. |
| single-session-preference | 0.6667 | 10 | Preference thường cần suy luận hoặc tổng hợp từ ngữ cảnh. |

Hai nhóm **multi-session** và **temporal-reasoning** tạo ra:

```text
55 + 39 = 94 failures
94 / 132 ≈ 71.2% tổng failures
```

Đây là insight quan trọng nhất của A0.

---

## 6. Retrieval Results

Retrieval metrics được tính trên **470 answerable examples**, tức không bao gồm 30 abstention examples.

| Metric | Mean | n |
|---|---:|---:|
| mrr_session | 0.7039 | 470 |
| session_recall@1 | 0.6298 | 470 |
| session_recall@3 | 0.7532 | 470 |
| session_recall@5 | 0.7915 | 470 |
| session_recall@10 | 0.8255 | 470 |
| session_recall@20 | 0.8574 | 470 |
| session_recall_all@1 | 0.3753 | 470 |
| session_recall_all@3 | 0.6543 | 470 |
| session_recall_all@5 | 0.7132 | 470 |
| session_recall_all@10 | 0.7588 | 470 |
| session_recall_all@20 | 0.8058 | 470 |
| session_ndcg@1 | 0.6298 | 470 |
| session_ndcg@3 | 0.6475 | 470 |
| session_ndcg@5 | 0.6694 | 470 |
| session_ndcg@10 | 0.6890 | 470 |
| session_ndcg@20 | 0.7040 | 470 |

### 6.1. Retrieval Interpretation

Với `top_k = 5`:

```text
session_recall@5 = 0.7915
session_recall_all@5 = 0.7132
```

Điều này nghĩa là:

- Khoảng 79.15% answerable examples có ít nhất một gold session trong top-5.
- Nhưng chỉ khoảng 71.32% có đủ toàn bộ gold sessions cần thiết trong top-5.
- Với các câu hỏi multi-session, việc thiếu một phần evidence có thể đủ làm model trả lời sai.

Tăng top-k từ 5 lên 10 hoặc 20 cải thiện recall, nhưng mức cải thiện không quá lớn:

```text
session_recall@5  = 0.7915
session_recall@10 = 0.8255
session_recall@20 = 0.8574
```

Trong khi đó, top-5 đã có prompt cost rất cao. Vì vậy, **chỉ tăng top-k session không phải next action tốt nhất**.

---

## 7. Efficiency Results

| Metric | Value |
|---|---:|
| Total prompt tokens | 8,394,972 |
| Total completion tokens | 106,942 |
| Avg prompt tokens/query | 16,789.9440 |
| Avg completion tokens/query | 213.8840 |
| Generation latency | 3,327.51 s |
| Evaluation latency | 1,065.33 s |
| Total latency | 4,392.84 s |
| Latency/query | 8.7857 s |

### 7.1. Efficiency Interpretation

A0 cho thấy session-level retrieval có chi phí context lớn:

```text
~16.8k prompt tokens/query với top-5 sessions
```

Điều này tạo cơ hội nghiên cứu rõ ràng cho memory manager:

> Nếu structured memory hoặc evidence-level retrieval có thể giữ hoặc tăng accuracy trong khi giảm prompt tokens/query, đây là một đóng góp có giá trị, ngay cả khi overall accuracy chỉ tăng nhẹ.

---

## 8. What Has Been Achieved in A0

A0 đã hoàn thành các việc sau:

- [x] Chạy full benchmark trên LongMemEval-S cleaned.
- [x] Xây được pipeline end-to-end: retrieval → generation → evaluation → reporting.
- [x] Có baseline BM25 session-level trên 500 examples.
- [x] Có overall accuracy.
- [x] Có task-averaged accuracy.
- [x] Có abstention accuracy.
- [x] Có accuracy theo từng eval type.
- [x] Có retrieval metrics: MRR, recall@k, recall_all@k, NDCG@k.
- [x] Có token usage: prompt tokens và completion tokens.
- [x] Có latency metrics.
- [x] Có failed cases file để phân tích lỗi.
- [x] Có đủ artifact để chuyển sang diagnostic phase.

A0 vì vậy đã đạt mục tiêu của một reproducible baseline.

---

## 9. Key Findings

### Finding 1 — BM25 session baseline is strong for simple factual/update questions

BM25 session-level retrieval đạt accuracy cao ở:

```text
single-session-user: 0.9219
knowledge-update: 0.9028
single-session-assistant: 0.8393
```

Điều này cho thấy baseline đã đủ mạnh để làm đối chứng, không phải một baseline quá yếu.

### Finding 2 — Multi-session reasoning is the largest bottleneck

Nhóm `multi-session` có accuracy thấp nhất:

```text
multi-session accuracy = 0.5455
multi-session failures = 55
```

Các lỗi tiềm năng gồm:

- không retrieve đủ evidence từ nhiều session;
- evidence có nhưng bị nhiễu bởi context dài;
- reader tổng hợp sai;
- reader đếm/cộng/deduplicate sai;
- reader không phân biệt evidence liên quan và không liên quan.

### Finding 3 — Temporal reasoning is the second largest bottleneck

Nhóm `temporal-reasoning` có:

```text
temporal-reasoning accuracy = 0.6929
temporal-reasoning failures = 39
```

Các lỗi tiềm năng gồm:

- lọc sai mốc thời gian;
- nhầm planned event với completed event;
- hiểu sai before/after/current/past;
- thiếu date metadata rõ ràng trong context;
- evidence có nhưng không được tổ chức thành event table.

### Finding 4 — Increasing session top-k is unlikely to be sufficient

Recall tăng chậm khi tăng top-k:

```text
recall@5  = 0.7915
recall@10 = 0.8255
recall@20 = 0.8574
```

Trong khi prompt cost hiện đã cao. Do đó, hướng tốt hơn là:

```text
session-level retrieval
→ turn/chunk-level evidence retrieval
→ structured memory / evidence-table reader
```

### Finding 5 — Token cost is a major research opportunity

A0 dùng trung bình gần 16.8k prompt tokens/query. Đây là tín hiệu mạnh cho hướng evidence-grounded memory:

```text
raw sessions are expensive;
structured memory and compact evidence may reduce context cost.
```

---

## 10. Limitations of A0

A0 là baseline thành công, nhưng vẫn có một số hạn chế cần xử lý trước khi báo cáo kết quả nghiên cứu chính thức.

### 10.1. Retrieval metrics need standardization

Cần đảm bảo một script duy nhất tính các metric sau:

```text
hit_any@k
hit_all@k
gold_fraction@k
mrr@k
ndcg@k
```

Và cần ghi rõ metric được tính trên:

```text
all examples?
answerable-only?
exclude abstention?
session-level hay turn-level?
```

### 10.2. Reader and judge use the same model family

A0 dùng `cx/gpt-5.2` cho cả reader và judge. Điều này ổn cho baseline nội bộ, nhưng cần manual audit để đảm bảo judge consistency.

### 10.3. Need oracle gap before changing architecture

Hiện chưa biết chính xác bao nhiêu lỗi đến từ retrieval và bao nhiêu lỗi đến từ reader/reasoning. Do đó, chưa nên triển khai full memory manager ngay.

### 10.4. Completion length may affect some answers

Average completion tokens là 213.8840. Nếu nhiều output chạm max generation length, có thể xảy ra lỗi do truncation hoặc final answer không rõ. Cần kiểm tra thêm:

```text
completion_tokens_p50
completion_tokens_p90
completion_tokens_p95
completion_tokens_max
hit_max_tokens_rate
no_final_answer_rate
```

---

## 11. Research Interpretation

A0 hỗ trợ luận điểm nghiên cứu sau:

> Long-term memory không chỉ là retrieve thêm nhiều raw session. Với các câu hỏi multi-session và temporal, hệ thống cần evidence-level indexing, structured event/preference memory, temporal metadata, và reader có khả năng tổng hợp evidence một cách có kiểm soát.

A0 cũng cho thấy contribution tiềm năng của hướng **Evidence-Grounded Adaptive Reflective Memory Manager**:

1. **Evidence-level memory** có thể giảm context noise và token cost.
2. **Event/preference memory** có thể cải thiện temporal và preference questions.
3. **Evidence-grounded reflection** có thể hỗ trợ multi-session synthesis.
4. **Temporal/event-aware retrieval** có thể giảm lỗi lọc thời gian và event status.
5. **Oracle/diagnostic framework** có thể tách lỗi retrieval khỏi lỗi reader.

---

## 12. Decision: Next Action

Dựa trên A0, next action chính là:

```text
A0.5 — Oracle + Diagnostic Benchmark
```

Không nên làm ngay:

```text
- Không chạy LongMemEval-M ngay.
- Không tăng top-k session lên 10/20 như hướng chính.
- Không implement full 5-module memory manager ngay.
- Không ưu tiên Versioned Updater ngay, vì knowledge-update hiện đã đạt 0.9028.
```

Lý do:

- Lỗi tập trung ở multi-session và temporal reasoning.
- Chưa biết oracle gap.
- Nhiều lỗi có thể đến từ reader/reasoning, không chỉ retrieval.
- Token cost của session-level context đã cao.

---

## 13. A0.5 Plan

### 13.1. Oracle Evidence Reader

Mục tiêu:

```text
Đo reader có trả lời đúng không nếu được đưa đúng gold evidence sessions.
```

Cách làm:

```text
For each answerable example:
  - use answer_session_ids
  - keep only gold evidence sessions
  - run reader
  - evaluate with same judge protocol
```

Bảng cần báo cáo:

| Setting | Examples | Accuracy | Avg prompt tokens | Notes |
|---|---:|---:|---:|---|
| A0 BM25 session@5, all examples | 500 | 0.7360 | 16,789.9440 | Current baseline |
| A0 BM25 session@5, answerable only | 470 | TBD | TBD | Exclude abstention |
| A0.5 Oracle evidence sessions | 470 | TBD | TBD | Gold evidence context |

### 13.2. Failed-case Taxonomy

Tạo file:

```text
A0_failed_case_taxonomy_sample57.csv
```

Audit tối thiểu:

```text
20 multi-session failures
20 temporal-reasoning failures
10 single-session-preference failures
7 abstention failures
```

Error labels:

```text
retrieval_miss
partial_evidence
retrieval_rank_low
reader_failure
aggregation_error
temporal_filter_error
event_status_error
semantic_category_error
preference_inference_error
abstention_false_answer
abstention_false_refusal
judge_noise_or_format
truncation_or_no_final
over_context_noise
```

### 13.3. Completion Length Diagnostic

Cần tính:

```text
completion_tokens_min
completion_tokens_p50
completion_tokens_p90
completion_tokens_p95
completion_tokens_p99
completion_tokens_max
hit_max_tokens_rate
no_final_answer_rate
```

### 13.4. Prompt Leakage and Judge Sanity Check

Checklist:

```text
- Reader không nhìn thấy gold answer.
- Reader không nhìn thấy answer_session_ids dưới dạng label.
- Judge mới nhìn thấy gold answer.
- Retrieval context chỉ gồm retrieved sessions/evidence.
- Manual audit 20 pass cases và 20 fail cases.
```

---

## 14. Expected Next Experiments After A0.5

Tùy kết quả oracle gap, quyết định như sau:

| A0.5 Result | Interpretation | Next action |
|---|---|---|
| Oracle accuracy cao hơn A0 nhiều | Retrieval/context selection là bottleneck chính | A1 turn/chunk retrieval |
| Oracle chỉ cao hơn A0 ít | Reader/reasoning là bottleneck chính | A2 evidence-table reader |
| Oracle temporal vẫn thấp | Cần event/time extraction | A3 Adaptive Writer + A5 Temporal Retriever |
| Oracle multi-session vẫn thấp | Cần synthesis/reflection | A4 Evidence-Grounded Reflection |
| Nhiều partial evidence | Session top-k thiếu đủ evidence | A1 turn/chunk + query decomposition |
| Nhiều aggregation error | Reader tổng hợp sai | A2 evidence-table reader |
| Nhiều abstention false answer | Evidence sufficiency yếu | Answerability detector / threshold |

---

## 15. Updated Roadmap After A0

| ID | System | Status | Main purpose |
|---|---|---|---|
| A0 | BM25 session@5 | DONE | Baseline full LongMemEval-S cleaned |
| A0.5 | Oracle + Diagnostic Benchmark | NEXT | Tách retrieval vs reader, phân loại lỗi |
| A1 | BM25 turn/chunk retrieval | TODO | Evidence-level retrieval, giảm context noise |
| A2 | Evidence-table reader | TODO | Giảm aggregation/temporal errors |
| A3 | Adaptive Writer | TODO | Structured fact/event/preference memory |
| A4 | Evidence-Grounded Reflection | TODO | Multi-session synthesis có evidence IDs |
| A5 | Temporal/Event-aware Retriever | TODO | Date/status/time-window reasoning |
| A6 | Versioned Memory Updater | TODO | Supersedes, valid_from, valid_until |
| A7 | Adaptive Retriever / Router | TODO | Route theo query intent |
| A8 | Retrospective Learner | TODO | Feedback-driven memory/retrieval score |

---

## 16. A0 Contribution Statement

A0 có thể được viết trong báo cáo nghiên cứu như sau:

> We reproduced a full LongMemEval-S cleaned BM25 session-level baseline and found that session retrieval performs strongly on single-session factual and knowledge-update questions, but substantially underperforms on multi-session and temporal reasoning. These two categories account for over 70% of all errors, suggesting that long-term memory systems require evidence-level indexing, structured event reasoning, and grounded multi-session synthesis rather than simply retrieving more raw sessions.

---

## 17. Final Conclusion

A0 đã hoàn thành vai trò baseline và diagnostic starting point.

Kết quả quan trọng nhất không chỉ là:

```text
overall_accuracy = 0.7360
```

Mà là insight:

```text
BM25 session-level retrieval mạnh ở factual/update questions,
nhưng yếu ở multi-session và temporal reasoning,
đồng thời có token cost cao.
```

Do đó, hướng phát triển đúng sau A0 là:

```text
A0.5 Oracle + Diagnostic
→ A1 Turn/chunk-level evidence retrieval
→ A2 Evidence-table reader
→ A3/A4 structured memory and grounded reflection
```

A0 đã tạo nền vững chắc để biến hướng **Evidence-Grounded Adaptive Reflective Memory Manager** thành một nghiên cứu có contribution rõ ràng.