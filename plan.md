# Kế hoạch phát triển hướng nghiên cứu: Evidence-Grounded Adaptive Reflective Memory Manager

**Phiên bản:** v2 — cập nhật sau khi hoàn thành A0
**Trạng thái hiện tại:** A0 đã chạy xong trên LongMemEval-S cleaned
**Next action chính:** A0.5 — Oracle + Diagnostic Benchmark trước khi triển khai A1/A2

---

## 0. Tóm tắt trạng thái hiện tại

Bạn đã hoàn thành bước đầu tiên:

```text
A0 = BM25 session-level retrieval baseline trên LongMemEval-S cleaned
```

Kết quả A0 hiện tại:

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

Accuracy theo nhóm:

| Eval type | n | Accuracy | Failures |
|---|---:|---:|---:|
| single-session-user | 64 | 0.9219 | 5 |
| knowledge-update | 72 | 0.9028 | 7 |
| single-session-assistant | 56 | 0.8393 | 9 |
| abstention | 30 | 0.7667 | 7 |
| temporal-reasoning | 127 | 0.6929 | 39 |
| single-session-preference | 30 | 0.6667 | 10 |
| multi-session | 121 | 0.5455 | 55 |

Retrieval metrics hiện tại:

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

---

## 1. Nhận định chính từ A0

A0 không chỉ là baseline, mà đã tạo ra một insight nghiên cứu rõ ràng.

### 1.1. BM25 session baseline khá mạnh ở factual/update questions

Các nhóm sau có accuracy tốt:

```text
single-session-user: 0.9219
knowledge-update: 0.9028
single-session-assistant: 0.8393
```

Điều này cho thấy baseline BM25 session-level không yếu toàn diện. Với fact đơn lẻ hoặc update rõ ràng, top-5 sessions thường đủ để reader trả lời.

### 1.2. Bottleneck chính nằm ở multi-session và temporal reasoning

Hai nhóm lỗi lớn nhất:

```text
multi-session: 55 failures
temporal-reasoning: 39 failures
```

Tổng cộng:

```text
55 + 39 = 94 / 132 failures
≈ 71.2% toàn bộ lỗi
```

Diễn giải:

```text
BM25 session-level retrieval không đủ tốt khi câu hỏi cần:
- gom nhiều evidence từ nhiều session;
- đếm, cộng, lọc, deduplicate;
- xử lý temporal constraint;
- phân biệt planned event với completed event;
- tổng hợp nhiều mẩu evidence thành một kết luận.
```

### 1.3. Không nên chỉ tăng top-k session

Top-k cao hơn chỉ cải thiện retrieval vừa phải:

```text
recall@5  = 0.7915
recall@10 = 0.8255
recall@20 = 0.8574
```

Trong khi top-5 đã tốn trung bình:

```text
16,789.94 prompt tokens/query
```

Vì vậy, tăng top-k session sẽ làm tăng token cost mạnh nhưng chưa chắc cải thiện QA tương xứng.

### 1.4. Lỗi không chỉ đến từ retrieval

Một số failed cases có `retrieval_miss = False`, nghĩa là gold evidence session đã nằm trong retrieved context nhưng answer vẫn sai.

Điều này cho thấy có ít nhất hai bottleneck:

```text
1. Retrieval bottleneck:
   BM25 session không luôn retrieve đủ evidence, đặc biệt với multi-session.

2. Reader / reasoning bottleneck:
   Khi evidence đã có, reader vẫn có thể:
   - đếm sai;
   - cộng sai;
   - lọc sai;
   - hiểu sai temporal condition;
   - nhầm kế hoạch với sự kiện đã xảy ra;
   - không đưa ra final answer đủ rõ.
```

### 1.5. Versioned Updater chưa nên là module đầu tiên

Ban đầu, Versioned Memory Updater được xem là một trong các module rất quan trọng. Điều đó vẫn đúng về mặt nghiên cứu dài hạn.

Tuy nhiên, A0 cho thấy:

```text
knowledge-update accuracy = 0.9028
```

Vì nhóm knowledge-update hiện chưa phải bottleneck lớn nhất, không nên ưu tiên Versioned Updater ngay sau A0.

Ưu tiên mới nên là:

