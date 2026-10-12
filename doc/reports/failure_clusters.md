# Phân tích lỗi Phase A

Nhóm: GiCungDuoc

Người phụ trách: Vũ Minh Hiếu — MSSV: 2A202602779 (Thành viên 1 — RAG Eval & Diagnostics Lead)

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T14:15:30.253409+00:00.
Generation mode/fallback và điều kiện chạy: Chạy trực tiếp qua pipeline RAG kết hợp Hybrid Search (BM25 + Qdrant Dense BAAI/bge-m3) + CrossEncoder Reranker (BAAI/bge-reranker-v2-m3) + Generation LLM (OpenAI gpt-4o-mini). Qdrant v1.9.7 containerized.
Models generation/evaluator: Generation: gpt-4o-mini-2024-07-18; Evaluator: gpt-4o-mini, text-embedding-3-small (báo cáo từ m4 evaluator metadata).

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 0.9333 | 0.4908 | 0.8000 |
| answer_relevancy | 0.4769 | 0.3356 | 0.4446 |
| context_precision | 0.9542 | 0.9333 | 0.9083 |
| context_recall | 0.9250 | 0.7792 | 0.6167 |
| avg_score | 0.8223 | 0.6347 | 0.6924 |

## Bottom 10 và nguyên nhân

Bảng danh sách 10 câu có `avg_score` thấp nhất từ Task 3:

| Rank | ID | Question | Distribution | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|:---:|:---:|---|:---:|:---:|:---:|---|---|
| 1 | 39 | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | multi_hop | 0.0000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 2 | 30 | So sánh quyền lợi bảo hiểm giữa nhân viên thử việc và nhân viên chính thức. | multi_hop | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 3 | 22 | Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT? | multi_hop | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 4 | 50 | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | adversarial | 0.3333 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 5 | 44 | Bao lâu phải đổi mật khẩu một lần? | adversarial | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 6 | 33 | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | multi_hop | 0.3750 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 7 | 9 | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | factual | 0.5000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 8 | 29 | Nhân viên tạm ứng 8 triệu, chưa thanh toán sau 30 ngày (quá hạn 15 ngày). Ai phê duyệt khoản này và phí phạt là bao nhiêu? | multi_hop | 0.5000 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |
| 9 | 21 | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | multi_hop | 0.5417 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 10 | 24 | Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu? | multi_hop | 0.6127 | faithfulness | LLM hallucinating | Tighten system prompt, lower temperature |

### Đối chiếu chi tiết các ca lỗi điển hình:
1. **Lỗi Retrieval & Multi-hop Reasoning (Câu ID 39, Avg 0.0000):**
   - *Câu hỏi:* So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA.
   - *Thực tế xảy ra:* Retrieval chỉ lấy được chunk của `mat_khau_v1.md` và `mat_khau_v2.md` nhưng bị thiếu phần chu kỳ đổi mật khẩu do chunk size 256 cắt rời bảng thông số. Khi prompt yêu cầu trả lời nghiêm ngặt "Chỉ dựa trên context, nếu không có nói Không tìm thấy", LLM đã trả về câu trả lời không đầy đủ hoặc fallback "Không tìm thấy thông tin", dẫn đến Faithfulness = 0 và Relevancy = 0.
2. **Lỗi Version Conflict Trap (Câu ID 44, Avg 0.3750):**
   - *Câu hỏi:* Bao lâu phải đổi mật khẩu một lần?
   - *Context:* Context chứa đồng thời 2 văn bản: v1.0 ghi "mỗi 90 ngày" và v2.0 (hiện hành) ghi "mỗi 120 ngày".
   - *Thực tế xảy ra:* Do cả hai tài liệu đều xuất hiện trong top-3 context, LLM gặp mâu thuẫn phiên bản trực tiếp. Nếu không có meta-prompt hướng dẫn ưu tiên tài liệu có hiệu lực mới nhất, LLM ngập ngừng và đưa ra câu trả lời từ chối hoặc bị phạt điểm faithfulness.
3. **Lỗi Policy Contradiction Trap (Câu ID 50, Avg 0.3333):**
   - *Câu hỏi:* Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không?
   - *Context:* Chính sách VPN v1.3 quy định rõ: "Không sử dụng VPN cá nhân (NordVPN, ExpressVPN) trong mạng công ty".
   - *Thực tế xảy ra:* LLM trả về fallback "Không tìm thấy" thay vì trích xuất rõ câu cấm phủ định, khiến answer_relevancy và faithfulness bị đánh giá 0 điểm.

