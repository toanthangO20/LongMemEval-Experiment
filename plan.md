# Kế hoạch phát triển hướng nghiên cứu: Evidence-Grounded Adaptive Reflective Memory Manager

**Phiên bản:** v3 — cập nhật sau khi chạy A0.5
**Trạng thái hiện tại:**
- A0 đã chạy xong trên LongMemEval-S cleaned.
- A0.5 đã chạy xong phần **Oracle Evidence Reader**.
- A0.5 diagnostic suite còn thiếu một số phần do notebook chưa tìm thấy A0 artifacts.

**Next action nhỏ ngay lập tức:**
`A0.5b — Complete Diagnostic Suite`

**Next action chính tiếp theo:**
`A1 — Turn/chunk-level evidence retrieval + compact evidence context`

---

## 0. Tóm tắt trạng thái hiện tại

Bạn đã hoàn thành hai bước quan trọng:

```text
A0   = BM25 session-level retrieval baseline trên LongMemEval-S cleaned
A0.5 = Oracle evidence reader diagnostic trên answerable examples
```

A0 tạo baseline đầy đủ.
A0.5 xác nhận rằng bottleneck lớn hiện tại là **retrieval/context selection**, vì khi dùng oracle evidence sessions, accuracy tăng rất mạnh.

---

## 1. Kết quả A0 — BM25 Session Baseline

### 1.1. Cấu hình A0

| Hạng mục | Giá trị |
|---|---:|
| Dataset | LongMemEval-S cleaned |
| Số examples | 500 |
| Retriever | BM25 |
| Granularity | Session |
| Top-k | 5 sessions |
| Reader | cx/gpt-5.2 |
| Judge | cx/gpt-5.2 |
| Overall accuracy | 0.7360 |
| Task-averaged accuracy | 0.7622 |
| Abstention accuracy | 0.7667 |
| Failed examples | 132 / 500 |
| Avg prompt tokens/query | 16,789.94 |
| Avg completion tokens/query | 213.88 |
| Total prompt tokens | 8,394,972 |
| Total completion tokens | 106,942 |
| Generation latency | 3,327.51 s |
| Evaluation latency | 1,065.33 s |
| Total latency/query | 8.7857 s |

### 1.2. Accuracy theo nhóm câu hỏi trong A0

| Eval type | n | Accuracy | Failures |
|---|---:|---:|---:|
| single-session-user | 64 | 0.9219 | 5 |
| knowledge-update | 72 | 0.9028 | 7 |
| single-session-assistant | 56 | 0.8393 | 9 |
| abstention | 30 | 0.7667 | 7 |
| temporal-reasoning | 127 | 0.6929 | 39 |
| single-session-preference | 30 | 0.6667 | 10 |
| multi-session | 121 | 0.5455 | 55 |

### 1.3. Retrieval metrics trong A0

| Retrieval metric | Value |
|---|---:|
| MRR session | 0.7039 |
| session recall@1 | 0.6298 |
| session recall@3 | 0.7532 |
| session recall@5 | 0.7915 |
| session recall@10 | 0.8255 |
| session recall@20 | 0.8574 |
| session recall_all@5 | 0.7132 |
| session recall_all@10 | 0.7588 |
| session recall_all@20 | 0.8058 |

### 1.4. Nhận định chính từ A0

A0 cho thấy BM25 session baseline khá mạnh ở các nhóm factual/update:

```text
single-session-user:       0.9219
knowledge-update:          0.9028
single-session-assistant:  0.8393
```

Nhưng yếu rõ ở:

```text
multi-session:             0.5455
temporal-reasoning:        0.6929
single-session-preference: 0.6667
```

Hai nhóm `multi-session` và `temporal-reasoning` tạo ra:

```text
55 + 39 = 94 failures
94 / 132 ≈ 71.2% tổng failures
```

Insight chính từ A0:

> Session-level BM25 retrieval không đủ tốt khi câu hỏi cần gom nhiều evidence, xử lý temporal constraint, đếm/cộng/lọc/deduplicate, hoặc phân biệt planned event với completed event.

---

## 2. Kết quả A0.5 — Oracle Evidence Reader

### 2.1. Cấu hình A0.5

A0.5 tạo oracle dataset từ `answer_session_ids`.

```text
Input:
  longmemeval_s_cleaned.json

For each answerable example:
  - lấy answer_session_ids
  - chỉ giữ các gold evidence sessions
  - bỏ 30 abstention examples khỏi oracle run

Output:
  longmemeval_s_cleaned_answerable_oracle.json
```

### 2.2. Kết quả chính A0.5

| Hạng mục | Giá trị |
|---|---:|
| Oracle answerable examples | 470 |
| Oracle retrieval rows | 470 |
| Missing gold sessions | 0 |
| Oracle correct | 431 |
| Oracle failed | 39 |
| Oracle overall accuracy | 0.9170 |
| Oracle task-averaged accuracy | 0.9158 |
| Oracle avg prompt tokens/query | 8,688.60 |
| Oracle generation latency | 2,871.74 s |
| Oracle evaluation latency | 965.26 s |
| Oracle total latency | 3,837.00 s |
| Oracle latency/example | 8.16 s |

### 2.3. So sánh A0 answerable-only với A0.5 Oracle

| Setting | Examples | Correct | Failures | Accuracy | Avg prompt tokens/query |
|---|---:|---:|---:|---:|---:|
| A0 BM25 session@5, answerable-only | 470 | 345 | 125 | 0.7340 | 16,789.94 |
| A0.5 Oracle evidence sessions | 470 | 431 | 39 | 0.9170 | 8,688.60 |

Oracle gap:

```text
0.9170 - 0.7340 = +0.1830
```

Oracle recovered:

```text
431 - 345 = 86 examples
86 / 125 A0 answerable failures ≈ 68.8%
```

Token reduction:

```text
16,789.94 - 8,688.60 = 8,101.34 fewer prompt tokens/query
8,101.34 / 16,789.94 ≈ 48.25% reduction
```

