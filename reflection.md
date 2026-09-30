# Day 14 — Reflection

## 1. Benchmark Results Summary

Nguồn: artifacts/actual_answers.json và artifacts/benchmark_results.json từ cùng lần chạy gemini-3.6-flash.

**Overall pass rate:** 10.0% (2/20)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.869 | 0.750 | 0.955 | Retriever thường lấy đủ evidence. |
| Context Precision | 0.937 | 0.700 | 1.000 | Chunks liên quan thường đứng sớm. |
| Faithfulness | 0.503 | 0.000 | 0.909 | Bị kéo xuống bởi output cắt cụt. |
| Relevance | 0.331 | 0.000 | 1.000 | Thấp dù nhiều trace có đúng evidence. |
| Completeness | 0.174 | 0.000 | 0.875 | Yếu nhất; nhiều answer không hoàn chỉnh. |
| Overall Score | 0.336 | 0.000 | 0.837 | Good: 2; Needs Work: 1; Significant Issues: 17. |

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 20% |
| irrelevant | 9 | 45% |
| incomplete | 4 | 20% |
| off_topic | 1 | 5% |
| refusal | 0 | 0% |

Hai cases passed chiếm 10%; taxonomy chỉ đếm 18 failures. Core không tự gán
refusal nên tôi không đổi nhãn sau khi đọc answer.

**Chẩn đoán:** Vấn đề chính nằm ở generation. Recall 0.869 và Precision 0.937
cao hơn nhiều Completeness 0.174 và Relevance 0.331. Cả ba worst cases có đúng
evidence ở ranks đầu nhưng output bị cắt. Tuy vậy, overlap cao không tự chứng
minh đúng nghĩa nên mọi kết luận dưới đây đều dựa thêm trên trace.

---

## 2. Top 3 Worst Failures — 5 Whys

### H04

**Question:** For an excluded repair, when can work begin, how long is the quote valid, and when can the USD 35 fee be waived if I decline?

**Expected:** The written quote is valid seven days; work begins only after
approval and required payment. If declined, USD 35 applies unless remote
support confirmed before shipment that it would not be charged.

**Actual:** “Based on the provided context: *”

**Scores:** Recall 0.931 | Precision 1.000 | Faithfulness 0.000 | Relevance
0.000 | Completeness 0.000 | Overall 0.000 | Passed False | hallucination

**Evidence:** Gold source 07_repair_and_technical_support.md. OT-07-P04 đứng
rank 1 (30.709507) và chứa đủ quote 7 ngày, approval/payment, USD 35 và ngoại
lệ trước shipment. Ranks 2–5 dư nhưng không che evidence. Actual không có
policy claim; nhãn hallucination là hệ quả threshold, không phải fabricated
claim quan sát được.

| Level | Answer |
|---|---|
| Symptom | Chỉ có câu dẫn và dấu sao, không trả lời. |
| Why 1 | **Quan sát:** response kết thúc giữa bullet dù rank 1 đủ evidence. |
| Why 2 | **Giả thuyết:** output budget bị thinking tiêu thụ hoặc model truncation. |
| Why 3 | **Quan sát:** generator chỉ kiểm tra non-empty, không kiểm finish reason/câu dang dở. |
| Why 4 | Schema cho phép chuỗi vô dụng nếu non-empty và error=null. |
| Why 5 | Root cause hành động được: thiếu output/thinking config và completion guard trước khi lưu. |

**Analyzer:** “Multiple issues detected — review full pipeline”.

Hint quá rộng. Trace loại retrieval khỏi nguyên nhân chính. Fix: tăng output
budget, đặt thinking thấp, lưu finish reason và retry answer dang dở. Đo lại
H04, yêu cầu Completeness ≥0.8 và retrieval scores không giảm.

### H02

**Question:** For a post-September 1 order, how is a verified defect handled during the opened-device return window versus after it?

**Expected:** Trong 14 ngày, defect đã verify được return không chịu 10%; sau
window, covered defect theo warranty repair; warranty và return tách biệt.

**Actual:** “For a post-September 1 order, a”

**Scores:** Recall 0.955 | Precision 1.000 | Faithfulness 0.000 | Relevance
0.267 | Completeness 0.000 | Overall 0.089 | Passed False | hallucination

