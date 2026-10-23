# Phần II. Coding Style & Định Dạng Mã Nguồn C#

Tài liệu chi tiết về phong cách viết mã C#/.NET: PascalCase, định dạng property, XML Documentation song ngữ, comment giải thích "vì sao", Guard clauses Invert `if`, quy tắc `var`, File-scoped namespace, và Style guidelines.

---

## 5. Property / Field trong DTO phải dùng PascalCase

Mọi property trong class C# (bao gồm DTO, Options, Entity) phải đặt tên theo **PascalCase** — đây là quy ước chuẩn của .NET.

Khi API bên ngoài (MoMo, ZaloPay...) yêu cầu payload JSON dạng `camelCase` hoặc `snake_case`, **không được** đặt tên property C# thành `camelCase`. Thay vào đó, dùng `[JsonPropertyName("camelCaseName")]` (System.Text.Json) để map.

Nếu property là tùy chọn (optional) và không muốn serialize khi mang giá trị `null`, sử dụng `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]`.

❌ **SAI**:

```csharp
public class MoMoAIOPaymentCreationRequestDto
{
    public string partnerCode { get; set; }
    public string requestId { get; set; }
    public long amount { get; set; }
    public string storeId { get; set; }
    public string extraData { get; set; }
}
```

✅ **ĐÚNG**:

```csharp
using System.Text.Json.Serialization;

public class MoMoAIOPaymentCreationRequestDto
{
    [JsonPropertyName("partnerCode")]
    public string PartnerCode { get; set; }

    [JsonPropertyName("requestId")]
    public string RequestId { get; set; }

    [JsonPropertyName("amount")]
    public long Amount { get; set; }

    [JsonPropertyName("storeId")]
    public string StoreId { get; set; }

    [JsonPropertyName("extraData")]
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? ExtraData { get; set; }
}
```

Quy tắc:
- Property bắt đầu bằng chữ HOA (PascalCase).
- Nếu tên JSON khác tên C#, dùng `[JsonPropertyName("...")]` để ánh xạ.
- Nếu serialize ra JSON mặc định cần camelCase cho toàn bộ, cấu hình global `JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase` trong Startup — không đặt từng property thành camelCase.
- Với các property optional/nullable cần bỏ qua khi `null`: dùng `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` hoặc global config.
- Không dùng `_` prefix cho property.

---

## 6. Property trong DTO/Options phải cách nhau một dòng trống

Mỗi property trong class DTO, Options (appsettings model), hoặc Entity phải được ngăn cách bởi **một dòng trống** để dễ đọc và dễ review diff.

❌ **SAI** — các property dính liền nhau:

```csharp
public class KredivoConnect
{
    public string Url { get; set; }
    public string ServerKey { get; set; }
    public string DealerCode { get; set; }
    public int TimedOut { get; set; } = 10000;
}
```

✅ **ĐÚNG** — mỗi property cách nhau 1 dòng:

```csharp
public class KredivoConnect
{
    public string Url { get; set; }

    public string ServerKey { get; set; }

    public string DealerCode { get; set; }

    public int TimedOut { get; set; } = 10000;
}
```

Quy tắc:
- Áp dụng cho **DTO**, **Options/Settings class**, và **Entity**.
- Nếu property có `/// <summary>` XML doc hoặc attribute (`[JsonPropertyName]`), dòng trống đặt **trước** block doc/attribute.
- Không áp dụng cho enum members hoặc constants gộp nhóm.

---

## 7. Options/DTO đọc từ tài liệu đối tác phải có XML comment song ngữ

Khi tạo class Options (đọc appsettings) hoặc DTO mà field được lấy/tham chiếu từ tài liệu API đối tác, **bắt buộc** phải có `/// <summary>` XML comment **song ngữ** (tiếng Việt trước, tiếng Anh sau) mô tả ý nghĩa field đó.