### 2.4. A0 vs Oracle theo eval type

| Eval type | A0 accuracy | A0 failures | Oracle accuracy | Oracle failures | Gap | Errors recovered |
|---|---:|---:|---:|---:|---:|---:|
| single-session-user | 0.9219 | 5 | 0.9688 | 2 | +0.0469 | 3 |
| single-session-assistant | 0.8393 | 9 | 1.0000 | 0 | +0.1607 | 9 |
| single-session-preference | 0.6667 | 10 | 0.8333 | 5 | +0.1666 | 5 |
| multi-session | 0.5455 | 55 | 0.8430 | 19 | +0.2975 | 36 |
| temporal-reasoning | 0.6929 | 39 | 0.9606 | 5 | +0.2677 | 34 |
| knowledge-update | 0.9028 | 7 | 0.8889 | 8 | -0.0139 | -1 |

### 2.5. Nhận định chính từ A0.5

A0.5 cho thấy:

```text
A0 errors are largely recoverable with correct evidence.
```

Đặc biệt:

```text
multi-session:
  A0 failures:     55
  Oracle failures: 19
  recovered:       36

temporal-reasoning:
  A0 failures:     39
  Oracle failures: 5
  recovered:       34
```

Tổng hai nhóm chính:

```text
A0 multi + temporal failures:     94
Oracle multi + temporal failures: 24
Oracle recovered:                 70 / 94 ≈ 74.5%
```

Diễn giải:

> Bottleneck lớn nhất hiện tại không phải reader không đủ mạnh, mà là hệ thống chưa chọn được đúng và đủ evidence ở granularity phù hợp.

---

## 3. Trạng thái A0.5 hiện tại

A0.5 đã hoàn thành phần quan trọng nhất:

```text
Oracle Evidence Reader = DONE
```

Nhưng A0.5 chưa hoàn thành toàn bộ diagnostic suite, vì notebook báo không tìm thấy A0 artifacts.

Notebook đã skip các phần:

```text
Standardized retrieval metrics
Failed-case taxonomy
Completion length diagnostic
```

Trạng thái chi tiết:

| Component | Status | Notes |
|---|---|---|
| Oracle evidence reader | DONE | Main A0.5 result completed |
| Oracle summary | DONE | Oracle accuracy = 0.9170 |
| Prompt leakage static check | DONE | Chưa thấy dấu hiệu reader đọc gold answer |
| Standardized retrieval metrics | SKIPPED | A0 artifacts not found |
| Failed-case taxonomy | SKIPPED | A0 artifacts not found |
| Completion length diagnostic | SKIPPED | A0 artifacts not found |
| Manual judge audit | TODO | Chưa làm |
| Actual prompt dump check | TODO | Cần dump prompt thật |

Vì vậy, trạng thái chính xác của A0.5 là:

```text
A0.5 = PARTIALLY DONE
A0.5 Oracle = DONE
A0.5 Diagnostics = INCOMPLETE
```

---

## 4. Ý nghĩa nghiên cứu sau A0 + A0.5

A0 + A0.5 cho phép rút ra kết luận mạnh hơn so với A0 đơn lẻ.

### 4.1. Kết luận chính

```text
BM25 session-level retrieval đạt 0.7340 answerable accuracy.
Oracle evidence sessions đạt 0.9170 answerable accuracy.
Oracle gap = +0.1830.
```

Điều này chứng minh:

> Long-term memory performance hiện bị giới hạn lớn bởi evidence selection và context construction.

### 4.2. Multi-session và temporal là mục tiêu tốt nhất

Hai nhóm này vừa là nhóm fail nhiều nhất ở A0, vừa là nhóm được oracle cứu nhiều nhất:

```text
multi-session gap:       +0.2975
temporal-reasoning gap:  +0.2677
```

Do đó, hướng tiếp theo nên tập trung vào:

```text
evidence-level retrieval
compact evidence context
event/time metadata
structured reasoning over evidence
```

### 4.3. A0.5 chưa phải final method contribution

A0.5 dùng gold `answer_session_ids`, nên không phải một phương pháp deployable.

Không nên claim:

```text
Our method achieves 0.9170 accuracy.
```

Nên claim:

```text
Oracle evidence diagnostic shows that correct evidence selection can recover 68.8% of A0 answerable failures.
```

A0.5 là một diagnostic/upper-bound result rất tốt, nhưng đóng góp chính tiếp theo phải là một phương pháp non-oracle đóng được một phần oracle gap.

---

## 5. Định vị đề tài nghiên cứu sau A0.5

Tên đề tài vẫn giữ:

> **Evidence-Grounded Adaptive Reflective Memory Manager for Long-Term Agentic Memory**

Nhưng claim nên được cập nhật:

> A BM25 session-level baseline performs reasonably on factual/update questions but fails on multi-session and temporal reasoning. Oracle evidence diagnostics show that 68.8% of answerable failures are recoverable when correct evidence sessions are supplied. This motivates a non-oracle evidence-grounded memory manager that retrieves compact evidence units, preserves evidence IDs, supports structured event reasoning, and later builds evidence-grounded reflections and versioned memories.

Claim tiếng Việt:

> BM25 session-level retrieval là baseline khá mạnh ở factual/update questions, nhưng yếu ở multi-session và temporal reasoning. A0.5 cho thấy khi cung cấp đúng evidence sessions, hệ thống cứu được 68.8% lỗi answerable. Vì vậy, hướng nghiên cứu nên tập trung vào memory manager có evidence-level retrieval, compact evidence context, structured event/preference memory, evidence-grounded reflection và temporal/versioned update.

---

## 6. Roadmap cập nhật sau A0.5

