# Phần XII. Bẫy Bất Đồng Bộ & Vòng Đời Dependency Injection (Async Pitfalls & DI Lifetime)

Bổ sung cho Rule 13–14 (hậu tố `Async`, `CancellationToken`): các lỗi async gây deadlock, cạn thread pool, mất exception; và quy tắc vòng đời DI để tránh captive dependency, chia sẻ `DbContext` giữa nhiều thread.

---

## 28. Async đúng cách: không chặn, không bỏ rơi Task, không chạy song song trên tài nguyên không thread-safe

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Thread pool không bị chặn nên service chịu tải cao hơn với cùng số pod. Exception không bị mất. Huỷ request dừng được công việc phía sau. |
| **Rủi ro nếu vi phạm** | `.Result` / `.Wait()` gây **thread pool starvation**: latency tăng vọt dưới tải dù CPU thấp, và deadlock trong môi trường có `SynchronizationContext`. `async void` làm crash cả process khi có exception. Fire-and-forget làm mất exception, và task bị giết khi pod restart. |

### 28.1. Quy tắc

1. **Async từ đầu tới cuối (async all the way).** Cấm sync-over-async: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`, `Task.WaitAll`. Ngoại lệ duy nhất: entry point không hỗ trợ async (rất hiếm từ C# 7.1 `async Main`), và phải có justification (Rule 42).
2. **Cấm `async void`**, trừ event handler của UI/Blazor. Method async luôn trả `Task` / `Task<T>` / `ValueTask`.
3. **Cấm fire-and-forget** (`_ = DoWorkAsync();`, `Task.Run(...)` không `await`) trong request. Công việc nền đẩy vào `Channel<T>` + `BackgroundService`, hoặc message queue / Hangfire / ABP `IBackgroundJobManager`. Job nền tự tạo scope DI riêng (Rule 29).
4. **Không bọc code đồng bộ bằng `Task.Run`** trong ASP.NET Core để "giả async", vì chỉ chuyển việc sang thread khác mà không giải phóng gì. `Task.Run` chỉ dùng cho việc CPU-bound nặng ở ứng dụng client/worker.
5. **Chạy song song tác vụ độc lập** bằng `Task.WhenAll`, nhưng **không bao giờ** trên cùng một `DbContext` hay tài nguyên không thread-safe. `DbContext` chỉ cho phép **một** thao tác tại một thời điểm.
6. **`ConfigureAwait(false)`** bắt buộc trong **library / shared package** (code có thể chạy dưới `SynchronizationContext` của caller). Trong ASP.NET Core application code thì không bắt buộc (không có `SynchronizationContext`).
7. **`ValueTask`** chỉ dùng khi đo được lợi ích (hot path thường hoàn thành đồng bộ, ví dụ cache hit). Một `ValueTask` **chỉ được await đúng một lần**, không lưu lại để await lần hai.
8. **`await using`** cho đối tượng `IAsyncDisposable` (`DbContext`, `IDbContextTransaction`, stream).
9. **Không `return` Task trực tiếp (elide `async`) khi có `using` hoặc `try`** bao quanh: object bị dispose trước khi task hoàn thành.
10. **Timeout** dùng `CancellationTokenSource.CreateLinkedTokenSource(cancellationToken)` + `CancelAfter(...)` hoặc `Task.WaitAsync(timeout, cancellationToken)` (.NET 6), không dùng `Task.Delay` chạy đua thủ công.

❌ **SAI**:

```csharp
public OrderOutputDto GetOrder(string orderCode)
{
    return GetOrderAsync(orderCode).Result;     // ❌ Sync-over-async: chặn thread pool
}

public async void SendNotification(Order order) // ❌ async void: exception làm crash process
{
    await _notificationClient.SendAsync(order);
}

