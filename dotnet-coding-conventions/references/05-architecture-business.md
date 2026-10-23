# Phần VI. Kiến Trúc & Xử Lý Nghiệp Vụ (Architecture & Business Logic)

Tài liệu chi tiết về kiến trúc tầng nghiệp vụ, Input Validation với `FluentValidation`, HTTP API client với `IHttpClientFactory`, và mô hình tích hợp API đối tác (Orchestrator + Functional Pipeline + Dynamic Proxy Interceptor + GoF Decorator).

---

## 17. Validation input bắt buộc dùng `FluentValidation`, không xử lý trong Business Service

Mọi logic kiểm tra tính hợp lệ của dữ liệu đầu vào (Input/Request Validation như: kiểm tra null/empty, độ dài chuỗi, khoảng giá trị số, format regex...) **phải** được tách riêng thành các class Validator sử dụng thư viện `FluentValidation`. **Không** viết các đoạn code `if (...) throw ...` kiểm tra input cơ bản rải rác bên trong Business Service/Domain Logic.

### Lợi ích:
- **Single Responsibility Principle (SRP)**: Service chỉ tập trung xử lý luồng nghiệp vụ chính.
- **Tự động hóa**: Tích hợp trực tiếp vào middleware/pipeline validation của ASP.NET Core để trả lỗi `400 Bad Request` trước khi chạm vào Service.
- **Dễ bảo trì & viết test**: Tách biệt unit test cho validation rules và business flow.

### Cấu trúc file Validator
Đặt trong thư mục `Validators/` cùng cấp hoặc theo từng module DTO:
- File đặt tên: `{DtoName}Validator.cs` (Kế thừa `AbstractValidator<{DtoName}>`).

❌ **SAI** — Viết logic validate input thủ công trong Business Service:

```csharp
public class MoMoAIOService : IMoMoAIOService
{
    public async Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
        MoMoAIOPaymentCreationInputDto input,
        CancellationToken cancellationToken = default)
    {
        // ❌ SAI: Validate rải rác trong Service làm loãng nghiệp vụ chính
        if (string.IsNullOrWhiteSpace(input.OrderId))
        {
            throw new CustomBusinessException("OrderId is required");
        }
        if (input.Amount <= 0 || input.Amount > 50_000_000)
        {
            throw new CustomBusinessException("Amount is invalid");
        }

        // Xử lý nghiệp vụ...
    }
}
```

✅ **ĐÚNG** — Dùng FluentValidation riêng biệt:

```csharp
// Validators/MoMoAIOPaymentCreationInputDtoValidator.cs
using FluentValidation;
using YourApp.Dto.MoMoAIO;

namespace YourApp.Validators;

public class MoMoAIOPaymentCreationInputDtoValidator : AbstractValidator<MoMoAIOPaymentCreationInputDto>
{
    public MoMoAIOPaymentCreationInputDtoValidator()
    {
        // Chỉ validate field nghiệp vụ do caller truyền — KHÔNG validate PartnerCode,
        // Signature, RequestId... vì theo Rule 4 các field đó không nằm trong InputDto
        RuleFor(x => x.OrderId)
            .NotEmpty().WithMessage("OrderId không được để trống.");

        RuleFor(x => x.Amount)
            .GreaterThan(0).WithMessage("Số tiền thanh toán phải lớn hơn 0.")
            .LessThanOrEqualTo(50_000_000).WithMessage("Số tiền thanh toán vượt quá hạn mức cho phép.");

        RuleFor(x => x.StoreId)
            .NotEmpty().WithMessage("StoreId không được để trống.");
    }
}
```

```csharp
// Service chỉ tập trung xử lý nghiệp vụ sạch sẽ:
public class MoMoAIOService : IMoMoAIOService
{
    public async Task<MoMoAIOPaymentCreationOutputDto> CreatePaymentAsync(
        MoMoAIOPaymentCreationInputDto input,
        CancellationToken cancellationToken = default)
    {
        // Service hoàn toàn tin tưởng input đã được validate hợp lệ ở tầng ngoài
        // Map InputDto (từ caller) → RequestDto (gửi đối tác), rồi inject config/signature
        var partnerPayload = _mapper.Map<MoMoAIOPaymentCreationRequestDto>(input);
        return await _momoApiClient.SendPaymentAsync(partnerPayload, cancellationToken);
    }
}
```

