# Báo Cáo Cá Nhân — Lab Day 08: RAG Pipeline

**Họ và tên:** Trần Trung Hiếu
**Vai trò trong nhóm:** Eval Owner
**Ngày nộp:** 13/04/2026
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi đã làm gì trong lab này? (100-150 từ)

Với vai trò Eval Owner, tôi chịu trách nhiệm chính ở Sprint 3 và Sprint 4. Cụ thể, tôi thiết kế và implement toàn bộ hệ thống đánh giá trong `eval.py`: implement 3 hàm chấm điểm tự động bằng LLM-as-Judge (`score_faithfulness`, `score_answer_relevance`, `score_completeness`) và tận dụng lại hàm `score_context_recall` đã có sẵn trong skeleton. Tôi cũng cấu hình `BASELINE_CONFIG` và `VARIANT_CONFIG`, chạy `run_scorecard()` cho cả hai trường hợp, sau đó gọi `compare_ab()` để sinh ra bảng so sánh và export `ab_comparison.csv`.

Phần công việc của tôi nằm ở cuối pipeline, phụ thuộc trực tiếp vào output của `rag_answer()` từ Tech Lead và Retrieval Owner. Kết quả scorecard tôi sinh ra cũng là input để Documentation Owner hoàn thành `tuning-log.md`.

---

## 2. Điều tôi hiểu rõ hơn sau lab này (100-150 từ)

Trước lab này, tôi nghĩ evaluation RAG chỉ đơn giản là so sánh answer với expected answer theo kiểu exact match. Nhưng thực tế phức tạp hơn nhiều — mỗi metric đo một chiều khác nhau của pipeline:

- **Context Recall** đo retrieval có tìm đúng tài liệu không — hoàn toàn độc lập với generation.
- **Faithfulness** đo generation có bám vào retrieved context không — độc lập với câu hỏi.
- **Completeness** đo answer có đủ ý so với expected không — đây mới là thứ gần với "chất lượng câu trả lời" nhất.

Nhờ tách ra 4 metrics như vậy, khi một câu hỏi bị điểm thấp, tôi biết ngay lỗi nằm ở tầng nào của pipeline thay vì phải đoán mò. Đây là điều tôi thấy giá trị nhất từ lab này.

---

## 3. Điều tôi ngạc nhiên hoặc gặp khó khăn (100-150 từ)

Điều tôi không ngờ nhất là kết quả của **q09** — câu hỏi về mã lỗi `ERR-403-AUTH` không có trong docs.

Baseline trả lời bằng cách bịa ra một quy trình Jira ticket hoàn toàn không liên quan, nhưng lại được LLM judge chấm **Relevance = 5** vì câu trả lời nghe "có vẻ liên quan". Trong khi đó, variant trả lời "Tôi không biết" — đây là hành vi **đúng theo yêu cầu abstain** — lại bị chấm **Relevance = 1**.

Điều này cho thấy LLM-as-Judge có điểm mù với abstain cases: judge đánh giá theo độ "hữu ích bề ngoài" chứ không phân biệt được "hữu ích đúng" vs "tự tin sai". Đây là hạn chế quan trọng cần lưu ý khi dùng LLM để tự động chấm điểm RAG pipeline.

---

## 4. Phân tích một câu hỏi trong scorecard (150-200 từ)

**Câu hỏi:** q07 — *"Approval Matrix để cấp quyền hệ thống là tài liệu nào?"*

**Phân tích:**

Đây là câu hỏi dùng tên cũ của tài liệu ("Approval Matrix for System Access") trong khi tài liệu hiện tại đã đổi tên thành "Access Control SOP". Đây là alias test — thử thách khả năng tìm kiếm khi user không biết tên mới.

Baseline đạt Completeness = 3: retrieval tìm đúng tài liệu (Recall = 5) nhờ dòng "Ghi chú: tài liệu này trước đây có tên..." được giữ lại trong quá trình indexing, nhưng answer chỉ nêu tên cũ mà không đề cập tên hiện tại "Access Control SOP" — thiếu thông tin quan trọng nhất mà user cần biết.

Variant cải thiện lên Completeness = 4: query expansion giúp sinh thêm variant query gần với tên mới hơn, kéo chunk có thông tin đầy đủ hơn vào context. Tuy nhiên vẫn chưa đạt 5 vì model không chủ động nêu rõ "tên hiện tại là...".

Lỗi nằm ở **generation**: prompt chưa hướng dẫn model khi tìm thấy alias phải nêu rõ cả tên cũ lẫn tên mới. Retrieval đã đúng từ đầu.

---

## 5. Nếu có thêm thời gian, tôi sẽ làm gì? (50-100 từ)

Tôi sẽ tách riêng **rubric cho abstain cases** trong LLM judge. Kết quả eval cho thấy judge hiện tại penalize "Tôi không biết" ngay cả khi đó là câu trả lời đúng (q09). Cụ thể, tôi sẽ thêm bước kiểm tra: nếu `expected_sources` rỗng, judge cần đánh giá xem model có **từ chối trả lời đúng cách** không thay vì chấm theo độ chi tiết của câu trả lời.
