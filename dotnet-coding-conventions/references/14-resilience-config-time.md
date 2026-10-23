# Phần XV. Resilience, Cấu Hình & Thời Gian / Tiền Tệ / Culture

Tài liệu chi tiết về resilience khi gọi HTTP (retry, timeout, circuit breaker), Options pattern có validate lúc khởi động, và các quy tắc xử lý thời gian, tiền, culture, so sánh chuỗi.

---

## 33. Gọi dịch vụ ngoài phải có resilience policy; chỉ retry khi an toàn

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Lỗi mạng chập chờn được tự xử lý. Đối tác chết không kéo sập cả hệ thống, vì circuit breaker cắt sớm thay vì giữ thread chờ timeout. Timeout có giới hạn tổng nên request của user không treo. |
| **Rủi ro nếu vi phạm** | Không timeout thì thread và connection bị giữ, gây hiệu ứng domino. **Retry lệnh tạo giao dịch không idempotent sẽ trừ tiền khách hai lần.** Retry dồn dập không backoff làm đối tác đang quá tải chết hẳn (retry storm). |

### 33.1. Quy tắc

1. **Mọi Named/Typed HttpClient** (Rule 18) gắn resilience pipeline qua `Microsoft.Extensions.Http.Resilience` (Polly v8). **Không** viết vòng lặp retry thủ công bằng `for` + `Task.Delay`.
2. **Thông số lấy từ Options theo từng đối tác** (Rule 19 §1.4), không hardcode.
3. **Chỉ retry lỗi tạm thời**: lỗi mạng, timeout, HTTP 408 / 429 / 5xx. **Không** retry 4xx còn lại (lỗi do request sai, retry vẫn sai).
4. **Không retry method không idempotent** (`POST` tạo giao dịch, chuyển tiền, gửi SMS) **trừ khi** request mang idempotency key / `requestId` mà đối tác cam kết dedupe (Rule 39). Mặc định tắt retry cho `POST`/`PATCH`.
5. **Retry có exponential backoff + jitter**, số lần hữu hạn (thường 2–3).
6. **Timeout hai tầng**: timeout mỗi lần thử (attempt) và timeout tổng (total). `HttpClient.Timeout` phải **lớn hơn hoặc bằng** timeout tổng của pipeline, nếu không nó sẽ cắt ngang trước khi pipeline kịp retry.
7. **Circuit breaker** cho mọi đối tác bên ngoài. Khi mạch mở, trả lỗi nhanh có mã rõ ràng (`PartnerUnavailable`), không chờ.
8. **Timeout khi gọi đối tác thanh toán không có nghĩa là thất bại**: giao dịch giữ trạng thái `Pending` và đi qua luồng tra cứu / đối soát (Rule 25.5).
9. **ABP Framework**: `IApiService` / `RemoteServiceOptions` cũng phải có cấu hình timeout và retry tương đương; với dynamic C# proxy dùng `AbpHttpClientBuilderOptions.ProxyClientBuildActions` để gắn resilience handler.

✅ **ĐÚNG**:

```csharp
builder.Services
    .AddHttpClient(EnvironmentSetting.MoMoRemote, (sp, client) =>
    {
        MoMoConnect options = sp.GetRequiredService<IOptions<MoMoConnect>>().Value;
        client.BaseAddress = new Uri(options.Url);

        // HttpClient.Timeout phải lớn hơn timeout tổng của pipeline, nếu không sẽ cắt ngang trước khi retry kịp chạy
        // HttpClient.Timeout must exceed the pipeline total timeout, otherwise it aborts before retries can run
        client.Timeout = options.TotalTimeout + TimeSpan.FromSeconds(5);
    })
    .AddStandardResilienceHandler()
    .Configure((HttpStandardResilienceOptions resilience, IServiceProvider sp) =>
    {
        MoMoConnect options = sp.GetRequiredService<IOptions<MoMoConnect>>().Value;

        resilience.AttemptTimeout.Timeout = options.AttemptTimeout;
        resilience.TotalRequestTimeout.Timeout = options.TotalTimeout;
        resilience.Retry.MaxRetryAttempts = options.MaxRetryAttempts;

        // POST tạo giao dịch không idempotent, retry có thể trừ tiền khách hai lần
        // Payment-creating POSTs are not idempotent, retrying could charge the customer twice
        resilience.Retry.DisableForUnsafeHttpMethods();
    });
```

### 33.2. Anti-patterns (KHÔNG làm)

- `HttpClient` không timeout, hoặc timeout mặc định 100 giây cho đối tác thanh toán.
- Retry `POST` tạo giao dịch không có idempotency key.
- Retry vô hạn hoặc không backoff.
- Retry lỗi 400 / 401 / 403 / 404 / 422.
- Vòng lặp retry tự viết.
- Coi timeout là thất bại rồi cho khách thanh toán lại.

---

