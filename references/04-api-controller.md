# Phần V. API Controller & Swagger Documentation

Tài liệu chi tiết về thiết kế API RESTful và tài liệu hoá Swagger (`[SwaggerOperation]`).

---

## 15. Controller đặt tên API theo chuẩn RESTful

Route dựa trên **danh từ số nhiều** cho resource, hành vi thể hiện qua HTTP verb. Không đưa động từ vào URL.

| ❌ Sai                          | ✅ Đúng                                   |
| ------------------------------ | ---------------------------------------- |
| `POST /MoMoAIO/create-payment` | `POST /momo-aio/payments`                |
| `GET /MoMo/check-transaction`  | `GET /momo/transactions/{transactionId}` |
| `POST /MoMo/create-qr`         | `POST /momo/qr-codes`                    |
| `POST /order/cancelOrder`      | `DELETE /orders/{orderId}`               |
| `GET /getUserById`             | `GET /users/{userId}`                    |

Quy tắc chi tiết:
- Resource là **danh từ số nhiều**, `kebab-case`: `payments`, `qr-codes`, `merge-requests`.
- HTTP verb thể hiện hành động: `GET` (đọc), `POST` (tạo), `PUT` (thay toàn bộ), `PATCH` (sửa một phần), `DELETE` (xoá).
- Identifier nằm trong path: `/payments/{paymentId}`.
- Filter/paging nằm trong query string: `?status=paid&page=1&pageSize=20`.
- `GET` **không** được nhận body; dùng query string.
- Sub-resource lồng theo quan hệ: `/payments/{paymentId}/refunds`.
- Không dùng `camelCase` hay động từ trong segment URL.

✅ **Ví dụ controller đúng**:

```csharp
[Route("api/momo-aio")]
public class MoMoAIOController : BaseApiController<IMoMoAIOService>
{
    /// <summary>Đẩy lệnh thanh toán xuống thiết bị All-In-One.</summary>
    [HttpPost("payments")]
    public Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
        [FromBody] MoMoAIOPaymentCreationInputDto request,
        CancellationToken cancellationToken = default)
        => _mainService.CreatePaymentAsync(request, cancellationToken);

    /// <summary>Lấy trạng thái một giao dịch.</summary>
    [HttpGet("transactions/{transactionId}")]
    public Task<MoMoAIOTransactionOutputDto> GetTransactionAsync(
        string transactionId,
        CancellationToken cancellationToken = default)
        => _mainService.GetTransactionAsync(transactionId, cancellationToken);
}
```

---

## 16. Controller action phải có `[SwaggerOperation]`

Mọi action method trong controller phải được decorate bằng `[SwaggerOperation]` (từ package `Swashbuckle.AspNetCore.Annotations`) để Swagger UI hiển thị mô tả rõ ràng cho từng endpoint.

❌ **SAI** — không có SwaggerOperation:

```csharp
[HttpPost("payments")]
public async Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
    [FromBody] MoMoAIOPaymentCreationInputDto request,
    CancellationToken cancellationToken = default)
    => await _mainService.CreatePaymentAsync(request, cancellationToken);
```

✅ **ĐÚNG**:

