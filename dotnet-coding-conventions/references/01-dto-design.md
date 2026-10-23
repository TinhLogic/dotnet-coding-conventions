# Phần I. Thiết Kế & Cấu Trúc DTO (Data Transfer Objects)

Tài liệu chi tiết về quy chuẩn DTO trong các dự án C#/.NET Enterprise.

---

## 1. DTO — mỗi class một file riêng

Không gộp nhiều class vào một file `.cs`. Không dùng class lồng nhau (nested class) làm container cho DTO.

Tên file phải trùng tên class.

❌ **SAI** — gộp nhiều DTO vào 1 file, bọc trong class container:

```csharp
// MoMoAIODto.cs
public class MoMoAIODto
{
    public class MoMoAIOPaymentCreationInputDto { }
    public class MoMoAIOPaymentCreationOutputDto { }
    public class MoMoAIOUserInfo { }
}
```

✅ **ĐÚNG** — mỗi class một file:

```
Dto/MoMoAIO/MoMoAIOPaymentCreationInputDto.cs
Dto/MoMoAIO/MoMoAIOPaymentCreationOutputDto.cs
Dto/MoMoAIO/MoMoAIOUserInfoDto.cs
Dto/MoMoAIO/MoMoAIODeliveryInfoDto.cs
```

```csharp
// Dto/MoMoAIO/MoMoAIOPaymentCreationInputDto.cs
namespace YourApp.Dto.MoMoAIO;

public class MoMoAIOPaymentCreationInputDto
{
    public string RequestId { get; set; }
}
```

**Lợi ích**: dễ tìm file, giảm conflict khi merge, không cần `using static ...`.

---

## 2. DTO phải kết thúc bằng hậu tố `Dto`

Mọi class dùng để truyền dữ liệu (request/response/nested object) đều phải có hậu tố `Dto`.

| ❌ Sai                 | ✅ Đúng                   |
| --------------------- | ------------------------ |
| `MoMoAIOUserInfo`     | `MoMoAIOUserInfoDto`     |
| `MoMoAIODeliveryInfo` | `MoMoAIODeliveryInfoDto` |
| `Product`             | `ProductDto`             |
| `PromotionInfo`       | `PromotionInfoDto`       |

Áp dụng cho cả class lồng bên trong DTO (nhưng vẫn phải tách ra file riêng theo mục 1).

Ngoại lệ: Options/Settings model (`MoMoAIOConnect`, `MBConnect`) và Entity không cần hậu tố `Dto`.

---

## 3. Tên class DTO phải là danh từ (noun), không dùng động từ

Tên class DTO phải đặt theo **danh từ** hoặc **cụm danh từ** mô tả dữ liệu mà nó chứa. Không đặt tên DTO bằng động từ hoặc cụm động từ — hành động thuộc về method trong Service/Controller, không phải tên DTO.

### Cấu trúc tên DTO

```
[Prefix/Module] + [Danh từ mô tả dữ liệu] + [Loại DTO] + Dto
```

Trong đó:
- **Prefix/Module**: tên đối tác hoặc module (ví dụ: `MoMoAIO`, `ZaloPay`, `YourApp`)
- **Danh từ mô tả dữ liệu**: mô tả "cái gì" chứ không phải "làm gì"
- **Loại DTO**: `Request`, `Input`, `Output`, `Response`, `Result`, `Info`, `Detail`...

### Bảng so sánh