## 34. Cấu hình qua Options pattern, có validate và fail ngay khi khởi động

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Sai cấu hình (thiếu URL đối tác, timeout = 0) làm pod **không khởi động được** và bị chặn ngay ở bước deploy, thay vì chạy "xanh" rồi lỗi lúc khách thanh toán. Cấu hình có kiểu rõ ràng, dễ test. |
| **Rủi ro nếu vi phạm** | `IConfiguration["Partner:Url"]` rải rác: sai tên key trả `null` âm thầm. Cấu hình sai chỉ lộ ra ở request đầu tiên trên production. |

### 34.1. Quy tắc

1. **Mỗi nhóm cấu hình một class Options** (`{Partner}Connect`, `{Feature}Options`), `sealed`, property có XML doc song ngữ (Rule 7). Section name khai báo bằng `const string SectionName`.
2. **Đăng ký bằng `AddOptions<T>().Bind(...)`** kèm **`ValidateDataAnnotations()`** (hoặc `IValidateOptions<T>` / FluentValidation cho rule phức tạp) và **`ValidateOnStart()`**.
3. **Cấm inject `IConfiguration`** vào business service. Chỉ dùng `IConfiguration` ở composition root (`Program.cs`, module `ConfigureServices`).
4. **Chọn đúng interface**:

   | Interface | Lifetime | Reload khi config đổi | Dùng khi |
   | --- | --- | --- | --- |
   | `IOptions<T>` | Singleton | Không | Mặc định, cấu hình cố định suốt vòng đời app |
   | `IOptionsSnapshot<T>` | Scoped | Có, mỗi request | Cần giá trị mới theo request. **Không** inject vào Singleton (Rule 29) |
   | `IOptionsMonitor<T>` | Singleton | Có, kèm `OnChange` | Singleton / BackgroundService cần giá trị mới; named options |

5. **Secret không nằm trong `appsettings.json`** commit lên Git (Rule 32). File chỉ chứa key rỗng hoặc placeholder, giá trị thật override bằng biến môi trường / Vault.
6. **Thời lượng khai báo kiểu `TimeSpan`** (`"00:00:30"`), không dùng `int` đơn vị mơ hồ (`TimedOut = 30000` là ms hay s?). Code cũ dùng `int` thì tên property phải kèm đơn vị (`TimeoutMilliseconds`).

✅ **ĐÚNG**:

```csharp
public sealed class MoMoConnect
{
    public const string SectionName = "MoMoConnect";

    /// <summary>
    /// Địa chỉ gốc API của MoMo.
    /// <para>Base URL of the MoMo API.</para>
    /// </summary>
    [Required, Url]
    public required string Url { get; init; }

    /// <summary>
    /// Thời gian chờ tối đa cho mỗi lần gọi.
    /// <para>Maximum wait time for each attempt.</para>
    /// </summary>
    [Range(typeof(TimeSpan), "00:00:01", "00:01:00")]
    public TimeSpan AttemptTimeout { get; init; } = TimeSpan.FromSeconds(10);

    /// <summary>
    /// Khoá bí mật ký request, nạp từ Vault, không commit vào appsettings.
    /// <para>Request signing secret, loaded from Vault and never committed to appsettings.</para>
    /// </summary>
    [Required]
    public required string SecretKey { get; init; }
}
```