### Quy tắc:
- Mọi Request/Input DTO có quy tắc ràng buộc dữ liệu phải có validator kế thừa `AbstractValidator<T>`.
- Đăng ký tự động qua DI trong `Startup.cs` / `Program.cs`:
  `services.AddValidatorsFromAssemblyContaining<Program>();`
- Trong Business Service chỉ kiểm tra các ràng buộc nghiệp vụ phụ thuộc state/database (ví dụ: *kiểm tra đơn hàng đã thanh toán trước đó chưa, kiểm tra số dư trong DB*), không kiểm tra format hay null/empty của request payload.

---

## 18. Gọi API HTTP bắt buộc chỉ sử dụng `IHttpClientFactory`, không dùng RestSharp hoặc thư viện bên thứ 3

Khi viết function hoặc service có phát sinh lời gọi HTTP API (gọi sang dịch vụ nội bộ hoặc cổng thanh toán / đối tác bên ngoài), **bắt buộc chỉ được sử dụng `IHttpClientFactory`**. **Tuyệt đối không** tự ý thêm các thư viện HTTP client bên thứ 3 như `RestSharp`, `Flurl`, `Refit`, `ServiceStack.HttpClient`... và **không** tự ý `new HttpClient()` thủ công trong code mới.

### Lợi ích:
- **Tối ưu vòng đời kết nối**: `IHttpClientFactory` tự động quản lý vòng đời của `HttpMessageHandler` bên dưới, tránh cạn kiệt socket (`socket exhaustion`) và cập nhật thay đổi DNS kịp thời.
- **Giảm dependency ngoài**: Tận dụng triệt để nền tảng chuẩn của Microsoft .NET, giảm footprint và rủi ro bảo mật từ bên thứ ba.
- **Dễ tích hợp & Cấu hình tập trung**: Dễ dàng tích hợp với `Polly` (resilience, retry, circuit breaker), Interceptor (Logging, Token injection) và Named/Typed Client.

### Quy cách áp dụng:

1. **Đăng ký Named Client hoặc Typed Client** trong `Program.cs` / `Startup.cs` hoặc trong các file cấu hình (module/extension methods) của project đang có.
2. **Inject `IHttpClientFactory`** vào constructor của Service và gọi `_httpClientFactory.CreateClient("PartnerName")` khi cần gửi request.

❌ **SAI** — Dùng RestSharp hoặc tạo mới `new HttpClient()` / `new RestClient()` thủ công:

```csharp
// ❌ SAI: Thêm dependency RestSharp hoặc new client thủ công
using RestSharp;

public class PartnerApiClient
{
    public async Task<PartnerResponseDto> SendAsync(PartnerRequestDto request)
    {
        var client = new RestClient("https://api.partner.com");
        var restRequest = new RestRequest("v1/payments", Method.Post);
        restRequest.AddJsonBody(request);
        var response = await client.ExecuteAsync<PartnerResponseDto>(restRequest);
        return response.Data;
    }
}
```

✅ **ĐÚNG** — Inject và sử dụng `IHttpClientFactory` (kết hợp `System.Text.Json`):

```csharp
// Đăng ký Named Client trong Program.cs:
// services.AddHttpClient("PartnerRemote", (sp, client) =>
// {
//     var options = sp.GetRequiredService<IOptions<PartnerConnect>>().Value;
//     client.BaseAddress = new Uri(options.Url);
//     client.Timeout = TimeSpan.FromMilliseconds(options.TimedOut);
// });

public class PartnerApiClient : IPartnerApiClient
{
    private readonly IHttpClientFactory _httpClientFactory;
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    public PartnerApiClient(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task<PartnerResponseDto> SendAsync(
        PartnerRequestDto request,
        CancellationToken cancellationToken = default)
    {
        var httpClient = _httpClientFactory.CreateClient("PartnerRemote");

        using StringContent content = new(
            JsonSerializer.Serialize(request, JsonOptions),
            Encoding.UTF8,
            "application/json");

        using HttpResponseMessage response = await httpClient.PostAsync("v1/payments", content, cancellationToken);
        response.EnsureSuccessStatusCode();

        string responseContent = await response.Content.ReadAsStringAsync(cancellationToken);

        // Body rỗng hoặc "null" vẫn deserialize thành null, phải chặn thay vì dùng toán tử ! (Rule 27)
        // An empty or "null" body still deserializes to null, so guard it instead of using the ! operator (Rule 27)
        return JsonSerializer.Deserialize<PartnerResponseDto>(responseContent, JsonOptions)
            ?? throw new PartnerIntegrationException(PartnerErrorCodes.EmptyResponse);
    }
}
```

