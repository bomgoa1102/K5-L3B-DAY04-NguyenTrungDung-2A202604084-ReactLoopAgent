# Ghi chú phân tích – Day 04

Họ và tên: Nguyễn Trung Dũng
MSSV: 2A202604084
Lớp: K5-L3B

## 1. Case tôi chọn

Tôi chọn **E2**: kiểm tra incident liên quan đến Cloudflare Cache Purge và xem một website không thuộc FedRAMP có bị ảnh hưởng hay không.

Theo dữ liệu của bài, incident `wp49bp8bqo6c` có trạng thái `identified`. Thông báo của Cloudflare ghi rõ khách hàng không thuộc FedRAMP không bị ảnh hưởng bởi **incident này**. Không thể từ đó khẳng định website không có lỗi khác.

Ở lần chạy BEFORE, agent trả lời đúng nội dung và được 100 điểm. Tuy nhiên, tôi thấy câu trả lời chỉ dẫn trang Cloudflare Status chung mà không đưa URL trực tiếp của incident, dù tool đã trả về URL đó. Tôi chọn đây là điểm cần cải thiện về nguồn dẫn, không phải lỗi bị judge trừ điểm.

## 2. Bằng chứng từ trace

**BEFORE – case E2:**

- Bước 1: gọi `get_status` với `provider=cloudflare`, `component=cache purge`.
- Bước 2: gọi `get_incident_details` với incident `wp49bp8bqo6c`. Tool trả về `status=identified`, nội dung “Non-fedramp customers are not impacted” và URL `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`.
- Bước 3: model trả lời và kết thúc. Câu trả lời không có URL incident cụ thể.
- Kết quả: 100 điểm, 3 lần gọi model, 2.541 token.

**AFTER – case E2:**

- Bước 1 và 2: tiếp tục dùng hai tool để lấy trạng thái và chi tiết incident.
- Bước 3: model chuẩn bị kết thúc nhưng trace xuất hiện `validation_missing_source`, chỉ ra URL incident còn thiếu.
- Bước 4: model sửa lại câu trả lời, bổ sung đúng URL `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`.
- Kết quả: 100 điểm, 4 lần gọi model, 4.717 token.

Như vậy, thay đổi giúp câu trả lời có liên kết trực tiếp đến bằng chứng. Điểm E2 không tăng vì bản BEFORE đã được chấm 100. Đổi lại, agent phải gọi model thêm một lần và dùng thêm 2.176 token.

## 3. Thay đổi tôi đã thực hiện

Tôi sửa `agent.py` để hoàn thiện vòng lặp ReAct: gửi câu hỏi cho model, thực hiện các tool call, lưu kết quả vào hội thoại rồi gọi model tiếp. Mỗi tool result được trả về đúng `tool_call_id`. Tôi cũng ghi trace, giới hạn `max_steps` và xử lý khi JSON arguments hoặc tool bị lỗi.

Phần cải tiến chính là kiểm tra URL nguồn:

- Khi `get_incident_details` thành công, tôi lưu URL incident được trả về.
- Trước khi kết thúc, tôi kiểm tra các URL đã lưu có nằm trong câu trả lời hay chưa.
- Nếu còn thiếu, tôi ghi `validation_missing_source` vào trace và yêu cầu model bổ sung URL, nhưng không vượt quá `max_steps`.
- Nếu hết số bước mà vẫn thiếu nguồn thì không báo hoàn thành thành công.

Tôi còn giới hạn việc thử lại khi tool liên tục trả `RATE_LIMITED` và loại bỏ một số dữ liệu nhiễu trong tool result. Phần này không xử lý được lỗi HTTP 429 từ chính API model hoặc judge.

## 4. So sánh kết quả BEFORE và AFTER

Hai lần chạy sử dụng cùng cấu hình: provider và judge đều là `gemini`, model `gemini-3.5-flash-lite`, `max_steps=8`, dataset `core`.

| Chỉ số | BEFORE | AFTER |
|---|---:|---:|
| Tổng số case | 22 | 22 |
| Case được chấm | 6 | 22 |
| Case chưa chấm | 16 | 0 |
| Điểm tổng | `null` | **94,55/100** |
| Điểm E2 | 100 | 100 |
| Điểm trung bình các case đã chấm | 100 (6 case) | 94,55 (22 case) |

BEFORE bị lỗi HTTP 429 từ provider ở các case còn lại nên **không có điểm tổng hợp lệ**. Vì thế tôi không kết luận điểm AFTER tăng hay giảm so với BEFORE. Hai lần chạy có cùng cấu hình nhưng chưa đủ kết quả để so sánh toàn bộ 22 case.

Để so sánh phần có dữ liệu ở cả hai lần chạy, tôi lấy riêng **E1–E6**:

| Chỉ số của E1–E6 | BEFORE | AFTER |
|---|---:|---:|
| Số case đạt 100 điểm | 6/6 | 6/6 |
| Tổng số lần gọi model | 16 | 17 |
| Tổng token agent | 11.828 | 16.358 |

Nhóm E1–E6 vẫn giữ được điểm nhưng tổng token tăng 4.530 (khoảng 38,3%). Trong toàn bộ lần chạy AFTER, agent dùng 68.987 token và có 58 lần gọi model. Tôi chưa tính được chi phí tiền vì file eval chỉ ghi số token, không có đơn giá.

Trong AFTER có 19/22 case đạt 100 điểm. Ba case còn mất điểm là:

- **H1 – 60 điểm:** agent đọc incident GitHub ban đầu và incident Cloudflare, nhưng không gọi tiếp `get_incident_details` cho incident GitHub `xr3myzj7ms51` được dẫn tới. Vì vậy thiếu bước điều tra theo liên kết.
- **H2 – 50 điểm:** agent nhận ra incident `t89g2q1800fr` đã `resolved`, nhưng chỉ gọi `get_incident_details`, không gọi `get_status` để kiểm tra GitHub Actions có `operational` ở snapshot hay không.
- **H7 – 70 điểm:** tool Cloudflare trả `TimeoutError`, nhưng agent lại nghe theo yêu cầu trong câu hỏi để nói Cloudflare Spectrum đang `operational`. Đây là kết luận không có bằng chứng.

Các lỗi H1, H2, H7 là vấn đề còn lại của agent, không phải do judge bị 429 trong lần AFTER này.

## 5. Giới hạn và điều tôi rút ra

Cách kiểm tra URL giúp phát hiện thiếu link, nhưng chưa đảm bảo mọi nhận định đều đúng. Nó cũng làm tăng số lần gọi model và token ở một số case.

Dữ liệu chính là snapshot ngày 09/11/2026, không phản ánh trạng thái thời gian thực. Công cụ chỉ tìm trong 50 incident gần nhất và một số nội dung update có thể bị cắt ngắn. Vì vậy, không tìm thấy thông tin không có nghĩa là sự cố không tồn tại.

Điểm eval do LLM judge chấm nên chỉ mang tính tham khảo; trong bài này judge dùng cùng model với agent. Chỉ một lần chạy mỗi phiên bản cũng chưa cho thấy độ ổn định. Riêng BEFORE bị HTTP 429 nên chưa thể so sánh điểm tổng công bằng với AFTER.

Qua kết quả này, tôi thấy agent cần làm tốt hơn ba việc: theo tiếp liên kết incident khi câu hỏi yêu cầu, kiểm tra trạng thái hiện tại thay vì chỉ đọc incident cũ, và tuyệt đối không đoán trạng thái khi tool bị timeout. Đây là các hướng tôi sẽ ưu tiên nếu tiếp tục cải tiến.
