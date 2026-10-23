# Phần XVII. Thiết Kế Bền Vững & Quản Trị Convention (Sustainable Design & Convention Governance)

Các nguyên tắc vượt ra ngoài cú pháp C#: tương thích ngược, idempotency, mô hình hoá domain, tiêu chí một MR phải chứng minh, và **quy trình được phép vi phạm convention**: bắt buộc ghi rõ lý do, lợi ích và rủi ro.

---

## 38. Thay đổi API, DTO, event, schema phải tương thích ngược

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Deploy service độc lập, không phải "big bang" đồng bộ mọi consumer (BFF, mobile app cũ chưa update, service khác, đối tác). Rollback an toàn vì bản cũ vẫn hiểu dữ liệu bản mới ghi. |
| **Rủi ro nếu vi phạm** | Đổi tên hay xoá một field làm app mobile phiên bản cũ crash hàng loạt (không thể ép người dùng update ngay). Consumer Kafka không deserialize được event, kẹt partition. Migration xoá cột làm bản cũ đang chạy song song lúc rolling update lỗi ngay. |

### 38.1. Thay đổi an toàn vs phá vỡ

| An toàn (non-breaking) | Phá vỡ (breaking) |
| --- | --- |
| Thêm field **optional** vào response / event | Xoá, đổi tên, đổi kiểu field |
| Thêm field **optional** vào request (có default) | Thêm field **bắt buộc** vào request |
| Thêm endpoint mới | Đổi route, HTTP method, status code thành công |
| Thêm giá trị enum **khi consumer chịu được giá trị lạ** | Đổi nghĩa của field / giá trị enum hiện có |
| Nới lỏng validation | Siết chặt validation (từ chối input trước đây hợp lệ) |

### 38.2. Quy tắc

1. **Mặc định chỉ thay đổi kiểu bổ sung (additive)**. Thay đổi phá vỡ bắt buộc ra **version mới** (`/api/v2/...` qua `Asp.Versioning.Mvc`, hoặc topic/event schema version mới), giữ version cũ chạy song song tới hết thời hạn deprecation đã thông báo.
2. **Consumer phải dung nạp (tolerant reader)**: bỏ qua field lạ (mặc định của `System.Text.Json`), enum từ bên ngoài deserialize an toàn (dùng string + fallback `Unknown`, không ném lỗi khi gặp giá trị mới).
3. **Không bao giờ đổi nghĩa** của một field đã phát hành. Cần nghĩa mới thì tạo field mới.
4. **Migration DB theo mô hình expand → migrate → contract**, mỗi bước một lần deploy:
   - *Expand*: thêm cột mới nullable / có default; code ghi cả hai cột.
   - *Migrate*: backfill dữ liệu; code đọc cột mới.
   - *Contract*: chỉ xoá cột cũ khi **không còn phiên bản nào** đang chạy đọc nó.
5. **Endpoint deprecated** đánh dấu `[Obsolete]` + `ApiVersion(..., Deprecated = true)`, có ticket theo dõi ngày gỡ.
6. MR có thay đổi hợp đồng phải ghi rõ "breaking / non-breaking" (Rule 41).

---

## 39. Mọi thao tác thay đổi trạng thái có thể bị lặp phải idempotent

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Retry (Rule 33), double-click của người dùng, message được giao lại (Kafka at-least-once, Outbox Rule 25.4), callback đối tác gửi nhiều lần: tất cả đều an toàn. |
| **Rủi ro nếu vi phạm** | Trừ tiền hai lần, tạo hai đơn, gửi hai SMS, cộng điểm loyalty hai lần. Đây là loại lỗi gây thiệt hại tài chính trực tiếp và khó đối soát. |

### 39.1. Quy tắc

1. **API `POST` tạo giao dịch / đơn hàng / thanh toán** nhận header `Idempotency-Key` (hoặc mã nghiệp vụ duy nhất như `OrderCode`, `RequestId`). Server lưu key + hash request + response:
   - Key mới: xử lý và lưu kết quả.
   - Key đã có, cùng hash: trả lại response cũ, **không** xử lý lại.
   - Key đã có, khác hash: trả `409 Conflict` / `422`.
