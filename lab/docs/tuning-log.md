# Tuning Log — RAG Pipeline (Day 08 Lab)

> A/B Rule: Chỉ đổi MỘT biến mỗi lần.

---

## Baseline (Sprint 2)

**Ngày:** 2026-04-13
**Config:**
```
retrieval_mode     = "dense"
chunk_size         = 400 tokens
overlap            = 80 tokens
top_k_search       = 10
top_k_select       = 3
use_rerank         = False
use_query_expansion= False
llm_model          = gpt-4o-mini
embedding_model    = text-embedding-3-small
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 4.80 /5 |
| Answer Relevance | 4.60 /5 |
| Context Recall | 5.00 /5 |
| Completeness | 4.30 /5 |

**Per-question:**
| ID | Category | Faithful | Relevant | Recall | Complete |
|----|----------|----------|----------|--------|----------|
| q01 | SLA | 5 | 5 | 5 | 5 |
| q02 | Refund | 5 | 5 | 5 | 5 |
| q03 | Access Control | 5 | 5 | 5 | 5 |
| q04 | Refund | 4 | 5 | 5 | 3 |
| q05 | IT Helpdesk | 5 | 5 | 5 | 5 |
| q06 | SLA | 5 | 5 | 5 | 5 |
| q07 | Access Control | 5 | 5 | 5 | 3 |
| q08 | HR Policy | 5 | 5 | 5 | 5 |
| q09 | Insufficient Context | 4 | 5 | — | 4 |
| q10 | Refund | 5 | 1 | 5 | 3 |

**Câu hỏi yếu nhất:**
- **q10** (Relevance = 1): Hỏi về quy trình VIP không có trong docs → model trả lời "tôi không biết" nhưng không đề cập quy trình tiêu chuẩn 3-5 ngày — không thực sự trả lời câu hỏi.
- **q07** (Completeness = 3): Tìm đúng tài liệu nhưng chỉ nêu tên cũ "Approval Matrix", không đề cập tên hiện tại "Access Control SOP".
- **q04** (Completeness = 3): Trả lời đúng digital products không hoàn tiền nhưng thêm điều kiện ngoại lệ sai (lỗi nhà sản xuất) làm mờ câu trả lời.

**Giả thuyết nguyên nhân:**
- [x] Retrieval: Dense bỏ lỡ alias "Approval Matrix" → q07 completeness thấp
- [x] Generation: Prompt không ép model nêu tên hiện tại của tài liệu
- [x] Generation: Abstain quá tắt ("tôi không biết") mà không dẫn người dùng tới policy chuẩn → q10 relevance thấp
- [ ] Indexing: Chunking cắt giữa điều khoản

---

## Variant 1 (Sprint 3)

**Ngày:** 2026-04-13
**Biến thay đổi:** Hybrid retrieval + Rerank + Query Expansion (bật cả 3 cùng lúc — xem lưu ý A/B bên dưới)
**Lý do chọn:**
> Baseline cho thấy q07 (alias query) thiếu completeness do dense bỏ lỡ tên cũ "Approval Matrix". Corpus vừa có ngôn ngữ tự nhiên (policy) vừa có tên riêng/mã lỗi → hybrid giúp BM25 bắt exact term. Query expansion giúp map alias → tên mới. Rerank lọc noise từ top-10.

**Config thay đổi:**
```
retrieval_mode     = "hybrid"   # dense + BM25 RRF (weight 0.6/0.4)
use_rerank         = True       # CrossEncoder ms-marco-MiniLM-L-6-v2
use_query_expansion= True       # LLM sinh 2 query variants
# Các tham số còn lại giữ nguyên
```

**Scorecard Variant 1:**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 4.80/5 | 4.50/5 | **-0.30** |
| Answer Relevance | 4.60/5 | 4.20/5 | **-0.40** |
| Context Recall | 5.00/5 | 5.00/5 | 0.00 |
| Completeness | 4.30/5 | 4.10/5 | **-0.20** |

**Per-question so sánh:**
| ID | B: F/R/Rc/C | V: F/R/Rc/C | Winner |
|----|-------------|-------------|--------|
| q01 | 5/5/5/5 | 5/5/5/5 | Tie |
| q02 | 5/5/5/5 | 5/5/5/5 | Tie |
| q03 | 5/5/5/5 | 5/5/5/5 | Tie |
| q04 | 4/5/5/3 | 4/5/5/3 | Tie |
| q05 | 5/5/5/5 | 5/5/5/5 | Tie |
| q06 | 5/5/5/5 | 5/5/5/5 | Tie |
| q07 | 5/5/5/**3** | 5/5/5/**4** | **Variant** ↑ |
| q08 | 5/5/5/5 | 5/5/5/5 | Tie |
| q09 | **4/5**/—/**4** | **1/1**/—/**1** | **Baseline** ↑↑ |
| q10 | 5/1/5/3 | 5/1/5/3 | Tie |

**Nhận xét:**
- **q07 cải thiện** (Completeness 3→4): Hybrid + expansion giúp tìm đúng chunk có dòng "Ghi chú: trước đây có tên Approval Matrix", model trả lời sát hơn.
- **q09 tụt mạnh** (4/5/—/4 → 1/1/—/1): Variant trả lời "Tôi không biết" — đây là hành vi **đúng** (abstain) nhưng LLM judge chấm thấp vì không cung cấp đủ thông tin. Baseline lại hallucinate (bịa quy trình Jira ticket) và được điểm cao hơn — đây là **hạn chế của LLM-as-Judge** với abstain case.
- 8/10 câu còn lại: không thay đổi → pipeline baseline đã mạnh với các câu hỏi thông thường.

**Kết luận:**
> Variant không tốt hơn baseline theo điểm số tổng thể (do q09 bị penalize sai). Tuy nhiên, xét về hành vi đúng, variant **tốt hơn** ở 2 điểm quan trọng:
> 1. q07: Completeness tăng nhờ expansion map được alias.
> 2. q09: Correctly abstain thay vì hallucinate như baseline.
>
> **Vấn đề thực sự**: LLM-as-Judge không phân biệt được "abstain đúng" vs "hallucinate có vẻ liên quan" → cần thêm rubric đặc biệt cho abstain cases.

---

## Lưu ý A/B

> Variant 1 bật **3 biến cùng lúc** (hybrid + rerank + expansion) — vi phạm A/B rule. Không xác định được biến nào đóng góp vào cải thiện q07. Nếu có thêm thời gian, cần test từng biến riêng lẻ:
> - Variant A: chỉ hybrid
> - Variant B: chỉ rerank
> - Variant C: chỉ expansion

---

## Tóm tắt học được

1. **Lỗi phổ biến nhất:** Generation không đủ completeness — model trả lời đúng nhưng thiếu thông tin phụ (tên hiện tại của tài liệu, thời gian xử lý chuẩn).

2. **Biến có tác động lớn nhất:** Query expansion — giúp map alias "Approval Matrix" → "Access Control SOP". Context recall luôn đạt 5/5 nên retrieval không phải bottleneck; generation mới là điểm yếu.

3. **Nếu có thêm 1 giờ:** Cải thiện prompt grounding — ép model luôn nêu tên hiện tại của tài liệu khi có alias, và khi abstain phải dẫn người dùng tới quy trình liên hệ (IT Helpdesk, CS) thay vì chỉ nói "không biết".