| Thứ tự | Mã | Trạng thái | Tên | Mục tiêu |
|---:|---|---|---|---|
| 1 | A0 | DONE | BM25 session baseline | Full benchmark trên LongMemEval-S cleaned |
| 2 | A0_Report | DONE | A0 report | Báo cáo baseline và phân tích lỗi sơ bộ |
| 3 | A0.5 | PARTIAL | Oracle + Diagnostic Benchmark | Oracle done, diagnostics incomplete |
| 4 | A0.5_Report | DONE | A0.5 report | Báo cáo oracle gap và insight retrieval bottleneck |
| 5 | A0.5b | NEXT-SMALL | Complete Diagnostic Suite | Attach A0 artifacts, taxonomy, completion diagnostic |
| 6 | A1 | NEXT-MAIN | Turn/chunk-level retrieval | Non-oracle method để đóng oracle gap |
| 7 | A2 | NEXT | Evidence-table reader | Giảm lỗi aggregation/temporal/event-status |
| 8 | A3 | TODO | Adaptive Writer | Tạo raw evidence + fact/event/preference memory |
| 9 | A4 | TODO | Evidence-Grounded Reflective Summarizer | Hỗ trợ multi-session synthesis có evidence_ids |
| 10 | A5 | TODO | Temporal/Event-aware Retriever | Lọc theo date, event status, time window |
| 11 | A6 | TODO | Versioned Memory Updater | Thêm supersedes, valid_from, valid_until, active_at |
| 12 | A7 | TODO | Adaptive Retriever / Router | Route retrieval theo query intent |
| 13 | A8 | TODO | Retrospective Learner | Học lại score từ evidence miss / judge label / correction |
| 14 | E1 | LATER | LongMemEval-M / LongMemEval-V2 | Mở rộng sang context lớn hơn hoặc agentic trajectory memory |

---

## 7. Next action nhỏ: A0.5b — Complete Diagnostic Suite

### 7.1. Lý do cần A0.5b

A0.5 oracle đã đủ để quyết định hướng A1.
Nhưng để báo cáo nghiên cứu chắc hơn, cần hoàn tất các diagnostic bị skip.

A0.5b không cần chạy lại oracle generation/evaluation nếu đã có output.
A0.5b chỉ cần attach đúng A0 artifacts rồi chạy các cell diagnostic.

### 7.2. A0 artifacts cần attach

Cần đảm bảo notebook A0.5b đọc được các file sau:

```text
A0_bm25_session_lme_s_cleaned_hypotheses.jsonl
A0_bm25_session_lme_s_cleaned_eval_log.jsonl
A0_bm25_session_lme_s_cleaned_retrieval_log.jsonl
A0_bm25_session_lme_s_cleaned_summary.json
A0_bm25_session_lme_s_cleaned_failed_cases.csv
```

Nếu tên file khác, cần sửa path trong notebook.

### 7.3. Output cần tạo trong A0.5b

```text
A0_retrieval_metrics_standardized.json
A0_retrieval_metrics_standardized.csv
A0_failed_case_taxonomy_sample57.csv
A0_failed_case_taxonomy_summary.json
A0_completion_length_diagnostic.json
A0_prompt_sample_3.txt
A05_oracle_prompt_sample_3.txt
A0_judge_manual_audit.csv
```

### 7.4. Checklist A0.5b

- [ ] Attach đúng A0 artifacts vào Kaggle notebook.
- [ ] Verify `HAS_A0_ARTIFACTS = True`.
- [ ] Chạy standardized retrieval metrics.
- [ ] Tạo failed-case taxonomy.
- [ ] Manual audit 57 failed cases trọng điểm.
- [ ] Tính completion length diagnostic.
- [ ] Dump 3 prompt thật của A0.
- [ ] Dump 3 prompt thật của A0.5 oracle.
- [ ] Manual check prompt không leak gold answer.
- [ ] Audit 20 pass + 20 fail cases để kiểm tra judge consistency.

### 7.5. Failed-case taxonomy cần có

Mỗi failed case nên có các trường:

```text
question_id
eval_type
question
gold_answer
hypothesis
judge_label
retrieval_miss
retrieved_session_ids
gold_answer_session_ids
num_gold_sessions
num_gold_sessions_retrieved
error_labels
notes
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

### 7.6. Manual audit tối thiểu

Audit trước:

```text
20 multi-session failures
20 temporal-reasoning failures
10 single-session-preference failures
7 abstention failures
```

Tổng:

```text
57 failed cases
```

### 7.7. Completion diagnostic cần tính

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

### 7.8. Prompt leakage check

Checklist:

```text
- Reader không nhìn thấy gold answer.
- Reader không nhìn thấy answer_session_ids dưới dạng label.
- Judge mới nhìn thấy gold answer.
- Retrieval context chỉ gồm retrieved sessions/evidence.
- Gold answer nếu có in stdout chỉ là logging, không nằm trong prompt.
```

---

## 8. Next action chính: A1 — Turn/chunk-level evidence retrieval

### 8.1. Vì sao A1 là next-main action

A0.5 cho thấy oracle evidence sessions tăng answerable accuracy:

```text
0.7340 → 0.9170
```

Và cứu:

```text
86 / 125 A0 answerable failures = 68.8%
```

Do đó, bước chính tiếp theo phải là một phương pháp **non-oracle** để chọn evidence tốt hơn.

Không nên đi ngay sang:

```text
LongMemEval-M
BM25 session@10/session@20 làm hướng chính
full 5-module memory manager
Versioned Updater trước
```

Lý do:

```text
- Oracle gap lớn chứng minh retrieval/context selection là bottleneck.
- Tăng top-k session chỉ tăng recall vừa phải nhưng tăng token cost.
- Session-level context quá dài và nhiễu.
- Need compact evidence, not more raw sessions.
```

### 8.2. Mục tiêu A1

A1 cần thay:

```text
retrieve top-5 full sessions
```

bằng:

```text
retrieve top-k compact evidence units
```

Các đơn vị evidence:

```text
turn-level
sliding chunk 2–4 turns
optionally session fallback
```

### 8.3. Indexing units trong A1

| Unit | Mô tả | Ưu điểm | Nhược điểm |
|---|---|---|---|
| session | Full session | Giữ nhiều context | Rất nhiễu, tốn token |
| turn | Một user/assistant turn | Chính xác, ít token | Dễ mất local context |
| sliding chunk | 2–4 turns liên tiếp | Cân bằng context và precision | Cần tune chunk size |
| session fallback | Full session khi evidence rời rạc | An toàn hơn | Tốn token nếu dùng nhiều |
| event/fact memory | Extracted memory object | Compact, structured | Để A3 |

### 8.4. Metadata bắt buộc cho evidence item

Mỗi retrieved item phải giữ:

```json
{
  "evidence_id": "session_12_turn_04",
  "session_id": "session_12",
  "turn_id": 4,
  "chunk_id": "session_12_chunk_02",
  "date": "2023-05-28",
  "role": "user",
  "text": "...",
  "retrieval_score": 12.4
}
```

### 8.5. Reader context format mới

Không đưa full sessions mặc định. Đưa compact evidence:

```text
[Evidence 1]
date: ...
session_id: ...
turn_id/chunk_id: ...
role: ...
text: ...

