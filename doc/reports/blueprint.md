# Blueprint CI/CD cho RAG và guardrails

Nhóm: GiCungDuoc

Người phụ trách: Vũ Minh Hiếu — MSSV: 2A202602779

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T14:18:47.620167+00:00; guard models: presidio-analyzer, nemo-guardrails-gpt-4o-mini.
Models generation/evaluator/judge từ A/B: generation: gpt-4o-mini-2024-07-18; evaluator: gpt-4o-mini, text-embedding-3-small; judge: gpt-4o-mini.

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Mô tả data flow và failure actions:
1. Tầng Input Guard (Early Termination / Short-circuit):
   - Presidio Scanner quét thông tin định danh cá nhân (PII) như số CCCD/CMND 9 hoặc 12 chữ số, số điện thoại Việt Nam và email cá nhân. Nếu phát hiện PII, hệ thống lập tức ẩn danh (redact) thay thế bằng nhãn entity trước khi chuyển tiếp.
   - NeMo Input Rail kiểm tra prompt injection, jailbreak attempts, và hành vi vi phạm đạo đức an toàn. Nếu kích hoạt input rail vi phạm, hệ thống ngắt chuỗi xử lý ngay lập tức (short-circuit), trả về thông điệp từ chối tiêu chuẩn được cấu hình trong `guardrails/rails.co` và ghi log cảnh báo an ninh.
2. Tầng RAG Core:
   - Truy vấn hợp lệ được mã hóa qua mô hình Dense Embedding (BAAI/bge-m3) kết hợp BM25 sparse để thực hiện Hybrid Search trên Qdrant Vector DB.
   - Top candidate chunks được tái xếp hạng qua BAAI/bge-reranker-v2-m3 để chọn ra ngữ cảnh có độ liên quan cao nhất gửi vào prompt template.
   - LLM sinh câu trả lời với ràng buộc chỉ dựa vào ngữ cảnh cung cấp.
3. Tầng Output Guard:
   - NeMo Output Rail kiểm tra tính trung thực, phát hiện rò rỉ thông tin nội bộ và ngôn từ độc hại trong câu trả lời đã sinh. Nếu vi phạm, thay thế bằng thông báo từ chối an toàn hoặc thực hiện redaction.
4. Quản lý Secrets và Fallback:
   - Toàn bộ API keys (OPENAI_API_KEY) và thông tin xác thực được quản lý qua GitHub Secrets hoặc biến môi trường bảo mật, tuyệt đối không lưu cứng trong mã nguồn.
   - Cơ chế Fallback: Khi dịch vụ OpenAI gặp sự cố timeout hoặc rate limit, hệ thống chuyển sang chế độ extractive fallback từ top retrieved chunks kèm thông báo cảnh báo suy giảm dịch vụ (graceful degradation).

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2; phương pháp: linear.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 1.85 | 3.42 | 4.10 | Đạt chuẩn hiệu năng regex scanner |
| NeMo input | chưa đo | chưa đo | chưa đo | Giới hạn C++ Build Tools trên Windows host, đề xuất runner Linux |
| Input guard sequential | chưa đo | chưa đo | chưa đo | Cần môi trường Linux runner có NeMo runtime đầy đủ |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: Mục tiêu thiết kế nhỏ hơn 500ms; đối với Presidio P95 đạt xấp xỉ 3.42ms (rất tối ưu). Phần NeMo input runtime cần môi trường Linux CI runner để đo đạc chính xác.
Kế hoạch đo đạc thực tế: Khi chạy trên CI runner Linux/Docker có cài đặt đầy đủ NeMo Guardrails, sẽ thực hiện 2 warm-up runs không tính điểm, sau đó lấy 20 mẫu timed trên tập clean controls `fixtures/clean_set_8.json` để nội suy phân vị tuyến tính P50, P95, P99 theo đúng hợp đồng `measure_p95_latency`.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.
Đề xuất lệnh kiểm tra tự động trên GitHub Actions:
```bash
# 1. Chạy unit tests offline và mocks
pytest tests/ -m "not integration" -q --tb=short

# 2. Chạy quality check contract
python scripts/check_lab.py --root .
```
Nếu bất kỳ chỉ số nào dưới ngưỡng cho phép, runner trả về exit code 1 làm thất bại workflow (fail fast), ngăn chặn việc merge code chưa đạt chuẩn vào nhánh production. Ghi rõ đây là workflow kiến trúc đề xuất cho quy trình triển khai sản phẩm.

## Monitoring và cải tiến

Quy trình giám sát sản phẩm:
- Thu thập mẫu production định kỳ (sample 5% lưu lượng thực tế) để thực hiện offline LLM-as-a-Judge đánh giá độ lệch (concept drift) và tỷ lệ ảo giác (hallucination rate).
- Giám sát độ trễ P95 với ngưỡng cảnh báo 500ms cho input layers; kích hoạt cảnh báo qua Slack/PagerDuty khi P95 vượt ngưỡng trong 5 phút liên tục.
- Giám sát tỷ lệ chặn nhầm (False Positive Rate): Định kỳ rà soát các truy vấn bị chặn bởi NeMo rails để cập nhật và tinh chỉnh tập luật trong Colang và Presidio custom recognizers.

RAGAS mean/worst distribution: factual_mean = 0.8223, multi_hop_mean = 0.6347, adversarial_mean = 0.6924; phân bố yếu nhất là multi_hop.
Cohen’s κ: 1.0000 (độ đồng thuận tuyệt đối với nhãn chuyên gia trên tập 10 câu đối chuẩn).
Attacks passed: Mục tiêu CI đạt ít nhất 18/20; clean false_positive_rate: mục tiêu = 0.0% (8/8 clean allowed).

3–5 giải pháp cải tiến gắn với bằng chứng thực nghiệm:
1. Cải thiện Multi-hop Reasoning: Tăng context window và áp dụng chiến lược Parent-Child Chunking hoặc Graph RAG để kết nối các thông tin liên quan bị phân tách giữa các điều khoản chính sách.
2. Xử lý Mâu thuẫn Phiên bản (Version Conflict Trap): Bổ sung trường metadata `version` và `effective_date` vào payload chunk của Qdrant; cập nhật system prompt hướng dẫn LLM luôn ưu tiên tài liệu có hiệu lực mới nhất thay vì phân vân giữa chính sách cũ và mới.
3. Tinh chỉnh Phủ định An toàn (Negative Constraint Prompting): Huấn luyện lại và tinh chỉnh prompt xử lý các câu hỏi bẫy cấm đoán (như lệnh cấm dùng VPN cá nhân) để LLM trích xuất rõ câu phủ định thay vì trả về không tìm thấy thông tin.
4. Tối ưu hóa Hiệu năng Triển khai: Đóng gói toàn bộ Guardrail Stack vào container Linux độc lập trên nền kiến trúc gRPC/FastAPI microservice để đảm bảo tính cô lập và đáp ứng ngân sách P95 nhỏ hơn 500ms.
