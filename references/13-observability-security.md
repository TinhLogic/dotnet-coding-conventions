# Phần XIV. Logging, Observability & Bảo Mật (Logging, Observability & Security Coding)

Tài liệu chi tiết về structured logging, tracing/metrics, health check và các quy tắc secure coding bắt buộc cho code .NET.

---

## 31. Logging có cấu trúc, đúng level, có correlation; hệ thống phải quan sát được

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Tìm được đúng request lỗi trong vài giây bằng `traceId` / `OrderCode` trên Kibana. Alert dựa trên metric thay vì chờ khách báo. `[LoggerMessage]` không cấp phát bộ nhớ khi level bị tắt. |
| **Rủi ro nếu vi phạm** | String interpolation trong log làm mất field có cấu trúc (không filter được theo `OrderCode`) và tốn CPU/RAM kể cả khi log bị tắt. Log PII hoặc token vi phạm Nghị định 13/2023 về bảo vệ dữ liệu cá nhân. Log object lớn làm phình index ELK. |

### 31.1. Quy tắc logging

1. **Dùng message template**, không string interpolation hay nối chuỗi. Tên placeholder PascalCase, cố định, có nghĩa (`{OrderCode}`, `{ElapsedMs}`).
2. **Hot path** (log trong vòng lặp, endpoint tải cao) dùng **`[LoggerMessage]` source generator**.
3. **Exception truyền vào tham số đầu tiên** `LogError(ex, "...")`, không log `ex.Message` / `ex.ToString()` vào message.
4. **Level theo bảng**:

   | Level | Dùng khi |
   | --- | --- |
   | `Trace` / `Debug` | Chi tiết kỹ thuật cho dev. **Tắt ở production** |
   | `Information` | Sự kiện nghiệp vụ có ý nghĩa (đơn tạo, thanh toán thành công), bắt đầu/kết thúc job |
   | `Warning` | Bất thường nhưng hệ thống tự xử lý được (retry, fallback cache, xung đột concurrency, lỗi nghiệp vụ dự kiến) |
   | `Error` | Thao tác thất bại cần người xem xét |
   | `Critical` | Hệ thống/thành phần không hoạt động (mất DB, không khởi động được) |

5. **Log một lần ở nơi xử lý lỗi cuối cùng** (Rule 26.2).
6. **Ngữ cảnh chung** (`OrderCode`, `ShopCode`, `PartnerCode`) đặt bằng `logger.BeginScope(...)` thay vì lặp ở mọi dòng log.
7. **KHÔNG log**: mật khẩu, token / JWT / API key / secret, chữ ký, số thẻ (PAN) / CVV, CCCD, số điện thoại / email đầy đủ, dữ liệu sức khoẻ, request/response body nguyên khối của đối tác thanh toán. Cần thì **mask** (`090****123`).
8. **Không log trong vòng lặp nóng** từng phần tử. Log tổng kết (số lượng, thời gian).

❌ **SAI**:

```csharp
_logger.LogInformation($"Create payment for order {input.OrderId}, phone {input.PhoneNumber}"); // ❌ Interpolation + PII
_logger.LogError("Error: " + ex.Message);                                                        // ❌ Mất stacktrace
_logger.LogInformation("Partner response {Response}", JsonSerializer.Serialize(response));       // ❌ Log body nguyên khối
```

✅ **ĐÚNG**:

```csharp
internal static partial class PaymentLog
{
    [LoggerMessage(EventId = 2001, Level = LogLevel.Information, Message = "Payment created for order {OrderCode} via {PartnerCode}")]
    public static partial void PaymentCreated(ILogger logger, string orderCode, string partnerCode);

    [LoggerMessage(EventId = 2002, Level = LogLevel.Warning, Message = "Partner {PartnerCode} returned result code {ResultCode} for order {OrderCode}")]
    public static partial void PartnerRejected(ILogger logger, string partnerCode, string resultCode, string orderCode);
}

using (_logger.BeginScope(new Dictionary<string, object> { ["OrderCode"] = input.OrderCode }))
{
    PaymentLog.PaymentCreated(_logger, input.OrderCode, PartnerCodes.MoMo);
}
```