| ❌ Sai (động từ / hành động) | ✅ Đúng (danh từ)              | Giải thích                       |
| --------------------------- | ----------------------------- | -------------------------------- |
| `CreatePaymentDto`          | `PaymentCreationRequestDto`   | "Creation" là danh từ            |
| `RefundPaymentDto`          | `PaymentRefundRequestDto`     | "Refund" dùng như danh từ        |
| `CheckTransactionDto`       | `TransactionQueryRequestDto`  | "Query" là danh từ               |
| `CancelOrderInputDto`       | `OrderCancellationInputDto`   | "Cancellation" là danh từ        |
| `GetUserResponseDto`        | `UserDetailResponseDto`       | "Detail" là danh từ              |
| `ValidateCardDto`           | `CardValidationRequestDto`    | "Validation" là danh từ          |
| `ProcessRefundDto`          | `RefundProcessingInputDto`    | "Processing" là danh từ (gerund) |
| `NotifyPartnerDto`          | `PartnerNotificationInputDto` | "Notification" là danh từ        |

> ⚠️ Bảng trên chỉ minh hoạ trục **động từ → danh từ**; đừng suy ra chiều dữ liệu từ suffix trong bảng này. Việc chọn `InputDto` / `OutputDto` / `RequestDto` / `ResponseDto` tuân theo bảng biên (boundary) ở mục 2 ngay bên dưới.

### Quy tắc chi tiết

**1. Tên DTO phải trả lời câu hỏi "Đây là cái gì?" — không phải "Làm gì?"**

```csharp
// ❌ SAI — tên nghe như method (hành động)
public class CreatePaymentRequestDto { }
public class RefundTransactionInputDto { }  // "Refund" ở đầu = động từ

// ✅ ĐÚNG — tên mô tả dữ liệu (danh từ)
public class PaymentCreationRequestDto { }
public class TransactionRefundInputDto { }  // "Refund" ở giữa = danh từ bổ nghĩa
```

**2. Loại DTO suffix giúp phân biệt vai trò**

Suffix đi theo **từng cặp, gắn với một biên (boundary) của hệ thống** — nhìn suffix là biết dữ liệu đang đi qua biên nào và theo chiều nào:

| Biên                      | Chiều vào (inbound) | Chiều ra (outbound) |
| ------------------------- | ------------------- | ------------------- |
| **Caller ↔ hệ thống mình** | `InputDto`          | `OutputDto`         |
| **Hệ thống mình ↔ đối tác** | `ResponseDto`       | `RequestDto`        |

| Suffix        | Vai trò                                              | Ví dụ                                |
| ------------- | ---------------------------------------------------- | ------------------------------------ |
| `InputDto`    | Caller/client gọi vào (không chứa field internal)    | `PaymentCreationInputDto`            |
| `OutputDto`   | Kết quả trả về cho caller                            | `PaymentCreationOutputDto`           |
| `RequestDto`  | Gửi sang đối tác (đầy đủ field: config, signature)   | `MoMoAIOPaymentRequestDto`           |
| `ResponseDto` | Nhận từ đối tác (raw response)                       | `MoMoAIOPaymentResponseDto`          |
| `ResultDto`   | Kết quả xử lý nội bộ                                 | `RefundResultDto`                    |
| `InfoDto`     | Thông tin bổ sung (nested object)                    | `CustomerInfoDto`, `DeliveryInfoDto` |
| `DetailDto`   | Chi tiết một entity                                  | `PaymentDetailDto`                   |
| `SummaryDto`  | Tóm tắt / danh sách                                  | `PaymentSummaryDto`                  |

Cách nhớ: **`Input` vào thì `Output` ra** (biên với caller) — **`Request` gửi đi thì `Response` nhận về** (biên với đối tác). Không được dùng lẫn cặp: `RequestDto` không bao giờ là DTO nhận từ caller, `OutputDto` không bao giờ là DTO nhận từ đối tác.

Luồng dữ liệu đầy đủ của một giao dịch:

```
Caller ──InputDto──▶ [Controller → Service] ──RequestDto──▶ Đối tác
Caller ◀─OutputDto── [Service ← Mapping]    ◀─ResponseDto── Đối tác
```

**3. Prefix đối tác/module đặt đầu tiên**

