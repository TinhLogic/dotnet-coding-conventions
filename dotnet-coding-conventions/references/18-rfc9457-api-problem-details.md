# Phần XIX. Chuẩn Lỗi API RFC 9457 & Problem Details Trong .NET

Tài liệu chi tiết về quy chuẩn định dạng lỗi HTTP API theo chuẩn quốc tế **RFC 9457** (*Problem Details for HTTP APIs* - tiêu chuẩn chính thức của IETF thay thế và kế thừa RFC 7807) dành riêng cho các ứng dụng **ASP.NET Core (.NET 8/9+)** (Controller, Minimal API, FluentValidation và Swagger/OpenAPI).

---

## 46. Toàn bộ API Controller / Minimal API phải trả về lỗi theo chuẩn RFC 9457

### Vì sao bắt buộc?
1. **Nhất quán 100% trên toàn bộ hệ thống**: Khách hàng, đội ngũ Frontend (React/Vue/Blazor), Mobile và các dịch vụ tích hợp bên ngoài chỉ cần viết 1 bộ giải mã lỗi (Error Interceptor) duy nhất.
2. **Không làm rò rỉ thông tin nhạy cảm (OWASP Top 10)**: Loại bỏ triệt để nguy cơ lộ mã nguồn, chuỗi kết nối SQL, tên server hoặc StackTrace ra ngoài môi trường Production.
3. **Đối soát lỗi tức thì qua `traceId`**: Khi người dùng gặp sự cố, mã `traceId` trong body JSON giúp đội ngũ kỹ thuật tìm thấy chính xác dòng log trong Application Insights, Serilog, Seq hoặc Datadog chỉ trong vài giây.

---

### 46.1. Header `Content-Type` Bắt Buộc

Mọi phản hồi lỗi (HTTP Status $4xx$ hoặc $5xx$) từ ASP.NET Core **bắt buộc** phải có header:
```http
Content-Type: application/problem+json
```

---

### 46.2. Cấu Trúc 5 Trường Tiêu Chuẩn Của `ProblemDetails`

Lớp `Microsoft.AspNetCore.Mvc.ProblemDetails` của .NET cung cấp đầy đủ 5 trường tiêu chuẩn của RFC 9457:

| Thuộc Tính .NET | Trường JSON | Bắt Buộc | Quy Chuẩn Thiết Lập |
| :--- | :--- | :--- | :--- |
| `Type` | `type` | Bắt buộc | URI trỏ tới trang tài liệu giải thích loại lỗi (hoặc `"about:blank"` nếu chưa có URI riêng). |
| `Title` | `title` | Bắt buộc | Tóm tắt ngắn gọn loại lỗi, viết cho con người đọc (cố định cho mọi lỗi cùng `type`). |
| `Status` | `status` | Bắt buộc | Mã HTTP status code số nguyên (`400`, `401`, `403`, `404`, `409`, `422`, `500`), **phải khớp 100% với HTTP Status của gói tin**. |
| `Detail` | `detail` | Khuyến nghị | Mô tả chi tiết nguyên nhân cụ thể phát sinh lỗi ở lần gọi hiện tại. |
| `Instance` | `instance` | Tùy chọn | URI định danh ngữ cảnh hoặc tài nguyên xảy ra lỗi (`httpContext.Request.Path`). |

---

### 46.3. Các Trường Mở Rộng Tiêu Chuẩn Enterprise Trong .NET (`Extensions`)

ASP.NET Core cho phép bổ sung các trường tùy chỉnh vào `problemDetails.Extensions`:

- **`traceId`**: Định danh truy vết phân tán (`httpContext.TraceIdentifier` hoặc `Activity.Current?.Id`).
- **`timestamp`**: Thời điểm xảy ra lỗi theo chuẩn ISO 8601 UTC (`DateTime.UtcNow`).
- **`errorCode`**: Mã lỗi nghiệp vụ dạng chuỗi (`OrderErrorCodes.InsufficientBalance`) để Frontend switch-case đa ngôn ngữ.
- **`invalidParams`** (hoặc `errors`): Mảng chi tiết các trường bị lỗi xác thực dữ liệu khi dùng FluentValidation.

---

### 46.4. Triển Khai Global Exception Handler Bằng `IExceptionHandler` (.NET 8+)

Tuyệt đối không dùng middleware `try-catch` tự chế cũ kỹ. Sử dụng `IExceptionHandler` chính thức của .NET 8/9:

