# Phần XI. Xử Lý Lỗi & Nullable Reference Types (Error Handling & Null Safety)

Tài liệu chi tiết về chiến lược exception toàn hệ thống (phân loại lỗi, nơi bắt lỗi, `ProblemDetails`, global handler) và bật Nullable Reference Types để loại bỏ `NullReferenceException` từ lúc compile.

---

## 26. Xử lý lỗi theo chiến lược thống nhất: phân loại lỗi, bắt ở biên, trả `ProblemDetails`

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Caller nhận lỗi có cấu trúc, có mã để xử lý tự động. Stacktrace được giữ nguyên nên debug nhanh. Không lộ thông tin nội bộ ra client. |
| **Rủi ro nếu vi phạm** | `catch` nuốt lỗi làm dữ liệu sai mà không ai biết. `throw ex;` làm mất stacktrace gốc. Trả message SQL hoặc stacktrace ra client là lỗ hổng bảo mật (OWASP A05). Cùng một lỗi bị log 3–4 lần ở mỗi tầng làm nhiễu cảnh báo. |

### 26.1. Phân loại lỗi — mỗi loại một cách xử lý

| Loại | Ví dụ | Cách xử lý | HTTP |
| --- | --- | --- | --- |
| **Lỗi input** | Sai format, thiếu field | FluentValidation chặn ở biên (Rule 17) | 400 / 422 |
| **Lỗi nghiệp vụ** (dự kiến được) | Không đủ số dư, đơn đã huỷ, xung đột concurrency | Exception nghiệp vụ có **mã lỗi** (`UserFriendlyException`/`BusinessException` của ABP, hoặc `DomainException` tự định nghĩa) | 400 / 404 / 409 |
| **Lỗi hạ tầng** (tạm thời) | Timeout, mất kết nối DB/Redis, đối tác 503 | Resilience policy retry (Rule 33); hết retry thì để lan lên global handler | 502 / 503 / 504 |
| **Lỗi lập trình** (bug) | `NullReferenceException`, `InvalidOperationException`, `ArgumentException` | **Không bắt**. Để crash request, global handler log `Error` và trả 500 chung chung | 500 |

### 26.2. Quy tắc

1. **Chỉ bắt exception mà mình xử lý được** (retry, chuyển sang lỗi nghiệp vụ, fallback có chủ đích). Bắt chỉ để log rồi ném lại thì **không bắt**, để global handler lo.
2. **Cấm `catch (Exception)`** trong business code. Chỉ được dùng ở **biên hệ thống**: global handler, vòng lặp consumer / `BackgroundService`, decorator cache fallback (Rule 19 §3.2). Khi dùng phải có comment nêu lý do.
3. **Ném lại bằng `throw;`**, không dùng `throw ex;`. Khi bọc sang exception khác, **luôn truyền `innerException`**.
4. **Log một lần, ở nơi xử lý cuối cùng**. Không log ở repository, log ở service rồi lại log ở controller cho cùng một lỗi.
5. **Không dùng exception để điều khiển luồng thông thường.** Với trường hợp "không tìm thấy" là bình thường, trả `null` hoặc dùng `TryXxx`.
6. **Mã lỗi khai báo tập trung** (`{Service}ErrorCodes`), không hardcode chuỗi message tại chỗ ném.
7. **Guard argument** dùng throw helper của .NET: `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegativeOrZero` (.NET 8).

❌ **SAI**:

```csharp
public async Task<OrderOutputDto> GetOrderAsync(string orderCode, CancellationToken cancellationToken = default)
{
    try
    {
        Order order = await _orderRepository.GetByCodeAsync(orderCode, cancellationToken);
        return _mapper.Map<OrderOutputDto>(order);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex.Message);       // ❌ Mất stacktrace, log không có ngữ cảnh
        throw new Exception(ex.Message);    // ❌ Mất innerException, mất kiểu lỗi, client nhận message nội bộ
    }
}
```

✅ **ĐÚNG**:

```csharp
public async Task<OrderOutputDto> GetOrderAsync(string orderCode, CancellationToken cancellationToken = default)
{
    Order? order = await _orderRepository.FindByCodeAsync(orderCode, cancellationToken);

    if (order is null)
    {
        // "Không tìm thấy" là lỗi nghiệp vụ dự kiến được, trả mã lỗi rõ ràng thay vì để NullReferenceException
        // "Not found" is an expected business error, so return an explicit code instead of a NullReferenceException
        throw new UserFriendlyException(OrderErrorCodes.OrderNotFound).WithData("OrderCode", orderCode);
    }

    return _mapper.Map<OrderOutputDto>(order);
}
```

✅ **ĐÚNG** — Bọc exception hạ tầng sang lỗi nghiệp vụ, giữ `innerException`:

```csharp
catch (HttpRequestException ex)
{
    throw new PartnerIntegrationException(PartnerErrorCodes.EnquiryFailed, ex);
}
```

### 26.3. Global exception handler & `ProblemDetails` (RFC 9457)

**ASP.NET Core thuần (.NET 8+)**: dùng `IExceptionHandler`, response theo chuẩn `ProblemDetails`.