Quy tắc:
- Chỉ inject và sử dụng `IHttpClientFactory`. Không tạo `new HttpClient()` thủ công.
- Không cài đặt thêm các package HTTP thứ 3 (`RestSharp`, `Flurl`, v.v.) vào project.
- Luôn truyền `cancellationToken` vào các hàm gửi HTTP (`GetAsync`, `PostAsync`, `SendAsync`, `ReadAsStringAsync`...).
- Kết hợp với `System.Text.Json` để serialize/deserialize payload.

---

## 19. Kiến trúc và Luồng Xử lý Giao dịch Tích hợp API Đối tác

### 1. Tổng quan Kiến trúc và Các Pattern Áp Dụng (Architectural & Design Patterns)

Hệ thống tích hợp api đối tác được xây dựng trên sự phối hợp chặt chẽ của các mẫu thiết kế hướng phân tách trách nhiệm (Separation of Concerns), tính linh hoạt và khả năng mở rộng không sửa đổi mã nguồn gốc (Open/Closed Principle):

1. **Orchestrator Pattern (Application Service Layer)**:
   * Lớp dịch vụ ứng dụng bề mặt (`PartnerService`) đóng vai trò là Nhạc trưởng (Orchestrator).
   * Lớp này chỉ chịu trách nhiệm: kiểm tra hợp lệ dữ liệu đầu vào mức cơ bản, điều phối các bước trong chu trình giao dịch qua pipeline, và ánh xạ dữ liệu kết quả trả về cho caller.
   * Lớp này hoàn toàn cô lập với các chi tiết kỹ thuật của đối tác như giao thức HTTP, định dạng JSON payload, giải thuật ký số, hay cơ chế xác thực.

2. **Template Method / Functional Pipeline Pattern**:
   * Toàn bộ chu trình tương tác với đối tác cho mọi loại giao dịch (Khởi tạo, Tra cứu, Hủy) được chuẩn hóa qua một đường ống hàm chung (`ExecutePipelineAsync`).
   * Pipeline này thực thi tuần tự 3 giai đoạn độc lập được truyền vào dưới dạng delegates hoặc functions:
     * Bước thứ nhất: Xây dựng ngữ cảnh yêu cầu (Build Request Context).
     * Bước thứ hai: Gửi yêu cầu qua kênh thực thi có đánh chặn/xác thực (Execute Request / Authorized).
     * Bước thứ ba: Tiếp nhận, kiểm tra và giải mã phản hồi (Handle & Parse Response).