2. **Ràng buộc duy nhất ở DB** (`UNIQUE` trên `IdempotencyKey` / `RequestId` / cặp `(PartnerCode, PartnerTransactionId)`), không chỉ kiểm tra `AnyAsync` ở code (Rule 25.2). Ưu tiên insert trước, bắt lỗi trùng sau.
3. **Consumer message** dedupe theo `EventId` (bảng inbox / processed-messages), ghi dấu đã xử lý **trong cùng transaction** với thay đổi nghiệp vụ.
4. **Callback / IPN từ đối tác** xử lý theo chuyển trạng thái có điều kiện: chỉ `Pending → Succeeded`. Callback đến lần hai thấy đã `Succeeded` thì trả thành công, không xử lý lại.
5. **Lời gọi tạo giao dịch sang đối tác** luôn mang `requestId` duy nhất do mình sinh và lưu **trước** khi gọi (Rule 25.5).
6. Thao tác cộng dồn (`+= amount`) là **không** idempotent. Chuyển thành "đặt trạng thái" hoặc gắn với id giao dịch đã dedupe.

```csharp
// Chuyển trạng thái có điều kiện: callback trùng sẽ không khớp WHERE nên không ghi lần hai
// Conditional state transition: a duplicate callback will not match WHERE, so nothing is written twice
int affectedRows = await _dbContext.Payments
    .Where(x => x.RequestId == callback.RequestId && x.Status == PaymentStatus.Pending)
    .ExecuteUpdateAsync(s => s
        .SetProperty(x => x.Status, PaymentStatus.Succeeded)
        .SetProperty(x => x.PartnerTransactionId, callback.TransId), cancellationToken);
```

---

## 40. Mô hình hoá domain: Value Object cho khái niệm có bất biến, hành vi nằm trong Entity

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Không thể truyền nhầm `shopCode` vào chỗ `orderCode` vì compiler chặn. Bất biến (số tiền không âm, số điện thoại đúng định dạng) được kiểm **một lần** khi tạo, mọi nơi dùng đều tin được. Logic nghiệp vụ tập trung, không rải khắp service. |
| **Rủi ro nếu vi phạm** | **Primitive obsession**: method nhận 4 tham số `string` liên tiếp rất dễ truyền sai thứ tự. **Anemic model**: entity chỉ có getter/setter, mọi service tự set `Status`, nên bất biến bị phá ở một chỗ nào đó không ai kiểm soát. |
| **Rủi ro khi áp dụng quá đà** | Bọc **mọi** `string` / `int` thành type riêng làm code nặng nề, tốn công mapping. Chỉ áp dụng cho khái niệm **có quy tắc định dạng hoặc bất biến**, hoặc **dễ nhầm lẫn** với khái niệm khác cùng kiểu. |

### 40.1. Quy tắc

1. **Value Object** cho: tiền (`Money` = số tiền + đơn vị tiền tệ), mã nghiệp vụ có định dạng (`OrderCode`, `PhoneNumber`, `Imei`), khoảng thời gian (`DateRange`). Khai báo `readonly record struct` (nhỏ, không null) hoặc `sealed record`. Tạo qua factory method có validate, constructor `private`. ABP có base `ValueObject`.
2. **Entity giữ hành vi**: thay đổi trạng thái qua method có tên nghiệp vụ (`order.Confirm()`, `order.Cancel(reason)`), setter `private` / `protected`. Method kiểm tra chuyển trạng thái hợp lệ (gợi ý State pattern khi ≥ 4 trạng thái, Rule 24).
3. **Validation format** ở biên vẫn thuộc FluentValidation (Rule 17). Value Object bảo vệ **bất biến** khi object được tạo từ bất kỳ đâu (kể cả code nội bộ, không chỉ request).
4. **Map EF Core** bằng `ComplexProperty` (EF Core 8), `OwnsOne` hoặc value converter. DTO vẫn dùng kiểu primitive để serialize (Rule 1–5); map ở tầng Application.
5. Codebase hiện có dùng entity POCO (như ABP hoặc hệ thống legacy): **không** refactor hàng loạt. Áp dụng cho module mới hoặc khi đang sửa đúng phần đó.