Mục đích: khi đọc lại code sau vài tháng, dev không cần mở lại tài liệu đối tác để hiểu field dùng làm gì.

✅ **ĐÚNG** — mỗi field có XML comment mô tả, tiếng Việt trước, tiếng Anh trong `<para>`:

```csharp
public class KredivoConnect
{
    /// <summary>
    /// Domain URL của Kredivo API (sandbox hoặc production)
    /// <para>Kredivo API domain URL (sandbox or production)</para>
    /// </summary>
    public string Url { get; set; }

    /// <summary>
    /// Server key do Kredivo cấp cho merchant để xác thực
    /// <para>Server key issued by Kredivo to the merchant for authentication</para>
    /// </summary>
    public string ServerKey { get; set; }

    /// <summary>
    /// Mã sub-merchant (dealer) dùng cho giao dịch offline, do Kredivo cấp
    /// <para>Sub-merchant (dealer) code issued by Kredivo, used for offline transactions</para>
    /// </summary>
    public string DealerCode { get; set; }

    /// <summary>
    /// Timeout (milliseconds) khi gọi API đối tác Kredivo
    /// <para>Timeout in milliseconds when calling the Kredivo partner API</para>
    /// </summary>
    public int TimedOut { get; set; } = 10000;
}
```

Quy tắc:
- Dòng **tiếng Việt có dấu** đứng trước (chuẩn phổ thông hoặc phong cách Nam Bộ, tránh từ ngữ vùng miền khác như *vâng*, *ạ*...), dòng **tiếng Anh** bọc trong `<para>` đứng sau.
- Viết ngắn gọn, mô tả nghiệp vụ (không phải mô tả kiểu dữ liệu), hai dòng cùng một nội dung.
- Áp dụng cho **Options class**, **DTO gửi sang đối tác** (`RequestDto`), và **DTO nhận về từ đối tác** (`ResponseDto`).

---

## 7.1. Quy tắc viết Comment trong code (Nội dung & XML Documentation)

### 1. Nguyên tắc nền: comment giải thích **VÌ SAO**, không mô tả lại **LÀM GÌ**

Code đã nói rõ nó làm gì. Comment phải trả lời phần code không nói được: lý do nghiệp vụ, ràng buộc từ tài liệu đối tác, quyết định kỹ thuật và cái giá của nó.

❌ **SAI** — comment lặp lại y nguyên code, thêm nhiễu mà không thêm thông tin:

```csharp
// Gán giá trị requestId
requestContext.RequestId = Guid.NewGuid().ToString();

// Serialize payload
string payload = JsonSerializer.Serialize(request, JsonOptions);
```

✅ **ĐÚNG** — comment nêu điều code không thể hiện được:

```csharp
// Đối tác yêu cầu requestId duy nhất theo từng lần gọi, kể cả khi retry cùng orderId
// Partner requires a unique requestId per call, even when retrying the same orderId
requestContext.RequestId = Guid.NewGuid().ToString();

// Đối tác chốt chữ ký trên chuỗi JSON đã sắp thuộc tính theo alphabet và không thụt dòng,
// lệch một ký tự là sai signature nên không được đổi JsonOptions ở đây
// Partner signs the JSON with alphabetically ordered properties and no indentation;
// a one-character difference breaks the signature, so do not swap JsonOptions here
string payload = JsonSerializer.Serialize(request, SignatureJsonOptions);
```

### 2. XML doc (`///`) — thẻ bắt buộc theo từng loại thành viên

| Thành viên                                  | Thẻ bắt buộc                                                        |
| ------------------------------------------- | ------------------------------------------------------------------- |
| `class` / `interface` / `record` / `enum`   | `<summary>`                                                         |
| Method `public` / `internal`                | `<summary>`, `<param>` cho **từng** tham số, `<returns>` nếu có trả về |
| Method ném exception nghiệp vụ chủ động     | thêm `<exception>` kèm ErrorCode cụ thể                             |
| Property / enum member                      | `<summary>` (bắt buộc nếu tên chưa tự giải thích hoặc theo Rule 7)  |
| Implementation / `override` / Decorator     | `<inheritdoc />` — **không** copy lại summary của interface          |