3. **Aspect-Oriented Programming (AOP) qua Bộ đôi Dynamic Proxy Interceptor & Interceptor Registrar**:
   * **Nguyên tắc phân tách Cross-Cutting Concerns**: Tầng xử lý nghiệp vụ (`PartnerBusinessService`) không trực tiếp nhúng cứng các logic phụ trợ (như lấy OAuth2 token, đính kèm token, log payload, retry, hoặc ký request động) vào thân hàm gửi HTTP.
   * **Cơ chế Triển khai Linh hoạt theo Nền tảng**:
     * **Đối với .NET thuần (Microsoft Dependency Injection kết hợp Castle DynamicProxy)**:
       * *Cơ chế Đăng ký*: Sử dụng `Castle.DynamicProxy.ProxyGenerator` khi đăng ký dịch vụ vào `IServiceCollection`. Ta có thể viết một hàm mở rộng (Extension Method) tự động quét qua các phương thức của Interface/Class. Nếu phương thức có gắn Custom Attribute định danh (ví dụ `[PartnerAuthorized]`), hệ thống sẽ khởi tạo một Dynamic Proxy bọc lấy đối tượng Business Service thật và đính kèm Interceptor tương ứng (`IAsyncInterceptor`).
       * *Cơ chế Đánh chặn*: Interceptor kế thừa `IAsyncInterceptor` chặn lời gọi bất đồng bộ, trích xuất đối tượng ngữ cảnh yêu cầu, nạp token/credentials vào header rồi mới cho phép hàm gốc chạy tiếp (`invocation.ProceedAsync()`).
     * **Đối với ABP Framework**:
       * Sử dụng cặp lớp kế thừa `AbpInterceptor` và lớp đăng ký tĩnh `PartnerAuthorizationInterceptorRegistrar`.
       * Registrar móc vào sự kiện `IOnServiceRegistredContext` của ABP khi ứng dụng khởi động. Nếu kiểu triển khai (`implementationType`) có phương thức gắn Custom Attribute (ví dụ `[PartnerAuthorized]`), Registrar sẽ đăng ký Interceptor vào Service Descriptor (`context.Interceptors.TryAdd<PartnerAuthorizationInterceptor>()`).
   * **Quy tắc điều hướng Interceptor tại Runtime**:
     * **Nếu đối tác dùng OAuth2 / Bearer Token**: Interceptor chặn lời gọi trước khi thực thi, trích xuất đối tượng ngữ cảnh yêu cầu (`PartnerApiRequestContext`), gọi dịch vụ token để lấy hoặc tái sử dụng token hợp lệ và tự động tiêm tiêu đề `Authorization: Bearer <token>` vào request context trước khi chuyển tiếp.
     * **Nếu đối tác dùng kỹ thuật xác thực khác (API Key, Custom Header, HMAC Header, mTLS)**: Tạo một Interceptor riêng cho loại xác thực đó để tự động bổ sung thông tin vào request context mà không sửa code nghiệp vụ.
     * **Nếu đối tác không yêu cầu xác thực**: Phương thức không gắn Attribute, hệ thống đăng ký sẽ bỏ qua Interceptor, lời gọi sẽ đi thẳng tới HTTP Client thực thi.

4. **Named HttpClient Registration & Timeout Configuration (Quản lý HttpClient theo Định danh Đối tác)**:
   * **Mỗi đối tác một mã định danh duy nhất (Partner Identifier)**: Định nghĩa hằng số định danh (ví dụ: `EnvironmentSetting.{Partner}Remote` như `TPBankRemote = "TPBank"`, `ZalopayOnlineConnect = "ZALOPAY_ONLINE"`). Mã này dùng làm key để cấu hình `RemoteServices` hoặc đăng ký Named `HttpClient`.
   * **Cấu hình Timeout riêng biệt**: Mỗi đối tác phải có cấu hình Timeout riêng (qua `appsettings.json` và Options pattern, ví dụ `{Partner}Connect.TimedOut`), được gắn trực tiếp vào client tương ứng khi khởi tạo hoặc khi gửi request.
   * **Cơ chế Triển khai**:
     * **Đối với .NET thuần**: Sử dụng `IHttpClientFactory` với Named Client (`services.AddHttpClient(EnvironmentSetting.{Partner}Remote, client => { client.BaseAddress = new Uri(options.Url); client.Timeout = TimeSpan.FromMilliseconds(options.TimedOut); })`).
     * **Đối với ABP Framework**: Cấu hình trong `RemoteServiceOptions` và thông qua `IApiService` / `PaymentGatewayAppService` truyền tham số `remoteRoot: EnvironmentSetting.{Partner}Remote` để tự động map đúng BaseUrl và Timeout của đối tác.

5. **Data Transformation & DTO Isolation Pattern**:
   * Cách ly hoàn toàn cấu trúc DTO nội bộ theo schema của nhà cung cấp (`PartnerResponseDto`) với cấu trúc DTO đầu ra phục vụ cho hệ thống nội bộ hoặc client (`PartnerResultDto`) thông qua công cụ ánh xạ đối tượng (`AutoMapper`, `Mapster`, hoặc `ObjectMapper`).