```csharp
public readonly record struct OrderCode
{
    public string Value { get; }

    private OrderCode(string value) => Value = value;

    public static OrderCode Create(string raw)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(raw);

        string normalized = raw.Trim().ToUpperInvariant();

        if (normalized.Length > OrderCodeRules.MaxLength)
        {
            throw new DomainException(OrderErrorCodes.InvalidOrderCode);
        }

        return new OrderCode(normalized);
    }

    public override string ToString() => Value;
}
```

```csharp
public sealed class Order
{
    public OrderStatus Status { get; private set; }

    public void Cancel(string reason, DateTimeOffset cancelledAt)
    {
        if (Status is OrderStatus.Completed or OrderStatus.Cancelled)
        {
            throw new DomainException(OrderErrorCodes.CannotCancel);
        }

        Status = OrderStatus.Cancelled;
        CancelReason = reason;
        CancelledAt = cancelledAt;
    }
}
```

---

## 41. Một Merge Request phải chứng minh được tính đúng, an toàn và khả năng rollback

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Reviewer (Tech Lead review trước khi merge vào `ci` theo git workflow) có đủ thông tin để quyết định trong một lần đọc. Sự cố sau deploy có sẵn kế hoạch rollback. |
| **Rủi ro nếu vi phạm** | MR "fix bug" không mô tả gì: reviewer approve theo cảm tính. Migration không có kế hoạch rollback: sự cố production phải sửa tay trên DB. Breaking change lọt qua review mà không ai biết. |

### 41.1. Mô tả MR bắt buộc có

| Mục | Nội dung |
| --- | --- |
| **What / Why** | Thay đổi gì, vì sao, link Jira |
| **Cách kiểm chứng** | Test đã thêm/sửa (Rule 37); kết quả `dotnet build` + `dotnet test`; bước test tay nếu có |
| **Tương thích** | Breaking / non-breaking cho API, DTO, event, schema (Rule 38) |
| **Dữ liệu** | Có migration không; migration có idempotent / rollback được không; ảnh hưởng dữ liệu hiện có; thời gian chạy ước tính trên bảng lớn |
| **Cấu hình** | Key cấu hình / secret / feature flag mới cần thêm ở từng môi trường (Rule 34) |
| **Rollback** | Cách quay lui: revert code, tắt feature flag, migration down, hoặc lý do không cần |
| **Ngoại lệ convention** | Danh sách suppress / ADR mới kèm lý do – lợi ích – rủi ro (Rule 42) |
| **TODO** | Mọi `TODO(<JIRA>)` mới thêm (Rule 7.1 §6) |

### 41.2. Quy tắc

1. MR nhỏ, **một mục đích**. Refactor tách khỏi thay đổi hành vi.
2. CI xanh (Rule 36.4) trước khi yêu cầu review.
3. Tự review diff trước khi gửi: không còn code debug, `Console.WriteLine`, secret, file rác (`.DS_Store`, `bin/`, `obj/`).
4. Commit theo Rule 20.

---

## 42. Ngoại lệ convention: chỉ được vi phạm khi ghi rõ LÝ DO – LỢI ÍCH – RỦI RO

Không có bộ convention nào đúng cho 100% tình huống. Rule này **cho phép** vi phạm có kiểm soát, đổi lại **mọi** ngoại lệ phải được ghi lại để người sau hiểu vì sao, đánh giá lại được, và gỡ bỏ khi không còn cần.

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Tránh hai thái cực: tuân thủ máy móc gây code tệ hơn (ví dụ ép Strategy cho 2 nhánh đơn giản), hoặc vi phạm âm thầm khiến convention mất hiệu lực. Mỗi ngoại lệ có chủ, có lý do, có rủi ro được nêu rõ nên reviewer quyết định có cơ sở. Khi bối cảnh thay đổi (nâng .NET, đối tác đổi API) có thể tìm lại toàn bộ ngoại lệ để gỡ. |
| **Rủi ro nếu vi phạm** | `#pragma warning disable` không giải thích: không ai dám gỡ vì không biết vì sao có nó. Suppress ở cấp project (`<NoWarn>`, `.editorconfig` `severity = none`) tắt rule cho **mọi** code tương lai chứ không chỉ chỗ đang gặp vấn đề. Ngoại lệ tích tụ thành "convention ngầm" mâu thuẫn với convention chính thức. |
| **Rủi ro khi áp dụng** | Quy trình quá nặng khiến dev né bằng cách không bật analyzer. Vì vậy chia hai cấp: **suppress cục bộ** (nhẹ, ghi ngay trong code) và **ADR** (cho quyết định kiến trúc hoặc ảnh hưởng rộng). |