### 31.2. Tracing, metrics & health check

1. **Correlation**: trace id propagate qua header W3C `traceparent` xuyên BFF → Core → Kafka. Khuyến nghị dùng **Elastic APM** (hoặc OpenTelemetry exporter sang APM Server). Log phải chứa `trace.id` để nhảy Log ↔ APM.
2. **Span nghiệp vụ tự định nghĩa** dùng `ActivitySource` (một `static readonly` cho mỗi service), không tự sinh correlation id riêng song song với trace id.
3. **Metric nghiệp vụ** dùng `System.Diagnostics.Metrics` (`IMeterFactory`, `Counter<T>`, `Histogram<T>`): số giao dịch theo đối tác/kết quả, latency gọi đối tác. Tag có số lượng giá trị hữu hạn (mã đối tác, kết quả), **không** dùng `OrderCode` / `UserId` làm tag (bùng nổ cardinality).
4. **Health check** tách 2 endpoint:
   - `/health/live`: chỉ trả process còn sống, **không** kiểm tra dependency (nếu không, DB chậm khiến Kubernetes restart hàng loạt pod).
   - `/health/ready`: kiểm tra DB, Redis, Kafka; fail thì pod bị rút khỏi load balancer.
   - Endpoint health không trả chi tiết lỗi/connection string ra ngoài.

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = x => x.Tags.Contains("live") });
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = x => x.Tags.Contains("ready") });
```

### 31.3. Anti-patterns (KHÔNG làm)

- `$"..."` hoặc `+` trong message log.
- Log PII / secret / token / body thanh toán.
- `Console.WriteLine` / `Debug.WriteLine` trong code production.
- Log `Error` cho lỗi nghiệp vụ dự kiến (gây báo động giả).
- Liveness probe kiểm tra DB.
- Metric tag theo id có cardinality vô hạn.

---

## 32. Secure coding: secret, injection, crypto, authorization

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Giảm bề mặt tấn công theo OWASP Top 10. Đáp ứng PCI DSS (thanh toán), Nghị định 13/2023 (dữ liệu cá nhân), compliance chuỗi y tế (VAC/LAB/PHARMACY). |
| **Rủi ro nếu vi phạm** | Secret commit lên Git **phải coi là đã lộ** dù đã xoá commit: phải rotate. SQL injection, so sánh chữ ký không constant-time (timing attack) giả mạo callback thanh toán, tắt verify TLS mở đường cho MITM. Hậu quả là mất tiền, mất dữ liệu, bị phạt pháp lý. |

### 32.1. Quy tắc

1. **Secret** (connection string, API key, secret key đối tác, private key):
   - **Cấm** hardcode trong code, `appsettings.json` commit lên Git, comment hay test.
   - Lấy từ biến môi trường / Vault / Kubernetes Secret, đọc qua Options pattern (Rule 34). Dev local dùng `dotnet user-secrets`.
   - Mỗi môi trường một bộ secret; production secret không dùng ở dev/uat.
2. **SQL / query injection**: chỉ dùng LINQ, `FromSql` / `FromSqlInterpolated`, Dapper với tham số (`new { ShopCode = shopCode }`). **Cấm** nối chuỗi input vào SQL, kể cả tên cột sắp xếp: map tên cột qua allow-list.
3. **So sánh chữ ký / HMAC / token** dùng `CryptographicOperations.FixedTimeEquals`, không dùng `==` hay `string.Equals` (lộ thông tin qua thời gian phản hồi).
4. **Sinh token, OTP, mã ngẫu nhiên bảo mật** dùng `RandomNumberGenerator`, không dùng `Random` / `Guid.NewGuid()` làm secret.
5. **Crypto chuẩn**: hash mật khẩu bằng `PasswordHasher<T>` (ASP.NET Core Identity) hoặc PBKDF2 `Rfc2898DeriveBytes.Pbkdf2` với SHA-256 trở lên; mã hoá đối xứng dùng `AesGcm`. Cấm MD5/SHA1 cho mục đích bảo mật, cấm tự chế thuật toán.
6. **TLS**: **cấm** `ServerCertificateCustomValidationCallback = (_, _, _, _) => true` hay `DangerousAcceptAnyServerCertificateValidator` ở mọi môi trường ngoài test cục bộ. Đối tác dùng mTLS thì nạp certificate qua `HttpClientHandler.ClientCertificates` từ secret store.
7. **Authorization** dùng policy (`[Authorize(Policy = ...)]`, `IAuthorizationService` cho resource-based) hoặc ABP permission (`[Authorize(XxxPermissions.Create)]`). Không tự `if (user.Role == "Admin")` trong service. **Luôn kiểm tra quyền sở hữu tài nguyên** (user chỉ xem đơn của mình/shop mình): chống IDOR.
8. **Callback / webhook từ đối tác**: verify chữ ký **trước** mọi xử lý, kiểm tra timestamp/nonce chống replay, xử lý idempotent (Rule 39).
9. **Deserialization**: chỉ `System.Text.Json` (Rule 11). Cấm `BinaryFormatter` (đã bị gỡ ở .NET 9), cấm `TypeNameHandling` khác `None` nếu buộc dùng Newtonsoft.
10. **File / path**: tên file từ input phải qua `Path.GetFileName`, path ghép xong phải kiểm tra nằm trong thư mục gốc (`Path.GetFullPath(...).StartsWith(root, StringComparison.Ordinal)`), kiểm tra loại file bằng magic bytes và giới hạn kích thước.
11. **Không lộ chi tiết lỗi** ra client (Rule 26.3), không trả field nội bộ ra DTO (Rule 4).

❌ **SAI**:

```csharp
private const string SecretKey = "<partner-secret-key>";              // ❌ Hardcode secret