```csharp
internal sealed class BusinessExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<BusinessExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        if (exception is not DomainException domainException)
        {
            // Trả false để handler kế tiếp (mặc định: 500 chung chung) xử lý lỗi lập trình/hạ tầng
            // Return false so the next handler (default: generic 500) deals with programming/infrastructure errors
            return false;
        }

        logger.LogWarning(exception, "Business rule violated: {ErrorCode}", domainException.ErrorCode);

        httpContext.Response.StatusCode = domainException.StatusCode;

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails =
            {
                Title = domainException.Message,
                Status = domainException.StatusCode,
                Extensions = { ["code"] = domainException.ErrorCode }
            }
        });
    }
}
```

```csharp
// Program.cs
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<BusinessExceptionHandler>();

app.UseExceptionHandler();
```

**ABP Framework**: ABP đã có `AbpExceptionFilter` và `IExceptionToErrorInfoConverter`, tự bọc lỗi thành `{ error: { code, message, details } }`. **Không** thêm global handler thứ hai. Tuỳ biến bằng `AbpExceptionHandlingOptions` (`SendExceptionsDetailsToClients = false` ở production) và map mã lỗi → HTTP status qua `AbpExceptionHttpStatusCodeOptions.Map(...)`.

**Quy tắc response lỗi:**
- Production **không** trả stacktrace, message SQL, tên server hay connection string.
- Response lỗi luôn kèm `traceId` để đối chiếu log (ASP.NET Core `ProblemDetails` tự thêm; ABP dùng correlation id).
- Lỗi 5xx chỉ trả message chung ("Hệ thống đang bận"), chi tiết chỉ nằm trong log.

---

## 27. Bật Nullable Reference Types và coi warning nullable là lỗi

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | `NullReferenceException` là lỗi runtime phổ biến nhất trong .NET. NRT chuyển phần lớn số lỗi này sang compile time. Chữ ký method nói rõ "có thể null hay không", nên người đọc và AI agent không phải đoán. |
| **Rủi ro nếu vi phạm** | Bật NRT nhưng rải `!` khắp nơi thì còn tệ hơn không bật: compiler im lặng mà lỗi vẫn còn. Để warning nullable trôi khiến hàng trăm warning tích tụ, warning thật bị chìm. |
| **Rủi ro khi áp dụng** | Bật cho project cũ sinh hàng loạt warning. Áp dụng **dần theo file** (`#nullable enable`) cho code cũ, bật toàn project cho code mới. |

### 27.1. Cấu hình bắt buộc

Project mới đặt trong `Directory.Build.props` (Rule 36):

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <WarningsAsErrors>$(WarningsAsErrors);nullable</WarningsAsErrors>
</PropertyGroup>
```

Project cũ: bật `#nullable enable` cho **file mới hoặc file đang sửa**. Không bật toàn project rồi suppress hàng loạt.

### 27.2. Quy tắc

1. **Kiểu phản ánh đúng sự thật**: có thể null thì khai báo `T?`, không bao giờ null thì `T`. Method "tìm" trả `T?` (`FindByCodeAsync`), method "lấy bắt buộc" trả `T` và ném lỗi nghiệp vụ khi không có (`GetByCodeAsync`).
2. **Cấm dùng `!` (null-forgiving) để tắt warning.** Chỉ được dùng khi compiler không suy luận được nhưng logic **chứng minh được** không null, và phải có comment giải thích vì sao. Trường hợp còn lại: kiểm tra null hoặc ném lỗi.
3. **Kết quả deserialize luôn có thể null**: `JsonSerializer.Deserialize<T>` trả `T?`, phải kiểm tra.
4. **DTO**: property bắt buộc dùng `required` (C# 11, Rule 22); property tuỳ chọn dùng `T?` kèm `[JsonIgnore(Condition = WhenWritingNull)]` (Rule 12). **Không** dùng `= null!;` để im compiler.
5. **Entity EF Core**: navigation bắt buộc có thể dùng `= null!;`. Đây là ngoại lệ được chấp nhận vì EF Core gán giá trị khi materialize. Collection navigation khởi tạo `= [];`.
6. Dùng attribute phân tích tĩnh khi viết helper: `[NotNullWhen(true)]`, `[MaybeNullWhen(false)]`, `[MemberNotNull(nameof(...))]`.
7. Dùng pattern `is null` / `is not null`, không dùng `== null` (tránh operator overload bị ghi đè).

❌ **SAI**:

```csharp
public string CustomerName { get; set; } = null!;   // ❌ DTO dùng null! để im compiler

PartnerResponseDto result = JsonSerializer.Deserialize<PartnerResponseDto>(json, JsonOptions)!; // ❌ Che giấu payload rỗng
return result.Data.TransactionId;   // NullReferenceException khi đối tác trả body rỗng
```

✅ **ĐÚNG**:

```csharp
public required string CustomerName { get; init; }

PartnerResponseDto? result = JsonSerializer.Deserialize<PartnerResponseDto>(json, JsonOptions);

if (result?.Data is null)
{
    throw new PartnerIntegrationException(PartnerErrorCodes.EmptyResponse);
}

return result.Data.TransactionId;
```

```csharp
public static bool TryParseOrderCode(string? raw, [NotNullWhen(true)] out OrderCode? orderCode)
{
    // ...
}
```

### 27.3. Anti-patterns (KHÔNG làm)

- `<Nullable>disable</Nullable>` trong project mới.
- `= null!;` trên DTO hoặc Options.
- `!` sau `Deserialize`, `FirstOrDefault`, `GetService`, `HttpContext.User.FindFirst(...)`.
- `#pragma warning disable CS8602` / `CS8618` không kèm justification (Rule 42).
- Thêm `?` vào mọi thứ "cho chắc", làm mất thông tin hợp đồng.