### 42.1. Khi nào được phép ngoại lệ

Chỉ khi rơi vào **ít nhất một** trường hợp:
- **False positive** của analyzer: rule báo sai, logic chứng minh được là đúng.
- **Ràng buộc bên ngoài**: đối tác, framework (ABP), thư viện, hoặc code generated buộc phải làm khác.
- **Tương thích ngược**: sửa theo convention sẽ phá hợp đồng đã phát hành (Rule 38).
- **Hiệu năng đo được**: có số liệu benchmark/profiling, không phải phỏng đoán.
- **Chi phí lớn hơn lợi ích rõ rệt**: ví dụ áp pattern làm code phức tạp hơn vấn đề nó giải quyết.

**Không** được viện lý do: "gấp deadline", "code cũ cũng thế", "AI sinh ra như vậy", "build đang đỏ".

### 42.2. Ba thông tin bắt buộc của mọi ngoại lệ

| Thành phần | Trả lời câu hỏi | Ví dụ |
| --- | --- | --- |
| **Lý do (Why)** | Vì sao không thể/không nên làm theo convention ở đây? | Đối tác bắt buộc field tên `partnerCode` dạng `UPPER_SNAKE` trong chuỗi ký |
| **Lợi ích (Benefit)** | Vi phạm mang lại gì cụ thể (đo được nếu có)? | Chữ ký khớp tài liệu đối tác, không cần lớp mapping riêng |
| **Rủi ro (Risk) & giảm thiểu** | Điều gì có thể sai, ai bị ảnh hưởng, làm gì để hạn chế? | Dev sau tưởng nhầm là lỗi đặt tên và sửa lại, làm chữ ký sai. Giảm thiểu: test chữ ký với vector mẫu của đối tác |

Kèm theo: **mã Jira** theo dõi, và **điều kiện gỡ bỏ** nếu ngoại lệ là tạm thời (ví dụ "gỡ khi shared lib lên v2.4").

### 42.3. Cấp 1 — Suppress cục bộ trong code

Dùng cho một dòng / một member. **Phạm vi hẹp nhất có thể.**

**Ưu tiên `[SuppressMessage]`** với `Justification` (có thể tìm kiếm, xuất hiện trong báo cáo analyzer):

```csharp
[SuppressMessage(
    "Design",
    "CA1031:Do not catch general exception types",
    Justification = "PH-2611 | Lý do: biên consumer Kafka, một message lỗi không được làm dừng cả partition " +
                    "| Lợi ích: các message sau vẫn được xử lý " +
                    "| Rủi ro: che giấu lỗi lập trình; giảm thiểu: log Error kèm offset và đẩy message vào DLQ")]
public async Task HandleAsync(ConsumeResult<string, string> message, CancellationToken cancellationToken)
```

**Khi bắt buộc dùng `#pragma`**: bao đúng dòng cần, luôn có `restore`, comment song ngữ (Rule 7.1) gồm đủ ba thành phần:

```csharp
// PH-2510 | Lý do: tài liệu tích hợp v1 của đối tác bắt buộc chữ ký MD5, chưa hỗ trợ SHA-256
// Lợi ích: tích hợp được đúng đặc tả | Rủi ro: MD5 yếu, có thể bị giả mạo; giảm thiểu: chỉ dùng cho đối tác này, kèm TLS + IP whitelist, gỡ khi đối tác lên v2
// PH-2510 | Why: the partner's v1 integration spec mandates MD5 signatures and does not support SHA-256 yet
// Benefit: integrates exactly to spec | Risk: MD5 is weak and forgeable; mitigation: scoped to this partner only, TLS + IP whitelist, remove once partner ships v2
#pragma warning disable CA5351
byte[] signature = MD5.HashData(Encoding.UTF8.GetBytes(rawData));
#pragma warning restore CA5351
```