```text
1. A0.5 Oracle + Diagnostic
2. A1 Turn/chunk-level evidence retrieval
3. A2 Evidence-table reader
4. A3 Adaptive Writer cho fact/event/preference memory
5. A4 Evidence-grounded reflection cho multi-session
6. A5 Temporal/event-aware retrieval
7. A6 Versioned Updater
```

---

## 2. Định vị đề tài nghiên cứu sau A0

Tên đề tài gợi ý:

> **Evidence-Grounded Adaptive Reflective Memory Manager for Long-Term Agentic Memory**

Claim chính nên điều chỉnh thành:

> Session-level BM25 retrieval là baseline khá mạnh cho factual và update-style memory, nhưng yếu rõ rệt ở multi-session và temporal reasoning. Một memory manager tốt cho Agentic AI cần evidence-level indexing, structured event/preference memory, evidence-grounded reflection, temporal/event-aware retrieval, và versioned update để giảm lỗi tổng hợp, lỗi thời gian, stale memory và unsupported reflection.

Không nên claim:

```text
Our method achieves SOTA.
```

Nên claim:

```text
We build an auditable long-term memory framework and show, through diagnostics and ablations, which memory mechanisms improve which long-memory abilities.
```

---

## 3. Roadmap cập nhật sau A0

| Thứ tự | Mã | Trạng thái | Tên | Mục tiêu |
|---:|---|---|---|---|
| 1 | A0 | DONE | BM25 session baseline | Đã có full benchmark trên LongMemEval-S cleaned |
| 2 | A0.5 | NEXT | Oracle + Diagnostic Benchmark | Tách lỗi retrieval vs reader, phân loại lỗi, chuẩn hóa metrics |
| 3 | A1 | TODO | BM25 turn/chunk retrieval | Giảm nhiễu context, tăng evidence precision |
| 4 | A2 | TODO | Evidence-table reader | Giảm lỗi đếm, cộng, lọc temporal/event |
| 5 | A3 | TODO | Adaptive Writer | Tạo raw evidence + fact/event/preference memory |
| 6 | A4 | TODO | Evidence-Grounded Reflective Summarizer | Hỗ trợ multi-session synthesis có evidence_ids |
| 7 | A5 | TODO | Temporal/Event-aware Retriever | Lọc theo date, event status, time window |
| 8 | A6 | TODO | Versioned Memory Updater | Thêm supersedes, valid_from, valid_until, active_at |
| 9 | A7 | TODO | Adaptive Retriever / Router | Route retrieval theo query intent, không dùng gold question_type |
| 10 | A8 | TODO | Retrospective Learner | Học lại score từ evidence miss / judge label / correction |
| 11 | E1 | LATER | LongMemEval-M / LongMemEval-V2 | Mở rộng sang context lớn hơn hoặc agentic trajectory memory |

---

## 4. Next action chính: A0.5 — Oracle + Diagnostic Benchmark

### 4.1. Mục tiêu

A0.5 cần trả lời câu hỏi:

```text
Trong 132 failures của A0:
- Bao nhiêu lỗi là do retrieval không lấy được evidence?
- Bao nhiêu lỗi là do retrieve được evidence nhưng reader suy luận sai?
- Bao nhiêu lỗi là do aggregation/counting/temporal filtering?
- Bao nhiêu lỗi là do abstention hoặc judge ambiguity?
```

Không nên chuyển sang LongMemEval-M hoặc xây full memory manager trước khi trả lời được các câu hỏi này.

---

## 5. A0.5.1 — Oracle evidence reader

### 5.1. Mục tiêu

Đo reader có trả lời đúng không nếu được đưa đúng gold evidence sessions.

Câu hỏi cần trả lời:

```text
Nếu retrieval hoàn hảo ở session-level, accuracy tăng lên bao nhiêu?
```

### 5.2. Cách tạo oracle setting

Tạo dataset oracle từ LongMemEval-S cleaned:

```text
Input:
  longmemeval_s_cleaned.json

For each answerable example:
  - lấy answer_session_ids
  - chỉ giữ các sessions có session_id nằm trong answer_session_ids
  - bỏ abstention examples hoặc xử lý riêng
```

Output:

```text
longmemeval_s_cleaned_answerable_oracle.json
```

### 5.3. Bảng cần báo cáo

