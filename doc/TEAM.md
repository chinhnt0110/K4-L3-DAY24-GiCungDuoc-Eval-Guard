# Nhóm thực hiện K4 Track 3A

Tên nhóm: GiCungDuoc

Repo chung: https://github.com/chinhnt0110/K4-L3-DAY24-GiCungDuoc-Eval-Guard

Trưởng nhóm/người nộp: Vũ Minh Hiếu - 2A202602779

Ngày lab/deadline áp dụng: 2026-10-12 23:59 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: c50730d9351c05ea7ac1f9ffc90078a30abae955

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Vũ Minh Hiếu | 2A202602779 | Thành viên 1 (RAG Eval & Diagnostics Lead): Chuẩn bị môi trường & Pipeline (m1-m5, Qdrant); Sinh 50 answers; Phase A (Tasks 1-4); Phase B (Tasks 5, 6 và hỗ trợ 7, 8); Soạn thảo failure_clusters.md và bias_report.md | Commit 846a59342778fac0f188b0031dac5b6c2afd8d21 ("Phase A and B"); artifacts/answers_50q.json, artifacts/ragas_50q.json, artifacts/judge_results.json; doc/reports/failure_clusters.md, doc/reports/bias_report.md |
| 2 | Nguyễn Thị Chinh | 2A202602876 | Thành viên 2 (Safety & Guardrail Lead): Chuẩn hóa adapter m4_eval (evaluator_models metadata); Phase B (Tasks 7, 8: Cohen's Kappa, Bias reporting); Phase C (Tasks 9a-12: Presidio PII scan, NeMo Guardrails, latency p95 benchmark); Tinh chỉnh prompt self_check_input triệt tiêu false positives; Tham gia soạn thảo blueprint.md | Commit c50730d9351c05ea7ac1f9ffc90078a30abae955 ("update phase C"); src/m4_eval.py, src/phase_c_guard.py, guardrails/config.yml, artifacts/guard_results.json, doc/reports/blueprint.md |

Người tích hợp/chạy batch/nộp link: Vũ Minh Hiếu & Nguyễn Thị Chinh — Tích hợp pipeline 5 module m1-m5 từ lab trước, chuẩn bị và nạp dữ liệu Qdrant Vector DB, chạy batch setup sinh 50 answers, thực thi batch experiments Phase A, Phase B và Phase C, kiểm tra offline unit tests (86/86 passed) và nộp link repository GitHub.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows 11 64-bit, Python 3.11.9 (quản lý bởi uv), Microsoft Visual C++ Build Tools chưa có trên Windows host.
Virtualenv/dependencies/imports: Tạo và kích hoạt môi trường ảo bằng uv; cài đặt dependencies lõi qua `uv pip install -r requirements/requirements.txt -c requirements/constraints-core.txt`. Kiểm tra imports thành công cho các thư viện chính (qdrant_client, openai, ragas, pytest); chạy offline unit test suite với lệnh `pytest tests/ -m "not integration"` đạt kết quả 86 passed, 0 failed.
Models/Qdrant/50 answers đã chuẩn bị: Qdrant Docker v1.9.7 containerized chạy tại localhost:6333, collection `lab24_production` đã nạp và index đầy đủ 107 chunks. Embedding sử dụng mô hình BAAI/bge-m3 dense kết hợp BM25 sparse (Hybrid search), CrossEncoder BAAI/bge-reranker-v2-m3, LLM generation OpenAI gpt-4o-mini-2024-07-18. File `artifacts/answers_50q.json` gồm đúng 50 câu (20 factual, 20 multi_hop, 10 adversarial) được sinh bằng LLM (generation_mode: llm, không dùng fallback).
Nếu cài full NeMo chưa thành công, ghi lỗi thực tế; không dùng unit test pass hoặc pip resolve để khẳng định runtime đã chạy: Thực tế khi cài đặt gói `nemoguardrails==0.10.0` trên môi trường Windows 11 bằng uv/pip, quá trình build wheel thất bại tại dependency `annoy==1.17.3` với lỗi: `error: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools"`. Do đó, runtime NeMo Guardrails đầy đủ chưa thể khởi chạy trực tiếp trên Windows host nếu thiếu MSVC C++ compiler. Toàn bộ logic cài đặt Phase C (PII Presidio regex scanning, async input rail check, output rail redaction, adversarial suite runner, linear percentile latency measurement) đã được hoàn thiện đầy đủ trong `src/phase_c_guard.py` và đã được kiểm chứng độc lập thông qua test suite unit/mock tests (86/86 passed). Để chạy live experiment Phase C với NeMo engine, khuyến nghị sử dụng môi trường Linux/WSL2 hoặc Docker container có sẵn GCC/g++.

## INDIVIDUAL

Sao chép mẫu cho mỗi thành viên; tiêu đề họ tên không dấu-MSSV.

### VuMinhHieu-2A202602779

- Phần được giao và phần thực tế đã làm:
  + Vai trò phân công: Thành viên 1 (RAG Eval & Diagnostics Lead) — Phụ trách chuẩn bị pipeline m1-m5, sinh 50 answers, Phase A (Tasks 1-4: đánh giá RAGAS, chẩn đoán cụm lỗi), và Phase B (Tasks 5, 6: Pairwise Judge A/B).
  + Thực tế đã hoàn thành:
    * Khởi tạo dịch vụ Qdrant Vector DB qua Docker container (v1.9.7), index toàn bộ 107 chunks từ corpus chính sách nội bộ; sửa adapter `src/m2_search.py` tương thích API `qdrant-client 1.9.2` (chuyển sang phương thức `search` chuẩn thay cho `query_points`); bổ sung cơ chế disk caching cho chunk enrichment trong `src/m5_enrichment.py`.
    * Chạy thành công `scripts/setup_answers.py --live` sinh đủ 50 câu trả lời (`artifacts/answers_50q.json`) chuẩn schema v2 (20 factual, 20 multi_hop, 10 adversarial) ở chế độ `generation_mode: llm`.
    * Triển khai hoàn chỉnh Task 1 (group_by_distribution), Task 2 (run_ragas_50q), Task 3 (bottom_10 với xếp hạng và đề xuất sửa lỗi chi tiết), Task 4 (cluster_analysis với ma trận lỗi và insight) trong `src/phase_a_ragas.py`.
    * Chạy thực nghiệm live Phase A tạo ra `artifacts/ragas_50q.json` với 4 metric RAGAS (faithfulness, answer_relevancy, context_precision, context_recall), đạt điều kiện thưởng Bonus Phase A: adversarial_mean (0.6924) < factual_mean (0.8223).
    * Soạn thảo báo cáo chẩn đoán lỗi `doc/reports/failure_clusters.md` phân tích ma trận lỗi `worst_metric × distribution`, đi sâu vào 3 ca lỗi tiêu biểu (ID 39, ID 44, ID 50) do mâu thuẫn chính sách phiên bản và context window.
    * Triển khai Task 5 (pairwise_judge) đánh giá độ chính xác dựa trên reference, Task 6 (swap_and_average) thực hiện hoán đổi 2 lượt để triệt tiêu position bias trong `src/phase_b_judge.py`. Đồng thời hỗ trợ hoàn thiện Task 7 (cohen_kappa) và Task 8 (bias_report).
    * Chạy thực nghiệm live Phase B tạo ra `artifacts/judge_results.json` với Cohen's Kappa đạt 1.0 (Hoàn hảo, vượt ngưỡng bonus 0.6) và position bias rate = 0.0%. Soạn thảo báo cáo phân tích bias `doc/reports/bias_report.md`.
    * Hỗ trợ triển khai các hàm PII scanning, async rail checks, latency percentiles trong `src/phase_c_guard.py` và bảo đảm toàn bộ test suite offline 86 test cases vượt qua hoàn hảo.
- Bằng chứng:
  + Các commit và mã nguồn: Commit 846a59342778fac0f188b0031dac5b6c2afd8d21 ("Phase A and B") — `src/m2_search.py`, `src/m5_enrichment.py`, `src/phase_a_ragas.py`, `src/phase_b_judge.py`.
  + File dữ liệu thực nghiệm đã sinh: `artifacts/answers_50q.json`, `artifacts/ragas_50q.json`, `artifacts/judge_results.json`.
  + Báo cáo phân tích chuyên sâu: `doc/reports/failure_clusters.md`, `doc/reports/bias_report.md`.
- Cách kiểm tra:
  + Kiểm tra test suite offline: `.\.venv-tests\Scripts\python.exe -m pytest tests/ -m "not integration" -q` (kết quả: 86 passed, 0 failed).
  + Kiểm tra pipeline answers: `.\.venv\Scripts\python.exe scripts/setup_answers.py --live` (kết quả: sinh 50 answers hợp lệ).
  + Kiểm tra Phase A live: `.\.venv\Scripts\python.exe scripts/run_experiments.py --phase a --live` (kết quả: tạo `artifacts/ragas_50q.json` khớp schema).
  + Kiểm tra Phase B live: `.\.venv\Scripts\python.exe scripts/run_experiments.py --phase b --live` (kết quả: tạo `artifacts/judge_results.json` khớp schema).
  + Kiểm tra tính hợp lệ của hợp đồng và artifact: `.\.venv-tests\Scripts\python.exe scripts/check_lab.py`.
- Điều đã học:
  + Bản chất toán học và ý nghĩa thực tế của 4 metric RAGAS: Faithfulness (độ trung thực so với context), Answer Relevancy (độ phù hợp với câu hỏi), Context Precision (tỷ lệ chunk liên quan được xếp hạng cao), Context Recall (độ bao phủ thông tin cần thiết so với ground truth).
  + Cơ chế chẩn đoán lỗi hệ thống RAG thông qua ma trận phân bố: Phân biệt rõ lỗi do Retrieval (thiếu context) và lỗi do Generation (hallucination hoặc mâu thuẫn phiên bản quy định giữa các tài liệu v1.0 và v2.0).
  + Tác động của Position Bias trong mô hình LLM-as-a-Judge và phương pháp swap-and-average để đảo vị trí hai ứng viên nhằm triệt tiêu thiên lệch.
  + Phương pháp thống kê Cohen's Kappa để đánh giá mức độ tương đồng giữa nhãn phân loại tự động của LLM Judge và chuyên gia con người (Human Annotation).
- Giới hạn/cách xử lý:
  + Hạn chế môi trường Windows khi cài đặt `annoy 1.17.3` của NeMo Guardrails: Đã xử lý bằng cách cài đặt đầy đủ logic mock/unit tests để đảm bảo mã nguồn chuẩn hợp đồng và test pass 100%; ghi chép trung thực hạn chế môi trường và đề xuất containerization cho production.
  + Giới hạn của chunk size 256 khiến các bảng quy định dài bị cắt rời: Đề xuất cải thiện bằng giải pháp Parent-Child chunking hoặc Document Hierarchy enrichment.
- Nguồn/AI hỗ trợ nếu có:
  + Tài liệu chính thức của RAGAS, Qdrant Client API docs, OpenAI Cookbook, NeMo Guardrails documentation.
  + Sử dụng trợ lý Antigravity AI để hỗ trợ gỡ lỗi tương thích phiên bản thư viện qdrant-client và cấu trúc schema Draft 2020-12; toàn bộ code logic, chạy thử nghiệm và phân tích số liệu đều được tác giả trực tiếp kiểm chứng và thực thi.

### NguyenThiChinh-2A202602876

- Phần được giao và phần thực tế đã làm:
  + Vai trò phân công: Thành viên 2 (Safety & Guardrail Lead) — Phụ trách chuẩn hóa adapter m4_eval, Phase B (Tasks 7, 8: tính Cohen's Kappa, thống kê Position & Verbosity Bias), Phase C (Tasks 9a-12: PII Detection Presidio, NeMo Guardrails input/output rails, benchmark độ trễ P95) và đồng tác giả blueprint.md.
  + Thực tế đã hoàn thành:
    * Hoàn thiện adapter `src/m4_eval.py`, bổ sung trường bắt buộc `metadata.evaluator_models` theo chuẩn `PIPELINE_INTERFACE.md`.
    * Triển khai Task 7 (cohen_kappa) tính toán thỏa thuận giữa LLM Judge và Human Annotation đạt Kappa = 1.0 (vượt ngưỡng bonus 0.6); Task 8 (bias_report) đo lường Position Inconsistency Rate (0.0%) và Verbosity Bias Rate.
    * Triển khai Task 9a (pii_scan) với Presidio Analyzer/Anonymizer nhận diện chính xác CCCD (12 số), CMND (9 số), SĐT Việt Nam và Email; lọc các thực thể mục tiêu để tránh false positives do Spacy NER tiếng Anh.
    * Triển khai Task 9b (check_input_rail) và Task 11 (check_output_rail) với NeMo Guardrails Colang 1.0; tinh chỉnh prompt `self_check_input` trong `guardrails/config.yml` phân biệt rõ câu hỏi chính sách bảo mật hợp lệ với yêu cầu trích xuất thông tin bí mật.
    * Triển khai Task 10 (run_adversarial_suite) và chạy thực nghiệm live đạt **28/28 passed** (20/20 attacks bị chặn, 8/8 clean inputs được cho qua, False Positive Rate = 0.0, đạt điều kiện thưởng Bonus Phase C).
    * Triển khai Task 12 (measure_p95_latency) đo đạc 20 samples thực tế sau 2 lượt warm-up, tính toán p50/p95/p99 nội suy tuyến tính và lưu đầy đủ vào `artifacts/guard_results.json`.
    * Đồng biên soạn bản thiết kế hệ thống `doc/reports/blueprint.md` đề xuất kiến trúc Guardrail Stack, Quality Gates cho CI/CD và chiến lược Monitoring khi lên Production.
- Bằng chứng:
  + Commit/PR mã nguồn: Commit c50730d9351c05ea7ac1f9ffc90078a30abae955 ("update phase C") — `src/m4_eval.py`, `src/phase_c_guard.py`, `guardrails/config.yml`.
  + File dữ liệu thực nghiệm: `artifacts/guard_results.json`.
  + Báo cáo Markdown: `doc/reports/blueprint.md`.
- Cách kiểm tra:
  + Chạy unit test Phase C: `/opt/miniconda3/envs/k4-day24/bin/python -m pytest tests/test_phase_c.py -q` (12 passed).
  + Chạy live Phase C runner: `/opt/miniconda3/envs/k4-day24/bin/python scripts/run_experiments.py --phase c --live --latency-samples 20` (28/28 passed).
  + Kiểm tra toàn diện bài nộp: `/opt/miniconda3/envs/k4-day24/bin/python scripts/check_lab.py --with-tests` (0 failed checks, 86 unit tests passed).
- Điều đã học:
  + Kiến trúc phòng thủ nhiều lớp (Defense-in-Depth): Presidio xử lý PII nhanh chóng ở tầng 1 trước khi chuyển sang LLM Guardrails ở tầng 2 để tối ưu hóa độ trễ và chi phí token.
  + Kỹ thuật kiểm soát Prompt Injection và Jailbreak bằng NeMo Guardrails kết hợp Colang 1.0.
  + Bản chất của bài toán đánh đổi (Trade-off) giữa Security và Usability trong Guardrails: Việc cấu hình quá khắt khe có thể gây chặn nhầm câu hỏi hợp lệ của người dùng (False Positives), đòi hỏi phải tinh chỉnh prompt phân biệt chính sách và hành vi tấn công.
  + Phương pháp đo lường độ trễ p95 theo chuẩn production, loại bỏ warm-up runs để số liệu phản ánh đúng runtime thực tế.
- Giới hạn/cách xử lý:
  + NeMo Guardrails phụ thuộc vào thư viện `annoy` khó build trên Windows: Đã thiết lập môi trường Unix/Mac (Python 3.11 Conda) để thực thi live experiment đầy đủ và tạo artifact thật.
  + Nguy cơ False Positives khi Presidio phân tích văn bản tiếng Việt với model Spacy tiếng Anh: Đã xử lý bằng cách lọc tập trung vào các regex pattern tùy chỉnh (VN_CCCD, VN_PHONE, EMAIL_ADDRESS).
- Nguồn/AI hỗ trợ nếu có:
  + Microsoft Presidio documentation, NeMo Guardrails Colang documentation.
  + Sử dụng AI Assistant để hỗ trợ phân tích nguyên nhân lỗi false positive và tối ưu hóa logic lọc thực thể PII.

## Cập nhật sau khi nộp

Không có bản cập nhật nào sau khi nộp.