[Evidence 2]
date: ...
session_id: ...
turn_id/chunk_id: ...
role: ...
text: ...
```

### 8.6. A1 variants nên chạy

| ID | Variant | Mục tiêu |
|---|---|---|
| A1.1 | BM25 turn@k | Kiểm tra granularity nhỏ nhất |
| A1.2 | BM25 chunk-2turn@k | Giữ context gần turn |
| A1.3 | BM25 chunk-4turn@k | Cân bằng context và recall |
| A1.4 | Turn/chunk + session fallback | Tránh mất context |
| A1.5 | Query decomposition cho multi-session | Retrieve nhiều evidence theo sub-questions |

### 8.7. k nên thử trong A1

Không nên dùng cùng top-k với session vì evidence units nhỏ hơn.

Đề xuất:

```text
turn/chunk top-k = 10, 20, 30, 50
session fallback = 0, 1, 2
```

### 8.8. Metrics chính của A1

| Metric | Kỳ vọng |
|---|---|
| `answerable_accuracy` | Tăng so với A0 |
| `overall_accuracy` | Tăng hoặc không giảm |
| `multi-session_accuracy` | Tăng rõ |
| `temporal_accuracy` | Tăng rõ |
| `avg_prompt_tokens/query` | Giảm mạnh |
| `oracle_gap` | Thu hẹp |
| `evidence_precision@k` | Tăng |
| `gold_session_hit_any@k` | Không giảm quá mạnh |
| `gold_session_hit_all@k` | Tăng hoặc giữ |
| `context_noise_rate` | Giảm |

### 8.9. A1 success criteria

A1 được xem là thành công nếu đạt một trong các điều kiện:

```text
- answerable accuracy > 0.7340
- multi-session accuracy tăng so với 0.5455
- temporal-reasoning accuracy tăng so với 0.6929
- prompt tokens/query giảm ít nhất 30%
- oracle gap giảm so với 0.1830
```

Kết quả rất tốt nếu:

```text
A1 answerable accuracy đạt 0.80–0.86
và prompt tokens/query giảm 30–60%
```

---

## 9. A2 — Evidence-table reader

### 9.1. Vì sao A2 cần làm sau hoặc song song với A1

A0.5 oracle vẫn còn 39 failures:

| Eval type | Oracle failures |
|---|---:|
| multi-session | 19 |
| knowledge-update | 8 |
| single-session-preference | 5 |
| temporal-reasoning | 5 |
| single-session-user | 2 |
| single-session-assistant | 0 |

Điều này nghĩa là ngay cả khi có đúng sessions, reader vẫn có thể sai ở:

```text
aggregation
counting
summing
temporal filtering
event status distinction
preference inference
update-chain reasoning
```

Do đó A2 nên cải thiện reader format.

### 9.2. Mục tiêu A2

Giảm lỗi:

```text
aggregation_error
temporal_filter_error
event_status_error
semantic_category_error
preference_inference_error
```

### 9.3. Prompt format đề xuất

```text
For each relevant evidence item, extract:
- evidence_id
- date
- event/entity
- value/count/amount if any
- include_or_exclude
- reason

Then answer using only included rows.
```

### 9.4. Output format mong muốn

```text
Evidence table:
| evidence_id | date | event | value | include? | reason |
|---|---|---|---:|---|---|
| s2_t4 | March 12 | charity walk | 250 | yes | user participated |
| s5_t2 | May 05 | bike-a-thon | 600 | yes | user participated |
| s4_t1 | June 01 | planned fundraiser | N/A | no | only planned |

