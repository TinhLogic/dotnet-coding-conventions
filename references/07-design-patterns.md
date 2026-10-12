# Phần VIII. Design Pattern & Quản Lý Độ Phức Tạp

Tài liệu chi tiết về ngưỡng định lượng và quy chuẩn tái cấu trúc mã nguồn phức tạp bằng mẫu thiết kế GoF (Gang of Four), đảm bảo nguyên tắc SOLID (đặc biệt là ISP và LSP).

---

## 24. Code phức tạp phải được xử lý bằng GoF Design Pattern, không nhồi thêm `if`/`else`

Khi một đoạn code vượt ngưỡng phức tạp ở mục 24.1, **bắt buộc** tái cấu trúc bằng mẫu thiết kế GoF phù hợp thay vì thêm nhánh điều kiện, thêm cờ `bool`, hay copy một khối logic sang chỗ khác.

Rule này quy định **khi nào** dùng pattern và **pattern nào** cho từng triệu chứng. Rule 19 quy định chi tiết các pattern dành riêng cho luồng tích hợp API đối tác (Orchestrator, Template Method, AOP Interceptor, Decorator) — khi hai rule cùng áp dụng được thì Rule 19 thắng vì nó cụ thể hơn.

### 24.1. Ngưỡng bắt buộc tái cấu trúc

Dùng số đo, không dùng cảm tính. Chạm **một** ngưỡng là phải tái cấu trúc:

| Ngưỡng | Giá trị |
| --- | --- |
| Số nhánh `if`/`else if`/`switch` phân loại trên cùng một enum hoặc một kiểu | ≥ **4** |
| Cùng một chuỗi phân loại lặp lại ở nhiều method | ≥ **2** method |
| Độ dài một method | > **80** dòng |
| Độ sâu lồng khối (`if`, `for`, `try`) | > **3** cấp |
| Số tham số của một method | > **5** |
| Số cờ `bool` điều khiển luồng trong một method | ≥ **2** |

Thêm một nhánh vào chuỗi đã chạm ngưỡng là **vi phạm**, kể cả khi nhánh đó chỉ một dòng — đó chính là cách một chuỗi 4 nhánh lớn thành 51 nhánh.

### 24.2. Triệu chứng → Pattern

| Triệu chứng trong code | Pattern GoF | Áp dụng thế nào |
| --- | --- | --- |
| `if/else` phân loại theo enum, mỗi nhánh một thuật toán khác nhau | **Strategy** | Một interface, mỗi nhánh một class, DI resolve theo key |
| Phải chọn/khởi tạo implementation theo tham số runtime | **Factory Method** / **Abstract Factory** | `I{X}Factory.Create(key)` trả về abstraction, không trả concrete |
| Nhiều bước cố định, chỉ khác nhau vài bước ở giữa | **Template Method** | Base class giữ khung, `abstract`/`virtual` cho bước biến thiên (Rule 19 §1.2) |
| Cần thêm quan tâm bổ trợ (log, cache, retry, phân quyền) mà không sửa nghiệp vụ | **Decorator** | Bọc quanh interface, xem ràng buộc đầy đủ ở Rule 19 §3 |
| Nhiều handler phải thử lần lượt cho tới khi có người xử lý | **Chain of Responsibility** | Mỗi handler giữ `Next`, tự quyết xử lý hay chuyển tiếp |
| Payload/response của đối tác có schema lệch nhau nhưng nghiệp vụ cần một shape | **Adapter** | Adapter riêng cho từng đối tác, map về DTO nội bộ (Rule 19 §1.5) |
| Một hành động phải phát sinh nhiều hệ quả độc lập, danh sách hệ quả còn thay đổi | **Observer** | Publish event, subscriber tự đăng ký; đừng gọi tuần tự trong thân hàm |
| Thuật toán khác nhau theo trạng thái, và trạng thái tự chuyển tiếp | **State** | Mỗi state một class, state tự quyết transition kế tiếp |
| Khởi tạo object có **bất biến phải kiểm tại thời điểm dựng**, hoặc dựng qua nhiều bước rời rạc | **Builder** | Chỉ dùng khi object initializer không diễn tả được. Nhiều field optional thuần thì dùng `required` + object initializer (Rule 10.2 §4), **không** dựng Builder |

