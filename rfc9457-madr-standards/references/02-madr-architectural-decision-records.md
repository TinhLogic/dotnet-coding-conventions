# Chuẩn Ghi Nhận Quyết Định Kiến Trúc (MADR - Markdown Architectural Decision Records)

Mọi quyết định kiến trúc quan trọng trong hệ thống (lựa chọn ngôn ngữ, cơ sở dữ liệu, mô hình giao tiếp đồng bộ/bất đồng bộ, chuẩn bảo mật, thiết kế microservices) **bắt buộc phải được ghi chép dưới dạng Architectural Decision Record (ADR)** theo chuẩn **MADR (Markdown Architectural Decision Records v3.0)**.

ADR phải được lưu trữ trực tiếp trong repository mã nguồn (dưới thư mục `docs/decisions/` hoặc `docs/adr/`) và được quản lý vòng đời bằng Git như mã nguồn.

---

## 1. Tại Sao Bắt Buộc Dùng MADR?

1. **Chấm dứt câu hỏi *"Tại sao ngày xưa lại làm như thế này?"***: Khi nhân sự mới vào hoặc sau 2–3 năm, toàn bộ đội ngũ vẫn hiểu rõ lý do, bối cảnh và các đánh đổi (trade-offs) đằng sau mỗi quyết định.
2. **Minh bạch hóa các phương án đã cân nhắc**: Tránh việc tranh cãi lặp đi lặp lại một vấn đề mà trước đó đội ngũ đã phân tích và bác bỏ.
3. **Theo dõi lịch sử tiến hóa kiến trúc**: Khi kiến trúc thay đổi, ta không xóa ADR cũ mà đánh dấu nó là `superseded` (bị thay thế) bởi ADR mới.

---

## 2. Quy Chuẩn Đặt Tên & Quản Lý Vòng Đời

### 2.1. Cấu trúc thư mục & Quy ước đặt tên file
```text
my-project/
├── docs/
│   └── decisions/
│       ├── 0000-use-markdown-architectural-decision-records.md
│       ├── 0001-use-postgresql-for-relational-data.md
│       ├── 0002-adopt-rfc9457-for-api-error-handling.md
│       └── 0003-use-redis-for-distributed-caching.md
```
- Số thứ tự có 4 chữ số (`0001`, `0002`...) tăng dần theo thời gian.
- Tên file viết bằng chữ thường, nối bằng dấu gạch ngang (`kebab-case`), bắt đầu bằng động từ thể hiện quyết định (`use-...`, `adopt-...`, `replace-...`).

### 2.2. Trạng thái vòng đời của một ADR (Lifecycle Statuses)
- **`proposed`**: Đang trong quá trình đề xuất và thảo luận (Review PR).
- **`accepted`**: Đã được đội ngũ kiến trúc/kỹ thuật phê duyệt và cam kết thực thi.
- **`rejected`**: Sau khi phân tích, phương án bị từ chối (vẫn lưu lại để ghi nhận lý do từ chối).
- **`deprecated`**: Quyết định không còn phù hợp với hiện tại nhưng chưa có giải pháp thay thế trực tiếp.
- **`superseded by [ADR-XXXX](XXXX-...)`**: Quyết định đã bị thay thế hoàn toàn bởi một ADR mới hơn.

---

## 3. Bản Mẫu Chuẩn MADR 3.0 (Official Template)

Mọi file ADR tạo mới phải tuân theo khung mẫu chuẩn sau:

