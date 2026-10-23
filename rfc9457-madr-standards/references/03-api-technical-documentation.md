# Quy Chuẩn Viết Tài Liệu Kỹ Thuật API (API Technical Specifications)

Tài liệu kỹ thuật API (API Tech Spec / Specification) là bản hợp đồng cam kết giữa các đội ngũ phát triển (Frontend, Backend, Mobile, DevOps, QA, Đối tác bên ngoài). Một tài liệu API chuẩn Enterprise phải rõ ràng, không mập mờ và cung cấp đầy đủ thông tin để lập trình viên có thể phát triển độc lập mà không cần hỏi lại.

---

## 1. Cấu Trúc Bắt Buộc Của Một Tài Liệu Kỹ Thuật API

Một tài liệu đặc tả API chuẩn phải bao gồm các phần sau:

```text
1. Tổng quan & Ngữ cảnh nghiệp vụ (Overview & Business Context)
2. Mô hình hóa tài nguyên RESTful (Resource Modeling & Endpoints)
3. Xác thực & Phân quyền (Authentication & Authorization Matrix)
4. Đặc tả chi tiết từng Endpoint (Endpoint Specifications)
   - Headers (Bắt buộc & Tùy chọn)
   - Parameters (Path, Query)
   - Request Body Schema (Kèm ràng buộc Validation)
   - Phản hồi thành công (Success Responses: 200, 201, 204)
   - Phản hồi lỗi chuẩn RFC 9457 (Error Responses: 400, 401, 403, 404, 409, 422, 500)
5. Cơ chế Idempotency & Retry an toàn (Idempotency Key)
6. Giới hạn tần suất & Hiệu năng (Rate Limiting & SLA)
```

---

## 2. Quy Chuẩn Đặt Tên & Thiết Kế URL RESTful

- **Danh từ số nhiều, chữ thường, nối bằng dấu gạch ngang (`kebab-case`)**:
  - ✅ **ĐÚNG**: `/api/v1/user-profiles`, `/api/v1/orders/{orderId}/order-items`
  - ❌ **SAI**: `/api/v1/getUserProfiles` (chứa động từ), `/api/v1/order_items` (snake_case)
- **Ánh xạ đúng phương thức HTTP (HTTP Verbs)**:
  - `GET`: Đọc tài nguyên (Safe, Idempotent, không có Body).
  - `POST`: Tạo mới tài nguyên hoặc kích hoạt quy trình nghiệp vụ (Không Idempotent, trả về `201 Created` kèm header `Location`).
  - `PUT`: Thay thế toàn bộ tài nguyên (Idempotent).
  - `PATCH`: Cập nhật một phần thuộc tính tài nguyên (JSON Merge Patch RFC 7396 hoặc JSON Patch RFC 6902).
  - `DELETE`: Xóa tài nguyên (Idempotent, trả về `204 No Content` hoặc `200 OK`).

---

## 3. Bản Mẫu Đặc Tả Một Endpoint Chi Tiết

### `POST /api/v1/payment-transfers` (Tạo lệnh chuyển tiền)

#### 1. Mô tả nghiệp vụ
Tạo một lệnh chuyển tiền mới từ tài khoản nguồn sang tài khoản thụ hưởng. Thao tác này trừ tiền tài khoản nguồn và ghi có tài khoản đích dưới giao dịch ACID.

#### 2. Quyền truy cập (Authorization)
- **Loại xác thực**: `Bearer JWT Token`
- **Scopes / Roles yêu cầu**: `payments:create` hoặc Role `Customer`, `FinanceOfficer`

#### 3. Headers yêu cầu
| Header Name | Kiểu | Bắt buộc | Mô tả |
| :--- | :--- | :--- | :--- |
| `Authorization` | `string` | Có | `Bearer <access_token>` |
| `Content-Type` | `string` | Có | `application/json` |
| `Idempotency-Key`| `string (UUID)` | Có | Khóa chống trùng lặp giao dịch khi người dùng click đúp hoặc retry mạng |

#### 4. Request Body Schema
```json
{
  "sourceAccountId": "acc-09182374",
  "destinationAccountId": "acc-11239845",
  "amount": 2500000,
  "currency": "VND",
  "description": "Thanh toan hop dong so HD-2026-09"
}
```