| Setting | Examples | Accuracy | Avg prompt tokens | Notes |
|---|---:|---:|---:|---|
| A0 BM25 session@5, all examples | 500 | 0.7360 | 16,789.94 | Current baseline |
| A0 BM25 session@5, answerable only | 470 | TBD | TBD | Exclude abstention |
| A0.5 Oracle evidence sessions | 470 | TBD | TBD | Gold evidence context |
| A0.5 Oracle + concise reader | 470 | TBD | TBD | Optional prompt ablation |

### 5.4. Cách đọc oracle gap

| Quan sát | Diễn giải | Next action |
|---|---|---|
| Oracle ≥ 0.85 | Retrieval/context selection là bottleneck lớn | Làm A1 turn/chunk retrieval |
| Oracle 0.75–0.85 | Retrieval và reader đều có lỗi | Làm A1 + A2 song song |
| Oracle gần A0 | Reader/prompt/reasoning là bottleneck chính | Ưu tiên A2 evidence-table reader |
| Oracle temporal vẫn thấp | Cần temporal event extraction | Làm A3/A5 sớm |
| Oracle multi-session vẫn thấp | Cần reflection/synthesis | Làm A4 sớm |

---

## 6. A0.5.2 — Failed-case taxonomy

### 6.1. Mục tiêu

Tạo `failed_cases_diagnostic.csv` cho 132 failed cases.

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

### 6.2. Error labels đề xuất

| Error label | Mô tả |
|---|---|
| `retrieval_miss` | Không retrieve được gold session nào trong top-k |
| `partial_evidence` | Retrieve được một phần gold sessions nhưng thiếu evidence khác |
| `retrieval_rank_low` | Gold evidence có nhưng rank thấp hoặc bị context nhiễu che mất |
| `reader_failure` | Evidence có trong context nhưng answer sai |
| `aggregation_error` | Đếm/cộng/tổng hợp sai |
| `temporal_filter_error` | Lọc sai theo thời gian |
| `event_status_error` | Nhầm planned/intended event với completed/actual event |
| `semantic_category_error` | Hiểu sai category cần tính hoặc cần lọc |
| `preference_inference_error` | Suy luận sai preference từ evidence |
| `abstention_false_answer` | Cần abstain nhưng model trả lời |
| `abstention_false_refusal` | Có answer nhưng model từ chối |
| `judge_noise_or_format` | Answer có vẻ đúng nhưng judge chấm sai hoặc final format không rõ |
| `truncation_or_no_final` | Output bị cắt hoặc không có final answer rõ ràng |
| `over_context_noise` | Có evidence nhưng quá nhiều session nhiễu khiến reader sai |

### 6.3. Audit thủ công tối thiểu

Không cần label tay toàn bộ 132 ngay. Audit trước:

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

Output:

```text
A0_failed_case_taxonomy_sample57.csv
A0_failed_case_taxonomy_summary.json
```

### 6.4. Bảng diagnostic cần có

| Eval type | Failures | Retrieval miss | Partial evidence | Reader failure | Aggregation error | Temporal error | Judge/format |
|---|---:|---:|---:|---:|---:|---:|---:|
| multi-session | 55 | TBD | TBD | TBD | TBD | TBD | TBD |
| temporal-reasoning | 39 | TBD | TBD | TBD | TBD | TBD | TBD |
| single-session-preference | 10 | TBD | TBD | TBD | TBD | TBD | TBD |
| abstention | 7 | TBD | TBD | TBD | TBD | TBD | TBD |

---

## 7. A0.5.3 — Completion length diagnostic

### 7.1. Lý do

A0 đang dùng generation length tương đối ngắn so với một số câu hỏi multi-session/temporal.

Average completion tokens:

```text
213.88
```

Nếu nhiều output chạm max token limit, một số answer có thể bị thiếu final answer hoặc thiếu bằng chứng.

### 7.2. Metrics cần thêm

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

### 7.3. Prompt ablation nhỏ

Chạy lại một subset failed cases với 3 format:

| Reader format | Mục tiêu |
|---|---|
| Current CoT-style | Baseline hiện tại |
| Concise final-only | Giảm truncation/format noise |
| Evidence-table reader | Giảm lỗi aggregation/temporal |