Những pattern **không** dùng trong codebase này:
- **Singleton** tự viết (`static` instance, `static` mutable state): dùng DI lifetime `AddSingleton` thay thế. `static` mutable state phá unit test và không an toàn đa luồng.
- **Service Locator** (`serviceProvider.GetService<T>()` rải trong thân hàm): inject qua constructor. Resolve ngầm giữa thân hàm làm dependency biến mất khỏi signature, người đọc không biết class thật sự phụ thuộc cái gì.

### 24.3. Ví dụ: chuỗi phân loại provider → Strategy

Triệu chứng thật trong `EWalletsAppService`: **51 nhánh** `if/else if` trên `EnmWalletProvider`, trải trên **7 method**, mỗi method lặp lại gần như cùng một danh sách provider. Thêm một đối tác phải sửa tới 7 chỗ và rất dễ bỏ sót một chỗ.

❌ **SAI** — mỗi nghiệp vụ một chuỗi phân loại, lặp lại ở mọi method:

```csharp
public async Task<CreateQrCodeOutputDto> CreateQrCodeAsync(CreateQrCodeInputDto input)
{
    if (input.ProviderQrCode == EnmWalletProvider.VNPAY) { /* ... */ }
    else if (input.ProviderQrCode == EnmWalletProvider.MoMo) { /* ... */ }
    else if (input.ProviderQrCode == EnmWalletProvider.Kredivo) { /* ... */ }
    // ...thêm 5 nhánh nữa
    throw new BusinessException(PaymentIntegrationErrorCodes.ERROR_PROVIDER_QRCODE_INVALID);
}

public async Task<CheckTransactionEwalletOutputDto> CheckTransactionPayedAsync(...)
{
    // Lặp lại gần như cùng danh sách provider, lần này 12 nhánh
}
```

✅ **ĐÚNG** — một abstraction cho mỗi nghiệp vụ, mỗi provider một implementation:

```csharp
public interface IEWalletProviderStrategy
{
    // Provider mà strategy này phụ trách
    EnmWalletProvider Provider { get; }

    Task<CreateQrCodeOutputDto> CreateQrCodeAsync(
        CreateQrCodeInputDto input,
        CancellationToken cancellationToken = default);
}
```

```csharp
internal sealed class KredivoEWalletProviderStrategy : IEWalletProviderStrategy
{
    private readonly IKredivoService _kredivoService;

    public KredivoEWalletProviderStrategy(IKredivoService kredivoService)
    {
        _kredivoService = kredivoService;
    }

    public EnmWalletProvider Provider => EnmWalletProvider.Kredivo;

    public async Task<CreateQrCodeOutputDto> CreateQrCodeAsync(
        CreateQrCodeInputDto input,
        CancellationToken cancellationToken = default)
    {
        // Chỉ còn logic riêng của Kredivo, không thấy provider nào khác
    }
}
```

Registry resolve theo enum, thay cho chuỗi phân loại:

```csharp
internal sealed class EWalletProviderStrategyResolver : IEWalletProviderStrategyResolver
{
    private readonly Dictionary<EnmWalletProvider, IEWalletProviderStrategy> _strategies;

    public EWalletProviderStrategyResolver(IEnumerable<IEWalletProviderStrategy> strategies)
    {
        _strategies = strategies.ToDictionary(s => s.Provider);
    }

    public IEWalletProviderStrategy Resolve(EnmWalletProvider provider)
    {
        // Provider chưa đăng ký strategy là lỗi cấu hình, phải ném sớm và rõ
        if (!_strategies.TryGetValue(provider, out IEWalletProviderStrategy strategy))
        {
            throw new BusinessException(PaymentIntegrationErrorCodes.ERROR_PROVIDER_QRCODE_INVALID)
                .WithData("Provider", provider);
        }

        return strategy;
    }
}
```

Orchestrator không còn biết provider nào tồn tại:

```csharp
public async Task<CreateQrCodeOutputDto> CreateQrCodeAsync(
    CreateQrCodeInputDto input,
    CancellationToken cancellationToken = default)
{
    IEWalletProviderStrategy strategy = _resolver.Resolve(input.ProviderQrCode);

    return await strategy.CreateQrCodeAsync(input, cancellationToken);
}
```

Thêm đối tác mới giờ là **thêm file**, không sửa file đang chạy — đúng Open/Closed.

### 24.4. Quy tắc áp dụng

