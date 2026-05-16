# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: 01 (Camera Stream → AI Vision)
- Product: Smart Campus Operations Platform
- Provider: AI Vision
- Consumer: Camera Stream
- Phiên: v1.0
- Ngày: 16/05/2026

---

## Issue #1: Định dạng dữ liệu truyền tải hình ảnh

- Raised by: Consumer (Camera Stream)
- Endpoint: `POST /vision/detect`
- Concern: Camera gửi frame ảnh dạng nhị phân (multipart/form-data) sẽ rất nặng và tốn thời gian truyền tải, có thể làm tắc nghẽn luồng xử lý.
- Proposal: Thay vì gửi file ảnh gốc, Camera Stream sẽ lưu ảnh lên một blob storage và chỉ gửi đường dẫn URL (`imageUrl`) dưới dạng JSON để AI Vision tự tải về xử lý.
- Resolution: Accepted
- Rationale: Giảm tải băng thông cho API Gateway và tăng tốc độ nhận request. Dữ liệu trao đổi thuần JSON dễ dàng validate (kiểm tra) hơn.
- Impact: Request body đổi thành `application/json`, có trường bắt buộc là `imageUrl`.

---

## Issue #2: Giới hạn kích thước và định dạng ảnh

- Raised by: Provider (AI Vision)
- Endpoint: `POST /vision/detect`
- Concern: Nếu URL ảnh trỏ đến một file 4K hoặc file rác quá lớn, server AI sẽ bị treo hoặc hết bộ nhớ (OOM) khi cố tải về.
- Proposal: Thống nhất chỉ hỗ trợ định dạng JPG/PNG và kích thước file tối đa khi AI tải về không vượt quá 5MB. Nếu vượt quá, AI Vision sẽ hủy xử lý và trả về lỗi.
- Resolution: Accepted
- Rationale: Cần bảo vệ server GPU của AI Vision khỏi các payload quá lớn dẫn đến sập hệ thống.
- Impact: AI Vision sẽ trả về lỗi `422 Unprocessable Entity` với thông báo "Kích thước ảnh vượt quá 5MB" nếu vi phạm.

---

## Issue #3: Cơ chế phản hồi Đồng bộ vs. Bất đồng bộ

- Raised by: Consumer (Camera Stream)
- Endpoint: `POST /vision/detect`
- Concern: Mô hình AI chạy nhận diện (detect) có thể mất vài giây. Nếu Camera Stream phải gọi đồng bộ (chờ AI xử lý xong mới nhận phản hồi) thì connection sẽ bị treo lâu dẫn tới Timeout.
- Proposal: Chuyển sang mô hình polling. Khi nhận request `POST`, Provider trả về HTTP `202 Accepted` kèm theo một `detectionId` ngay lập tức. Sau đó Consumer tự gọi `GET /vision/detections/{detectionId}` để kiểm tra kết quả định kỳ.
- Resolution: Accepted
- Rationale: Đảm bảo luồng chạy của Camera Stream không bị chặn (block) khi đợi AI, nhất là khi có nhiều camera cùng phát hiện chuyển động.
- Impact: Cần thiết kế thêm endpoint `GET /vision/detections/{detectionId}`.

---

## Issue #4: Xử lý trùng lặp (Idempotency)

- Raised by: Provider (AI Vision)
- Endpoint: `POST /vision/detect`
- Concern: Nếu mạng chập chờn, Camera retry gửi yêu cầu phân tích cho cùng một frame ảnh nhiều lần, AI sẽ phải xử lý lặp lại gây lãng phí tài nguyên GPU.
- Proposal: Consumer bắt buộc phải gửi kèm một `eventId` (UUID) trong mỗi request. Provider sẽ dùng nó làm khóa chống trùng (idempotency key). Nếu thấy `eventId` đã tồn tại, Provider chỉ trả lại `detectionId` cũ mà không chạy lại AI.
- Resolution: Accepted
- Rationale: Tiết kiệm tối đa tài nguyên xử lý đắt đỏ của hệ thống AI.
- Impact: Bổ sung trường bắt buộc `eventId` vào schema của request body.

---

## Issue #5: Xử lý trường hợp không có vật thể (Low Confidence)

- Raised by: Consumer (Camera Stream)
- Endpoint: `GET /vision/detections/{detectionId}`
- Concern: Nếu mô hình không tìm thấy ai trong ảnh, hoặc độ tin cậy (confidence) quá thấp, API sẽ trả về lỗi hay thành công?
- Proposal: Vẫn trả về HTTP `200 OK`, nhưng mảng `objects` sẽ là mảng rỗng `[]` và trường `riskLevel` sẽ có giá trị `null`.
- Resolution: Accepted
- Rationale: Không nhận diện được vật thể là một kết quả nghiệp vụ hợp lệ, không phải là lỗi hệ thống (4xx/5xx). Chuẩn OpenAPI 3.1 có hỗ trợ gán giá trị null bằng union type.
- Impact: Schema của `riskLevel` phải định nghĩa là `type: [string, "null"]` thay vì nullable.

---

## Issue #6: Chuẩn hóa định dạng báo lỗi

- Raised by: Provider (AI Vision)
- Endpoint: Tất cả các Endpoints
- Concern: Cần thống nhất cấu trúc báo lỗi để Consumer có thể lập trình bắt lỗi (parse error) một cách tự động và nhất quán cho mọi tình huống (400, 401, 409, 422, 500...).
- Proposal: Sử dụng chung schema `Problem` theo chuẩn RFC 7807 (`application/problem+json`) cho mọi phản hồi lỗi.
- Resolution: Accepted
- Rationale: Tuân thủ đúng quy định kỹ thuật của Lab 02 và giúp Consumer không cần viết nhiều khối if/else để xử lý các loại cấu trúc lỗi khác nhau.
- Impact: Thêm schema `Problem` vào `components/schemas` và gắn vào phần `responses` của mọi operation.

---

# Chốt hợp đồng v1.0

Provider sign-off:  <Điền tên bạn đóng vai AI Vision>
Consumer sign-off:  <Điền tên của bạn>
Witness (GV/TA):    (Để trống hoặc điền tên giảng viên)
Date:               16/05/2026

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| (Hiện chưa có) | | |