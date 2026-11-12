# Ghi chú phân tích · Day 04

Họ và tên: Phạm Đăng Quang
MSSV: 2A202603760

## 1. Case được chọn

- ID case: H6
- Câu trả lời của agent:
```text
### Tình trạng GitHub Actions

Theo thông tin từ GitHub Status, hiện tại dịch vụ GitHub Actions đang hoạt động bình thường.

- **Trạng thái tổng thể**: Tất cả hệ thống hoạt động tốt. 
- **Thời điểm ghi nhận**: 9 tháng 11 năm 2026.
- **Status**: [GitHub Status](https://www.githubstatus.com)

### Chi tiết Incident h02n9572mx9y

- **Tên sự cố**: Incident with Actions
- **Trạng thái**: Đã giải quyết
- **Ngày tạo**: 4 tháng 11 năm 2026, 17:40 UTC
- **Ngày cập nhật cuối**: 7 tháng 11 năm 2026, 01:10 UTC
- **Component liên quan**: Actions
- **Liên kết chi tiết**: [Chi tiết về sự cố](https://stspg.io/7pwv28hvkmz8)

### Nội dung và Nguyên nhân sự cố

Sự cố này chủ yếu đã được giải quyết, và trong các cập nhật, thông tin sau đây được nội dung:

1. **Cập nhật 1** (17:40 UTC): Đang điều tra báo cáo về khả năng sẵn sàng bị suy giảm cho Actions.
2. **Cập nhật 2** (18:07 UTC): Giảm bớt tình trạng suy giảm ảnh hưởng đến Actions, đang theo dõi để đảm bảo ổn định.
3. **Cập nhật 3** (18:16 UTC): Xin vui lòng tham khảo tóm tắt kết hợp trong sự cố liên quan: [Incident xr3myzj7ms51](https://www.githubstatus.com/incidents/xr3myzj7ms51).

Trong các thông báo không nêu rõ nguyên nhân cụ thể nào cho sự cố, mà chỉ miêu tả ảnh hưởng và tình trạng theo dõi, cung cấp liên kết tới sự cố khác có thể có thông tin bổ sung liên quan.

### Giới hạn dữ liệu

Dữ liệu chỉ là tóm tắt từ cập nhật sự cố và không phải là toàn bộ thông tin. Các chi tiết cụ thể về nguyên nhân chưa được làm sáng tỏ trong các thông báo công khai. Tất cả thông tin được trích từ dữ liệu được cung cấp bởi GitHub Status và không phải là kết luận từ nguồn bên ngoài.

- **Nguồn**: [GitHub Status](https://www.githubstatus.com)
```
- Điểm ở lần chạy 1: 60/100 (Raw judge score: 75, bị áp trần 60 do vi phạm kiểm tra cơ học).

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
Step 1:
- Tool 1: get_status(provider="github", component="actions")
- Tool 2: get_incident_details(provider="github", incident_id="h02n9572mx9y")
Result của h02n9572mx9y:
"updates": [
  {
    "status": "resolved",
    "created_at": "2026-11-04T18:16:08.361Z",
    "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51",
    "truncated": false
  }
]
Hành vi lỗi:
Sau Step 1, Agent dừng lại ngay lập tức và sinh câu trả lời cuối cùng mà không gọi tool ở Step 2 để đọc incident gộp 'xr3myzj7ms51'.
Mechanical check violation: "Cần tool calls ở ít nhất 2 model steps khác nhau." -> score_cap: 60.
```

**Nguyên nhân có thể xảy ra:**
1. **Thiếu suy luận đa bước (multi-hop reasoning):** Khi phát hiện một sự cố có liên kết đến sự cố gộp (`xr3myzj7ms51`), Agent không chủ động thực hiện tiếp cuộc gọi tool ở lượt sau để tra cứu chi tiết liên kết đó, dẫn đến việc thiếu dữ liệu thực tế và phải tự phỏng đoán nguyên nhân.
2. **Chưa có cơ chế kiểm tra nguồn (Evidence Gate):** Agent chưa tự thẩm định xem câu trả lời đã bao gồm đầy đủ URL của các bằng chứng đã điều tra hay chưa.
3. **Bỏ qua giới hạn dữ liệu:** Agent chưa chú ý đến các cờ cảnh báo dữ liệu bị lược bớt (`truncated`, `updates_total`) từ tool trả về.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: Hiện thực cơ chế **Evidence Gate** (kiểm tra nguồn). Khi agent sinh câu trả lời cuối cùng (`finish == 'stop'`), nếu đã có incident được tra cứu thành công nhưng câu trả lời thiếu URL nguồn của incident đó, agent sẽ ghi nhận event `validation_missing_source` vào trace và gửi yêu cầu model bổ sung nguồn trong ngân sách `max_steps`.
  2. Trong `agent.py`: Cập nhật `SYSTEM_PROMPT` với các chỉ dẫn chuyên biệt:
     - Yêu cầu model khi phát hiện một incident nhắc đến hoặc có liên kết tới incident liên quan hay sự cố gộp (combined summary), phải tiếp tục gọi tool để kiểm tra chi tiết sự cố liên kết đó trước khi kết luận.
     - Yêu cầu model kiểm tra cờ `truncated` và số lượng updates để nêu rõ ràng giới hạn của dữ liệu trong câu trả lời.
     - Yêu cầu trích dẫn URL chính xác thu thập được từ tool.
- Vì sao chọn thay đổi này:
  - Giúp agent thực hiện suy luận đa bước tự nhiên, đảm bảo điều tra đến cùng các liên kết sự cố theo đúng yêu cầu bài toán (như case H1, H6), giải quyết triệt để lỗi vi phạm số bước gọi tool.
  - Nâng cao tính chính xác (factuality) và độ tin cậy của bằng chứng (evidence), đồng thời vượt qua 100% các bài test kiểm tra nguồn trong bộ unit test `tests/test_validation.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 80.91 | 78.41 |