6. **Decorator Pattern (GoF) cho các quan tâm bổ trợ ở tầng nghiệp vụ**:
   * Mọi quan tâm **có thể thêm/bớt mà không đổi nghiệp vụ gốc** — tiền xử lý/chuẩn hoá input trước khi gửi đối tác, caching response qua Redis, lọc/phân trang, phân quyền dữ liệu, logging nghiệp vụ — phải được bọc thành từng lớp Decorator quanh `IPartnerService`, **không** nhồi vào `PartnerService`.
   * Phân định với AOP Interceptor ở mục 3: Decorator áp ở tầng **nghiệp vụ** (`IPartnerService`, thấy tham số có kiểu rõ ràng); Interceptor áp ở tầng **gửi HTTP** (`IPartnerBusinessService`, chỉ chạm headers/context).

---

### 2. Quy Trình Xử Lý Chi Tiết Của Từng Nghiệp Vụ Giao Dịch

Bất kể đối tác yêu cầu cơ chế xác thực nào (OAuth2, Custom Header hay No-Auth) và chạy trên nền tảng .NET thuần hay Framework, chu trình xử lý của 3 nghiệp vụ Khởi tạo (Init), Tra cứu (Enquiry/Get) và Hủy (Cancel) đều tuân thủ 5 bước chuẩn hóa:

#### Bước 1: Tiếp nhận, Kiểm tra và Chuẩn hóa Dữ liệu Đầu vào (Validation & Defaulting)
* **Khởi tạo (Init)**: Nhận DTO thông tin yêu cầu (`PartnerInitialInputDto`), kiểm tra null. Nếu loại giao dịch (`TransactionType`) rỗng, tự động điền giá trị mặc định theo quy ước nghiệp vụ (ví dụ: `"PURCHASE"`).
* **Tra cứu (Enquiry / Get)**: Nhận DTO tra cứu (`PartnerEnquiryInputDto`), kiểm tra null đối với tham số hoặc mã yêu cầu.
* **Hủy (Cancel)**: Nhận trực tiếp chuỗi định danh mã yêu cầu (`paymentRequestCode` / `requestCode`).

#### Bước 2: Xây dựng Ngữ cảnh Yêu cầu & Ký số (Build Request Context & Signing)
Tầng Orchestrator gọi tầng Business Service để khởi tạo đối tượng ngữ cảnh yêu cầu (`PartnerApiRequestContext`):
* **Tuần tự hóa JSON bất biến (Deterministic Serialization)**: Payload được serialize sang chuỗi JSON theo các quy tắc nghiêm ngặt của đối tác (ví dụ: sắp xếp thuộc tính theo bảng chữ cái alphabet, không thụt dòng, không escape Unicode nếu cần đối soát chuỗi ký).
* **Tạo Chữ ký số (HMAC / RSA Signature)**: Nếu đối tác yêu cầu chữ ký số trên từng request, tầng Business sử dụng cặp khóa hoặc Secret kết hợp với JSON payload để tính toán chữ ký và đính vào Header hoặc Body (ví dụ: `X-Signature`, `X-Api-Key`).
* **Đóng gói Ngữ cảnh**: Thiết lập endpoint URL, HTTP Method, Content-Type, JSON body và các custom headers cần thiết vào đối tượng `PartnerApiRequestContext`.

#### Bước 3: Đánh chặn và Xử lý qua Dynamic Proxy Pipeline (AOP Interception Execution)
* Khi Orchestrator kích hoạt hàm thực thi request (`ExecuteAuthorizedAsync` hoặc `ExecuteRequestAsync`), Dynamic Proxy (đã được bọc từ lúc khởi động DI) sẽ kích hoạt Interceptor:
  * **Nếu có Interceptor xác thực (OAuth2 / Custom Auth)**: Interceptor lấy đối tượng `PartnerApiRequestContext`, kiểm tra nếu chưa có thông tin xác thực thì tự động nạp token/credentials vào headers của context rồi mới cho phép tiếp tục (`ProceedAsync`).
  * **Nếu có các Interceptor mở rộng khác (Audit Log, Metrics, Signing Interceptor, Retry)**: Lời gọi lần lượt đi qua chuỗi Interceptor theo đúng thứ tự (Chain of Responsibility).
  * **Nếu không có Interceptor**: Lời gọi tiến thẳng vào thực thi HTTP.