**Quy tắc cấp 1:**
1. Không có đủ Lý do – Lợi ích – Rủi ro thì reviewer **phải reject**.
2. Cấm `#pragma warning disable` không có `restore`, hoặc bao cả file / class.
3. `GlobalSuppressions.cs` chỉ được chứa entry có `Scope` và `Target` cụ thể kèm `Justification` đủ ba thành phần. Cấm suppress toàn assembly không target.
4. Mọi suppress mới được liệt kê trong mô tả MR (Rule 41).

### 42.4. Cấp 2 — Architecture Decision Record (ADR)

**Bắt buộc ADR** khi ngoại lệ:
- Tắt hoặc hạ severity một rule ở cấp **project / solution** (`<NoWarn>`, `.editorconfig` `dotnet_diagnostic.X.severity = none`, `WarningsNotAsErrors`).
- Đi ngược một **rule kiến trúc** (Rule 17, 18, 19, 24, 25, 29...) ở phạm vi module trở lên.
- Chọn công nghệ / thư viện khác chuẩn (ví dụ dùng `Refit` thay `IHttpClientFactory` thuần, dùng Newtonsoft cho module mới).
- Áp dụng lâu dài hoặc nhiều team bị ảnh hưởng.

**Vị trí**: `docs/adr/NNNN-<tieu-de-kebab>.md` trong repo service (đánh số tăng dần, không sửa ADR đã `Accepted`; thay đổi quyết định thì tạo ADR mới với trạng thái `Supersedes NNNN`). Link ADR trong `.editorconfig` / `Directory.Build.props` ngay cạnh dòng tắt rule.

**Template:**

```markdown
# ADR-0007: Tắt CA2007 (ConfigureAwait) cho project HttpApi.Host

- Trạng thái: Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
- Ngày: 2026-10-16
- Người quyết định: <Tech Lead>, <người đề xuất>
- Jira: PH-2700
- Rule liên quan: Rule 28.1 §6

## Bối cảnh
Tình huống, ràng buộc, số liệu dẫn tới cần quyết định.

## Quyết định
Làm gì, áp dụng ở phạm vi nào (project / namespace / module).

## Lý do
Vì sao convention không phù hợp ở đây.

## Lợi ích
Kết quả cụ thể mang lại; số liệu nếu có.

## Rủi ro & biện pháp giảm thiểu
| Rủi ro | Khả năng | Ảnh hưởng | Giảm thiểu |
| --- | --- | --- | --- |
| ... | Thấp/Trung bình/Cao | ... | ... |

## Phương án đã cân nhắc
Các lựa chọn khác và vì sao không chọn.

## Điều kiện xem xét lại / gỡ bỏ
Khi nào quyết định này hết hiệu lực (ngày review lại, phiên bản .NET, thay đổi đối tác...).
```

Ví dụ dòng tắt rule có liên kết ADR:

```ini
# ADR-0007: ASP.NET Core không có SynchronizationContext nên ConfigureAwait(false) không mang lại lợi ích ở host
# ADR-0007: ASP.NET Core has no SynchronizationContext, so ConfigureAwait(false) brings no benefit in the host
[src/**/HttpApi.Host/**.cs]
dotnet_diagnostic.CA2007.severity = none
```

### 42.5. Rà soát định kỳ

- Mỗi đợt nâng version .NET / ABP hoặc mỗi quý: tìm toàn bộ `SuppressMessage`, `#pragma warning disable`, `NoWarn`, `severity = none`, ADR `Accepted` và đánh giá lại "điều kiện gỡ bỏ".
- Ngoại lệ không còn lý do: gỡ và sửa code theo convention, ADR chuyển `Deprecated`.
- Một ngoại lệ lặp lại ở ≥ 3 nơi / ≥ 2 repo là tín hiệu **convention cần sửa**: đề xuất cập nhật chính bộ skill này thay vì tiếp tục suppress.

### 42.6. Anti-patterns (KHÔNG làm)

- Suppress không có `Justification`, hoặc justification chung chung ("false positive", "không cần thiết", "theo yêu cầu").
- Thiếu một trong ba: Lý do, Lợi ích, Rủi ro.
- `#pragma warning disable` không `restore`; bao cả file.
- Tắt rule cấp project không có ADR.
- Sửa nội dung ADR đã `Accepted` thay vì tạo ADR mới.
- AI agent tự thêm suppress để build xanh mà không báo trong phần tổng kết task.
