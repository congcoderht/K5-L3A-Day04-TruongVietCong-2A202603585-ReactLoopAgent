# Ghi chú phân tích · Day 04

Họ và tên: Trương Việt Công  
MSSV: 2A202603585

## 1. Case được chọn

- ID case: A5
- Câu hỏi: Cần danh sách đầy đủ incident GitHub Actions trong năm 2025.
- Câu trả lời ban đầu: Agent nói không có incident năm 2025, dù dữ liệu không đủ để xác minh toàn bộ năm.
- Điểm ở lần chạy 1: 50/100.

## 2. Bằng chứng trong trace

Trace A5 gọi `get_status(provider="github", component="Actions")`. Tool báo `matching_incidents_in_last_50 = 23`, nhưng chỉ trả tối đa 5 incident gần nhất. Tool cũng ghi chú: “không tìm thấy không chứng minh không có sự cố”. Các incident trả về thuộc tháng 11/2026; dữ liệu không cho biết toàn bộ năm 2025 có những gì.

Nguyên nhân có thể: model đã biến “không thấy incident năm 2025 trong kết quả giới hạn” thành kết luận “không có incident năm 2025”. Đây là kết luận vượt quá bằng chứng, dù phần sau của câu trả lời có nêu giới hạn.

## 3. Thay đổi đã thực hiện

- Sửa ở đâu: bổ sung hướng dẫn trong `SYSTEM_PROMPT` của `agent.py`.
- Thay đổi: với yêu cầu lịch sử hoặc một khoảng ngày, agent phải nói rõ không thể xác minh nếu kết quả 50 incident gần nhất không bao phủ khoảng đó; không được kết luận không có incident hoặc danh sách đầy đủ chỉ vì không tìm thấy.
- Vì sao chọn thay đổi này: nó nhắm trực tiếp vào lỗi suy luận của A5, không thay đổi dữ liệu hay rubric.

## 4. Kết quả so sánh

| Chỉ số | `eval_before.json` | `eval_after.json` mới |
|---|---:|---:|
| Điểm tổng | 78.64/100 | 77.95/100 |
| Số case được chấm | 22/22 | 22/22 |
| Điểm case A5 | 50/100 | 75/100 |
| Model calls ở A5 | 2 | 6 |
| Token A5 | 1,418 | 4,418 |
| Chi phí A5 | $0.000431 | $0.000975 |

Eval đầy đủ mới dùng cùng cấu hình: OpenRouter, `openai/gpt-4o-mini`, judge cùng model, `max_steps=8`. Không có case nào unscored.

Sau khi thêm hướng dẫn prompt, A5 không còn khẳng định rằng năm 2025 không có incident và nói rõ dữ liệu không đủ để xác minh danh sách đầy đủ. Điểm tăng từ 50 lên 75. Tuy nhiên, câu trả lời vẫn liệt kê dài các incident năm 2026 không thuộc khoảng thời gian được hỏi; judge nhận xét phần này gây nhiễu. A5 dùng 6 tool calls thay vì 2, nên cải tiến còn tốn thêm token và chi phí.

Chạy riêng A5 trong `eval_a5_after_prompt.json` cũng được 75/100. Hai lần chạy A5 cho kết quả tương tự, nhưng điểm LLM judge vẫn chỉ là ước lượng. Điểm tổng của eval đầy đủ giảm từ 78.64 xuống 77.95; các case khác cũng dao động nên không thể kết luận toàn bộ agent đã tốt hơn chỉ từ điểm A5.

Lỗi A5 được giảm nhưng chưa xử lý hết: nên yêu cầu câu trả lời theo đúng khoảng thời gian, chỉ nói không đủ dữ liệu cho năm 2025 và không liệt kê incident năm khác trừ khi người dùng yêu cầu.

## 5. Giới hạn

`get_status` chỉ tìm trong 50 incident gần nhất và trả tối đa 5 kết quả; nó không phải công cụ tra cứu toàn bộ lịch sử theo năm. Snapshot cố định cũng không cho biết trạng thái trực tiếp hiện tại. Điểm LLM judge có thể dao động giữa các lần chạy.