```markdown
# [Tiêu đề: Giải pháp đã chọn cho vấn đề cần giải quyết]

* **Trạng thái**: [proposed | accepted | rejected | deprecated | superseded by ADR-XXXX]
* **Người quyết định**: [Danh sách tên / GitHub usernames của những người tham gia]
* **Ngày quyết định**: [YYYY-MM-DD]
* **Tài liệu liên quan**: [Link PR, issue, hoặc ticket Jira]

## Bối Cảnh & Bài Toán (Context and Problem Statement)

[Mô tả bối cảnh hiện tại của hệ thống và vấn đề kỹ thuật/nghiệp vụ đang gặp phải. Tại sao chúng ta phải đưa ra quyết định này ngay lúc này? Phạm vi ảnh hưởng là gì?]

## Yếu Tố Thúc Đẩy Quyết Định (Decision Drivers)

* [Yếu tố 1: ví dụ Hiệu năng cao (đáp ứng 10.000 RPS)]
* [Yếu tố 2: ví dụ Chi phí vận hành hạ tầng đám mây tối ưu]
* [Yếu tố 3: ví dụ Dễ bảo trì và làm quen cho nhân sự mới]
* [Yếu tố 4: ví dụ Tuân thủ chuẩn bảo mật PCI-DSS / HIPAA]

## Các Phương Án Đã Cân Nhắc (Considered Options)

1. [Phương án 1 - ví dụ: Sử dụng PostgreSQL]
2. [Phương án 2 - ví dụ: Sử dụng MongoDB]
3. [Phương án 3 - ví dụ: Sử dụng Amazon DynamoDB]

## Kết Quả Quyết Định (Decision Outcome)

Phương án được chọn: **[Tên phương án được chọn]**, bởi vì:
[Giải thích ngắn gọn lý do cốt lõi tại sao phương án này chiến thắng các phương án còn lại dựa trên các Decision Drivers ở trên].

### Hệ Quả Tích Cực (Positive Consequences)
* [Lợi ích 1: Đạt được tính toàn vẹn giao dịch ACID tuyệt đối]
* [Lợi ích 2: Hệ sinh thái công cụ hỗ trợ phong phú, cộng đồng lớn]

### Hệ Quả Tiêu Cực & Đánh Đổi (Negative Consequences & Trade-offs)
* [Rủi ro / Điểm yếu: Cần thiết lập Read-Replica để mở rộng quy mô đọc lớn]
* [Chi phí: Đội ngũ cần học thêm về tối ưu hóa chỉ mục JSONB]

---

## Phân Tích Đa Chiều Các Phương Án (Pros and Cons of the Options)

### [Phương án 1]
[Mô tả ngắn gọn về phương án 1]

* 👍 **Ưu điểm**: [Ưu điểm A]
* 👍 **Ưu điểm**: [Ưu điểm B]
* 👎 **Nhược điểm**: [Nhược điểm X]
* 👎 **Nhược điểm**: [Nhược điểm Y]

### [Phương án 2]
[Mô tả ngắn gọn về phương án 2]

* 👍 **Ưu điểm**: [Ưu điểm A]
* 👎 **Nhược điểm**: [Nhược điểm X]

---

## Kế Hoạch Xác Thực (Validation / Confirmation)

[Làm thế nào để xác nhận quyết định này đã được thực thi thành công trong thực tế? Ví dụ: Chạy bài kiểm thử tải BenchmarkDotNet, đo lường chỉ số Latency P99 trên Datadog sau 30 ngày triển khai.]

## Thông Tin Tham Khảo Thêm (More Information)

* [Link tài liệu kỹ thuật của công nghệ]
* [Link bài viết blog kiến trúc / Whitepaper liên quan]
```

---

## 4. Ví Dụ ADR Hoàn Chỉnh Thực Tế