✅ **ĐÚNG** — khai báo trên interface song ngữ:

```csharp
/// <summary>
/// Khởi tạo giao dịch thanh toán phía đối tác và trả về thông tin để điều hướng khách hàng.
/// <para>Initiates a payment transaction on the partner side and returns the customer redirect info.</para>
/// </summary>
/// <param name="input">
/// Thông tin giao dịch do caller truyền lên (đã qua FluentValidation).
/// <para>Transaction information supplied by the caller (already validated by FluentValidation).</para>
/// </param>
/// <param name="cancellationToken">
/// Token hủy tác vụ.
/// <para>Token used to cancel the operation.</para>
/// </param>
/// <returns>
/// Kết quả khởi tạo đã chuẩn hoá cho hệ thống nội bộ.
/// <para>The initialization result, normalized for internal consumers.</para>
/// </returns>
/// <exception cref="BusinessException">
/// Ném <c>ERROR_PARTNER_INITIAL_FAILED</c> khi đối tác trả <c>resultCode</c> khác mã thành công
/// hoặc chữ ký phản hồi không khớp.
/// <para>Throws <c>ERROR_PARTNER_INITIAL_FAILED</c> when the partner returns a non-success
/// <c>resultCode</c> or the response signature does not match.</para>
/// </exception>
Task<PartnerInitialResultDto> InitialAsync(
    PartnerInitialInputDto input,
    CancellationToken cancellationToken = default);
```

Implementation chỉ cần `<inheritdoc />`:

```csharp
/// <inheritdoc />
public async Task<PartnerInitialResultDto> InitialAsync(
    PartnerInitialInputDto input,
    CancellationToken cancellationToken = default)
{
    // ...
}
```

### 3. Những chỗ bắt buộc phải có comment `//` trong thân hàm

| Tình huống                                        | Comment phải nêu rõ                                       |
| ------------------------------------------------- | --------------------------------------------------------- |
| Magic number / giá trị cố định theo đối tác       | Con số đó từ đâu ra, nghĩa gì                             |
| Workaround cho hành vi lạ của đối tác             | Hiện tượng gặp phải + mã ticket theo dõi                  |
| Rule/guard nghiệp vụ bị **tắt hoặc nới** có chủ đích | Mã ticket + lý do nghiệp vụ + điều kiện để bật lại        |
| Thứ tự thực thi quan trọng                        | Vì sao không được đổi thứ tự                              |
| Regex phức tạp                                    | Regex match cái gì, ví dụ chuỗi hợp lệ                    |
| `catch` cố ý không ném lại (swallow)              | Vì sao lỗi này không được phép làm fail nghiệp vụ         |
| Code trông dư thừa nhưng cần thiết                | Vì sao xoá đi sẽ sinh bug                                 |

### 4. Mọi comment đều phải song ngữ: tiếng Việt trước, tiếng Anh sau

**Mọi** comment trong source — `//` trong thân hàm, XML doc `///`, `TODO`/`FIXME` — **bắt buộc** có đủ hai ngôn ngữ, **tiếng Việt đứng trước, tiếng Anh đứng sau**.

#### a. Comment `//` trong thân hàm:
```csharp
// PH-2510: Bỏ ràng buộc Price > 0 — sản phẩm giá 0đ (quà tặng, hàng khuyến mãi) phải đẩy được qua Kredivo
// PH-2510: Price guard removed — zero-price items (gifts, promo) must pass through to Kredivo
List<string> missing = [];
```