public async Task<OrderOutputDto> CreateOrderAsync(OrderCreationInputDto input, CancellationToken cancellationToken = default)
{
    // ...
    _ = _notificationClient.SendAsync(order);   // ❌ Fire-and-forget: mất exception, mất task khi pod restart

    Task<Customer> customerTask = _dbContext.Customers.SingleAsync(x => x.Id == input.CustomerId, cancellationToken);
    Task<Shop> shopTask = _dbContext.Shops.SingleAsync(x => x.Code == input.ShopCode, cancellationToken);
    await Task.WhenAll(customerTask, shopTask); // ❌ Hai query song song trên cùng DbContext → InvalidOperationException
}
```

✅ **ĐÚNG**:

```csharp
public async Task<OrderOutputDto> CreateOrderAsync(OrderCreationInputDto input, CancellationToken cancellationToken = default)
{
    // Hai lời gọi HTTP độc lập, không chung tài nguyên, nên chạy song song để giảm latency
    // Two independent HTTP calls with no shared resource, so run them concurrently to cut latency
    Task<CustomerProfileDto> profileTask = _customerClient.GetProfileAsync(input.CustomerId, cancellationToken);
    Task<PromotionDto> promotionTask = _promotionClient.GetActiveAsync(input.ShopCode, cancellationToken);
    await Task.WhenAll(profileTask, promotionTask);

    // ... ghi đơn hàng trong một Unit of Work (Rule 25)

    // Thông báo đi qua queue nền thay vì fire-and-forget, để không mất khi lỗi hoặc pod restart
    // Notifications go through a background queue instead of fire-and-forget so they survive failures and restarts
    await _notificationQueue.EnqueueAsync(new OrderNotificationDto(order.Id), cancellationToken);

    return _mapper.Map<OrderOutputDto>(order);
}
```

✅ **ĐÚNG** — Timeout liên kết với token của request:

```csharp
using CancellationTokenSource timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
timeoutCts.CancelAfter(_options.Value.EnquiryTimeout);

PartnerEnquiryResponseDto response = await _partnerClient.EnquiryAsync(request, timeoutCts.Token);
```

### 28.2. Anti-patterns (KHÔNG làm)

- `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` trong code chạy dưới request.
- `async void` ngoài event handler.
- `Task.Run` trong controller/service ASP.NET Core để giả async.
- `Task.WhenAll` / `Parallel.ForEachAsync` trên cùng một `DbContext`.
- Await một `ValueTask` hai lần, hoặc `.Result` trên `ValueTask` chưa hoàn thành.
- `Thread.Sleep` trong code async (dùng `await Task.Delay(..., cancellationToken)`).
- Bỏ qua `cancellationToken` khi gọi xuống tầng dưới (Rule 14).

---

## 29. Vòng đời Dependency Injection phải đúng: không captive dependency, không service locator

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Mỗi request có `DbContext` và state riêng, không rò dữ liệu giữa các user. Lỗi cấu hình DI bị phát hiện lúc khởi động thay vì lúc chạy production. Phụ thuộc hiện rõ trên constructor nên dễ test. |
| **Rủi ro nếu vi phạm** | **Captive dependency** (Singleton giữ Scoped) làm `DbContext` bị dùng chung giữa các request: lỗi "A second operation was started on this context", dữ liệu tracking cũ, và có thể **lộ dữ liệu user A cho user B**. Service locator che giấu phụ thuộc nên lỗi chỉ lộ ra ở runtime. |

### 29.1. Bảng chọn lifetime

| Lifetime | Dùng cho | Không dùng cho |
| --- | --- | --- |
| **Singleton** | Stateless thuần, cache in-memory, `IHttpClientFactory`, `JsonSerializerOptions`, `TimeProvider`, `IOptionsMonitor<T>` | Bất cứ thứ gì giữ `DbContext`, user context, hoặc state theo request |
| **Scoped** | `DbContext`, repository, Unit of Work, application service, `ICurrentUser`, `IOptionsSnapshot<T>` | Inject vào Singleton |
| **Transient** | Object nhẹ, không state, tạo mới mỗi lần resolve | Object implement `IDisposable` được resolve từ root (rò bộ nhớ đến khi app tắt) |

**Quy tắc phụ thuộc**: service chỉ được phụ thuộc vào service có lifetime **dài bằng hoặc dài hơn** nó. Singleton → chỉ Singleton. Scoped → Scoped / Singleton. Transient → mọi loại (nhưng hấp thụ lifetime của service chứa nó).

### 29.2. Quy tắc

1. **Bật kiểm tra DI** khi khởi động. `WebApplicationBuilder` đã bật mặc định ở môi trường Development; **không được tắt**, và nên bật `ValidateOnBuild` cho mọi môi trường để fail sớm:

   ```csharp
   builder.Host.UseDefaultServiceProvider(options =>
   {
       options.ValidateScopes = builder.Environment.IsDevelopment();
       options.ValidateOnBuild = true;
   });
   ```

2. **Constructor injection** là mặc định. **Cấm** inject `IServiceProvider` vào business service để `GetService<T>()` (service locator). Ngoại lệ: factory / composition root (đăng ký decorator ở Rule 19 §3.4, Strategy resolver theo key).
3. **Singleton hoặc `BackgroundService` cần dịch vụ Scoped** thì inject `IServiceScopeFactory` và tạo scope cho **mỗi đơn vị công việc** (mỗi message, mỗi vòng lặp), không tạo một scope dùng mãi.
4. **Cần `DbContext` ngoài request** (job, consumer, xử lý song song): dùng `IDbContextFactory<TContext>` (`AddDbContextFactory`), mỗi luồng một instance, dispose ngay sau khi dùng.
5. **Typed HttpClient không inject vào Singleton**: typed client là Transient, bị giữ trong Singleton thì handler không bao giờ được xoay vòng (mất lợi ích DNS refresh của Rule 18). Singleton dùng `IHttpClientFactory.CreateClient(name)` mỗi lần gọi.
6. **Không tự `new` service có phụ thuộc**, trừ trong factory đăng ký DI.
7. **Keyed services** (.NET 8 `AddKeyedScoped<TService, TImpl>(key)` + `[FromKeyedServices(key)]`) là cách ưu tiên để resolve Strategy theo key (Rule 24), thay cho `IEnumerable<T>` + `FirstOrDefault`.
8. **ABP Framework**: lifetime khai báo qua marker interface (`ITransientDependency`, `IScopedDependency`, `ISingletonDependency`) hoặc `[Dependency(ServiceLifetime.X)]`. Quy tắc phụ thuộc ở §29.1 vẫn áp dụng.

❌ **SAI** — Captive dependency:

```csharp
// Đăng ký Singleton nhưng giữ DbContext (Scoped)
services.AddSingleton<IExchangeRateCache, ExchangeRateCache>();

