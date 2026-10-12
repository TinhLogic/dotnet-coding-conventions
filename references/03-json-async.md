# Phần III & IV. JSON Serialization & Xử Lý Bất Đồng Bộ

Tài liệu chi tiết về JSON Serialization (`System.Text.Json`), Async/Await và `CancellationToken`.

---

# III. JSON Serialization & Mapping

## 11. Dùng `System.Text.Json` thay cho `Newtonsoft.Json`

Code **mới tạo** phải sử dụng `System.Text.Json` (built-in từ .NET Core 3.0+). Không thêm dependency `Newtonsoft.Json` cho code mới.

❌ **SAI** — dùng Newtonsoft cho code mới:

```csharp
using Newtonsoft.Json;

var json = JsonConvert.SerializeObject(request);
var result = JsonConvert.DeserializeObject<PartnerResponseDto>(responseContent);
```

✅ **ĐÚNG** — dùng System.Text.Json:

```csharp
using System.Text.Json;

var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};

var json = JsonSerializer.Serialize(request, options);
var result = JsonSerializer.Deserialize<PartnerResponseDto>(responseContent, options);
```

Attribute mapping:

| Newtonsoft                         | System.Text.Json                                               |
| ---------------------------------- | -------------------------------------------------------------- |
| `[JsonProperty("name")]`           | `[JsonPropertyName("name")]`                                   |
| `[JsonIgnore]`                     | `[JsonIgnore]`                                                 |
| `NullValueHandling.Ignore`         | `DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull` |
| `JsonConvert.SerializeObject`      | `JsonSerializer.Serialize`                                     |
| `JsonConvert.DeserializeObject<T>` | `JsonSerializer.Deserialize<T>`                                |

**Ngoại lệ**: Code cũ đã dùng `Newtonsoft.Json` thì **không bắt buộc** phải migrate ngay (tránh regression). Nhưng khi sửa method đó, **ưu tiên** chuyển sang `System.Text.Json` nếu không phá vỡ contract hiện tại.

Tip: Nếu cần tương thích cả hai trong giai đoạn chuyển đổi, có thể dùng `[JsonPropertyName("x")]` (System.Text.Json) cùng `[JsonProperty("x")]` (Newtonsoft) trên cùng property — cả hai attribute sẽ hoạt động song song với serializer tương ứng.

---

## 12. Property nullable trong DTO phải có `[JsonIgnore(Condition = WhenWritingNull)]`

Khi serialize DTO ra JSON để gửi đối tác hoặc trả về caller, các property nullable (`string?`, `int?`, `Guid?`, `List<T>?`...) mà chưa được gán giá trị **không nên** xuất hiện trong payload JSON (tránh gửi `"FieldName": null` thừa, giảm kích thước payload và tránh lỗi ở phía đối tác khi nhận field null không mong đợi).

**Cách áp dụng**: Dùng `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` trên từng property nullable, hoặc cấu hình global `DefaultIgnoreCondition` trong `JsonSerializerOptions`.

### Cách 1: Attribute trên từng property (ưu tiên khi chỉ vài field nullable)

❌ **SAI** — property nullable không có ignore:

```csharp
public class YourAppRefundOutputDto
{
    public bool IsSuccess { get; set; }

    public int Code { get; set; }

    public string Message { get; set; }

    public Guid? ExpenseId { get; set; }

    public long? FindCode { get; set; }
}
// → serialize khi ExpenseId = null: {"IsSuccess":false,"Code":400,"Message":"...","ExpenseId":null,"FindCode":null}
```

✅ **ĐÚNG** — có JsonIgnore WhenWritingNull:

```csharp
using System.Text.Json.Serialization;

public class YourAppRefundOutputDto
{
    public bool IsSuccess { get; set; }

    public int Code { get; set; }

    public string Message { get; set; }

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public Guid? ExpenseId { get; set; }

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public long? FindCode { get; set; }
}
// → serialize khi ExpenseId = null: {"IsSuccess":false,"Code":400,"Message":"..."}
```

### Cách 2: Global config (ưu tiên khi toàn bộ DTO đều cần ignore null)

```csharp
private static readonly JsonSerializerOptions JsonOptions = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
```

Quy tắc:
- **DTO gửi sang đối tác** (RequestDto): nếu field optional (không bắt buộc), dùng `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` hoặc global config.
- **DTO trả về cho caller** (OutputDto): nếu field có thể null khi thất bại, dùng attribute để payload gọn hơn.
- **DTO nhận từ đối tác** (ResponseDto): **KHÔNG** cần attribute này — khi deserialize, null field tự động map thành default value.
- Nếu đã dùng global `DefaultIgnoreCondition = WhenWritingNull` trong `JsonSerializerOptions`, không cần đặt attribute lặp lại trên từng property.
- **KHÔNG** dùng `JsonIgnoreCondition.Always` trừ khi field đó thực sự cần bị loại bỏ hoàn toàn khỏi cả serialize lẫn deserialize.

---

# IV. Xử Lý Bất Đồng Bộ (Async / Concurrency)

## 13. Hàm `async` phải có hậu tố `Async`

Mọi method trả về `Task` / `Task<T>` / `ValueTask<T>` đều phải kết thúc bằng `Async`. Áp dụng cho cả interface, implementation, private method và controller action.

❌ **SAI**:

```csharp
Task<MoMoAIOPaymentCreationOutputDto> CreatePayment(MoMoAIOPaymentCreationInputDto request);
private async Task WriteQrHistory(...) { }
```

✅ **ĐÚNG**:

```csharp
Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(MoMoAIOPaymentCreationInputDto request);
private async Task WriteQrHistoryAsync(...) { }
```

Method đồng bộ (không trả `Task`) thì **không** được thêm `Async`.

---

## 14. Luôn có `CancellationToken` với giá trị default

Mọi method async **mới tạo** phải nhận `CancellationToken` là tham số cuối cùng, có default `= default`, và phải truyền token xuống các lời gọi async bên trong.

✅ **ĐÚNG**:

```csharp
public async Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
    MoMoAIOPaymentCreationInputDto request,
    CancellationToken cancellationToken = default)
{
    var response = await _apiService.InvokePartnerApiResponseAsync(..., cancellationToken);
    var content = await response.Content.ReadAsStringAsync(cancellationToken);
    await WriteQrHistoryAsync(logQr, cancellationToken);
    return result;
}
```

Quy tắc:
- Tham số `CancellationToken` đứng **cuối cùng** trong danh sách tham số.
- Luôn có `= default` để không phá vỡ caller hiện tại.
- **Không** được nhận token rồi bỏ không dùng — phải forward xuống tầng dưới.
- Nếu API tầng dưới (ví dụ `IApiService`) chưa hỗ trợ token, ghi chú `// TODO` và nêu rõ trong phần tổng kết, không im lặng bỏ qua.