```csharp
// Infrastructure/Exceptions/GlobalExceptionHandler.cs
using System.Diagnostics;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

namespace Enterprise.Infrastructure.Exceptions;

/// <summary>
/// Xử lý exception toàn cục và chuyển đổi sang chuẩn RFC 9457 Problem Details.
/// Global exception handler converting unhandled exceptions to RFC 9457 Problem Details.
/// </summary>
internal sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        // 1. Phân loại mã trạng thái HTTP dựa trên kiểu Exception
        int statusCode = exception switch
        {
            DomainException domainEx => domainEx.StatusCode,
            FluentValidation.ValidationException => StatusCodes.Status422UnprocessableEntity,
            KeyNotFoundException => StatusCodes.Status404NotFound,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            InvalidOperationException => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status500InternalServerError
        };

        string title = exception switch
        {
            DomainException => "Lỗi vi phạm quy tắc nghiệp vụ / Business rule violation",
            FluentValidation.ValidationException => "Dữ liệu đầu vào không hợp lệ / Validation failed",
            KeyNotFoundException => "Không tìm thấy tài nguyên / Resource not found",
            UnauthorizedAccessException => "Chưa xác thực danh tính / Unauthorized",
            InvalidOperationException => "Xung đột trạng thái / Conflict",
            _ => "Lỗi xử lý nội bộ máy chủ / Internal server error"
        };

        string detail = statusCode == StatusCodes.Status500InternalServerError
            ? "Đã xảy ra lỗi không mong muốn trên hệ thống. Vui lòng liên hệ quản trị viên kèm mã traceId."
            : exception.Message;

        logger.LogError(exception, "Xảy ra ngoại lệ {ExceptionType} tại {Path} với TraceId {TraceId}",
            exception.GetType().Name, httpContext.Request.Path, httpContext.TraceIdentifier);

        httpContext.Response.StatusCode = statusCode;

        // 2. Đóng gói ProblemDetails theo chuẩn RFC 9457
        var problemDetails = new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = detail,
            Instance = httpContext.Request.Path,
            Type = $"https://api.mycompany.com/errors/{statusCode}"
        };

        // Bổ sung các trường mở rộng
        problemDetails.Extensions["traceId"] = Activity.Current?.Id ?? httpContext.TraceIdentifier;
        problemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        if (exception is DomainException domainException && !string.IsNullOrWhiteSpace(domainException.ErrorCode))
        {
            problemDetails.Extensions["errorCode"] = domainException.ErrorCode;
        }

        // Nếu là lỗi FluentValidation, đóng gói danh sách chi tiết từng trường
        if (exception is FluentValidation.ValidationException valEx)
        {
            problemDetails.Extensions["invalidParams"] = valEx.Errors.Select(e => new
            {
                name = e.PropertyName,
                reason = e.ErrorMessage,
                code = e.ErrorCode
            }).ToList();
        }

        // 3. Ghi trực tiếp ra response qua IProblemDetailsService
        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = problemDetails,
            Exception = exception
        });
    }
}
```

---

### 46.5. Đăng Ký Trong `Program.cs`

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Bật dịch vụ ProblemDetails và ExceptionHandler của .NET 8+
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        // Tự động bổ sung traceId và timestamp cho mọi ProblemDetails được sinh ra
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

var app = builder.Build();

// Kích hoạt middleware ExceptionHandler đầu pipeline
app.UseExceptionHandler();
app.UseStatusCodePages(); // Bọc cả các lỗi 404, 401 tự sinh bởi ASP.NET Core thành RFC 9457
```

---

### 46.6. Quy Chuẩn Action Trong Controller & Swagger Documentation

Mọi Controller trả về lỗi trực tiếp phải dùng hàm `Problem(...)` hoặc để Exception tự ném ra cho Handler xử lý. Bắt buộc khai báo attribute `[ProducesResponseType]` cho Swagger UI hiển thị đúng schema `application/problem+json`:

```csharp
[ApiController]
[Route("api/v1/payment-transfers")]
[Produces("application/json")]
public sealed class PaymentTransfersController(IPaymentTransferService transferService) : ControllerBase
{
    /// <summary>Thực hiện lệnh chuyển tiền nội bộ.</summary>
    [HttpPost]
    [SwaggerOperation(Summary = "Tạo lệnh chuyển tiền", Description = "Thực hiện trừ tiền tài khoản nguồn và ghi có tài khoản đích.")]
    [ProducesResponseType(typeof(PaymentTransferOutputDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest, "application/problem+json")]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status422UnprocessableEntity, "application/problem+json")]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status500InternalServerError, "application/problem+json")]
    public async Task<IActionResult> CreateTransferAsync(
        [FromBody] PaymentTransferCreationInputDto request,
        [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey,
        CancellationToken cancellationToken = default)
    {
        if (string.IsNullOrWhiteSpace(idempotencyKey))
        {
            return Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: "Thiếu Header bắt buộc",
                detail: "Header 'Idempotency-Key' là bắt buộc để ngăn chặn trùng lặp giao dịch.",
                type: "https://api.mycompany.com/errors/missing-header"
            );
        }

        PaymentTransferOutputDto result = await transferService.TransferAsync(request, idempotencyKey, cancellationToken);

        return CreatedAtAction(nameof(GetTransferByIdAsync), new { transferId = result.TransferId }, result);
    }
}
```

---

### 46.7. So Sánh Phản Diện (Bad) vs Chính Diện (Good) Trong C#

❌ **SAI** — Trả về chuỗi thô, trả về anonymous object tự chế, hoặc lộ stacktrace:
```csharp
// SAI 1: Trả về chuỗi thô (Content-Type: text/plain)
return BadRequest("Số dư không đủ");

// SAI 2: Trả về Anonymous object tự chế
return NotFound(new { success = false, message = "Không tìm thấy đơn hàng" });

// SAI 3: Trả về 200 OK nhưng body chứa thông tin lỗi (Anti-pattern nguy hiểm)
return Ok(new { isError = true, errorMessage = "Tài khoản bị khoá" });

// SAI 4: Bắt exception rồi trả trực tiếp ex.Message hoặc ex.ToString()
catch (Exception ex)
{
    return StatusCode(500, ex.ToString()); // ❌ Lộ chuỗi kết nối SQL / cấu trúc server
}
```

✅ **ĐÚNG** — Trả về `ProblemDetails` chuẩn RFC 9457:
```csharp
// ĐÚNG 1: Trả về Problem trực tiếp
return Problem(
    statusCode: StatusCodes.Status422UnprocessableEntity,
    title: "Số dư tài khoản không đủ",
    detail: $"Số dư hiện tại là {account.Balance:N0} ₫, không đủ cho giao dịch {request.Amount:N0} ₫.",
    type: "https://api.mycompany.com/errors/insufficient-funds"
);

// ĐÚNG 2: Ném Exception nghiệp vụ để GlobalExceptionHandler tự đóng gói ProblemDetails
if (account.IsLocked)
{
    throw new DomainException(OrderErrorCodes.AccountLocked, "Tài khoản người dùng hiện đang bị tạm khóa.");
}
```