```csharp
using Swashbuckle.AspNetCore.Annotations;

[HttpPost("payments")]
[SwaggerOperation(
    Summary = "Tạo lệnh thanh toán trên thiết bị AIO",
    Description = "Đẩy lệnh thanh toán (thẻ/QR) xuống thiết bị EDC All-In-One.\n" +
                  "- Hỗ trợ instrumentType:\n" +
                  "  - QR: EDC hiển thị mã QR đa năng (AIO QR)\n" +
                  "  - CARD: EDC chờ đọc thẻ (chạm/cắm/quẹt)\n" +
                  "  - POS: EDC quét mã QR của khách hàng\n" +
                  "- storeId bắt buộc, phải được liên kết với thiết bị EDC\n" +
                  "- serialNumber tuỳ chọn, bỏ trống thì tự lấy serial mới nhất\n" +
                  "- autoCapture mặc định true, set false nếu muốn capture sau",
    OperationId = "MoMoAIO_CreatePayment",
    Tags = new[] { "MoMo AIO" })]
[SwaggerResponse(200, "Tạo lệnh thành công, thiết bị đã được kích hoạt", typeof(MoMoAIOPaymentCreationOutputDto))]
[SwaggerResponse(400, "Dữ liệu không hợp lệ (thiếu storeId, orderId, requestId hoặc amount ngoài hạn mức)")]
[SwaggerResponse(408, "Timeout khi gọi đối tác thanh toán")]
public async Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
    [FromBody] MoMoAIOPaymentCreationInputDto request,
    CancellationToken cancellationToken = default)
    => await _mainService.CreatePaymentAsync(request, cancellationToken);
```

Quy tắc:
- `Summary`: mô tả ngắn gọn (hiển thị trên Swagger UI danh sách API). Viết tiếng Việt hoặc tiếng Anh tuỳ team.
- `Description`: mô tả chi tiết hơn (hiển thị khi expand endpoint).
  - **Mỗi ý phải xuống dòng** (`\n`) và có **gạch đầu dòng** (`-`) cho dễ đọc.
  - Ý phụ dùng indent 2 space + gạch đầu dòng (`  -`).
  - Không viết 1 đoạn văn dài liền nhau.
- `OperationId`: format `{ControllerGroup}_{ActionName}`, dùng PascalCase, unique toàn bộ project. Ví dụ: `MoMoAIO_CreatePayment`, `MoMo_CheckTransaction`, `ZaloPay_CreateQrCode`.
- `Tags`: nhóm các endpoint liên quan trên Swagger UI. Dùng tên đối tác hoặc module.
- `[SwaggerResponse]`: khai báo các HTTP status code có thể trả về. Tối thiểu phải có:
  - `200` (hoặc `201` cho POST tạo resource) với type trả về
  - `400` cho lỗi validation
  - Các status đặc biệt nếu có (408 timeout, 404 not found...)

**Không để lộ thông tin nhạy cảm trong SwaggerOperation**:

Swagger UI là tài liệu public (hoặc semi-public cho internal team). Tuyệt đối **KHÔNG** đưa các thông tin sau vào `Summary`, `Description`, hoặc bất kỳ annotation nào trên controller:
- URL/endpoint API đối tác (ví dụ: `v2/gateway/api/create`, `https://payment.momo.vn/...`)
- Secret key, access key, mã partner
- Thuật toán ký (HmacSHA256) và chuỗi rawHash
- Thông tin signature, cách tính chữ ký
- Credentials, token, certificate

❌ **SAI** — lộ endpoint đối tác:

```csharp
[SwaggerOperation(
    Summary = "Tạo lệnh thanh toán trên thiết bị AIO",
    Description = "Gọi MoMo API v2/gateway/api/create với HmacSHA256 signature")]
```

✅ **ĐÚNG** — chỉ mô tả nghiệp vụ:

```csharp
[SwaggerOperation(
    Summary = "Tạo lệnh thanh toán trên thiết bị AIO",
    Description = "Đẩy lệnh thanh toán (thẻ/QR) xuống thiết bị EDC All-In-One. " +
                  "Hỗ trợ instrumentType: QR, CARD, POS.")]
```

Nếu cần link tài liệu kỹ thuật cho developer nội bộ, đặt ở `/// <summary>` XML doc trong source code — không đặt trong SwaggerOperation (vì Swagger có thể expose ra ngoài).

**Package cần cài**: `Swashbuckle.AspNetCore.Annotations`

Nếu project đã cài `Swashbuckle.AspNetCore` (thường có sẵn), chỉ cần bật trong `Startup`:

```csharp
services.AddSwaggerGen(c =>
{
    c.EnableAnnotations();
});
```

