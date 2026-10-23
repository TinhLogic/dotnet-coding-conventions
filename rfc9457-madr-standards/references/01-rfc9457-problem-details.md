# Quy Chuẩn Định Dạng Lỗi HTTP API (RFC 9457 Problem Details)

Mọi HTTP RESTful API trong hệ thống doanh nghiệp khi trả về phản hồi lỗi (HTTP Status Code $4xx$ hoặc $5xx$) **bắt buộc phải tuân theo chuẩn RFC 9457** (*Problem Details for HTTP APIs* - tiêu chuẩn chính thức của IETF thay thế và kế thừa RFC 7807).

Tuyệt đối không tự ý bịa ra cấu trúc lỗi tùy tiện (như `{ "error": "Something went wrong", "code": 123 }`) gây khó khăn cho việc tích hợp giữa các dịch vụ (Microservices), Frontend và các đối tác bên thứ ba.

---

## 1. Tiêu Chuẩn Media Type & 5 Trường Cốt Lõi

Khi một API trả về lỗi, header `Content-Type` bắt buộc phải là:
```http
Content-Type: application/problem+json
```
*(hoặc `application/problem+xml` nếu dịch vụ hỗ trợ XML).*

### 5 Trường Tiêu Chuẩn Của RFC 9457 (Kế Thừa RFC 7807):

| Trường (Field) | Kiểu Dữ Liệu | Bắt Buộc | Mô Tả Quy Chuẩn | Ví Dụ |
| :--- | :--- | :--- | :--- | :--- |
| **`type`** | `URI reference` | Bắt buộc | Định danh duy nhất phân loại lỗi. Trỏ tới trang tài liệu giải thích loại lỗi này. Mặc định dùng `"about:blank"` nếu không có URI riêng. | `"https://api.example.com/errors/insufficient-funds"` hoặc `"about:blank"` |
| **`title`** | `string` | Bắt buộc | Tóm tắt ngắn gọn loại lỗi, viết cho con người đọc. **Tuyệt đối không đổi theo từng lần xảy ra lỗi** (dùng chung cho mọi lỗi cùng `type`). | `"Tài khoản không đủ số dư"` hoặc `"Invalid Parameter"` |
| **`status`** | `number` | Bắt buộc | Mã trạng thái HTTP status code phát sinh ở máy chủ, **phải trùng khớp 100% với HTTP Status của gói tin**. | `400`, `404`, `409`, `422`, `500` |
| **`detail`** | `string` | Khuyến nghị | Mô tả chi tiết cụ thể giải thích riêng cho lần phát sinh lỗi hiện tại. Viết rõ nguyên nhân để lập trình viên hoặc người dùng khắc phục. | `"Số dư tài khoản là 150.000 ₫, không đủ thực hiện giao dịch 500.000 ₫."` |
| **`instance`** | `URI reference` | Tùy chọn | URI định danh duy nhất tài nguyên hoặc ngữ cảnh xảy ra lỗi này (giúp tra cứu log nhanh). | `"/transfers/txn-9842145"` hoặc `"/orders/10294"` |

---

## 2. Trường Mở Rộng Tiêu Chuẩn Doanh Nghiệp (Extension Members)

RFC 9457 cho phép mở rộng các trường tùy chỉnh (Extension members). Trong môi trường Enterprise, các trường sau được quy chuẩn hóa bắt buộc:

- **`traceId`**: Mã định danh phân tán (W3C Trace Context / OpenTelemetry / Correlation ID) để truy vết log từ Frontend sang Backend và Database.
- **`timestamp`**: Thời điểm xảy ra lỗi theo chuẩn ISO 8601 UTC (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **`errorCode`** (hoặc `code`): Mã nghiệp vụ nội bộ (Machine-readable code) để frontend dựa vào đó switch-case dịch đa ngôn ngữ hoặc hiển thị màn hình tương ứng.
- **`invalidParams`** (hoặc `errors`): Danh sách chi tiết các trường bị lỗi xác thực dữ liệu đầu vào.

### Cấu Trúc JSON Hoàn Chỉnh Mẫu (Validation Error 422 / 400):

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Dữ liệu đầu vào không hợp lệ",
  "status": 422,
  "detail": "Có 2 trường dữ liệu không vượt qua kiểm tra tính hợp lệ.",
  "instance": "/api/v1/customers",
  "errorCode": "VALIDATION_FAILED",
  "timestamp": "2026-10-23T07:45:00.000Z",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "invalidParams": [
    {
      "name": "email",
      "reason": "Email không đúng định dạng chuẩn RFC 5322.",
      "code": "EMAIL_INVALID"
    },
    {
      "name": "age",
      "reason": "Độ tuổi khách hàng phải từ 18 trở lên.",
      "code": "MIN_VALUE_EXCEEDED"
    }
  ]
}
```

---

## 3. So Sánh Phản Diện (Bad) vs Chính Diện (Good)

### ❌ SAI — Trả về cấu trúc lỗi tự chế hoặc lỗi bảo mật:

```json
// SAI 1: Tự chế cấu trúc lỗi không đồng nhất, thiếu status code
{
  "success": false,
  "error": "Error: User not found",
  "code": 404
}

// SAI 2: Trả về HTTP 200 OK nhưng body lại chứa lỗi (Anti-pattern nguy hiểm)
HTTP/1.1 200 OK
{
  "isError": true,
  "message": "Sai mật khẩu"
}

