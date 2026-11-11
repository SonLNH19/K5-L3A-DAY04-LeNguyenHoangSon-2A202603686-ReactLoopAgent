# Ghi chú phân tích · Day 04

Họ và tên: Lê Nguyễn Hoàng Sơn<br>
MSSV: 2A202603686

## 1. Case được chọn

- **ID case:** A1 — điều tra incident GitHub Actions và Cloudflare Spectrum.
- **Câu trả lời của agent:** Lượt đầu có một số URL incident, nhưng thiếu URL mà kiểm tra nguồn yêu cầu; agent tiếp tục sửa đến hết `max_steps`.
- **Điểm ở lần chạy 1:** 0/100.

## 2. Bằng chứng trong trace

Trong trace A1 ở `eval_before.json`, các lần gọi `get_incident_details` đều thành công. Sau đó agent lặp yêu cầu bổ sung cùng một URL từ bước 3 đến bước 8:

```json
{"event":"validation_missing_source","step":3,"missing_sources":["https://stspg.io/lv4q3xm9r0rk"]}
{"event":"validation_missing_source","step":4,"missing_sources":["https://stspg.io/lv4q3xm9r0rk"]}
...
{"event":"validation_missing_source","step":8,"missing_sources":["https://stspg.io/lv4q3xm9r0rk"]}
```

Trace cho thấy lỗi nằm ở điều kiện kiểm tra nguồn: code đòi câu trả lời có URL của từng incident. Model đã đưa một số link khác, nhưng bỏ sót URL trên, nên vòng sửa dùng hết số bước.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:** Trong `agent.py`, chỉ yêu cầu câu trả lời có ít nhất một URL incident đã quan sát từ lần gọi `get_incident_details` thành công. Nếu thiếu, agent ghi `validation_missing_source` và yêu cầu model sửa trong giới hạn `max_steps`. Nếu không có incident lấy chi tiết thành công thì không bắt buộc URL.
- **Vì sao chọn thay đổi này:** Trace A1 cho thấy yêu cầu từng URL gây lặp dù câu trả lời đã có các nguồn incident khác. Cách sửa vẫn kiểm tra sự hiện diện của nguồn, nhưng không buộc model liệt kê URL cho mọi incident. Kiểm tra này không chứng minh mọi claim đều có nguồn.

## 4. So sánh hai lần chạy

Hai lượt dùng cùng cấu hình: OpenRouter, model và judge `openai/gpt-4o-mini`, `max_steps=8`, dataset `core`. Mỗi lượt chấm đủ 22/22 case.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 76.14/100 | 80.45/100 |
| Điểm case đã chọn (A1) | 0/100 | 75/100 |
| Số model calls ghi nhận | 75 | 71 |
| Token, chi phí nếu có | 108,357 tokens; $0.0214719 | 75,933 tokens; $0.0167910 |

- **Lỗi đã giải quyết:** A1 chuyển từ `max_steps` sang `completed`; điểm tăng từ 0 lên 75. Tổng điểm tăng 4.31. Tổng token giảm 32,424; chi phí usage giảm $0.0046809.
- **Lỗi còn lại hoặc mới xuất hiện:** A1 vẫn được 75 vì judge nhận xét còn chi tiết thời gian/trạng thái chưa chính xác. Một số điểm giảm giữa hai lượt: E4, A2 và H8 giảm 15; E2 và H2 giảm 10. Các lượt eval và judge chỉ chạy một lần nên chưa thể kết luận những biến động này do thay đổi trong `agent.py`.

Điểm từng case (trước → sau): E1 75→75, E2 100→90, E3 65→95, E4 100→85, E5 75→75, E6 75→75; A1 0→75, A2 100→85, A3 75→75, A4 100→100, A5 65→75, A6 70→75, A7 85→75, A8 80→100; H1 60→60, H2 100→90, H3 85→100, H4 100→100, H5 75→75, H6 60→60, H7 50→65, H8 80→65.

## 5. Giới hạn

Chạy `python3 -m unittest discover -s tests -v`: **37 tests passed**. Điểm judge là ước lượng và có thể dao động; mỗi phiên bản chỉ có một lượt eval nên chưa chứng minh độ ổn định. Snapshot thể hiện dữ liệu tại thời điểm ghi nhận, còn tool giới hạn số incident và nội dung update. Việc tìm thấy URL không xác minh được mọi nhận định trong câu trả lời.