Final answer: $850
```

### 9.5. A2 evaluation trước khi chạy full

Chạy trước trên subset:

```text
all A0 multi-session failures
all A0 temporal-reasoning failures
all oracle remaining multi-session failures
all oracle remaining temporal failures
```

Sau đó mới chạy full 500 nếu tốt.

### 9.6. A2 success criteria

A2 thành công nếu:

```text
- giảm aggregation_error_rate
- giảm temporal_filter_error_rate
- giảm event_status_error_rate
- tăng multi-session accuracy
- tăng temporal-reasoning accuracy
- không làm completion_tokens tăng quá nhiều
```

---

## 10. A3 — Adaptive Writer

### 10.1. Mục tiêu

Sau A1/A2, bắt đầu chuyển từ retrieval đơn thuần sang memory manager có cấu trúc.

Các lớp memory:

```text
raw evidence
fact memory
event memory
preference memory
reflection memory
versioned memory
```

### 10.2. Schema raw evidence

```json
{
  "evidence_id": "sess_012_turn_003",
  "question_id": "...",
  "session_id": "sess_012",
  "turn_id": 3,
  "role": "user",
  "content": "...",
  "timestamp": "2023-08-21"
}
```

### 10.3. Schema memory object

```json
{
  "memory_id": "mem_000001",
  "memory_type": "fact | event | preference | reflection",
  "subject": "user",
  "predicate": "prefers",
  "object": "Italian restaurants",
  "content": "The user prefers Italian restaurants for casual dinners.",
  "evidence_ids": ["sess_012_turn_003"],
  "source_session_ids": ["sess_012"],
  "created_at": "2023-08-21",
  "valid_from": "2023-08-21",
  "valid_until": null,
  "supersedes": [],
  "confidence": 0.82,
  "status": "active"
}
```

### 10.4. Writer focus sau A0.5

Vì A0.5 cho thấy multi-session và temporal được oracle cứu nhiều nhất, writer nên ưu tiên:

```text
event extraction
date extraction
value/count extraction
preference extraction
event status extraction: planned / completed / cancelled / intended
evidence_ids for every memory
```

### 10.5. Metrics cho Adaptive Writer

| Metric | Mục tiêu |
|---|---|
| `evidence_coverage` | Gold sessions có được memory cover không |
| `memory_compression_ratio` | Raw tokens / memory tokens |
| `memory_support_rate` | Tỷ lệ memory có evidence_ids |
| `unsupported_memory_rate` | Tỷ lệ memory không support được |
| `event_date_extraction_accuracy` | Riêng temporal |
| `preference_extraction_accuracy` | Riêng preference |
| `memory_build_cost` | Token/latency để build memory |

---

## 11. A4 — Evidence-Grounded Reflective Summarizer

### 11.1. Mục tiêu

A0 multi-session thấp nhất:

```text
multi-session accuracy = 0.5455
```

A0.5 oracle cải thiện mạnh:

```text
multi-session oracle accuracy = 0.8430
```

Điều này cho thấy multi-session cần đúng evidence và cần synthesis tốt.

Reflection nên hỗ trợ synthesis, nhưng phải evidence-grounded.

### 11.2. Nguyên tắc

Reflection không được là summary tự do.

Mọi reflection phải có:

```text
evidence_ids
supporting_memory_ids
confidence
created_at
valid_from
valid_until
status
```

### 11.3. Reflection schema

```json
{
  "memory_id": "refl_000023",
  "memory_type": "reflection",
  "content": "The user tends to prioritize budget and convenience when planning trips.",
  "scope": "travel_preferences",
  "evidence_ids": [
    "sess_003_turn_002",
    "sess_018_turn_005",
    "sess_027_turn_001"
  ],
  "supporting_memory_ids": [
    "mem_000041",
    "mem_000118",
    "mem_000155"
  ],
  "confidence": 0.74,
  "created_at": "2024-05-01",
  "valid_from": "2024-05-01",
  "valid_until": null,
  "status": "active"
}
```

### 11.4. Reflection validator

```text
Input:
  reflection content
  evidence snippets

Output:
  supported: true/false
  unsupported_claims: [...]
  confidence_adjustment: -0.2 / 0 / +0.1
```

Nếu unsupported:

```text
- không lưu reflection;
- hoặc hạ confidence;
- hoặc lưu trạng thái draft, không dùng cho retrieval chính.
```

### 11.5. Metrics

| Metric | Mục tiêu |
|---|---|
| `reflection_support_rate` | Cao |
| `unsupported_reflection_error_rate` | Thấp |
| `reflection_helpfulness` | Tăng multi-session accuracy |
| `reflection_usage_rate` | Reader có dùng reflection |
| `token_saving_vs_raw_sessions` | Giảm context tokens |

---

## 12. A5 — Temporal/Event-aware Retriever

### 12.1. Mục tiêu

A0 temporal-reasoning còn thấp:

```text
temporal-reasoning accuracy = 0.6929
```

A0.5 oracle rất cao:

```text
temporal-reasoning oracle accuracy = 0.9606
```

Điều này cho thấy temporal errors phần lớn đến từ evidence selection/context construction.

A5 cần retrieval hiểu:

```text
date
time window
event status
question_date
before / after / during
planned vs completed
current vs past
```

### 12.2. Strategy

| Query type | Strategy |
|---|---|
| ask current state | Ưu tiên active/latest memory |
| ask past state | Lọc memory valid tại thời điểm được hỏi |
| ask event count | Retrieve event memories + date/value/status |
| ask before/after | Time-window filter |
| ask changed/updated | Retrieve old + new evidence |
| ask plan vs actual | Ưu tiên completed/confirmed events, loại planned-only nếu question yêu cầu actual |

### 12.3. Metrics

| Metric | Mục tiêu |
|---|---|
| `temporal_accuracy` | Tăng |
| `temporal_filter_error_rate` | Giảm |
| `event_status_error_rate` | Giảm |
| `date_extraction_error_rate` | Giảm |
| `validity_filter_precision` | Tăng |

---

## 13. A6 — Versioned Memory Updater

### 13.1. Mục tiêu

Dù không phải bottleneck đầu tiên sau A0/A0.5, Versioned Updater vẫn là module quan trọng để định vị Agentic AI memory.

Không ghi đè thô. Dùng:

```text
supersedes
valid_from
valid_until
status
active_at(question_date)
```

### 13.2. Vì sao chưa ưu tiên ngay

A0 cho thấy:

```text
knowledge-update accuracy = 0.9028
```

A0.5 oracle cho thấy:

```text
knowledge-update oracle accuracy = 0.8889
```

Do đó, update không phải bottleneck lớn nhất hiện tại. Versioned Updater nên làm sau A1/A2/A3/A5.

### 13.3. Update schema

Memory cũ:

```json
{
  "memory_id": "mem_000012",
  "content": "The user lives in Boston.",
  "status": "superseded",
  "valid_from": "2022-01-01",
  "valid_until": "2024-02-17",
  "superseded_by": "mem_000089"
}
```

Memory mới:

```json
{
  "memory_id": "mem_000089",
  "content": "The user lives in Seattle.",
  "status": "active",
  "valid_from": "2024-02-17",
  "valid_until": null,
  "supersedes": ["mem_000012"]
}
```

### 13.4. Conflict/update detection labels

```text
SAME_FACT
ELABORATION
CONTRADICTION
PREFERENCE_CHANGE
TEMPORAL_UPDATE
IRRELEVANT
```

### 13.5. Metrics

| Metric | Mục tiêu |
|---|---|
| `stale_memory_error_rate` | Giảm |
| `active_fact_accuracy` | Tăng |
| `past_fact_accuracy` | Tăng |
| `update_chain_recall` | Tăng |
| `conflict_detection_accuracy` | Tăng |

---

## 14. A7 — Adaptive Retriever / Query Router

### 14.1. Nguyên tắc

Không dùng gold `question_type` trong final system.

Có thể dùng gold type cho:

```text
oracle-router diagnostic
upper bound
analysis only
```

Final system phải tự predict intent.

### 14.2. Query intent labels

```text
factual
preference
temporal
update
multi_session
abstention_risk
```

### 14.3. Retrieval strategy theo intent

| Intent | Strategy |
|---|---|
| factual | fact memory + raw evidence fallback |
| preference | preference memory + active/latest preference |
| temporal | event memory + date/time filter |
| update | active memory + superseded chain |
| multi_session | reflection + supporting evidence |
| abstention_risk | evidence sufficiency threshold |

### 14.4. Scoring function

```text
score =
  lexical_score
