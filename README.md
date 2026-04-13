# Nhóm 14 — E402 — Lab Day 08: RAG Pipeline

## Thông tin bài Lab

| Thông tin | Chi tiết |
|-----------|----------|
| **Môn học** | AI in Action (AICB-P1) |
| **Chủ đề** | RAG Pipeline: Indexing → Retrieval → Generation → Evaluation |
| **Thời gian** | 4 giờ (4 sprints x 60 phút) |
| **Mục tiêu** | Xây dựng trợ lý nội bộ cho khối CS + IT Helpdesk, trả lời câu hỏi về chính sách, SLA ticket, quy trình cấp quyền và FAQ |

### Mô tả
Nhóm xây dựng hệ thống RAG hoàn chỉnh: từ indexing tài liệu chính sách, retrieval thông tin, generation câu trả lời có citation, đến đánh giá pipeline bằng scorecard và A/B comparison.

---

## Thành viên nhóm

| STT | Họ và tên | Vai trò | Trách nhiệm chính |
|-----|-----------|---------|-------------------|
| 1 | **Chu Tuấn Nghĩa** | Tech Lead | Code chính RAG pipeline, implement rerank và transform_query |
| 2 | **Mai Đức Thuận** | Documentation Owner | Viết architecture.md, tuning-log.md; implement sparse/hybrid retrieval |
| 3 | **Trần Trung Hiếu** | Eval Owner | Thiết kế và implement hệ thống đánh giá (eval.py), scorecard, A/B comparison |

---

## Cấu trúc dự án & Đường dẫn file

### Code

| File | Mô tả | Sprint |
|------|-------|--------|
| [`lab/index.py`](lab/index.py) | Indexing pipeline: Preprocess → Chunk → Embed → Store (ChromaDB) | 1 |
| [`lab/rag_answer.py`](lab/rag_answer.py) | Retrieval + Generation: Dense/Hybrid search, Rerank, Query Expansion, Grounded Answer | 2, 3 |
| [`lab/eval.py`](lab/eval.py) | Evaluation: Scorecard, A/B Comparison, LLM-as-Judge scoring | 4 |

### Tài liệu thiết kế

| File | Mô tả |
|------|-------|
| [`lab/docs/architecture.md`](lab/docs/architecture.md) | Mô tả kiến trúc pipeline: chunking decision, retrieval config, embedding model, diagram |
| [`lab/docs/tuning-log.md`](lab/docs/tuning-log.md) | Ghi lại A/B experiments: baseline vs variant, phân tích per-question, kết luận |

### Báo cáo cá nhân

| File | Thành viên |
|------|-----------|
| [`lab/reports/individual/chu_tuan_nghia.md`](lab/reports/individual/chu_tuan_nghia.md) | Chu Tuấn Nghĩa — Tech Lead |
| [`lab/reports/individual/Mai_Duc_Thuan.md`](lab/reports/individual/Mai_Duc_Thuan.md) | Mai Đức Thuận — Documentation Owner |
| [`lab/reports/individual/Tran_Trung_Hieu.md`](lab/reports/individual/Tran_Trung_Hieu.md) | Trần Trung Hiếu — Eval Owner |

### Kết quả đánh giá

| File | Mô tả |
|------|-------|
| [`lab/results/scorecard_baseline.md`](lab/results/scorecard_baseline.md) | Scorecard baseline (dense retrieval) |
| [`lab/results/scorecard_variant.md`](lab/results/scorecard_variant.md) | Scorecard variant (hybrid + rerank + query expansion) |
| [`lab/results/ab_comparison.csv`](lab/results/ab_comparison.csv) | Bảng so sánh A/B dưới dạng CSV |


---

## Kiến trúc hệ thống

```
[Raw Docs] → [index.py: Preprocess → Chunk → Embed → Store] → [(ChromaDB)]
                                                              ↓
[User Query] → [rag_answer.py: Retrieve → Rerank → Generate] → [Answer + Citation]
                                                              ↓
                                              [eval.py: Scorecard + A/B Comparison]
```

---

## Kết quả 
| Metric | Baseline | Variant | Delta |
|--------|----------|---------|-------|
| Faithfulness | 4.80/5 | 4.50/5 | -0.30 |
| Answer Relevance | 4.60/5 | 4.20/5 | -0.40 |
| Context Recall | 5.00/5 | 5.00/5 | 0.00 |
| Completeness | 4.30/5 | 4.10/5 | -0.20 |

---

## Cách chạy

```bash
# 1. Cài dependencies
pip install -r lab/requirements.txt

# 2. Tạo file .env
cp lab/.env.example lab/.env
# Điền OPENAI_API_KEY hoặc GOOGLE_API_KEY

# 3. Chạy end-to-end
python lab/index.py
python lab/rag_answer.py
python lab/eval.py
```

---