- **Pattern phải giải quyết độ phức tạp đang có thật.** Code chưa chạm ngưỡng 24.1 mà đã bọc 3 lớp abstraction là over-engineering, cũng là vi phạm rule này. Không có nhánh thứ hai thì chưa cần Strategy.
- **Tên class phải chứa tên pattern** khi class đó tồn tại chỉ vì pattern: `KredivoEWalletProviderStrategy`, `PartnerServiceCacheDecorator`, `EWalletProviderStrategyResolver`. Người đọc nhận ra vai trong pattern mà không phải đọc thân class.
- **Một class một vai.** Không để một class vừa là Factory vừa là Strategy.
- **Phụ thuộc qua abstraction**, không qua concrete class — nếu không, pattern mất toàn bộ giá trị thay thế được.
- **Không chồng pattern để cho sang.** Mỗi lớp abstraction thêm vào là một lớp người đọc phải lần qua khi debug. Chỉ thêm khi nó xoá được một ngưỡng ở 24.1.
- **Refactor sang pattern phải là commit riêng** (`refactor(<JIRA>): ...`), tách khỏi commit thêm tính năng (Rule 20 §6). Hai việc trộn một commit thì không review được và không revert riêng được.
- **Refactor không được đổi hành vi.** Chưa có test bao phủ vùng sắp refactor thì viết test trước, hoặc báo rõ trong phần tổng kết là chưa verify được.

### 24.5. Ràng buộc SOLID của pattern

Áp pattern **không** tự động đạt SOLID. Bảng 24.2 chọn đúng pattern giúp đạt SRP (mỗi class một vai), OCP (thêm file thay vì sửa file) và DIP (phụ thuộc abstraction). Hai nguyên tắc còn lại — **ISP** và **LSP** — pattern không tự bảo đảm, phải tuân thủ có ý thức. Đây cũng là hai chỗ vỡ nhiều nhất khi tách một chuỗi phân loại lớn.

| Pattern | Phục vụ sẵn | Vỡ ở đâu nếu làm sai |
| --- | --- | --- |
| Strategy | SRP, OCP, DIP | **ISP** nếu gom nhiều nghiệp vụ vào một interface; **LSP** nếu strategy không làm được hết hợp đồng |
| Template Method | SRP, OCP | **LSP** nếu subclass đổi nghĩa bước đã khai báo ở base |
| Decorator | SRP, OCP, DIP | **LSP** nếu decorator thêm public member hoặc đổi hợp đồng (Rule 19 §3) |
| Chain of Responsibility | SRP, OCP | **LSP** nếu một handler nuốt lỗi rồi trả kết quả rỗng như thể đã xử lý |
| Adapter | SRP, DIP | **LSP** nếu adapter không map đủ field và trả giá trị mặc định giả |
| State / Observer | SRP, OCP | **LSP** nếu một state/subscriber ném exception ngoài hợp đồng |

#### a. ISP — tách interface theo **nghiệp vụ**, không theo **đối tượng**

Mỗi nghiệp vụ một interface riêng. **Không** gom nhiều nghiệp vụ vào một interface "cho đủ bộ" chỉ vì chúng cùng nói về một đối tượng.

Lý do cụ thể trong codebase này: tập provider của từng nghiệp vụ **lệch nhau**.

| Nghiệp vụ | Số provider |
| --- | --- |
| `CreateQrCode` | 8 |
| `CheckTransactionPayed` | 12 |
| `CheckTransactionOnlinePayed` | 5 (chỉ nghiệp vụ này có Moca) |
| `CheckTransactionOnline` | 5 (chỉ nghiệp vụ này có TPBank) |

Hợp lại là 14 provider trên 4 nghiệp vụ = 56 ô, nhưng chỉ **30 ô** có logic thật — gần một nửa ma trận trống. Chỉ VNPAY và Zalopay có mặt ở cả 4 nghiệp vụ.

❌ **SAI** — một interface béo gom cả 4 nghiệp vụ, buộc 26 implementation rỗng:

```csharp
public interface IEWalletProvider
{
    Task<CreateQrCodeOutputDto> CreateQrCodeAsync(...);

    Task<CheckTransactionEwalletOutputDto> CheckTransactionPayedAsync(...);

    Task<CheckTransactionOnlineEwalletOutputDto> CheckTransactionOnlinePayedAsync(...);

    Task<CheckTransactionOnlineEwalletOutputDto> CheckTransactionOnlineAsync(...);
}
```

```csharp
internal sealed class MocaEWalletProvider : IEWalletProvider
{
    // Moca không hỗ trợ tạo QR, buộc phải viết stub nói dối về hợp đồng
    public Task<CreateQrCodeOutputDto> CreateQrCodeAsync(...)
    {
        throw new NotSupportedException();
    }
}
```