#### b. XML doc `///`: Dòng tiếng Anh bọc trong `<para>`:
```csharp
/// <summary>
/// Khởi tạo giao dịch thanh toán phía đối tác và trả về thông tin để điều hướng khách hàng.
/// <para>Initiates a payment transaction on the partner side and returns the customer redirect info.</para>
/// </summary>
```

#### c. `TODO` / `FIXME`: marker chỉ ghi ở dòng tiếng Việt, mã ticket ghi cả hai dòng:
```csharp
// TODO(PH-1289): IApiService chưa nhận CancellationToken, bổ sung khi shared lib lên v2.4 (Rule 14)
// PH-1289: IApiService does not accept CancellationToken yet, wire it up when shared lib reaches v2.4
```

### 5. Những điều không được làm
- Không để code bị comment lại (dead code) — hãy xoá sạch.
- Không ghi changelog, tên người, ngày sửa trong comment (Git đã lưu).
- Không để comment lỗi thời sau khi sửa code.
- Không đưa secret/key/token/signature thật hay PII khách hàng vào comment.

---

## 8. Block `if`/`else` luôn phải có dấu ngoặc nhọn `{ }`

Mọi khối `if`, `else if`, `else` **bắt buộc** phải bọc trong `{ }`, kể cả khi chỉ có một dòng code duy nhất bên trong.

❌ **SAI** — thiếu `{ }`:

```csharp
if (response == null)
    throw new CustomBusinessException("YOURAPP_NULL_RESPONSE");
```

✅ **ĐÚNG** — luôn có `{ }`:

```csharp
if (response == null)
{
    throw new CustomBusinessException("YOURAPP_NULL_RESPONSE");
}
```

---

## 8.1. Ưu tiên Invert `if` (Guard Clauses / Early Return) để làm phẳng luồng xử lý

Khi kiểm tra điều kiện đầu vào hoặc điều kiện xử lý trong hàm, **bắt buộc ưu tiên kỹ thuật Invert `if`** (Guard Clauses / Early Exit / Early Return) để loại bỏ các trường hợp không thỏa mãn trước (qua `return`, `throw`, `continue`, `break`). Tránh lồng `if` nhiều cấp (*Arrow Anti-Pattern / Pyramid of Doom*).

❌ **SAI** — lồng `if` nhiều tầng làm luồng chính bị đẩy sâu vào trong:

```csharp
public void ProcessOrder(OrderDto? order)
{
    if (order != null)
    {
        if (order.IsActive)
        {
            if (order.TotalAmount > 0)
            {
                SendToPaymentGateway(order);
                NotifyCustomer(order);
            }
        }
    }
}
```

✅ **ĐÚNG** — Đảo ngược điều kiện `if` (Guard Clauses & Early Return):

```csharp
public void ProcessOrder(OrderDto? order)
{
    if (order == null)
    {
        return;
    }

    if (!order.IsActive)
    {
        _logger.LogWarning("Đơn hàng chưa kích hoạt");
        return;
    }

    if (order.TotalAmount <= 0)
    {
        _logger.LogWarning("Số tiền đơn hàng không hợp lệ");
        return;
    }

    // Luồng xử lý chính (Happy path) luôn nằm ở cấp ngoài cùng
    SendToPaymentGateway(order);
    NotifyCustomer(order);
}
```

---

## 9. Quy tắc sử dụng `var` vs khai báo kiểu tường minh

| Kịch bản | Khuyến nghị | Ví dụ |
| --- | --- | --- |
| Kiểu dữ liệu đã hiển thị rõ ở vế phải | Dùng `var` hoặc Target-typed `new` | `var user = new UserDto();` hoặc `UserDto user = new();` |
| Kiểu dữ liệu bị ẩn (gọi hàm/phương thức) | Dùng kiểu tường minh | `UserDto user = await _userService.GetByIdAsync(id);` |
| Kiểu nguyên thủy (primitive types) | Dùng kiểu tường minh nếu giá trị không rõ ràng | `int count = 10;` `decimal price = 99.9m;` |
| Kiểu vô danh (anonymous types) | Bắt buộc dùng `var` | `var anon = new { Name = "A", Age = 20 };` |