* HTTP Client (`HttpClient` hoặc `IApiService`) sử dụng Named Client theo mã định danh `EnvironmentSetting.{Partner}Remote` (kèm Timeout cấu hình riêng) nhận `PartnerApiRequestContext` đã được hoàn thiện đầy đủ dữ liệu và headers để gửi HTTP Request thực tế sang cổng tích hợp api đối tác.

#### Bước 4: Kiểm tra và Phân tích Phản hồi (Handle Response & Verification)
* Tầng Business tiếp nhận phản hồi HTTP (`HttpResponseMessage`) từ cổng tích hợp api đối tác:
* **Kiểm tra trạng thái HTTP & Lỗi nghiệp vụ**: Đánh giá StatusCode và mã phản hồi từ phía đối tác. Nếu kết quả thất bại, hệ thống ném ra ngoại lệ nghiệp vụ (`BusinessException` hoặc `CustomException`) với ErrorCode cụ thể cho từng nghiệp vụ (như `ERROR_PARTNER_INITIAL_FAILED`, `ERROR_PARTNER_ENQUIRY_FAILED`, `ERROR_PARTNER_CANCEL_FAILED`).
* **Xác thực chữ ký phản hồi (Response Signature Verification)**: Nếu đối tác trả về chữ ký xác thực, tầng Business tiến hành băm và kiểm tra lại dữ liệu phản hồi để chống giả mạo trước khi tin tưởng dữ liệu.
* **Deserialize DTO**: Giải mã chuỗi JSON phản hồi thành DTO nội bộ tương ứng (`PartnerInitialResponseDto`, `PartnerEnquiryResponseDto`, `PartnerCancelResponseDto`).

#### Bước 5: Ánh xạ và Trả về Kết quả (Result Mapping)
* Orchestrator nhận đối tượng DTO phản hồi từ tầng Business.
* Sử dụng công cụ ánh xạ (`AutoMapper`, `Mapster`, hoặc `ObjectMapper`) để chuyển đổi đối tượng phản hồi từ đối tác sang DTO kết quả chuẩn hóa (`PartnerInitialResultDto`, `PartnerEnquiryResultDto`, `PartnerCancelResultDto`) và trả về cho tầng Controller hoặc caller.

---

### 3. Decorator Pattern — Tiền Xử Lý Input & Caching Response

#### 3.1. Bốn vai theo GoF

| Vai GoF | Trong kiến trúc này |
| --- | --- |
| **Component** | `IPartnerService` — hợp đồng nghiệp vụ công khai |
| **ConcreteComponent** | `PartnerService` — Orchestrator gọi đối tác thật |
| **Decorator** (abstract) | `PartnerServiceDecoratorBase` — giữ tham chiếu Component, delegate toàn bộ |
| **ConcreteDecorator** | `PartnerServiceRequestShapingDecorator`, `PartnerServiceCacheDecorator`... |

Năm tính chất **bắt buộc** của một Decorator:
1. **Cùng kiểu với Component**: implement đúng `IPartnerService`, **không** thêm public member nào ngoài interface. Caller không phân biệt được đang giữ bản gốc hay bản đã bọc.
2. **Giữ tham chiếu qua abstraction**: field là `IPartnerService`, **không phải** `PartnerService`. Nhờ vậy decorator bọc được decorator khác → mới compose được chuỗi.
3. **Có Decorator base trừu tượng delegate toàn bộ**: ConcreteDecorator chỉ `override` method nó quan tâm. Thêm method mới vào interface → sửa **một** chỗ là base.
4. **Một decorator một quan tâm duy nhất**, tên class phải nói rõ quan tâm đó.
5. **KHÔNG kế thừa class hạ tầng/framework** (`AppServiceBase`, `PaymentIntegrationAppService`...). Kế thừa base của framework biến decorator thành một service thật có state và capability riêng — không còn là wrapper trong suốt.