Không cần chạy full 500 ngay. Chạy trước trên:

```text
20 multi-session failures
20 temporal-reasoning failures
```

---

## 8. A0.5.4 — Chuẩn hóa retrieval metrics

Hiện cần thống nhất một bộ retrieval metrics duy nhất.

### 8.1. Population cần ghi rõ

Mỗi retrieval metric phải nói rõ tính trên tập nào:

```text
all examples?
answerable-only?
non-abstention only?
exclude no-user-target examples?
session-level hay turn-level?
```

### 8.2. Metrics chuẩn đề xuất

| Metric | Định nghĩa |
|---|---|
| `hit_any@k` | Có ít nhất một gold session trong top-k |
| `hit_all@k` | Tất cả gold sessions nằm trong top-k |
| `gold_fraction@k` | Tỷ lệ gold sessions được retrieve |
| `mrr@k` | Reciprocal rank của gold session đầu tiên |
| `ndcg@k` | Ranking quality |
| `avg_context_tokens@k` | Context cost theo k |
| `qa_accuracy@k` | End-to-end QA theo k nếu chạy generation |

### 8.3. Không nên dùng lẫn lộn

Tránh tình trạng:

```text
run_retrieval.py báo một bộ số
print_retrieval_metrics.py báo một bộ số khác
notebook recompute báo một bộ số khác
```

Cần tạo một script duy nhất:

```text
evaluation/standardize_retrieval_metrics.py
```

Output:

```text
A0_retrieval_metrics_standardized.json
A0_retrieval_metrics_standardized.csv
```

---

## 9. A0.5.5 — Prompt leakage và judge sanity check

### 9.1. Prompt leakage check

Kiểm tra 5–10 prompt thực tế.

Checklist:

```text
- Reader không nhìn thấy gold answer.
- Reader không nhìn thấy answer_session_ids dưới dạng gold label.
- Judge mới nhìn thấy gold answer.
- Retrieval context chỉ gồm retrieved sessions/evidence.
```

### 9.2. Judge consistency check

Manual audit:

```text
20 pass cases
20 fail cases
```

Mục tiêu:

```text
- phát hiện judge chấm sai;
- phát hiện answer đúng nhưng format không rõ;
- phát hiện answer gần đúng nhưng thiếu chi tiết;
- phát hiện hallucination.
```

Output:

```text
A0_judge_manual_audit.csv
```

---

## 10. Decision sau A0.5

Sau A0.5, quyết định bước tiếp theo theo bảng sau.

| Kết quả A0.5 | Diễn giải | Hành động |
|---|---|---|
| Oracle accuracy cao hơn A0 nhiều | Retrieval/context selection là bottleneck chính | Làm A1 turn/chunk retrieval |
| Oracle chỉ cao hơn A0 ít | Reader/prompt/reasoning là bottleneck chính | Làm A2 evidence-table reader trước |
| Oracle tốt ở factual nhưng vẫn thấp ở temporal | Cần event/time extraction | Làm A3 + A5 sớm |
| Oracle tốt ở factual nhưng vẫn thấp ở multi-session | Cần synthesis/reflection | Làm A4 sớm |
| Completion hit max token cao | Output bị truncate hoặc quá dài | Rút gọn prompt hoặc tăng max tokens |
| Nhiều `partial_evidence` | Session-level top-k không đủ multi-evidence | A1 turn/chunk + query decomposition |
| Nhiều `aggregation_error` | Reader không tổng hợp tốt | A2 evidence-table reader |
| Nhiều `abstention_false_answer` | Evidence sufficiency chưa calibrated | Thêm answerability detector / threshold |

---

## 11. A1 — BM25 turn/chunk retrieval

### 11.1. Mục tiêu

Thay vì retrieve full sessions, retrieve đơn vị evidence nhỏ hơn:

```text
turn-level
chunk-level
event-level về sau
```

Mục tiêu:

```text
- giảm context tokens;
- tăng evidence precision;
- giảm over-context noise;
- hỗ trợ reader tập trung vào đúng bằng chứng.
```

### 11.2. Indexing units