- **Vòng lặp `for`**: Dùng `var` cho biến chỉ mục: `for (var i = 0; i < count; i++)`
- **Vòng lặp `foreach`**: **Khai báo kiểu tường minh** cho biến lặp: `foreach (PaymentItemDto item in paymentList)`

---

## 10. Khai báo Namespace: File-scoped namespace (.NET 6+) vs Block-scoped namespace (.NET 5 trở xuống)

- **Từ .NET 6 trở lên (C# 10+)**: **Bắt buộc** sử dụng **File-scoped namespace** (`namespace MyNamespace;`).
- **Từ .NET 5 trở xuống / .NET Framework**: Sử dụng **Block-scoped namespace** (`namespace MyNamespace { ... }`).

```csharp
// .NET 6+ (C# 10+)
namespace YourApp.Services;

public class MoMoAIOService : IMoMoAIOService { }
```

---

## 10.1. Đặt chỉ thị `using` bên ngoài khai báo Namespace

Mọi chỉ thị `using` (import thư viện/namespace) phải được đặt ở **đầu file (bên ngoài khai báo namespace)** để đảm bảo fully qualified name và tránh xung đột resolution.

---

## 10.2. Hướng dẫn về Phong cách & Định dạng Mã nguồn (Style Guidelines)

1. **Thụt lề và Ngắt dòng**:
   - Sử dụng **4 khoảng trắng (spaces)** cho mỗi cấp thụt lề. Không dùng Tab.
   - Phong cách ngoặc nhọn **Allman**: `{` và `}` luôn nằm trên một dòng riêng biệt.
   - Mỗi dòng chỉ chứa một câu lệnh hoặc một khai báo biến.
   - Ngắt câu lệnh/biểu thức điều kiện dài **trước toán tử nhị phân** (`&&`, `||`, `+`...).
2. **Comment Style**: Mọi comment song ngữ Việt trước, Anh sau.
3. **Chuỗi**: Nội suy chuỗi `$"{a} {b}"`, Raw String Literals `"""..."""` cho JSON/multiline.
4. **Primary Constructor**: Dùng PascalCase trên Record, camelCase trên Class (C# 12+). Dùng `required` thay vì constructor dài.
5. **Collection Expressions (C# 12+)**: `string[] allowed = ["VND", "USD"];`
6. **Delegates**: Dùng `Action<>`, `Func<>` và Method Group.
7. **Exception & Resource**: `using var` (using declaration), bắt ngoại lệ cụ thể.
8. **Object Initializer**: Khởi tạo bằng `{ ... }`.

---

## 10.3. Quy ước Đặt tên Định danh & Kiểu Dữ liệu Chuẩn .NET (Identifier & Type Conventions)

Kế thừa chuẩn mực từ .NET Runtime team và Microsoft C# Guidelines:

### 1. Từ khóa nguyên thủy thay vì CLR Runtime Types
- **Luôn sử dụng từ khóa C# nguyên thủy** (`string`, `int`, `long`, `bool`, `byte`, `nint`, `nuint`...) thay vì kiểu runtime .NET (`String`, `Int32`, `Int64`, `Boolean`, `Byte`, `IntPtr`...).
- **Ưu tiên `int` có dấu** thay vì các kiểu không dấu (`uint`, `ulong`) cho các biến đếm, số lượng và chỉ mục thông thường. Kiểu `int` là chuẩn chung phổ quát trong C# và tương thích tối đa với các thư viện .NET. Chỉ sử dụng kiểu không dấu khi làm việc với interop native, bitmask hoặc tài liệu phần cứng/đối tác quy định bắt buộc.

❌ **SAI**:
```csharp
System.String customerName = "Nguyen Van A";
Int32 retryCount = 3;
Boolean isSuccess = true;
```

✅ **ĐÚNG**:
```csharp
string customerName = "Nguyen Van A";
int retryCount = 3;
bool isSuccess = true;
```

### 2. Tiền tố Field tĩnh (`s_`) và ThreadStatic (`t_`)
- **Instance field** (`private` / `internal`): Bắt đầu bằng dấu gạch dưới `_` + camelCase (ví dụ: `_logger`, `_httpClientFactory`).
- **Static field** (`private` / `internal static`): Bắt đầu bằng tiền tố **`s_`** + camelCase để phân biệt rõ ràng giữa trạng thái tĩnh dùng chung và trạng thái theo instance.
- **ThreadStatic field** (`[ThreadStatic]`): Bắt đầu bằng tiền tố **`t_`** + camelCase.

✅ **ĐÚNG**:
```csharp
public class PaymentProcessingWorker
{
    private static readonly Lock s_syncLock = new();
    private static readonly HttpClient s_sharedClient = new();

    [ThreadStatic]
    private static int t_executionDepth;

    private readonly ILogger<PaymentProcessingWorker> _logger;
}
```

### 3. Hằng số (`const`) luôn dùng PascalCase
- Mọi hằng số (`const` field trong class, hằng số local trong method) **bắt buộc dùng PascalCase**.
- **Cấm** dùng định dạng chữ hoa gạch dưới (`SCREAMING_SNAKE_CASE` hoặc `UPPER_SNAKE_CASE`).

❌ **SAI**:
```csharp
private const int MAX_RETRY_ATTEMPTS = 3;
public const string DEFAULT_CURRENCY = "VND";
```

✅ **ĐÚNG**:
```csharp
private const int MaxRetryAttempts = 3;
public const string DefaultCurrency = "VND";
```

### 4. Quy ước Đặt tên Enum (Số ít vs Số nhiều)
- **Enum thông thường (Non-flags)**: Dùng **danh từ số ít** (ví dụ: `OrderStatus`, `PaymentMethodType`, `InvoiceStatus`).
- **Enum cờ bit (Có thuộc tính `[Flags]`)**: Dùng **danh từ số nhiều** (ví dụ: `UserPermissions`, `FileAccessOptions`, `AuditLogCategories`).

✅ **ĐÚNG**:
```csharp
public enum PaymentStatus
{
    Pending,
    Processing,
    Completed,
    Failed
}

[Flags]
public enum UserPermissions
{
    None = 0,
    Read = 1 << 0,
    Write = 1 << 1,
    Delete = 1 << 2,
    All = Read | Write | Delete
}
```

### 5. Tham số Kiểu Generic (Type Parameters)
- Khi generic chỉ có duy nhất **1 tham số**: Dùng chữ cái đơn **`T`** (`List<T>`, `IResultDto<T>`).
- Khi generic có **nhiều tham số**: Bắt buộc bắt đầu bằng tiền tố **`T`** kết hợp với danh từ mô tả rõ vai trò (`Converter<TInput, TOutput>`, `ISessionChannel<TSession>`, `IMapper<TSource, TDestination>`).

### 6. Gọi Static Member qua đúng tên Class khai báo
- Luôn gọi static member thông qua chính xác tên lớp khai báo ban đầu (`ClassName.StaticMember`), **không** gọi static member của lớp cha qua tên của lớp con kế thừa.
- Tránh nguy cơ hiểu nhầm ngữ nghĩa và tránh gãy code khi lớp con vô tình bổ sung static member cùng tên trong tương lai.

### 7. Quy tắc Cú pháp Bổ sung
- **Không dùng 2 dấu gạch dưới liên tiếp (`__`)** trong bất kỳ định danh nào (đây là cú pháp dành riêng cho compiler Roslyn sinh mã).
- **Toán tử so sánh điều kiện**: Luôn dùng `&&` và `||` (toán tử ngắn mạch - short-circuiting), cấm dùng `&` và `|` trong biểu thức điều kiện boolean (trừ khi thao tác bitwise trên số hoặc enum flags).

---

## 10.4. Nguyên tắc Sealed by Default & Kế thừa (Inheritance & Sealed Guidelines)

Tuân thủ triết lý cốt lõi của OOP hiện đại: **"Design for inheritance, or prohibit it"** (Thiết kế có chủ đích cho kế thừa, hoặc phải cấm kế thừa):

### 1. "Sealed by Default" cho Concrete Classes
Mọi concrete class (Service, Strategy, Resolver, Repository, Validator, Decorator, Adapter...) nếu **không được thiết kế có chủ đích để làm Base class** thì **bắt buộc đánh dấu `sealed`** (hoặc `internal sealed`).

Lợi ích bắt buộc:
1. **Bảo vệ tính đóng gói & Hợp đồng (Encapsulation)**: Ngăn chặn triệt để hành vi kế thừa tự phát (Inheritance abuse) phá vỡ logic nội bộ hoặc tạo ra sự phụ thuộc chặt chẽ ngoài ý muốn.
2. **Kích hoạt JIT Devirtualization (.NET 6/7/8/9)**: JIT compiler khi phát hiện class là `sealed` sẽ tự động chuyển đổi các lời gọi ảo (virtual call / interface dispatch) thành **direct method call** và tự động **inline** mã nguồn, giúp tăng thông lượng xử lý CPU và giảm cấp phát tài nguyên đáng kể.

❌ **SAI**:
```csharp
internal class KredivoEWalletProviderStrategy : IEWalletProviderStrategy
{
    // Để class hở, cho phép kế thừa ngoài ý muốn và JIT không devirtualize được tối đa
}
```

✅ **ĐÚNG**:
```csharp
internal sealed class KredivoEWalletProviderStrategy : IEWalletProviderStrategy
{
    // Đóng kín class, bảo vệ hợp đồng và hỗ trợ tối ưu JIT
}
```

### 2. Sử dụng `sealed override`
Khi một class con ghi đè một phương thức `virtual` hoặc `abstract` từ base class và muốn **chấm dứt chuỗi ghi đè** (không cho phép các lớp kế thừa tiếp theo sửa đổi hành vi), sử dụng từ khóa kết hợp `sealed override`.

```csharp
public class BaseWorker
{
    public virtual void Execute() { }
}

public class StandardWorker : BaseWorker
{
    // Chặn mọi lớp kế thừa từ StandardWorker không được override lại Execute
    public sealed override void Execute()
    {
        // Core implementation
    }
}
```

### 3. Quy chuẩn sử dụng `abstract`
- Chỉ dùng `abstract class` cho các cấu trúc hạ tầng hoặc pattern có **chủ đích thiết kế kế thừa rõ ràng**:
  - **Template Method**: Abstract class giữ khung luồng xử lý cố định (Rule 19 & Rule 24).
  - **Decorator Base**: Abstract class giữ tham chiếu Component và delegate cuộc gọi (`PartnerServiceDecoratorBase`).
  - **Validator Base**: Kế thừa `AbstractValidator<T>` (Rule 17).
- Cấm tùy tiện tạo abstract class khi chưa có nhu cầu kế thừa thực tế (chống Over-engineering, Rule 24).
- Mọi implementation của một abstract class phải tuân thủ trọn vẹn hợp đồng LSP (Rule 24.5b) — cấm ném `NotSupportedException` trong derived class.

---

## 📚 Tài Liệu Tham Chiếu Microsoft (Official References)

Để tra cứu chi tiết và đối chiếu các tiêu chuẩn của Microsoft Runtime & Language teams:
- [.NET Common C# Code Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [C# Identifier Naming Rules & Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/identifier-names)
- [Abstract and Sealed Classes and Class Members](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/abstract-and-sealed-classes-and-class-members)