```csharp
public abstract class PartnerServiceDecoratorBase : IPartnerService
{
    protected readonly IPartnerService Inner;

    protected PartnerServiceDecoratorBase(IPartnerService inner)
    {
        Inner = inner ?? throw new ArgumentNullException(nameof(inner));
    }

    /// <inheritdoc />
    public virtual Task<PartnerInitialResultDto> InitialAsync(
        PartnerInitialInputDto input,
        CancellationToken cancellationToken = default)
        => Inner.InitialAsync(input, cancellationToken);

    // ... delegate mọi method còn lại của IPartnerService
}
```

#### 3.2. Hai loại decorator hay dùng nhất

**Tiền xử lý input** (chạy *trước* khi gọi đối tác) — chuẩn hoá dữ liệu (trim, upper mã, chuẩn hoá số điện thoại), làm giàu tham số từ config (map mã shop nội bộ → mã đối tác), điền default khi caller bỏ trống. Biến đổi `input` rồi mới gọi `base.XxxAsync(input, ct)`.

**KHÔNG** validate ở decorator này — null/empty/range/regex thuộc FluentValidation (Rule 17). Decorator chỉ **biến đổi**, không từ chối dữ liệu.

**Caching response** (chạy *sau* khi có kết quả) — cache-aside: đọc cache → hit trả ngay, miss thì `base.XxxAsync(...)` rồi ghi cache.

Bốn nguyên tắc bắt buộc cho cache decorator:
- Phụ thuộc **abstraction** `IDistributedCache`, **không** inject trực tiếp `StackExchange.Redis.IDatabase`. Redis là chi tiết hạ tầng cấu hình ở DI. Vì vậy tên class là `...CacheDecorator`, **không phải** `...RedisDecorator`.
- **TTL lấy từ Options** (`{Partner}Connect.XxxCacheExpirySeconds`), **không** hardcode `TimeSpan.FromDays(7)` tại call site.
- **Lỗi cache không được làm fail nghiệp vụ**: bắt lỗi, log `Warning`, fallback gọi đối tác. Áp dụng cho cả lỗi kết nối Redis và `JsonException` khi payload cache sai định dạng.
- **Chỉ ghi cache khi payload hợp lệ / không rỗng** — để một lần gọi đối tác thất bại không đầu độc cache.
- **Cache key khai báo tập trung** trong `{Partner}CacheKeys`, không nối chuỗi rải rác. Quy ước key: `<hệ thống>:<module>:<đối tác>:<nghiệp vụ>[:<tham số>]`, toàn bộ lowercase, phân cách bằng `:`.

**Chỉ cache nghiệp vụ đọc.** Nghiệp vụ thay đổi trạng thái (Init, Cancel) **không** cache — không override method đó, để Decorator base tự delegate.

#### 3.3. Thứ tự bọc

Bọc từ trong ra ngoài. **Lớp bọc sau cùng là lớp ngoài cùng, chạy đầu tiên** với request và **cuối cùng** với response.

```
Caller
  └─▶ VisibilityDecorator        (ngoài cùng — cắt dữ liệu theo quyền)
       └─▶ QueryDecorator        (filter / paging theo tham số request)
            └─▶ CacheDecorator   (cache-aside)
                 └─▶ RequestShapingDecorator  (chuẩn hoá + làm giàu input)
                      └─▶ PartnerService       (gọi đối tác thật)
```

Hai ràng buộc kiến trúc:
- **Decorator biến đổi input phải nằm TRONG cache.** Nếu không, cùng một yêu cầu logic (`"shop01"` và `" SHOP01 "`) sẽ sinh ra nhiều cache key khác nhau → cache gần như luôn miss.
- **Decorator filter/paging/phân quyền phải nằm NGOÀI cache.** Để Redis luôn lưu **payload đầy đủ** của đối tác, không phải một mẩu đã cắt theo một request cụ thể. Đặt cache ngoài filter thì user A paging trang 1 sẽ làm user B nhận đúng trang đó.

**Bắt buộc ghi ràng buộc thứ tự vào XML comment của từng decorator** (`<para>Phải nằm trong/ngoài ...</para>`).

#### 3.4. Đăng ký DI — factory tường minh