## Matrix và nhóm yếu nhất

Ma trận phân bố lỗi theo `worst_metric × distribution` trên toàn bộ 50 câu:

| Worst Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng câu |
|---|:---:|:---:|:---:|:---:|
| faithfulness | 1 | 9 | 2 | 12 |
| answer_relevancy | 18 | 10 | 4 | 32 |
| context_precision | 0 | 0 | 1 | 1 |
| context_recall | 1 | 1 | 3 | 5 |

- **Nhóm yếu theo mean thấp nhất:** `multi_hop`; mean `avg_score`: **0.6347** (so với factual: 0.8223 và adversarial: 0.6924).
- **Worst metric xuất hiện nhiều nhất:** `answer_relevancy`; số câu: **32 câu** (trong đó có 18 câu factual, 10 câu multi_hop, 4 câu adversarial).

### Phân tích nguyên nhân và đề xuất cải tiến:
- **Vì sao multi_hop là nhóm yếu nhất?** 
  Các câu hỏi multi-hop đòi hỏi phải tổng hợp thông tin từ 2–3 nguồn tài liệu khác nhau (ví dụ: kết hợp thâm niên ở tài liệu nhân sự và mức phụ cấp ở tài liệu tài chính). Context Recall của multi_hop đạt 0.7792 (thấp hơn factual 0.9250) và Faithfulness chỉ đạt 0.4908 (so với factual 0.9333). Khi thiếu một phần context, LLM buộc phải suy đoán hoặc trả lời thiếu ý, dẫn đến suy giảm nghiêm trọng cả độ trung thực lẫn độ phù hợp.
- **Đề xuất khắc phục cụ thể:**
  1. *Prompt Engineering:* Bổ sung hướng dẫn Chain-of-Thought (CoT) vào generation prompt: yêu cầu LLM trích xuất các điều kiện tiên quyết trước khi đưa ra kết luận số liệu.
  2. *Chunking & Enrichment:* Nâng `HIERARCHICAL_PARENT_SIZE` từ 2048 lên 3072 để khi trả context từ child chunk về, parent context bao quát đủ cả điều khoản phụ cấp và thâm niên.
  3. *Quy trình kiểm chứng:* Chạy lại `python scripts/setup_answers.py --live` và `python scripts/run_experiments.py --phase a --live` để đối chiếu xem `multi_hop` avg_score có vượt ngưỡng 0.75 hay không.

## Bẫy adversarial và giới hạn

### Phân tích so sánh với factual và multi-hop:
- **Điểm trung bình Adversarial (0.6924) thấp hơn rõ rệt so với Factual (0.8223):**
  - Trong bộ kiểm thử, 10 câu adversarial chứa hai dạng bẫy thiết kế có chủ ý:
    1. **Bẫy xung đột phiên bản (Version Conflicts):** Điển hình là chính sách mật khẩu (v1.0 yêu cầu 8 ký tự, 90 ngày vs v2.0 yêu cầu 12 ký tự, 120 ngày) và chính sách nghỉ phép (v2023 12 ngày vs v2024 15 ngày). Context recall của nhóm adversarial rơi xuống mức thấp nhất toàn hệ thống (**0.6167**), do hệ thống retrieval lấy nhầm các chunk của phiên bản cũ hoặc chỉ lấy được 1 trong 2 phiên bản cần so sánh.
    2. **Bẫy phủ định và ngoại lệ (Negation & Policy Contradiction):** Câu ID 46 (thử việc không có phép năm), Câu ID 48 (thử việc không có bảo hiểm PVI), Câu ID 50 (cấm dùng VPN cá nhân). Khi người dùng hỏi dạng câu hỏi khẳng định có vẻ hợp lý ("Manager có thể dùng VPN cá nhân để tăng bảo mật không?"), LLM thường có xu hướng trả lời mang tính đồng thuận (sycophancy) hoặc bối rối trước văn bản phủ định nếu không trích dẫn chính xác từ khóa "cấm / không được".
- **Giới hạn thực nghiệm:**
  - Bộ mẫu adversarial có quy mô 10 câu, phản ánh được các bẫy cơ bản của văn bản nội bộ nhưng chưa bao quát toàn bộ các kịch bản prompt injection tinh vi.
  - Quá trình generation sử dụng `gpt-4o-mini` ở temperature=0; một số câu bị timeout hoặc trả về fallback "Không tìm thấy thông tin" làm điểm relevancy bị hạ thấp xuống 0.0.