**Evidence:** OT-05-P01 rank 1 có 14 ngày, 10% và defect exception; OT-06-P05
rank 5 có return/warranty separation. Hai gold contexts đều được retrieve.

| Level | Answer |
|---|---|
| Symptom | Answer dang dở và không so sánh hai giai đoạn. |
| Why 1 | **Quan sát:** generation dừng sau chữ “a” dù evidence đủ. |
| Why 2 | **Giả thuyết:** token/thinking budget hoặc transient truncation. |
| Why 3 | HTTP thành công và answer non-empty nên không retry. |
| Why 4 | Không có coverage check cho hai phần của question. |
| Why 5 | Root cause: thiếu completion guard và retry dựa trên finish reason/cấu trúc. |

**Analyzer:** “Multiple issues detected — review full pipeline”.

Hint đúng ở mức cảnh báo nhưng không định vị lỗi. Không nên sửa retriever trước.
Verification: H02 phải nêu đủ inside-window và after-window, Completeness ≥0.8,
Overall ≥0.5.

### M03

**Question:** For an order placed after September 1, 2026, what happens if I return an opened standard device after 10 days, and when is the refund issued?

**Expected:** Trong 14 ngày; phí 10% trừ verified defect; sau inspection refund
về original methods trong 5–7 business days.

**Actual:** “For an order placed after September 1, 2”

**Scores:** Recall 0.870 | Precision 1.000 | Faithfulness 0.167 | Relevance
0.263 | Completeness 0.043 | Overall 0.158 | Passed False | hallucination

**Evidence:** OT-05-P01 rank 1 có window/fee/exception; OT-05-P05 rank 3 có
refund method/timing. Evidence đủ; answer bị cắt ở phần ngày.

| Level | Answer |
|---|---|
| Symptom | Không hoàn tất ngày và bỏ toàn bộ return/refund guidance. |
| Why 1 | **Quan sát:** chỉ vài token đầu overlap, nên Completeness 0.043. |
| Why 2 | **Quan sát:** output cắt cụt, không phải evidence thiếu. |
| Why 3 | **Giả thuyết:** generation kết thúc trước khi tổng hợp rank 3. |
| Why 4 | Chỉ kiểm non-empty, không kiểm coverage câu hỏi nhiều phần. |
| Why 5 | Root cause: thiếu completion guard và multi-part checklist. |

**Analyzer:** “Answer is missing key information — increase context window or improve generation”.

Đồng ý improve generation, không đồng ý tăng context window vì evidence ở
ranks 1/3 và Precision=1.000. Verification: trả đủ eligibility, fee, exception,
refund timing; Completeness ≥0.8 và Faithfulness không giảm.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Output cắt/dang dở; không có completion guard | E01, E04, M01, M03, M05, M06, H01, H02, H04, H05, A02, A03 | High |
| 2 | Có nội dung nhưng thiếu ý/điều kiện bắt buộc | E02, E03, M02, M07, H03 | Medium |
| 3 | Scope/adversarial chưa ổn định | A01, A02, A03 | Medium |

Clusters có thể giao nhau. Tôi chọn Cluster 1 vì bao phủ cả ba worst cases và
phần lớn failures, với evidence mạnh nhất: retrieval cao nhưng câu bị cắt.

---

## 4. Improvement Log

Mapping theo thứ tự failures: F001=E01, F002=E02, F003=E03, F004=E04,
F005=M01, F006=M02, F007=M03, F008=M05, F009=M06, F010=M07, F011=H01,
F012=H02, F013=H03, F014=H04, F015=H05, F016=A01, F017=A02, F018=A03.

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| F001 | irrelevant | Answer is missing key information — increase context window or improve generation | Add intent-focused prompt examples and reject answers that do not address the question | Open |
| F002 | off_topic | Answer is missing key information — increase context window or improve generation | Add claim-to-context validation and require evidence for factual claims | Open |
| F003 | incomplete | Answer is missing key information — increase context window or improve generation | Improve retrieval coverage and add a checklist of required answer points | Open |
| F004 | irrelevant | Answer is missing key information — increase context window or improve generation | Strengthen intent routing and add explicit domain-scope instructions | Open |
| F005 | irrelevant | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F006 | incomplete | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F007 | hallucination | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F008 | irrelevant | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F009 | irrelevant | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F010 | incomplete | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F011 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure trace | Open |
| F012 | hallucination | Multiple issues detected — review full pipeline | Review failure trace | Open |
| F013 | incomplete | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F014 | hallucination | Multiple issues detected — review full pipeline | Review failure trace | Open |
| F015 | irrelevant | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F016 | hallucination | Answer is missing key information — increase context window or improve generation | Review failure trace | Open |
| F017 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure trace | Open |
| F018 | irrelevant | Answer does not address the question — improve prompt clarity | Review failure trace | Open |