```csharp
// Core impl đăng ký theo KIỂU CONCRETE để factory resolve được nó
context.Services.AddTransient<PartnerService>();
context.Services.AddDecorators();
```

```csharp
public static class Decorator
{
    public static void AddDecorators(this IServiceCollection services)
    {
        services.AddScoped<IPartnerService>(provider =>
        {
            // Base: Orchestrator gọi API đối tác thật
            IPartnerService pipeline = provider.GetRequiredService<PartnerService>();

            // Chuẩn hoá + làm giàu input — phải nằm TRONG cache
            pipeline = new PartnerServiceRequestShapingDecorator(
                pipeline,
                provider.GetRequiredService<ILogger<PartnerServiceRequestShapingDecorator>>(),
                provider.GetRequiredService<IOptions<PartnerConnect>>());

            // Cache-aside — phải nằm NGOÀI shaping, TRONG filter/paging/phân quyền
            pipeline = new PartnerServiceCacheDecorator(
                pipeline,
                provider.GetRequiredService<IDistributedCache>(),
                provider.GetRequiredService<ILogger<PartnerServiceCacheDecorator>>(),
                provider.GetRequiredService<IOptions<PartnerConnect>>());

            return pipeline;
        });
    }
}
```

Năm quy tắc của cách đăng ký này:
1. **Core impl đăng ký theo kiểu concrete** (`AddTransient<PartnerService>()`). **KHÔNG** đăng ký `AddScoped<IPartnerService, PartnerService>()`.
2. **Interface map vào factory**, factory trả về lớp ngoài cùng của chuỗi.
3. **Mỗi dòng `new ...Decorator(...)` có comment** nói rõ quan tâm của nó và ràng buộc thứ tự.
4. **Gán lại cùng một biến `pipeline`** theo thứ tự trong → ngoài.
5. **Lifetime thống nhất** cho cả chuỗi.

#### 3.5. Hai interface, hai tầng — vị trí và modifier

| | `IPartnerService` | `IPartnerBusinessService` |
| --- | --- | --- |
| Tầng | Nghiệp vụ | Kỹ thuật đối tác |
| Assembly | `*.Application.Contracts` | `*.Application` |
| Modifier | `public` | `internal` |
| Ai gọi | Controller, service khác, `HttpApi.Client` | Chỉ `PartnerService` |
| Kiểu dữ liệu | `InputDto` → `ResultDto` | `PartnerApiRequestContext`, `HttpResponseMessage`, `ResponseDto` |
| Bị bọc bởi | **Decorator** (cache, shaping) | **Interceptor** (auth) |

Hệ quả bắt buộc khi `IPartnerBusinessService` là `internal`:
- `PartnerBusinessService` **và** `PartnerService` cũng phải `internal`.
- Phải thêm `[assembly: InternalsVisibleTo("DynamicProxyGenAssembly2")]`.

#### 3.6. Decorator hay Interceptor?

| | Decorator | Interceptor (AOP) |
| --- | --- | --- |
| Tầng áp dụng | `IPartnerService` (nghiệp vụ) | `IPartnerBusinessService` (gửi HTTP) |
| Thấy được gì | Tham số + kiểu trả về **có kiểu rõ ràng** | `object[] Arguments` — phải cast |
| Dùng cho | Biến đổi input, caching, filter, phân quyền dữ liệu | Auth token, ký request, audit log, retry hạ tầng |
| Compile-time safety | Có | Không |
| Chi phí | Không reflection | Dynamic proxy + reflection |

#### 3.7. Mã lỗi và mã kết quả là hai thứ khác nhau

- `{Partner}ResultCode` — mã **do đối tác trả về** trong payload (`resultCode`). Dùng `const string`.
- `{Partner}ErrorCode` — mã **hệ thống mình ném ra** cho caller. **Mỗi nghiệp vụ một mã riêng** (`ERROR_PARTNER_INITIAL_FAILED`, `ERROR_PARTNER_ENQUIRY_FAILED`, `ERROR_PARTNER_CANCEL_FAILED`).
- `HTTP 200` **không** có nghĩa là giao dịch thành công. Luôn xét thêm `ResultCode` trước khi tin dữ liệu.