```csharp
builder.Services
    .AddOptions<MoMoConnect>()
    .Bind(builder.Configuration.GetSection(MoMoConnect.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

### 34.2. Anti-patterns (KHÔNG làm)

- `_configuration["MoMo:Url"]` trong service.
- Options không validate, hoặc validate nhưng thiếu `ValidateOnStart()`.
- `IOptionsSnapshot<T>` trong Singleton.
- Timeout kiểu `int` không ghi đơn vị.
- Giá trị secret thật trong `appsettings.Development.json` commit lên Git.

---

## 35. Thời gian, tiền tệ, culture & so sánh chuỗi phải tường minh

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Kết quả giống nhau trên mọi máy, mọi container, mọi múi giờ. Test được logic phụ thuộc thời gian (hết hạn, lịch tiêm, SLA) mà không cần chờ thật. Không lệch tiền do làm tròn. |
| **Rủi ro nếu vi phạm** | `DateTime.Now` trên server UTC lệch 7 tiếng giờ Việt Nam, làm đơn ghi sai ngày và báo cáo cuối ngày sai. Làm tròn mặc định của .NET là **banker's rounding** (`2.5 → 2`), lệch tiền khi đối soát. `ToString()` / `Parse()` theo culture của máy: `1.5` thành `1,5` khi culture là `vi-VN`, khiến chữ ký gửi đối tác sai. `ToUpper()` theo culture gây lỗi kinh điển "Turkish I". |

### 35.1. Thời gian

1. **Inject `TimeProvider`** (.NET 8, đăng ký `TimeProvider.System` Singleton) và dùng `timeProvider.GetUtcNow()`. **Cấm** `DateTime.Now` / `DateTime.UtcNow` / `DateTimeOffset.Now` trực tiếp trong business code. ABP dùng `IClock` (cấu hình `AbpClockOptions.Kind = DateTimeKind.Utc`).
2. **Lưu trữ và truyền tải bằng UTC** (`DateTimeOffset` hoặc `DateTime` có `Kind = Utc`). Chỉ chuyển sang giờ Việt Nam ở **biên hiển thị** hoặc khi nghiệp vụ định nghĩa theo ngày địa phương ("cuối ngày", "trong ngày").
3. **Múi giờ Việt Nam** dùng id IANA `"Asia/Ho_Chi_Minh"` (chạy được trên Linux container và Windows từ .NET 6 có ICU), khai báo một chỗ duy nhất.
4. **Ngày không có giờ** (ngày sinh, ngày hẹn tiêm) dùng `DateOnly`; giờ trong ngày dùng `TimeOnly`.
5. **Test** dùng `FakeTimeProvider` (`Microsoft.Extensions.TimeProvider.Testing`).

### 35.2. Tiền tệ

1. **Tiền dùng `decimal`**, cấm `float` / `double` (Rule 25.2).
2. **Làm tròn luôn chỉ định `MidpointRounding`** theo quy định nghiệp vụ (thường `MidpointRounding.AwayFromZero`). Không gọi `Math.Round(x)` mặc định.
3. **VND không có phần thập phân**: làm tròn về số nguyên **trước** khi gửi đối tác / ký số, và làm tròn **một lần ở cuối** phép tính, không làm tròn từng bước trung gian.
4. Chia tiền (trả góp, chia khuyến mãi theo dòng hàng) phải phân bổ phần dư vào một dòng để **tổng sau chia bằng tổng trước chia**.

### 35.3. Culture & chuỗi

1. **Dữ liệu máy đọc** (payload đối tác, chuỗi ký, key cache, log, file CSV) format/parse bằng `CultureInfo.InvariantCulture`. Dữ liệu hiển thị cho người dùng mới dùng culture `vi-VN`.
2. **So sánh chuỗi luôn chỉ định `StringComparison`**: mã, key, id dùng `Ordinal` / `OrdinalIgnoreCase`; văn bản hiển thị dùng `CurrentCulture`. Đổi hoa/thường dùng `ToUpperInvariant()` / `ToLowerInvariant()`.
3. **Dictionary key chuỗi** không phân biệt hoa thường khởi tạo với `StringComparer.OrdinalIgnoreCase`.
4. Container chạy `InvariantGlobalization=true` thì không có dữ liệu culture và timezone đầy đủ. Service cần `vi-VN` hoặc `Asia/Ho_Chi_Minh` phải tắt chế độ này hoặc cài ICU.

❌ **SAI**:

```csharp
order.CreatedAt = DateTime.Now;                                  // ❌ Phụ thuộc múi giờ server, không test được
decimal fee = Math.Round(amount * 0.015m);                       // ❌ Banker's rounding
string raw = $"amount={amount}&orderId={orderId}";               // ❌ Format theo culture máy, chữ ký có thể sai
if (status.ToUpper() == "SUCCESS") { }                           // ❌ Culture-sensitive, không chỉ định comparison
```

✅ **ĐÚNG**:

```csharp
order.CreatedAt = _timeProvider.GetUtcNow();

decimal fee = Math.Round(amount * 0.015m, 0, MidpointRounding.AwayFromZero);

string raw = string.Create(CultureInfo.InvariantCulture, $"amount={amount}&orderId={orderId}");

if (string.Equals(status, PartnerResultCodes.Success, StringComparison.OrdinalIgnoreCase)) { }
```

```csharp
public static class VietnamTime
{
    public static readonly TimeZoneInfo Zone = TimeZoneInfo.FindSystemTimeZoneById("Asia/Ho_Chi_Minh");

    // "Ngày nghiệp vụ" tính theo giờ Việt Nam, không theo UTC, để chốt doanh thu đúng ngày
    // The business date follows Vietnam time, not UTC, so daily revenue closes on the correct day
    public static DateOnly BusinessDate(DateTimeOffset utcNow)
        => DateOnly.FromDateTime(TimeZoneInfo.ConvertTime(utcNow, Zone).DateTime);
}
```

### 35.4. Anti-patterns (KHÔNG làm)

- `DateTime.Now` / `DateTime.UtcNow` trong business code.
- Lưu giờ địa phương không kèm offset.
- `double` cho tiền; `Math.Round` không chỉ định `MidpointRounding`.
- `ToString()` / `Parse()` / interpolation không chỉ định culture cho dữ liệu máy đọc.
- `==` / `ToUpper()` / `ToLower()` để so sánh mã không phân biệt hoa thường.
- Hardcode `"SE Asia Standard Time"` (chỉ có trên Windows).