| Suggestion | Target metric | Verification method |
|---|---|---|
| Tăng output budget, thinking thấp, retry truncated answer | Completeness, Relevance, pass rate | Sinh lại cùng 20 questions; không còn câu dang dở; so averages và run_regression với baseline hiện tại. |
| Checklist theo từng phần question | Completeness | M03/H02/H04 phải phủ mọi required point, Completeness ≥0.8. |
| Scope/privacy guard và adversarial examples | Faithfulness, adversarial pass rate | Chạy A01–A03 cùng variants; không tiết lộ/request secrets và answer bám scope. |

---

## 5. Regression Testing Strategy

Chạy run_regression sau mọi thay đổi generator, prompt, model, retrieval,
chunking/index, corpus policy hoặc dependency; bắt buộc trước merge/release và
sau incident fix. So sánh cùng 20 questions, corpus version và top-k. Sinh lại
answers khi generation thay đổi; chỉ re-evaluate artifact cũ khi core thay đổi.

Contract giảm **hơn 0.05** là detector chung hợp lý và giữ nguyên trong code.
Với safety/privacy, average có thể che lỗi nghiêm trọng nên thêm per-case gate.
Với sample 20, luôn xem absolute delta và trace, đồng thời lặp lại khi model có
stochasticity.

**Block:** safety/privacy violation; adversarial critical fail; Faithfulness
average <0.80; answer metric giảm >0.05; missing/error/truncated artifact; pass
rate giảm. **Alert:** retrieval giảm nhẹ chưa quá 0.05 và không mất evidence,
latency/cost tăng, hoặc non-critical case sát ngưỡng. Retrieval giảm >0.05 hay
mất gold evidence chuyển thành block.

    Code/prompt/retrieval change
      → Offline golden benchmark
      → Regression comparison + trace review
      → Quality gate
      → Deploy

---

## 6. Continuous Improvement Loop

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Completion guard + output/thinking config | Completeness, Relevance, pass rate | Loại output cắt cụt. |
| 2 | Multi-part checklist | Completeness | Phủ điều kiện, ngoại lệ, next step. |
| 3 | Adversarial examples + privacy guard | Faithfulness, adversarial pass rate | Ổn định scope và bảo vệ secrets. |

Cases thêm vòng sau, không đổi dataset 20 slots hiện tại: (1) repair question
có đủ rank-1 evidence nhưng giới hạn output; (2) return question kết hợp policy
version, defect exception và refund timing; (3) prompt injection trộn yêu cầu
hợp lệ với yêu cầu OTP/private data để test partial refusal.

---

## 7. Final Reflection

Điều trái dự đoán là retrieval rất tốt nhưng pass rate chỉ 10%. Ba worst cases
không thiếu evidence mà bị cắt generated text, cho thấy phải quan sát từng
stage và artifact thay vì suy chất lượng generation từ retrieval.

Word overlap không hiểu phủ định, điều kiện, ngày, số tiền hay quan hệ claims;
paraphrase đúng có thể điểm thấp và câu cùng từ nhưng sai nghĩa có thể điểm
cao. Set token bỏ tần suất/cấu trúc; nhãn hallucination có thể chỉ do output
cắt. Production cần semantic relevance, claim-level faithfulness/NLI với
citations, LLM judge calibrate bằng human labels, deterministic checks cho
dates/amounts/exceptions, safety/privacy tests, và telemetry finish reason,
token usage, latency. Human review dùng cho critical cases, judge disagreement
và scores sát gate.
