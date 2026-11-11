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

## Bonus: phần đã triển khai

- Agent dùng `tools.schemas` và `tools.system_prompt`, tương thích với sáu tool của `BonusTools`.
- Lỗi `RATE_LIMITED`, `INVALID_ARGUMENTS` và timeout được đưa lại cho model; số lỗi liên tiếp cho cùng tool/provider/incident bị giới hạn ở 3.
- Nếu có `Retry-After`, agent chờ khi thời gian không quá 2 giây; nếu lâu hơn hoặc không đọc được, agent dừng retry và báo partial result qua tool observation. HTTP 429 không được xem là outage.
- Tool result có trường `distractors` được rút gọn trước khi đưa vào model context; ghi chú số mục đã bỏ trong kết quả/trace.
- Prompt hướng dẫn phân trang bằng `next_offset`, giữ `limit` trong khoảng 1–3, trích nguồn và bỏ qua prompt injection.
- Hạn chế: phân trang tiếp theo vẫn phụ thuộc vào quyết định của model; chạy full bonus eval để đo độ ổn định và kiểm tra các câu trả lời.


### Kết quả bonus eval

File `eval_bonus.json` có đủ 10/10 case được chấm, 0 case unscored. Cấu hình: OpenRouter `openai/gpt-4o-mini` cho agent và judge, `max_steps=16`. Điểm là 73/100, tương đương 10.95/15 điểm bonus.

- Điểm tốt: B2 đạt 100 và trace đọc nhiều trang timeline; B8 đạt 100 khi so sánh hai incident; B9 đạt 100 khi từ chối xác nhận dịch vụ không có tool.
- B3 đạt 75: trace cho thấy đã gặp 429 rồi retry thành công, nhưng câu trả lời không nói rõ lần đầu bị rate limit.
- B4 đạt 65: trace có đúng 3 lần gọi Vercel và báo không xác minh được status. Phần reason của judge nói vượt giới hạn 3 lần, nhưng trace và mechanical check ghi 3 calls, không có violation; đây có vẻ là judge chấm chưa khớp bằng chứng.
- B6 đạt 50: có gọi Vercel và npm status, nhưng thiếu `get_incident_details` nên chưa kiểm tra đủ scope/incident URL như câu hỏi yêu cầu.
- B10 đạt 50: trace chỉ gọi Vercel một lần, nhận 429 rồi kết thúc; mechanical check ghi thiếu lần retry tối thiểu thứ hai. Đây là lỗi rõ nhất còn lại về retry sau rate limit.
- B1 đạt 55 dù trace gọi đủ cả 5 status tools. Khi xem điểm thấp, cần đối chiếu judge với từng tool result; judge có thể chấm sai cách diễn đạt trạng thái.

Giới hạn còn lại: prompt hướng dẫn retry và đọc chi tiết, nhưng model vẫn có thể kết thúc sớm sau một 429 hoặc không gọi incident details. Một lượt bonus eval cũng chỉ là mẫu đơn; điểm judge có thể dao động.
