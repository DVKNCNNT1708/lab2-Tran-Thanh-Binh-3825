# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: Pair 01 (Camera Stream → AI Vision)
- Product: A / B 
- Consumer service: Camera Stream
- Provider service: AI Vision
- Người viết: Trần Thanh Bình
- Ngày: 16/5/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `DetectionResult` | Nhận kết quả phân tích ảnh từ AI Vision để chuyển tiếp cho Core Business khi phát hiện có bất thường. | `detectionId`, `objects`, `confidence`, `riskLevel` | `model_version`, `bounding_boxes` |
| `Frame/Metadata` | Gửi dữ liệu ảnh cho AI Vision khi phát hiện có chuyển động (motion). | `cameraId`, `timestamp`, `frame_data_or_url` | `resolution` |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| POST | `/vision/detect` | Gọi ngay khi camera phát hiện có chuyển động (motion). | Trả về `detectionId`, danh sách `objects`, `confidence` và `riskLevel` thật nhanh. |
| GET | `/vision/detections/{detectionId}` | Gọi khi cần polling kết quả (nếu AI xử lý lâu) hoặc cần lấy lại thông tin cũ. | Chi tiết của lần nhận diện đó. |
| GET | `/vision/models/info` | Gọi lúc khởi động hệ thống để biết AI đang dùng model nào. | Thông tin version model AI. |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Dữ liệu sai định dạng (thiếu ảnh hoặc id). | Sửa payload/ghi log lỗi, không gửi lại. |
| 401 | Thiếu token hoặc sai chứng chỉ. | Cảnh báo lỗi cấu hình xác thực của Camera. |
| 409 | Trùng event/request gây xử lý lặp. | Bỏ qua request này, coi như đã được ghi nhận. |
| 422 | Frame ảnh bị lỗi không thể đọc/xử lý. | Cảnh báo phần cứng/chất lượng Camera. |
| 504 | Timeout (AI Vision phản hồi quá chậm). | Đưa request vào hàng đợi để retry (thử lại) sau. |

---

## 4. Giả định bổ sung

- Giả định 1: Camera sẽ gửi ảnh dưới dạng Base64 hoặc URL (cần chốt với Provider).
- Giả định 2: Camera sẽ tự tạo `correlationId` gắn vào mỗi request để dễ dàng trace log sau này.

---

## 5. Câu hỏi cho Provider

1. Ảnh gửi dạng multipart hay URL?
2. Giới hạn kích thước frame là bao nhiêu?
3. AI Vision trả kết quả đồng bộ (chờ AI chạy xong) hay trả `detectionId` để Camera polling lấy kết quả sau?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider xử lý quá chậm (Timeout). | Mất dữ liệu cảnh báo quan trọng, nghẽn luồng Camera. | Chốt thời gian phản hồi tối đa (SLA) và cơ chế async nếu cần. |
| Trùng lặp sự kiện gửi đi. | AI Vision xử lý lặp lại gây tốn tài nguyên. | Provider phải thiết kế API hỗ trợ idempotency (dùng `eventId` để lọc trùng). |