| Unit | Mô tả | Ưu điểm | Nhược điểm |
|---|---|---|---|
| session | Full session | Giữ nhiều context | Rất nhiễu, tốn token |
| turn | Một user/assistant turn | Chính xác, ít token | Dễ mất context |
| sliding chunk | 2–4 turns liên tiếp | Cân bằng context và precision | Cần tune chunk size |
| event/fact memory | Extracted memory object | Compact, structured | Cần writer/extractor |

### 11.3. Metadata bắt buộc

Mỗi retrieved item phải giữ:

```json
{
  "evidence_id": "session_12_turn_4",
  "session_id": "session_12",
  "turn_id": 4,
  "date": "2023-05-28",
  "role": "user",
  "text": "...",
  "score": 12.4
}
```

### 11.4. Output reader context mới

Không đưa full session. Đưa compact evidence:

```text
[Evidence 1]
date: ...
session_id: ...
turn_id: ...
role: ...
text: ...

[Evidence 2]
...
```

### 11.5. Metrics chính

| Metric | Kỳ vọng |
|---|---|
| `turn_recall@k` | Tăng |
| `evidence_precision@k` | Tăng |
| `avg_prompt_tokens` | Giảm mạnh |
| `multi-session accuracy` | Tăng nếu đủ evidence |
| `temporal accuracy` | Tăng nếu giữ date metadata |

---

## 12. A2 — Evidence-table reader

### 12.1. Mục tiêu

Giảm lỗi:

```text
aggregation_error
temporal_filter_error
event_status_error
semantic_category_error
```

Đây là các lỗi rất có khả năng xuất hiện trong multi-session và temporal-reasoning.

### 12.2. Prompt format đề xuất

Reader nên được yêu cầu tạo bảng evidence ngắn trước khi trả lời.

Format:

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

### 12.3. Output format

```text
Evidence table:
| evidence_id | date | event | value | include? | reason |
|---|---|---|---:|---|---|
| s2_t4 | March 12 | charity walk | 250 | yes | user participated |
| s5_t2 | May 05 | bike-a-thon | 600 | yes | user participated |
| s4_t1 | June 01 | planned fundraiser | N/A | no | only planned |

Final answer: $850
```

### 12.4. Evaluation

Chạy trước trên subset:

```text
all multi-session failures from A0
all temporal-reasoning failures from A0
```

Sau đó mới chạy full 500 nếu kết quả tốt.

Metrics:

| Metric | Kỳ vọng |
|---|---|
| `aggregation_error_rate` | Giảm |
| `temporal_filter_error_rate` | Giảm |
| `event_status_error_rate` | Giảm |
| `multi-session accuracy` | Tăng |
| `temporal-reasoning accuracy` | Tăng |
| `completion_tokens` | Có thể tăng nhẹ, cần kiểm soát |

---

## 13. A3 — Adaptive Writer

### 13.1. Mục tiêu

Tạo memory store có cấu trúc từ raw history.

Các lớp memory:

```text
raw evidence
fact memory
event memory
preference memory
reflection memory
versioned memory
```

### 13.2. Schema raw evidence

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

### 13.3. Schema memory object

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

### 13.4. Writer focus sau A0

Vì A0 yếu nhất ở multi-session và temporal, writer nên ưu tiên:

```text
event extraction
date extraction
value/count extraction
preference extraction
event status extraction: planned / completed / cancelled / intended
evidence_ids for every memory
```

### 13.5. Metrics

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

## 14. A4 — Evidence-Grounded Reflective Summarizer

### 14.1. Mục tiêu

A0 cho thấy multi-session là nhóm yếu nhất:

```text
multi-session accuracy = 0.5455
failures = 55
```

Do đó reflection nên được ưu tiên trước Versioned Updater.

### 14.2. Nguyên tắc

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

### 14.3. Reflection schema

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

### 14.4. Reflection validator

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

### 14.5. Metrics

| Metric | Mục tiêu |
|---|---|
| `reflection_support_rate` | Cao |
| `unsupported_reflection_error_rate` | Thấp |
| `reflection_helpfulness` | Tăng multi-session accuracy |
| `reflection_usage_rate` | Reader có dùng reflection |
| `token_saving_vs_raw_sessions` | Giảm context tokens |

---

## 15. A5 — Temporal/Event-aware Retriever

### 15.1. Mục tiêu

A0 temporal-reasoning còn thấp:

```text
temporal-reasoning accuracy = 0.6929
failures = 39
```