+ dense_score
+ evidence_support_score
+ recency_score_if_update
+ temporal_validity_score
+ event_status_score
+ memory_confidence
- staleness_penalty
- unsupported_reflection_penalty
```

### 14.5. Evaluation

| Setting | Dùng gold type? | Mục đích |
|---|---|---|
| A7-oracle-router | Có | Upper bound |
| A7-predicted-router | Không | Kết quả chính |
| A7-fixed-retriever | Không | Baseline |
| A7-without-temporal-filter | Không | Ablation |
| A7-without-confidence-threshold | Không | Ablation abstention |

---

## 15. A8 — Retrospective Learner

### 15.1. Mục tiêu

Sau khi trả lời, dùng feedback để cập nhật:

```text
memory_score
retrieval_score
confidence_threshold
staleness_penalty
router weights
```

### 15.2. Protocol tránh leakage

Không dùng gold label của final test để tune rồi báo cáo trên chính test đó.

Dùng:

```text
dev_stratified: tune thresholds, weights, router prompt
test_final: freeze config, run once
```

### 15.3. Feedback signals

| Signal | Dùng ở đâu |
|---|---|
| retrieved evidence used by reader | online/self-supervised |
| LLM citation agreement | online/self-supervised |
| judge label | dev only |
| gold answer_session_ids | dev/analysis only |
| user correction | simulated hoặc future interactive |
| abstention false positive | dev calibration |

### 15.4. Score update rules

```text
memory_score += 0.1 nếu memory được dùng trong answer đúng
memory_score -= 0.1 nếu memory retrieved nhưng gây answer sai
retriever_weight_temporal += nếu temporal query bị retrieval_miss
confidence_threshold += nếu nhiều abstention false positives
staleness_penalty += nếu knowledge-update dùng old memory
```

---

## 16. Bảng ablation cập nhật

| ID | System | Status | Overall | Answerable | Multi-session | Temporal | Update | Abstention | Tokens/query | Mục đích |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| A0 | BM25 session@5 | DONE | 0.7360 | 0.7340 | 0.5455 | 0.6929 | 0.9028 | 0.7667 | 16,789.94 | Baseline |
| A0.5 | Oracle evidence reader | ORACLE DONE | N/A | 0.9170 | 0.8430 | 0.9606 | 0.8889 | N/A | 8,688.60 | Upper-bound diagnostic |
| A0.5b | Complete diagnostics | NEXT-SMALL | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Taxonomy + prompt/completion audit |
| A1 | BM25 turn/chunk | NEXT-MAIN | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Non-oracle evidence retrieval |
| A2 | Evidence-table reader | NEXT | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Giảm aggregation/temporal errors |
| A3 | Adaptive Writer | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Structured memory |
| A4 | Evidence-grounded Reflection | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Multi-session synthesis |
| A5 | Temporal/Event-aware Retriever | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Temporal reasoning |
| A6 | Versioned Updater | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Stale/update handling |
| A7 | Adaptive Retriever | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Intent-aware retrieval |
| A8 | Retrospective Learner | TODO | TBD | TBD | TBD | TBD | TBD | TBD | TBD | Feedback-driven scoring |

---

## 17. Metrics cần báo cáo xuyên suốt

### 17.1. QA metrics

| Metric | Mục đích |
|---|---|
| `overall_accuracy` | Kết quả end-to-end |
| `answerable_accuracy` | Accuracy không tính abstention |
| `macro_accuracy_by_type` | Tránh overall bị che bởi nhóm lớn |
| `accuracy_by_eval_type` | Biết module nào giúp nhóm nào |
| `abstention_accuracy` | Đánh giá refusal/no-answer |
| `false_answer_rate` | Tỷ lệ bịa khi nên abstain |

### 17.2. Retrieval metrics

| Metric | Mục đích |
|---|---|
| `hit_any@k` | Có ít nhất một gold evidence không |
| `hit_all@k` | Có đủ toàn bộ gold evidence không |
| `gold_fraction@k` | Tỷ lệ gold evidence retrieved |
| `mrr@k` | Evidence rank |
| `ndcg@k` | Ranking quality |
| `evidence_precision@k` | Giảm nhiễu |
| `oracle_gap` | Tách retrieval vs reader |

### 17.3. Reader/reasoning metrics

| Metric | Mục đích |
|---|---|
| `aggregation_error_rate` | Đếm/cộng/tổng hợp sai |
| `temporal_filter_error_rate` | Lọc thời gian sai |
| `event_status_error_rate` | Nhầm planned vs actual |
| `semantic_category_error_rate` | Lọc sai category |
| `no_final_answer_rate` | Output không rõ final |
| `hit_max_tokens_rate` | Bị giới hạn output |

### 17.4. Memory metrics

| Metric | Mục đích |
|---|---|
| `memory_compression_ratio` | Raw tokens / memory tokens |
| `memory_support_rate` | Memory có evidence support |
| `unsupported_memory_rate` | Memory unsupported |
| `reflection_support_rate` | Reflection grounded |
| `unsupported_reflection_error_rate` | Reflection gây hallucination |
| `stale_memory_error_rate` | Dùng memory hết hiệu lực |
| `validity_filter_precision` | Lọc đúng memory theo thời gian |

### 17.5. Efficiency metrics

| Metric | Mục đích |
|---|---|
| `avg_prompt_tokens` | Context cost |
| `avg_completion_tokens` | Answer cost |
| `latency_per_query` | Practicality |
| `memory_build_cost` | Chi phí build memory |
| `storage_size` | Dung lượng memory store |

---

## 18. Implementation structure cập nhật

```text
memory_manager/
  data/
    raw/
      longmemeval_s_cleaned.json
    processed/
      oracle/
      turn_chunks/
      diagnostics/
    memory_store/
  src/
    loaders/
      longmemeval_loader.py
    retrieval/
      bm25_session.py
      bm25_turn.py
      bm25_chunk.py
      dense.py
      hybrid.py
      reranker.py
      temporal_retriever.py
      adaptive_router.py
    reader/
      standard_reader.py
      concise_reader.py
      evidence_table_reader.py
    memory/
      writer.py
      event_extractor.py
      preference_extractor.py
      updater.py
      reflection.py
      validator.py
      store.py
    evaluation/
      qa_eval.py
      retrieval_eval.py
      standardize_retrieval_metrics.py
      diagnostics.py
      error_taxonomy.py
      judge_audit.py
    experiments/
      run_A0_bm25_session.py
      run_A05_oracle_diagnostic.py
      run_A05b_complete_diagnostic.py
      run_A1_bm25_turn_chunk.py
      run_A2_evidence_table_reader.py
      run_A3_writer.py
      run_A4_reflection.py
      run_A5_temporal_retriever.py
      run_A6_versioned.py
      run_A7_adaptive.py
  results/
    A0_bm25_session/
    A05_oracle_diagnostic/
    A05b_complete_diagnostic/
    A1_bm25_turn_chunk/
    A2_evidence_table_reader/
  reports/
    A0_Report.md
    A0.5_Report.md