internal sealed class ExchangeRateCache(AppDbContext dbContext) : IExchangeRateCache // ❌ DbContext sống mãi, dùng chung mọi request
{
}
```

✅ **ĐÚNG** — Singleton tạo scope khi cần:

```csharp
internal sealed class ExchangeRateRefreshWorker(
    IServiceScopeFactory scopeFactory,
    TimeProvider timeProvider,
    ILogger<ExchangeRateRefreshWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using PeriodicTimer timer = new(TimeSpan.FromMinutes(5), timeProvider);

        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            // Mỗi vòng lặp một scope riêng để DbContext không giữ tracking cũ và được dispose đúng lúc
            // One scope per iteration so the DbContext does not keep stale tracking and is disposed on time
            await using AsyncServiceScope scope = scopeFactory.CreateAsyncScope();
            IExchangeRateService service = scope.ServiceProvider.GetRequiredService<IExchangeRateService>();

            try
            {
                await service.RefreshAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                // Biên của BackgroundService: một vòng lỗi không được làm dừng worker vĩnh viễn (Rule 26.2)
                // BackgroundService boundary: one failed iteration must not stop the worker permanently (Rule 26.2)
                logger.LogError(ex, "Exchange rate refresh failed");
            }
        }
    }
}
```

### 29.3. Anti-patterns (KHÔNG làm)

- Singleton phụ thuộc Scoped (`DbContext`, repository, `ICurrentUser`, `IOptionsSnapshot<T>`).
- `IServiceProvider.GetService<T>()` rải trong business code.
- Một scope tạo lúc khởi động `BackgroundService` rồi dùng cho mọi vòng lặp.
- Lưu `HttpContext` / `ICurrentUser` vào field của Singleton.
- Static field giữ service hoặc state theo request.
- Tắt `ValidateScopes` / `ValidateOnBuild` để "chạy được".