Cần retrieval có hiểu:

```text
date
time window
event status
question_date
before / after / during
planned vs completed
current vs past
```

### 15.2. Strategy

| Query type | Strategy |
|---|---|
| ask current state | Ưu tiên active/latest memory |
| ask past state | Lọc memory valid tại thời điểm được hỏi |
| ask event count | Retrieve event memories + date/value/status |
| ask before/after | Time-window filter |
| ask changed/updated | Retrieve old + new evidence |
| ask plan vs actual | Ưu tiên completed/confirmed events, loại planned-only nếu question yêu cầu actual |

### 15.3. Metrics

| Metric | Mục tiêu |
|---|---|
| `temporal_accuracy` | Tăng |
| `temporal_filter_error_rate` | Giảm |
| `event_status_error_rate` | Giảm |
| `date_extraction_error_rate` | Giảm |
| `validity_filter_precision` | Tăng |

---

## 16. A6 — Versioned Memory Updater

### 16.1. Mục tiêu

Dù không phải bottleneck đầu tiên sau A0, Versioned Updater vẫn là module quan trọng để định vị Agentic AI memory.

Không ghi đè thô. Dùng:

```text
supersedes
valid_from
valid_until
status
active_at(question_date)
```

### 16.2. Update schema

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

### 16.3. Conflict/update detection labels

```text
SAME_FACT
ELABORATION
CONTRADICTION
PREFERENCE_CHANGE
TEMPORAL_UPDATE
IRRELEVANT
```

### 16.4. Metrics

| Metric | Mục tiêu |
|---|---|
| `stale_memory_error_rate` | Giảm |
| `active_fact_accuracy` | Tăng |
| `past_fact_accuracy` | Tăng |
| `update_chain_recall` | Tăng |
| `conflict_detection_accuracy` | Tăng |

---

## 17. A7 — Adaptive Retriever / Query Router

### 17.1. Nguyên tắc

Không dùng gold `question_type` trong final system.

Có thể dùng gold type cho:

```text
oracle-router diagnostic
upper bound
analysis only
```

Final system phải tự predict intent.

### 17.2. Query intent labels

```text
factual
preference
temporal
update
multi_session
abstention_risk
```

### 17.3. Retrieval strategy theo intent

| Intent | Strategy |
|---|---|
| factual | fact memory + raw evidence fallback |
| preference | preference memory + active/latest preference |
| temporal | event memory + date/time filter |
| update | active memory + superseded chain |
| multi_session | reflection + supporting evidence |
| abstention_risk | evidence sufficiency threshold |

### 17.4. Scoring function

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

### 17.5. Evaluation

| Setting | Dùng gold type? | Mục đích |
|---|---|---|
| A7-oracle-router | Có | Upper bound |
| A7-predicted-router | Không | Kết quả chính |
| A7-fixed-retriever | Không | Baseline |
| A7-without-temporal-filter | Không | Ablation |
| A7-without-confidence-threshold | Không | Ablation abstention |

---

## 18. A8 — Retrospective Learner

### 18.1. Mục tiêu

Sau khi trả lời, dùng feedback để cập nhật:

```text
memory_score
retrieval_score
confidence_threshold
staleness_penalty
router weights
```

### 18.2. Protocol tránh leakage

Không dùng gold label của final test để tune rồi báo cáo trên chính test đó.

Dùng:

```text
dev_stratified: tune thresholds, weights, router prompt
test_final: freeze config, run once
```

### 18.3. Feedback signals

| Signal | Dùng ở đâu |
|---|---|
| retrieved evidence used by reader | online/self-supervised |
| LLM citation agreement | online/self-supervised |
| judge label | dev only |
| gold answer_session_ids | dev/analysis only |
| user correction | simulated hoặc future interactive |
| abstention false positive | dev calibration |

### 18.4. Score update rules

```text
memory_score += 0.1 nếu memory được dùng trong answer đúng
memory_score -= 0.1 nếu memory retrieved nhưng gây answer sai
retriever_weight_temporal += nếu temporal query bị retrieval_miss
confidence_threshold += nếu nhiều abstention false positives
staleness_penalty += nếu knowledge-update dùng old memory
```

---

## 19. Bảng ablation cập nhật