string sql = $"SELECT * FROM Orders WHERE ShopCode = '{shopCode}' ORDER BY {sortColumn}"; // ❌ SQL injection

if (computedSignature == request.Signature) { }                     // ❌ Timing attack

string otp = new Random().Next(100000, 999999).ToString();          // ❌ Random không an toàn cho mục đích bảo mật
```

✅ **ĐÚNG**:

```csharp
// Secret qua Options, giá trị thật nằm ở Vault / Kubernetes Secret
string secretKey = _options.Value.SecretKey;

// Tên cột sắp xếp đi qua allow-list vì không tham số hoá được tên cột
// Sort column goes through an allow-list because column names cannot be parameterized
string sortColumn = OrderSortColumns.Resolve(input.SortBy);

byte[] expected = HMACSHA256.HashData(Encoding.UTF8.GetBytes(secretKey), Encoding.UTF8.GetBytes(rawData));
byte[] actual = Convert.FromHexString(request.Signature);

if (!CryptographicOperations.FixedTimeEquals(expected, actual))
{
    throw new UserFriendlyException(PaymentErrorCodes.InvalidSignature);
}

string otp = RandomNumberGenerator.GetInt32(0, 1_000_000).ToString("D6", CultureInfo.InvariantCulture);
```

### 32.2. Anti-patterns (KHÔNG làm)

- Secret trong source, `appsettings*.json` commit, comment, log, unit test.
- Nối chuỗi SQL với input; `FromSqlRaw` với interpolation.
- `==` để so sánh chữ ký.
- `Random` cho token / OTP / mã giảm giá.
- MD5 / SHA1 cho mật khẩu hoặc chữ ký mới. Đối tác bắt buộc thuật toán cũ thì ghi ngoại lệ theo Rule 42.
- Tắt kiểm tra TLS certificate.
- Kiểm tra quyền bằng chuỗi role rải rác; không kiểm tra quyền sở hữu tài nguyên.
- Xử lý callback đối tác trước khi verify chữ ký.