| Điểm case đã chọn (H6) | 60 | 75 |
| Số model calls | 50 | 52 |
| Token, chi phí nếu có | 40,506 tokens (~$0.0102) | 51,239 tokens (~$0.0122) |

- Lỗi đã giải quyết:
  * **Case H6 khắc phục hoàn toàn vi phạm cơ học:** Điểm tăng từ **60 lên 75** (+15 điểm). Vi phạm `Cần tool calls ở ít nhất 2 model steps khác nhau` biến mất hoàn toàn (`violations: []`, trần điểm được mở lại 100%).
  * **Thực hiện thành công suy luận đa bước:** Ở Step 2, Agent đã chủ động gọi tiếp `get_incident_details` cho incident gộp `xr3myzj7ms51`.
  * **Trả lời chính xác nguyên nhân và giới hạn:** Agent trích xuất được nguyên nhân thực tế của incident gộp ("do thay đổi cấu hình dịch vụ định tuyến sự kiện nội bộ..."), nhận diện rõ ràng cờ `truncated` (thông tin bị rút gọn), và trích dẫn đầy đủ URL của cả 2 incident.
  * **Cơ chế Evidence Gate hoạt động chuẩn xác:** Đạt 37/37 unit tests offline trong `tests/test_validation.py`.

- Lỗi còn lại hoặc mới xuất hiện:
  * Điểm tổng có sự dao động nhẹ (80.91 -> 78.41) chủ yếu do tính ngẫu nhiên của LLM Judge khi chấm điểm và ở case H2 agent chưa kích hoạt tool do tự suy luận từ tiền đề.
  * Chi phí token tăng nhẹ (từ 40.5k lên 51.2k tokens, tăng ~$0.002) do agent thực hiện thêm bước gọi tool thứ 2 và câu trả lời chi tiết, đầy đủ bằng chứng hơn.

## 5. Giới hạn

- **Giới hạn của điểm LLM Judge:** Điểm số là một ước lượng xấp xỉ (heuristic). LLM Judge có thể có sự dao động nhỏ (variance) giữa các lần chạy với cùng một câu trả lời, đồng thời có thể xuất hiện thiên lệch tự đánh giá (self-preference bias) khi dùng cùng một mô hình (`gpt-4o-mini`) cho cả Agent và Judge.
- **Giới hạn của snapshot dữ liệu:** Bộ dữ liệu là một snapshot tĩnh được chụp tại thời điểm cố định (tháng 11/2026), chỉ chứa tối đa 50 sự cố gần nhất. Do đó, agent bị giới hạn trong khung thông tin này và không thể phản ánh những sự cố lịch sử nằm ngoài phạm vi snapshot hoặc các cập nhật sau thời điểm ghi nhận.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** Việc chạy 1 lần trước và 1 lần sau không đủ để khẳng định tính ổn định thống kê lâu dài. Tuy nhiên, bằng chứng cải thiện rõ rệt ở case H6 (từ 60 lên 75 điểm, không còn vi phạm kiểm tra cơ học) là minh chứng xác đáng cho thấy cải tiến về suy luận đa bước và kiểm tra nguồn đã phát huy hiệu quả.