| ID | System | Status | Overall | Multi-session | Temporal | Update | Abstention | Tokens/query | Mục đích |
|---|---|---|---:|---:|---:|---:|---:|---:|---|
| A0 | BM25 session@5 | DONE | 0.7360 | 0.5455 | 0.6929 | 0.9028 | 0.7667 | 16,789.94 | Baseline |
| A0.5 | Oracle evidence reader | NEXT | TBD | TBD | TBD | TBD | TBD | TBD | Tách retrieval vs reader |
| A1 | BM25 turn/chunk | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Evidence-level retrieval |
| A2 | Evidence-table reader | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Giảm aggregation/temporal errors |
| A3 | Adaptive Writer | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Structured memory |
| A4 | Evidence-grounded Reflection | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Multi-session synthesis |
| A5 | Temporal/Event-aware Retriever | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Temporal reasoning |
| A6 | Versioned Updater | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Stale/update handling |
| A7 | Adaptive Retriever | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Intent-aware retrieval |
| A8 | Retrospective Learner | TODO | TBD | TBD | TBD | TBD | TBD | TBD | Feedback-driven scoring |

---

## 20. Metrics cần báo cáo xuyên suốt

### 20.1. QA metrics

| Metric | Mục đích |
|---|---|
| `overall_accuracy` | Kết quả end-to-end |
| `macro_accuracy_by_type` | Tránh overall bị che bởi nhóm lớn |
| `accuracy_by_eval_type` | Biết module nào giúp nhóm nào |
| `abstention_accuracy` | Đánh giá refusal/no-answer |
| `false_answer_rate` | Tỷ lệ bịa khi nên abstain |

### 20.2. Retrieval metrics

| Metric | Mục đích |
|---|---|
| `hit_any@k` | Có ít nhất một gold evidence không |
| `hit_all@k` | Có đủ toàn bộ gold evidence không |
| `gold_fraction@k` | Tỷ lệ gold evidence retrieved |
| `mrr@k` | Evidence rank |
| `ndcg@k` | Ranking quality |
| `evidence_precision@k` | Giảm nhiễu |
| `oracle_gap` | Tách retrieval vs reader |

### 20.3. Reader/reasoning metrics

| Metric | Mục đích |
|---|---|
| `aggregation_error_rate` | Đếm/cộng/tổng hợp sai |
| `temporal_filter_error_rate` | Lọc thời gian sai |
| `event_status_error_rate` | Nhầm planned vs actual |
| `semantic_category_error_rate` | Lọc sai category |
| `no_final_answer_rate` | Output không rõ final |
| `hit_max_tokens_rate` | Bị giới hạn output |

### 20.4. Memory metrics

| Metric | Mục đích |
|---|---|
| `memory_compression_ratio` | Raw tokens / memory tokens |
| `memory_support_rate` | Memory có evidence support |
| `unsupported_memory_rate` | Memory unsupported |
| `reflection_support_rate` | Reflection grounded |
| `unsupported_reflection_error_rate` | Reflection gây hallucination |
| `stale_memory_error_rate` | Dùng memory hết hiệu lực |
| `validity_filter_precision` | Lọc đúng memory theo thời gian |

### 20.5. Efficiency metrics

| Metric | Mục đích |
|---|---|
| `avg_prompt_tokens` | Context cost |
| `avg_completion_tokens` | Answer cost |
| `latency_per_query` | Practicality |
| `memory_build_cost` | Chi phí build memory |
| `storage_size` | Dung lượng memory store |

---

## 21. Implementation structure cập nhật

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
    A1_bm25_turn_chunk/
    A2_evidence_table_reader/
  reports/
```

---

## 22. Chuẩn output cho mỗi experiment

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

A0.5 nên có thêm:

```text
A05_oracle_diagnostic/
  oracle_dataset.json
  oracle_hypotheses.jsonl
  oracle_eval_log.jsonl
  oracle_summary.json
  failed_case_taxonomy_sample57.csv
  failed_case_taxonomy_summary.json
  completion_length_diagnostic.json
  retrieval_metrics_standardized.json
  judge_manual_audit.csv
  prompt_leakage_check.md
