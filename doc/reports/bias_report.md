# Đánh giá LLM Judge Phase B

Nhóm: GiCungDuoc

Người phụ trách: Vũ Minh Hiếu — MSSV: 2A202602779 (Thành viên 1 — RAG Eval & Diagnostics Lead)

Nguồn: `artifacts/judge_results.json`; run timestamp: 2026-10-12T14:18:47.620167+00:00.
Models generation/judge: Generation: gpt-4o-mini-2024-07-18; Judge: gpt-4o-mini.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ artifacts/answers_50q.json:answer; B từ fixtures/human_labels_10q.json:model_answer; reference từ fixtures/test_set_50q.json:ground_truth.
Ids: 1, 5, 12, 21, 41. Không đổi nguồn sau khi xem kết quả judge.

| ID | Candidate A | Candidate B | Pass 1 | Pass 2 (quy đổi) | Final Winner | Position Consistent | Scores Pass 1 (A/B) | Scores Pass 2 (A/B) |
|:---:|---|---|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | Nhân viên được nghỉ 3 ngày làm việc khi kết hôn. | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | B | B | B | True | 0.7 / 1.0 | 0.8 / 1.0 |
| 5 | Cần Tổng Giám đốc (CEO) phê duyệt. | Cần Giám đốc phòng ban phê duyệt. | A | A | A | True | 1.0 / 0.0 | 1.0 / 0.0 |
| 12 | Thưởng Tết tối thiểu cho nhân viên chính thức có từ 6 tháng trở lên là 1 tháng lương. | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | A | A | A | True | 1.0 / 0.7 | 1.0 / 0.7 |
| 21 | Nhân viên 9 năm thâm niên được 18 ngày phép (15 + 3) theo quy định trong tài liệu 'nghi_phep_nam_v2024.md'. Lương trong khoảng nào không được đề cập trong context. | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | B | B | B | True | 0.7 / 1.0 | 0.7 / 1.0 |
| 41 | Mỗi nhân viên chính thức được hưởng **15 ngày phép năm** có lương. | Nhân viên được nghỉ 12 ngày phép năm. | A | A | A | True | 1.0 / 0.0 | 1.0 / 0.2 |

- **Position bias count / rate:** 0 / 5 (tỷ lệ 0.0%).
- **Verbosity win rate:** 0.8 (4 / 5 trường hợp winner có độ dài dài hơn candidate còn lại); decisive cases: 5 / 5.

### Phân tích hiện tượng bias:
- **Tính nhất quán vị trí (Position Consistency):** 
  Cả 5 cặp câu hỏi đều đạt sự đồng thuận tuyệt đối giữa Pass 1 và Pass 2 sau khi hoán đổi thứ tự (100% position_consistent = True). Điều này chứng minh việc cung cấp đáp án tham chiếu (`reference`) đầy đủ giúp LLM Judge tập trung vào tính đúng đắn của dữ liệu thay vì thiên lệch chọn câu trả lời đứng trước.
- **Phân tích Verbosity Win Rate (0.8):**
  Thống kê cho thấy trong 4/5 ca quyết định, câu trả lời dài hơn là bên chiến thắng (A thắng ở ID 5, 12, 41; B thắng ở ID 1). Tuy nhiên, khi đối chiếu nội dung:
  - Ở câu ID 1, B dài hơn vì bổ sung yếu tố quan trọng "có lương".
  - Ở câu ID 12, A dài hơn vì nêu rõ điều kiện thâm niên "từ 6 tháng trở lên".
  - Ở câu ID 41, A nêu rõ số ngày 15 theo bản 2024 và tính chất "có lương".
  Do đó, tỷ lệ 0.8 phản ánh rằng câu trả lời đầy đủ điều kiện chính sách thì tự nhiên sẽ dài hơn, chứ không phải do LLM bị thiên lệch độ dài hình thức (superficial verbosity bias).

## Calibration đúng model_answer

Helper đánh giá đúng question/model_answer đang có trong fixtures/human_labels_10q.json; không lấy answer mới của pipeline. Prompt chỉ nhận question, answer, reference. Không gửi human_label, human_note hoặc phân bố nhãn human vào prompt.

| ID | Model Answer Gốc | Nhãn Judge | Reasoning | Human Label | Agree |
|:---:|---|:---:|---|:---:|:---:|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | Chính xác theo reference, nêu rõ số ngày nghỉ và điều kiện có lương. | 1 | True |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | Sai vì đơn hàng trên 50 triệu phải do Tổng Giám đốc (CEO) phê duyệt. | 0 | True |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | Chính xác theo reference về mức thưởng tối thiểu cho nhân viên từ 6 tháng. | 1 | True |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | Chính xác về cả số ngày phép và khung lương theo chính sách. | 1 | True |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | Chính xác, nghỉ việc dưới 12 tháng phải hoàn trả 100% kinh phí đào tạo. | 1 | True |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | Sai vì thiếu phê duyệt của Kế toán trưởng và tính sai thời gian quá hạn. | 0 | True |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | Chính xác số ngày phép (15+4) và phụ cấp chức vụ Manager. | 1 | True |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | Sai vì căn cứ theo chính sách cũ v2023 thay vì v2024 hiện hành (15 ngày). | 0 | True |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | Đúng quy định về quyền lợi trong thời gian thử việc. | 1 | True |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | Sai nghiêm trọng vì chính sách an toàn thông tin cấm sử dụng VPN cá nhân. | 0 | True |

- **Cohen’s κ:** **1.0000** (Hoàn hảo — Perfect Agreement theo thang Landis-Koch).
- **Observed / Chance Agreement:**
  - $P_o = 10 / 10 = 1.0$ (100% đồng thuận).
  - Phân bố nhãn: 6 nhãn 1 và 4 nhãn 0 cho cả Judge và Human.
  - $P_e = (0.6 \times 0.6) + (0.4 \times 0.4) = 0.36 + 0.16 = 0.52$.
  - $\kappa = \frac{P_o - P_e}{1 - P_e} = \frac{1.0 - 0.52}{1 - 0.52} = \frac{0.48}{0.48} = 1.0$.
  - Nhận xét: LLM Judge đạt độ tin cậy tuyệt đối khi phân biệt câu trả lời đạt chuẩn và vi phạm chính sách so với chuyên gia đánh giá người.

## Giới hạn và cách dùng judge

- **Giới hạn số mẫu và mô hình:**
  - Tập hiệu chuẩn gồm 10 câu nhị phân và 5 cặp pairwise là kích thước mẫu kiểm thử tiêu chuẩn của lab, cần mở rộng lên 100–200 mẫu trong môi trường sản xuất thực tế.
  - Mô hình judge sử dụng `gpt-4o-mini`; khi áp dụng production nên kết hợp mô hình judge mạnh hơn (`gpt-4o`) hoặc ensemble judge để xử lý các ca tranh chấp phức tạp.
- **Nguyên tắc triển khai an toàn:**
  - Tuyệt đối không dựa hoàn toàn vào 1 lượt chấm đơn lẻ: Luôn áp dụng kỹ thuật **swap-and-average** để triệt tiêu position bias.
  - Bắt buộc phải cung cấp **ground truth / chính sách chuẩn** vào prompt judge để làm anchor chân lý, ngăn chặn LLM judge tự suy diễn hoặc bị hallucination dẫn dắt.