Interface béo ở đây tệ **hơn** chuỗi `if/else` ban đầu: chuỗi `if/else` không hứa điều nó không làm được, còn interface béo thì hứa rồi ném exception lúc runtime.

✅ **ĐÚNG** — một interface cho mỗi nghiệp vụ, provider chỉ implement cái nó làm được:

```csharp
public interface ICreateQrCodeStrategy
{
    EnmWalletProvider Provider { get; }

    Task<CreateQrCodeOutputDto> CreateQrCodeAsync(
        CreateQrCodeInputDto input,
        CancellationToken cancellationToken = default);
}
```

Moca không đăng ký `ICreateQrCodeStrategy` thì resolver trả lỗi cấu hình rõ ràng ngay khi có request sai, thay vì để `NotSupportedException` bắn ra từ giữa luồng nghiệp vụ.

#### b. LSP — implementation phải tuân thủ **trọn** hợp đồng

Mọi implementation của một abstraction phải thay thế được cho nhau mà caller không cần biết đang giữ bản nào. **Cấm** các cách lách sau trong implementation:

| Cấm | Vì sao |
| --- | --- |
| `throw new NotSupportedException()` / `NotImplementedException()` | Hứa trong interface rồi từ chối lúc runtime — caller không có cách nào biết trước |
| Trả `null` khi hợp đồng nói trả object | Caller phải thêm null-check cho **mọi** implementation, mất hết giá trị thay thế |
| No-op im lặng (nhận lệnh, không làm gì, báo thành công) | Lỗi trôi xuống tận cuối luồng, không truy được nguồn |
| Siết tiền điều kiện (`require` chặt hơn các implementation khác) | Caller gọi được bản A nhưng vỡ với bản B cùng input |
| Ném exception không khai báo ở `<exception>` của interface | Hợp đồng ghi trong XML doc (Rule 7.1 §2) trở thành sai |

Ba ràng buộc ở Rule 19 §3 vốn viết cho Decorator, nay áp cho **mọi** pattern thay thế (Strategy, State, Adapter, Chain of Responsibility):
1. **Cùng kiểu với abstraction**, không thêm public member nào ngoài interface.
2. **Giữ tham chiếu qua abstraction**, không qua concrete class.
3. **Không kế thừa class hạ tầng/framework** — kế thừa base framework biến implementation thành một service có state và capability riêng, không còn thay thế được.

Provider không hỗ trợ một nghiệp vụ thì **không đăng ký** strategy cho nghiệp vụ đó. Việc "thiếu" được phát hiện ở tầng resolve bằng lỗi cấu hình tường minh, không bị che bằng một stub nói dối.

#### c. Kiểm tra nhanh trước khi coi là xong

- Viết được một test chạy **cùng một bộ assertion** cho mọi implementation của abstraction mà không cần `if` theo kiểu cụ thể → LSP đạt.
- Mọi implementation dùng **hết** các method của interface, không có stub nào → ISP đạt.
- `grep NotSupportedException\|NotImplementedException` trong các class implement abstraction → phải không có kết quả.

### 24.6. Checklist khi áp dụng pattern

- [ ] Đã chỉ ra được **ngưỡng cụ thể** ở 24.1 mà code đang vi phạm
- [ ] Pattern chọn khớp triệu chứng theo bảng 24.2
- [ ] Không trùng lặp pattern đã quy định ở Rule 19 cho luồng đối tác
- [ ] Tên class chứa tên pattern, một class một vai
- [ ] Phụ thuộc qua interface, không qua concrete class
- [ ] Không dùng `static` mutable state hay Service Locator
- [ ] Interface tách theo **nghiệp vụ**, không gom nhiều nghiệp vụ vào một interface béo (24.5a)
- [ ] Mọi implementation dùng hết method của interface, không có stub (24.5a)
- [ ] Không có `NotSupportedException`, `null` hay no-op im lặng trong implementation của abstraction (24.5b)
- [ ] Viết được một bộ assertion dùng chung cho mọi implementation, không cần `if` theo kiểu cụ thể (24.5c)
- [ ] Validate vẫn nằm ở FluentValidation, không chuyển vào Strategy/Decorator (Rule 17)
- [ ] Hành vi không đổi, hoặc đã nêu rõ phần chưa verify được
- [ ] Commit refactor tách riêng khỏi commit feature (Rule 20)
- [ ] `dotnet build` → 0 errors (Rule 21)