```

---

## 23. Checklist ngay bây giờ

### A0 — Done / cần đóng gói

- [x] Chạy full A0 BM25 session trên LongMemEval-S cleaned.
- [x] Có overall accuracy.
- [x] Có accuracy theo eval_type.
- [x] Có retrieval metrics.
- [x] Có token/cost/latency.
- [x] Có failed examples.
- [ ] Chuẩn hóa retrieval metrics bằng một script duy nhất.
- [ ] Đóng gói A0 summary thành `A0_bm25_session_report.md`.

### A0.5 — Next action

- [ ] Tạo oracle dataset từ `answer_session_ids`.
- [ ] Chạy oracle evidence reader trên answerable examples.
- [ ] So sánh A0 vs Oracle.
- [ ] Tạo failed-case taxonomy.
- [ ] Manual audit 57 failed cases trọng điểm.
- [ ] Tính completion length diagnostic.
- [ ] Kiểm tra prompt leakage.
- [ ] Audit judge consistency trên 20 pass + 20 fail cases.
- [ ] Ra quyết định A1 hay A2 là bước tiếp theo dựa trên oracle gap.

---

## 24. Việc không nên làm ngay

Không nên làm các việc sau ở thời điểm hiện tại:

```text
- Không chạy LongMemEval-M ngay.
- Không tăng top-k session lên 10/20 như hướng chính.
- Không implement full 5-module memory manager ngay.
- Không ưu tiên Versioned Updater trước khi hiểu rõ lỗi multi-session/temporal.
- Không dùng gold question_type trong final retriever.
- Không chỉ báo cáo overall accuracy.
- Không claim SOTA.
- Không bỏ qua token cost.
```

Lý do:

```text
A0 đã cho thấy vấn đề chính không đơn thuần là thiếu nhiều session hơn.
Vấn đề nằm ở evidence granularity, multi-evidence synthesis, temporal/event reasoning,
và khả năng reader xử lý structured evidence.
```

---

## 25. Contribution statements cập nhật

### Contribution 1 — Reproducible A0 baseline and diagnostic insight

> We reproduce a full LongMemEval-S cleaned BM25 session-level baseline and show that session retrieval performs strongly on single-session factual and knowledge-update questions, but fails substantially on multi-session and temporal reasoning.

### Contribution 2 — Oracle-gap diagnostic protocol

> We introduce an oracle-evidence diagnostic protocol to separate retrieval failures from reader/reasoning failures in long-term memory evaluation.

### Contribution 3 — Evidence-level retrieval and structured reading

> We propose turn/chunk-level evidence retrieval combined with evidence-table reading to reduce context noise and address aggregation, temporal filtering, and event-status errors.

### Contribution 4 — Evidence-grounded memory representation

> We propose a memory representation where every fact, event, preference and reflection is linked to evidence_ids and confidence, enabling auditability and reducing unsupported memory use.

### Contribution 5 — Grounded reflection for multi-session memory

> We investigate evidence-grounded reflection as a mechanism for improving multi-session synthesis while preserving traceability to raw evidence.

### Contribution 6 — Temporal and versioned memory

> We extend the memory manager with temporal/event-aware retrieval and versioned update to reduce stale-memory and temporal-validity errors.

---

## 26. Kết luận cập nhật

A0 đã xác nhận hướng nghiên cứu là hợp lý.

Kết quả quan trọng nhất không phải là:

```text
BM25 session đạt 0.7360 accuracy.
```

Mà là:

```text
BM25 session-level retrieval khá tốt ở factual/update questions,
nhưng yếu rõ ở multi-session và temporal reasoning,
trong khi token cost rất cao và nhiều lỗi không chỉ do retrieval miss.
```

Vì vậy, next action đúng là:

```text
A0.5 Oracle + Diagnostic Benchmark
```

Sau đó nhiều khả năng đi tiếp theo hướng:

```text
A1 BM25 turn/chunk retrieval
+
A2 Evidence-table reader
```

Đây là đường đi ngắn nhất để biến kết quả A0 thành một đóng góp nghiên cứu rõ ràng:

> Memory tốt cho Agentic AI không chỉ là retrieve nhiều hơn, mà là retrieve đúng evidence ở đúng granularity, tổ chức evidence thành memory có cấu trúc, hỗ trợ temporal/event reasoning, và đảm bảo mọi reflection đều truy ngược được về bằng chứng.