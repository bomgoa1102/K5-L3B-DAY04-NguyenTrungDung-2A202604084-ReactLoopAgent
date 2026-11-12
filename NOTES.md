# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Trung Dũng
MSSV: 2A202604084

## 1. Case được chọn

Tôi chọn case A2, hỏi về incident Cloudflare cache purge và phạm vi ảnh hưởng đến khách hàng không thuộc FedRAMP. Case này cần đọc thông báo chi tiết và dẫn đúng URL nguồn.

Do API key cũ đã hết hạn, tôi chưa chạy được eval với LLM thật. Để kiểm tra lỗi trước khi chạy eval, tôi dùng FakeProvider trong bài test offline, cố ý tạo một câu trả lời đúng ý nhưng thiếu URL.

- Câu trả lời trong mô phỏng trước cải tiến: "Cloudflare cache purge được xác định, non-FedRAMP không bị ảnh hưởng."
- Điểm A2 lần chạy 1: chưa có, cần lấy từ `eval_before.json` sau khi chạy bằng key mới.

## 2. Bằng chứng trong trace

Tool `get_incident_details` trả về incident `wp49bp8bqo6c`, trạng thái `identified`, URL `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`. Thông báo ngày 06/11/2026 ghi: "Non-fedramp customers are not impacted." Điều này chỉ nói khách hàng ngoài FedRAMP không bị ảnh hưởng bởi **incident này**, không chứng minh website của họ luôn bình thường.

Kết quả mô phỏng offline trên snapshot ngày 09/11/2026:

```text
Trước cải tiến:
  model (step 1) -> get_incident_details -> model (step 2)
  status: completed
  URL trong answer: không có

Sau cải tiến:
  model (step 1) -> get_incident_details -> model (step 2)
  validation_missing_source (step 2)
  model sửa lại (step 3)
  status: completed
  URL trong answer: có
```

Đây là trace chạy bằng FakeProvider, không phải điểm hay trace của bộ eval LLM.

## 3. Thay đổi đã thực hiện

Tôi sửa `agent.py` để lưu URL của những lần `get_incident_details` thành công. Khi model chuẩn bị trả lời, chương trình kiểm tra xem các URL đã quan sát có trong câu trả lời chưa. Nếu thiếu, tôi thêm yêu cầu sửa câu trả lời vào lịch sử hội thoại và gọi model tiếp, nhưng vẫn phải tuân theo `max_steps`. Khi đã hết bước mà vẫn thiếu URL thì không báo `completed`.

Tôi cũng giữ nguyên assistant message và `tool_call_id`, trả lỗi JSON/tool cho model, giới hạn retry khi gặp 429 và bỏ bớt các mục nhiễu không liên quan trong tool result. Cách này giúp agent hoạt động với bộ tools chính và tools bonus mà không dùng framework agent.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (22 case) | Chưa chạy | Chưa chạy |
| Điểm case A2 | Chưa chạy | Chưa chạy |
| Số model calls | Chưa có dữ liệu eval | Chưa có dữ liệu eval |
| Token, chi phí nếu có | Chưa có dữ liệu eval | Chưa có dữ liệu eval |

Trong mô phỏng offline riêng, số lần gọi FakeProvider tăng từ 2 lên 3 và lỗi thiếu URL được sửa. Kết quả này chỉ chứng minh điều kiện kiểm tra nguồn hoạt động trong tình huống đã dựng; chưa chứng minh điểm LLM judge sẽ tăng.

Sau khi có key mới, tôi cần chạy đủ 22 case cho bản trước và sau với cùng provider, model, judge và `max_steps=8`, rồi điền điểm, số lượt gọi model, token và lỗi phát sinh từ hai file eval vào mục này.

## 5. Giới hạn

- Snapshot được lưu ngày 09/11/2026, không thể dùng làm bằng chứng trạng thái dịch vụ thời gian thực.
- Tool bài chính chỉ tìm trong 50 incident gần nhất; việc không tìm thấy không có nghĩa là sự cố không tồn tại.
- Một số bản cập nhật bị rút gọn, nên có thể chưa đủ thông tin về nguyên nhân hoặc phạm vi.
- Kiểm tra URL chỉ bảo đảm có dẫn link incident đã đọc, không kiểm chứng tất cả kết luận trong câu trả lời.
- LLM judge có thể chấm sai hoặc thiên lệch. Một lần chạy mỗi phiên bản không chứng minh agent luôn ổn định.
- Tests offline kiểm tra logic, không thay thế việc chạy eval với model thật.