```

---

## 19. Chuẩn output cho mỗi experiment

Mỗi experiment nên có cùng format:

```text
experiment_id/
  config.yaml
  hypotheses.jsonl
  retrieval_log.jsonl
  eval_log.jsonl
  diagnostic_log.jsonl
  summary.json
  failed_cases.csv
  cost_latency.csv
```

A0.5b nên có thêm:

```text
A05b_complete_diagnostic/
  A0_retrieval_metrics_standardized.json
  A0_retrieval_metrics_standardized.csv
  A0_failed_case_taxonomy_sample57.csv
  A0_failed_case_taxonomy_summary.json
  A0_completion_length_diagnostic.json
  A0_prompt_sample_3.txt
  A05_oracle_prompt_sample_3.txt
  A0_judge_manual_audit.csv
```

A1 nên có:

```text
A1_bm25_turn_chunk/
  config.yaml
  turn_index.jsonl
  chunk_index.jsonl
  retrieval_log.jsonl
  retrieval_metrics.json
  token_estimate.json
  hypotheses.jsonl
  eval_log.jsonl
  summary.json
  failed_cases.csv
```

---

## 20. Checklist hiện tại

### A0 — Done

- [x] Chạy full A0 BM25 session trên LongMemEval-S cleaned.
- [x] Có overall accuracy.
- [x] Có answerable-only accuracy.
- [x] Có accuracy theo eval_type.
- [x] Có retrieval metrics.
- [x] Có token/cost/latency.
- [x] Có failed examples.
- [x] Viết `A0_Report.md`.
- [ ] Chuẩn hóa retrieval metrics bằng một script duy nhất.
- [ ] Hoàn tất manual judge audit.

### A0.5 — Partially Done

- [x] Tạo oracle dataset từ `answer_session_ids`.
- [x] Chạy oracle evidence reader trên 470 answerable examples.
- [x] So sánh A0 vs Oracle.
- [x] Tính oracle gap.
- [x] Tính token reduction oracle vs A0.
- [x] Static prompt leakage check.
- [x] Viết `A0.5_Report.md`.
- [ ] Tạo failed-case taxonomy.
- [ ] Manual audit 57 failed cases trọng điểm.
- [ ] Tính completion length diagnostic.
- [ ] Dump actual prompts.
- [ ] Audit judge consistency trên 20 pass + 20 fail cases.

### A0.5b — Next-small

- [ ] Attach A0 artifacts vào A0.5b notebook.
- [ ] Verify artifacts path.
- [ ] Chạy skipped diagnostics.
- [ ] Cập nhật `A0.5_Report.md` nếu có phát hiện mới.
- [ ] Chốt error taxonomy để hỗ trợ A1/A2.

### A1 — Next-main

- [ ] Build turn-level index.
- [ ] Build chunk-level index.
- [ ] Implement BM25 turn retrieval.
- [ ] Implement BM25 chunk retrieval.
- [ ] Implement compact evidence context.
- [ ] Run retrieval-only metrics trước.
- [ ] Estimate token cost.
- [ ] Run generation/evaluation nếu retrieval-only tốt.
- [ ] Compare A1 vs A0 vs Oracle.

---

## 21. Việc không nên làm ngay

Không nên làm các việc sau ở thời điểm hiện tại:

```text
- Không chạy LongMemEval-M ngay.
- Không tăng top-k session lên 10/20 như hướng chính.
- Không implement full 5-module memory manager ngay.
- Không ưu tiên Versioned Updater trước A1/A2.
- Không dùng gold question_type trong final retriever.
- Không claim A0.5 oracle là final method.
- Không chỉ báo cáo overall accuracy.
- Không claim SOTA.
- Không bỏ qua token cost.
```

Lý do:

```text
A0.5 đã cho thấy oracle evidence cứu được 68.8% answerable failures.
Vấn đề trước mắt là đóng oracle gap bằng phương pháp non-oracle,
không phải thêm nhiều session hơn hoặc chuyển ngay sang benchmark lớn hơn.
```

---

## 22. Contribution statements cập nhật

### Contribution 1 — Reproducible A0 baseline

> We reproduce a full LongMemEval-S cleaned BM25 session-level baseline and show that session retrieval performs strongly on single-session factual and knowledge-update questions, but fails substantially on multi-session and temporal reasoning.

### Contribution 2 — Oracle-gap diagnostic result

> We run an oracle evidence diagnostic showing that replacing BM25 session retrieval with gold evidence sessions raises answerable accuracy from 0.7340 to 0.9170 and recovers 68.8% of answerable failures.

### Contribution 3 — Evidence selection bottleneck analysis

> The largest oracle gains occur in multi-session and temporal reasoning, suggesting that long-term memory systems require evidence-level selection and structured context construction rather than more raw session retrieval.

### Contribution 4 — Evidence-level retrieval and structured reading

> We propose turn/chunk-level evidence retrieval combined with evidence-table reading to reduce context noise and address aggregation, temporal filtering, and event-status errors.

### Contribution 5 — Evidence-grounded memory representation

> We propose a memory representation where every fact, event, preference and reflection is linked to evidence_ids and confidence, enabling auditability and reducing unsupported memory use.

### Contribution 6 — Grounded reflection for multi-session memory

> We investigate evidence-grounded reflection as a mechanism for improving multi-session synthesis while preserving traceability to raw evidence.

### Contribution 7 — Temporal and versioned memory

> We extend the memory manager with temporal/event-aware retrieval and versioned update to reduce stale-memory and temporal-validity errors.

---

## 23. Safe vs unsafe claims sau A0.5

### Safe claims

Có thể viết:

```text
A0.5 is an oracle diagnostic, not a deployable method.
```

```text
Oracle evidence sessions improve answerable accuracy from 0.7340 to 0.9170.
```

```text
A0.5 recovers 86 out of 125 A0 answerable failures, equal to 68.8% error recovery.
```

```text
The largest gains are on multi-session and temporal reasoning.
```

```text
These results motivate turn/chunk-level evidence retrieval and compact evidence context.
```

### Unsafe claims

Không nên viết:

```text
Our method achieves 0.9170 accuracy.
```

```text
We solve LongMemEval-S.
```

```text
Oracle evidence reader is our proposed memory manager.
```

```text
Evidence selection bottleneck has never been studied before.
```

Lý do:

```text
A0.5 dùng gold answer_session_ids.
Oracle setting là upper bound/diagnostic, không phải deployable system.
```

---

## 24. Kết quả nghiên cứu có thể báo cáo ngay

A0 + A0.5 đủ để báo cáo như một preliminary/diagnostic result:

```text
1. A0: BM25 session baseline đạt 0.7360 overall accuracy.
2. A0 answerable-only accuracy là 0.7340.
3. A0.5 oracle evidence đạt 0.9170 answerable accuracy.
4. Oracle gap là +0.1830.
5. Oracle recovered 68.8% answerable failures.
6. Multi-session và temporal reasoning là hai nhóm được oracle cứu nhiều nhất.
7. Oracle context giảm prompt tokens khoảng 48.25% so với A0 session@5.
```

Cách viết contribution hiện tại:

> We present a diagnostic study showing that evidence selection is the dominant bottleneck for our BM25 session baseline on LongMemEval-S cleaned. Oracle evidence sessions recover 68.8% of answerable failures and reduce prompt tokens by 48.25%, motivating a non-oracle evidence-level memory manager.

---

## 25. Quyết định tiếp theo

### 25.1. Next-small

Chạy:

```text
A0.5b — Complete Diagnostic Suite
```

Mục tiêu:

```text
- Có failed-case taxonomy.
- Có completion diagnostic.
- Có standardized retrieval metrics.
- Có prompt dump check.
- Có judge audit.
```

### 25.2. Next-main

Triển khai:

```text
A1 — Turn/chunk-level evidence retrieval + compact evidence context
```

Mục tiêu:

```text
- Đóng một phần oracle gap.
- Giảm token cost.
- Tăng multi-session accuracy.
- Tăng temporal-reasoning accuracy.
```

### 25.3. Next-after-A1

Triển khai:

```text
A2 — Evidence-table reader
```

Mục tiêu:

```text
- Giảm aggregation errors.
- Giảm temporal filtering errors.
- Giảm event-status errors.
- Cải thiện các oracle remaining failures.
```

---

## 26. Kết luận cập nhật

A0 đã cho thấy baseline BM25 session-level có hiệu quả vừa đủ nhưng yếu ở multi-session và temporal reasoning.

A0.5 đã xác nhận mạnh hơn:

```text
Nếu đưa đúng evidence sessions,
answerable accuracy tăng từ 0.7340 lên 0.9170.
```

Do đó, hướng nghiên cứu hiện tại là đúng, nhưng priority cần rõ:

```text
A0.5b diagnostics
→ A1 turn/chunk-level evidence retrieval
→ A2 evidence-table reader
→ A3 structured memory writer
→ A4 evidence-grounded reflection
→ A5 temporal/event-aware retrieval
→ A6 versioned updater
```

Kết luận nghiên cứu quan trọng nhất hiện tại:

> Memory tốt cho Agentic AI không chỉ là retrieve nhiều hơn, mà là retrieve đúng evidence ở đúng granularity, giảm context noise, giữ metadata thời gian/sự kiện, và đảm bảo mọi memory hoặc reflection đều truy ngược được về evidence_ids.