**Ràng buộc kiểm tra tính hợp lệ (Validation Rules):**
- `sourceAccountId`: `string`, UUID hợp lệ, bắt buộc.
- `destinationAccountId`: `string`, UUID hợp lệ, bắt buộc, phải khác `sourceAccountId`.
- `amount`: `integer`, tối thiểu `10.000`, tối đa `500.000.000`, bắt buộc.
- `currency`: `string`, mã ISO 4217, độ dài đúng 3 ký tự (chỉ hỗ trợ `VND`, `USD`).
- `description`: `string`, tối đa 255 ký tự, không chứa ký tự điều khiển.

#### 5. Phản hồi thành công (Success Response: `201 Created`)
- **Headers**:
  - `Content-Type: application/json`
  - `Location: /api/v1/payment-transfers/txn-99882211`
- **Body**:
```json
{
  "transactionId": "txn-99882211",
  "sourceAccountId": "acc-09182374",
  "destinationAccountId": "acc-11239845",
  "amount": 2500000,
  "currency": "VND",
  "status": "COMPLETED",
  "createdAt": "2026-10-23T07:50:00.000Z"
}
```

#### 6. Các phản hồi lỗi chuẩn RFC 9457 (Error Responses)
- **`400 Bad Request`**: Dữ liệu JSON sai cú pháp hoặc thiếu Header `Idempotency-Key`.
- **`401 Unauthorized`**: Token không hợp lệ hoặc đã hết hạn.
- **`403 Forbidden`**: Tài khoản nguồn không thuộc quyền sở hữu của người dùng hiện tại.
- **`409 Conflict`**: Khóa `Idempotency-Key` đang được xử lý trong một request song song khác.
- **`422 Unprocessable Entity`**: Số dư tài khoản không đủ để thực hiện giao dịch.
  ```json
  {
    "type": "https://api.example.com/errors/insufficient-balance",
    "title": "Số dư không đủ",
    "status": 422,
    "detail": "Số dư khả dụng hiện tại là 1.200.000 VND, không đủ cho giao dịch 2.500.000 VND.",
    "instance": "/api/v1/payment-transfers",
    "errorCode": "INSUFFICIENT_FUNDS",
    "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  }
  ```

---

## 4. Quy Chuẩn Cơ Chế Idempotency Cho Các Thao Tác Nhạy Cảm

Đối với các API nhạy cảm (Tạo đơn hàng, Thanh toán, Trừ điểm thưởng, Gửi email thông báo):
- Bắt buộc yêu cầu client gửi Header: `Idempotency-Key: <UUIDv4>`.
- **Cơ chế hoạt động tại máy chủ**:
  1. Kiểm tra cache Redis xem `Idempotency-Key` đã tồn tại chưa (TTL: 24h).
  2. **Nếu chưa tồn tại**: Lưu khóa vào Redis với trạng thái `PROCESSING`, thực thi nghiệp vụ, ghi nhận kết quả và cập nhật trạng thái `COMPLETED` cùng toàn bộ body phản hồi.
  3. **Nếu đang `PROCESSING`**: Trả về `409 Conflict` yêu cầu client chờ.
  4. **Nếu đã `COMPLETED`**: Trả về ngay lập tức kết quả đã lưu trong cache mà **không thực thi lại nghiệp vụ thanh toán một lần nữa**.

---

## 5. Quy Chuẩn Phân Trang (Pagination Standards)

Hệ thống cung cấp 2 chuẩn phân trang rõ ràng tùy ngữ cảnh:

### 1. Phân trang theo Offset (Dành cho trang Web Admin có số trang cụ thể)
- Tham số truy vấn: `?pageIndex=1&pageSize=20` (hoặc `?skip=0&take=20`)
- Kết quả phản hồi:
  ```json
  {
    "totalCount": 1420,
    "pageIndex": 1,
    "pageSize": 20,
    "items": [ ... ]
  }
  ```

### 2. Phân trang theo Con trỏ (Keyset / Cursor Pagination - Dành cho Infinite Scroll & Bảng siêu lớn)
- Tham số truy vấn: `?limit=25&cursor=eyJpZCI6MTAyOSwidGltZSI6MTcyOTY2OTYwMH0=`
- Kết quả phản hồi:
  ```json
  {
    "items": [ ... ],
    "nextCursor": "eyJpZCI6MTA1NCwidGltZSI6MTcyOTY2OTkwMH0=",
    "hasMore": true
  }
  ```