// SAI 3: Lộ StackTrace kỹ thuật ra ngoài môi trường Production
HTTP/1.1 500 Internal Server Error
{
  "exception": "System.NullReferenceException: Object reference not set to an instance of an object at Services.OrderService.CreateOrderAsync() line 142..."
}
```

### ✅ ĐÚNG — Tuân thủ chuẩn RFC 9457:

```json
// ĐÚNG: Lỗi 404 Not Found
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/resource-not-found",
  "title": "Không tìm thấy tài nguyên",
  "status": 404,
  "detail": "Không tìm thấy tài khoản người dùng với ID 'usr-98124'.",
  "instance": "/api/v1/users/usr-98124",
  "traceId": "00-98f92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}

// ĐÚNG: Lỗi 409 Conflict (Trùng lặp dữ liệu)
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/duplicate-entity",
  "title": "Xung đột dữ liệu",
  "status": 409,
  "detail": "Địa chỉ email 'contact@example.com' đã được đăng ký bởi một tài khoản khác.",
  "instance": "/api/v1/registrations",
  "errorCode": "DUPLICATE_EMAIL",
  "traceId": "00-11a92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

---

## 4. Triển Khai Trên Các Nền Tảng Kỹ Thuật

### 4.1. ASP.NET Core (.NET 7/8/9)
ASP.NET Core tích hợp sẵn chuẩn RFC 9457 qua `Results.Problem()` và Middleware `AddProblemDetails()`:

```csharp
// Program.cs
builder.Services.AddProblemDetails(options => {
    options.CustomizeProblemDetails = context => {
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});

// Trong Controller hoặc Minimal API
app.MapPost("/api/v1/orders", (CreateOrderRequest request) => {
    if (request.Amount <= 0) {
        return Results.Problem(
            statusCode: StatusCodes.Status400BadRequest,
            title: "Giá trị đơn hàng không hợp lệ",
            detail: "Số tiền thanh toán phải lớn hơn 0.",
            type: "https://api.example.com/errors/invalid-amount"
        );
    }
    return Results.Ok();
});
```

### 4.2. Node.js / Express / Fastify / NestJS
Sử dụng class chuẩn hoá `HttpProblem` hoặc filter toàn cục:

```typescript
// NestJS Custom Exception Filter
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class ProblemDetailsFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse() as any;

    response
      .status(status)
      .header('Content-Type', 'application/problem+json')
      .json({
        type: exceptionResponse.type || 'about:blank',
        title: exceptionResponse.error || exception.message,
        status: status,
        detail: exceptionResponse.message || exception.message,
        instance: request.url,
        traceId: request.headers['x-trace-id'] || request.id,
        timestamp: new Date().toISOString()
      });
  }
}
```

### 4.3. Python / FastAPI
FastAPI cho phép định nghĩa Exception Handler kế thừa cấu trúc Problem Details:

```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse

app = FastAPI()

class ProblemDetailsException(Exception):
    def __init__(self, status_code: int, title: str, detail: str, type_uri: str = "about:blank"):
        self.status_code = status_code
        self.title = title
        self.detail = detail
        self.type_uri = type_uri

@app.exception_handler(ProblemDetailsException)
async def problem_details_handler(request: Request, exc: ProblemDetailsException):
    return JSONResponse(
        status_code=exc.status_code,
        headers={"Content-Type": "application/problem+json"},
        content={
            "type": exc.type_uri,
            "title": exc.title,
            "status": exc.status_code,
            "detail": exc.detail,
            "instance": str(request.url.path),
            "traceId": request.headers.get("x-trace-id", "")
        }
    )
```

---

## 5. Bảng Ánh Xạ Mã Lỗi HTTP Phổ Biến

| HTTP Status | Title Chuẩn | Khi Nào Sử Dụng? |
| :--- | :--- | :--- |
| **`400 Bad Request`** | `"Yêu cầu không hợp lệ"` | Cú pháp JSON sai, thiếu Header bắt buộc. |
| **`401 Unauthorized`** | `"Chưa xác thực danh tính"` | Thiếu Bearer Token, Token hết hạn hoặc chữ ký không hợp lệ. |
| **`403 Forbidden`** | `"Không có quyền truy cập"` | Đã đăng nhập nhưng Role/Permission không được phép thao tác tài nguyên này. |
| **`404 Not Found`** | `"Không tìm thấy tài nguyên"` | ID thực thể không tồn tại trong cơ sở dữ liệu. |
| **`409 Conflict`** | `"Xung đột trạng thái tài nguyên"` | Trùng lặp Email/SKU, vi phạm ràng buộc Optimistic Lock / Versioning. |
| **`422 Unprocessable Entity`** | `"Dữ liệu không vượt qua xác thực"` | JSON đúng cú pháp nhưng vi phạm luật nghiệp vụ (Validation rules). |
| **`429 Too Many Requests`** | `"Vượt quá giới hạn tần suất gọi"` | Bị chặn bởi Rate Limiter (kèm header `Retry-After`). |
| **`500 Internal Server Error`** | `"Lỗi xử lý nội bộ máy chủ"` | Exception không mong muốn (che giấu stacktrace, chỉ trả về `traceId`). |
| **`503 Service Unavailable`** | `"Dịch vụ tạm thời ngưng phục vụ"` | Database quá tải hoặc dịch vụ phụ thuộc đang bảo trì. |