```markdown
# 0002. Chuẩn Hóa Định Dạng Phản Hồi Lỗi API Theo RFC 9457 (Problem Details)

* **Trạng thái**: accepted
* **Người quyết định**: @duytinh, @tech-lead, @frontend-lead
* **Ngày quyết định**: 2026-10-23
* **Tài liệu liên quan**: PR #45, Issue #120

## Bối Cảnh & Bài Toán (Context and Problem Statement)

Hiện tại, các microservices trong hệ thống đang trả về lỗi với nhiều cấu trúc JSON không đồng nhất:
* Auth Service trả về: `{"error": "Unauthorized", "code": 401}`
* Payment Service trả về: `{"success": false, "message": "Số dư không đủ"}`
* Order Service trả về: `{"errors": ["Tên sản phẩm không hợp lệ"]}`

Sự bất đồng nhất này khiến đội ngũ Frontend và Mobile App phải viết hàng chục khối `try-catch` và `switch-case` riêng biệt cho từng service, làm tăng 40% thời gian phát triển và khó khăn khi tích hợp đối tác bên thứ ba.

## Yếu Tố Thúc Đẩy Quyết Định (Decision Drivers)

* Đạt được định dạng lỗi duy nhất, chuẩn hóa trên 100% API nội bộ và công khai.
* Hỗ trợ truy vết lỗi phân tán (Distributed Tracing) qua `traceId`.
* Tương thích tốt với các thư viện sinh client tự động (OpenAPI Generator).
* Tiêu chuẩn quốc tế được cộng đồng IETF công nhận.

## Các Phương Án Đã Cân Nhắc (Considered Options)

1. Tự thiết kế định dạng JSON riêng của công ty (`EnterpriseErrorResponse`).
2. Chuẩn Google Cloud JSON Style Guide (`{ "error": { "code": 400, "message": ... } }`).
3. Chuẩn IETF RFC 9457 (Problem Details for HTTP APIs - thay thế RFC 7807).

## Kết Quả Quyết Định (Decision Outcome)

Phương án được chọn: **Chuẩn IETF RFC 9457**, bởi vì:
* Đây là tiêu chuẩn chính thức của IETF (kế thừa và hoàn thiện RFC 7807), được hỗ trợ nguyên bản (out-of-the-box) bởi ASP.NET Core, Spring Boot 3+, NestJS và FastAPI.
* Cung cấp header chuẩn `application/problem+json` giúp client dễ dàng phân biệt giữa dữ liệu bình thường và phản hồi lỗi.

### Hệ Quả Tích Cực
* Toàn bộ Frontend/Mobile có thể dùng 1 bộ giải mã lỗi (Error Handler Interceptor) duy nhất cho mọi request.
* Tự động đính kèm `traceId` giúp đội ngũ hỗ trợ khách hàng tra cứu log trên Datadog/Grafana trong dưới 30 giây.

### Hệ Quả Tiêu Cực & Đánh Đổi
* Các service cũ cần viết Exception Middleware để chuyển đổi cấu trúc lỗi cũ sang RFC 9457.

---

## Phân Tích Đa Chiều Các Phương Án

### Phương Án 1: Tự thiết kế định dạng riêng
* 👍 **Ưu điểm**: Đội ngũ tự do thêm bớt các trường theo sở thích.
* 👎 **Nhược điểm**: Không có tài liệu tiêu chuẩn quốc tế; các công cụ OpenAPI/Swagger không tự động hỗ trợ.

### Phương Án 2: Chuẩn Google Cloud Style
* 👍 **Ưu điểm**: Đã được Google kiểm chứng qua hàng trăm API lớn.
* 👎 **Nhược điểm**: Lồng ghép sâu (`error.details[0]`), thiếu tính linh hoạt cho các trường mở rộng cấp cao.

### Phương Án 3: Chuẩn RFC 9457
* 👍 **Ưu điểm**: Tiêu chuẩn IETF chính thức, cấu trúc phẳng, hỗ trợ mở rộng không giới hạn (`invalidParams`, `traceId`).
* 👍 **Ưu điểm**: Tích hợp sẵn trong .NET 7/8/9 (`AddProblemDetails()`) và Spring Boot 3 (`ProblemDetail`).

---

## Kế Hoạch Xác Thực (Validation / Confirmation)

* Tạo một Pull Request tích hợp `ProblemDetailsMiddleware` vào Template Service nền tảng.
* Toàn bộ 5 core services chuyển đổi thành công trong Sprint 24.
```