```csharp
// ✅ ĐÚNG — prefix module + danh từ + loại
public class MoMoAIOPaymentCreationInputDto { }      // Caller gọi vào
public class YourAppCustomerRefundOutputDto { }      // Trả về cho caller
public class ZaloPayRefundRequestDto { }             // Gửi sang ZaloPay
public class KredivoInstallmentPlanResponseDto { }   // Kredivo trả về
```

**4. Ngoại lệ chấp nhận được**

Một số pattern phổ biến trong .NET được chấp nhận dù có vẻ như động từ:

```csharp
// Chấp nhận — đây là danh từ ghép phổ biến trong domain
public class PaymentCallbackDto { }       // "Callback" là danh từ
public class WebhookNotificationDto { }   // "Notification" là danh từ
public class TransactionLookupResultDto { } // "Lookup" dùng như danh từ
public class RefundDto { }                // "Refund" đứng một mình = danh từ
```

**5. Method trong Service/Controller mới dùng động từ**

```csharp
// Method = hành động (động từ) — nhận InputDto từ caller, trả OutputDto cho caller
Task<PaymentCreationOutputDto> CreatePaymentAsync(PaymentCreationInputDto input);
Task<TransactionRefundOutputDto> RefundTransactionAsync(TransactionRefundInputDto input);
Task<OrderCancellationOutputDto> CancelOrderAsync(OrderCancellationInputDto input);
```

### Tóm tắt nguyên tắc

- **Class DTO = danh từ** → mô tả "dữ liệu là gì"
- **Method = động từ** → mô tả "hành động làm gì"
- Khi phân vân, thử đọc: *"Đây là một [tên class]"* — nếu nghe tự nhiên thì đúng.
  - ✅ "Đây là một PaymentCreationRequest" → OK
  - ❌ "Đây là một CreatePayment" → không tự nhiên

---

## 4. Không expose field internal ra DTO nhận từ client

Các field mà service tự generate hoặc lấy từ config **không được** đưa vào DTO mà client/caller gửi lên. Tách rõ 2 DTO theo đúng biên ở Rule 3:

- **InputDto** (caller gọi vào): chỉ chứa field nghiệp vụ mà caller cần truyền.
- **RequestDto** (gửi sang đối tác): chứa đầy đủ field bao gồm cả config, signature...

Các field điển hình **KHÔNG** nằm trong InputDto:

| Field                     | Lý do                                    | Cách resolve                                        |
| ------------------------- | ---------------------------------------- | --------------------------------------------------- |
| `requestId`               | Internal tracing, không để caller tự đặt | Lấy từ `traceIdNode` > `traceId` > `Guid.NewGuid()` |
| `partnerCode`             | Config                                   | Lấy từ appsettings                                  |
| `secretKey` / `accessKey` | Secret                                   | Lấy từ appsettings                                  |
| `ipnUrl` / `redirectUrl`  | Config                                   | Lấy từ appsettings                                  |
| `signature`               | Computed                                 | Service tự tính từ rawHash + secretKey              |
| `requestType`             | Cố định per flow                         | Lấy từ appsettings                                  |
| `lang`                    | Config default                           | Lấy từ appsettings                                  |
| `autoCapture`             | Cố định = true                           | Hardcode trong service                              |

✅ **ĐÚNG** — InputDto chỉ chứa field caller cần:

```csharp
public class MoMoAIOPaymentCreationInputDto
{
    public string OrderId { get; set; }      // Caller truyền
    public string OrderInfo { get; set; }    // Caller truyền
    public long Amount { get; set; }         // Caller truyền
    public string StoreId { get; set; }      // Caller truyền
    public string InstrumentType { get; set; } // Caller chọn QR/CARD/POS
    // KHÔNG có: RequestId, PartnerCode, IpnUrl, Signature, Lang, AutoCapture
}
```

Service sẽ map InputDto → RequestDto bằng AutoMapper, rồi inject các field còn lại (config, signature, requestId) vào RequestDto trước khi gửi sang đối tác.

