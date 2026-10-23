# Phần X. Quản Lý Giao Dịch & Toàn Vẹn Dữ Liệu ACID (Database Transactions & ACID)

Tài liệu chi tiết về quy chuẩn quản lý giao dịch cơ sở dữ liệu (Database Transactions), đảm bảo 4 thuộc tính ACID (*Atomicity, Consistency, Isolation, Durability*), xử lý tương tranh (*Concurrency Control*), và kiến trúc cách ly giao dịch khi tích hợp API đối tác/tin nhắn phân tán.

---

## 29. Quản lý Giao dịch Cơ sở Dữ liệu & Tính Toàn vẹn ACID

Mọi logic nghiệp vụ ghi dữ liệu vào cơ sở dữ liệu (Database Write Operations) có sự phụ thuộc lẫn nhau giữa nhiều bảng hoặc nhiều bản ghi **bắt buộc** phải được thực thi trong một phạm vi giao dịch (Transaction Scope) an toàn, tuân thủ nghiêm ngặt 4 thuộc tính **ACID**.

### 29.1. Bốn Trụ Cột ACID Trong Phát Triển .NET & EF Core

| Thuộc tính | Ý nghĩa kỹ thuật | Quy chuẩn bắt buộc trong mã nguồn C# / EF Core |
| :--- | :--- | :--- |
| **Atomicity** *(Nguyên tố)* | "Tất cả hoặc không gì cả" — Hoàn tất 100% hoặc hủy bỏ hoàn toàn khi xảy ra lỗi. | Mọi thao tác cập nhật đa bảng phải nằm trong cùng một Transaction (`IDbContextTransaction` hoặc Unit of Work). Chỉ gọi `SaveChangesAsync()` một lần duy nhất tại điểm kết thúc luồng nghiệp vụ. |
| **Consistency** *(Nhất quán)* | Dữ liệu chuyển từ trạng thái hợp lệ này sang trạng thái hợp lệ khác, không vi phạm bất biến nghiệp vụ (*Invariants*). | Kiểm soát toàn vẹn ở 2 cấp độ: **Application Domain Rules** (kiểm tra điều kiện tiên quyết trước khi biến đổi) và **Database Constraints** (Foreign Keys, Unique Indexes, Check Constraints). |
| **Isolation** *(Cô lập)* | Các giao dịch thực thi đồng thời không được can thiệp hoặc nhìn thấy trạng thái dở dang của nhau. | Thiết lập `IsolationLevel` phù hợp với độ nhạy cảm của dữ liệu (`ReadCommitted` mặc định; `Snapshot` cho báo cáo/đọc phức tạp; `Serializable` cho biến động số dư tài chính). Áp dụng **Optimistic Concurrency Control** (`RowVersion` / `[Timestamp]`) để chống mất mát dữ liệu (*Lost Updates*). |
| **Durability** *(Bền vững)* | Dữ liệu đã commit phải được lưu trữ bền vững trên ổ đĩa, không bị mất khi ứng dụng crash hoặc mất điện. | Xác nhận Transaction commit thành công trước khi phản hồi caller. Khi tích hợp hệ thống phân tán, áp dụng **Transactional Outbox Pattern** để đảm bảo sự kiện được gửi tin cậy sau khi DB ghi nhận. |

---

### 29.2. Quy Tắc Vàng: CẤM Gọi API Bên Ngoài / Network I/O Trong Database Transaction

❌ **TUYỆT ĐỐI CẤM**: Mở Database Transaction rồi thực hiện các tác vụ mạng có độ trễ cao (gọi Third-party API, gửi HTTP request qua `IHttpClientFactory`, gửi Email/SMS, gọi gRPC đối tác) bên trong Transaction đó.

#### Lý do:
1. **Cạn kiệt Connection Pool (Connection Starvation)**: Mỗi transaction đang mở chiếm dụng một kết nối DB vật lý. Chờ API ngoài 3-10 giây sẽ nhanh chóng làm cạn kiệt connection pool, khiến toàn bộ ứng dụng bị treo.
2. **Khóa dữ liệu kéo dài (Lock Contention & Deadlocks)**: Các bản ghi bị khóa (Row/Page locks) trong suốt thời gian chờ phản hồi mạng, gây tắc nghẽn dây chuyền cho các request khác.
3. **Mất an toàn Atomicity thực tế**: Nếu API bên thứ ba đã trừ tiền nhưng sau đó database cục bộ lỗi khi commit, hệ thống sẽ rơi vào trạng thái bất nhất không thể tự động rollback phía đối tác.

```
❌ SAI:
Mở DB Transaction ──────▶ Ghi DB (Pending) ──────▶ GỌI HTTP PARTNER (Chờ 5s, giữ DB Lock) ──────▶ Ghi DB (Success) ──────▶ Commit DB

✅ ĐÚNG (Tách biệt 3 pha):
[Pha 1: DB Transaction 1] ──▶ Ghi trạng thái "Pending" ──▶ Commit DB ngay (< 10ms)
                                                                 │
[Pha 2: HTTP Call ngoài DB] ───────────────────────────────▶ Gọi HTTP API đối tác (5s)
                                                                 │
[Pha 3: DB Transaction 2] ──▶ Ghi trạng thái "Success/Failed" theo Idempotency Key ──▶ Commit DB (< 10ms)
```

---

### 29.3. Triển Khai Giao Dịch Chuẩn Với EF Core & `IExecutionStrategy`

Khi ứng dụng cấu hình cơ chế tự động thử lại khi đứt kết nối mạng (`EnableRetryOnFailure`), EF Core yêu cầu các transaction thủ công phải được thực thi thông qua `IExecutionStrategy.ExecuteAsync()` để đảm bảo khả năng retry toàn vẹn.

❌ **SAI** — Gọi `BeginTransactionAsync()` trực tiếp không qua Execution Strategy khi retry được bật:

```csharp
// ❌ SAI: Sẽ ném InvalidOperationException nếu DbContext bật EnableRetryOnFailure()
public async Task ProcessOrderPaymentAsync(Guid orderId, CancellationToken cancellationToken = default)
{
    await using var transaction = await _dbContext.Database.BeginTransactionAsync(cancellationToken);
    
    // Ghi dữ liệu...
    await _dbContext.SaveChangesAsync(cancellationToken);
    await transaction.CommitAsync(cancellationToken);
}
```

✅ **ĐÚNG** — Bọc trong `IExecutionStrategy` và quản lý Unit of Work tập trung:

```csharp
namespace YourApp.Services;

/// <summary>
/// Dịch vụ xử lý đơn hàng và quyết toán tài chính an toàn giao dịch.
/// <para>Service for order processing and transaction-safe financial settlement.</para>
/// </summary>
public sealed class OrderSettlementService : IOrderSettlementService
{
    private readonly AppDbContext _dbContext;
    private readonly ILogger<OrderSettlementService> _logger;

    public OrderSettlementService(
        AppDbContext dbContext,
        ILogger<OrderSettlementService> logger)
    {
        _dbContext = dbContext ?? throw new ArgumentNullException(nameof(dbContext));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc />
    public async Task<OrderSettlementResultDto> SettleOrderAsync(
        OrderSettlementInputDto input,
        CancellationToken cancellationToken = default)
    {
        // 1. Sử dụng ExecutionStrategy để tương thích hoàn toàn với Connection Resiliency (Retry Policy)
        // Use ExecutionStrategy to ensure full compatibility with Connection Resiliency
        var strategy = _dbContext.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            // 2. Mở Transaction với mức độ cô lập tường minh (Explicit IsolationLevel)
            // Open transaction with explicit isolation level
            await using var transaction = await _dbContext.Database.BeginTransactionAsync(
                IsolationLevel.ReadCommitted,
                cancellationToken);

            try
            {
                // Bước 1: Kiểm tra bất biến nghiệp vụ (Consistency Check)
                // Step 1: Check domain invariants
                var order = await _dbContext.Orders
                    .FirstOrDefaultAsync(o => o.Id == input.OrderId, cancellationToken);

                if (order is null)
                {
                    throw new BusinessException(OrderErrorCodes.OrderNotFound)
                        .WithData("OrderId", input.OrderId);
                }

                if (order.Status != OrderStatus.Pending)
                {
                    // Guard clause: Chống xử lý trùng lặp (Idempotency Guard)
                    // Guard clause: Prevent duplicate processing
                    _logger.LogWarning(
                        "Đơn hàng {OrderId} đã ở trạng thái {Status}, bỏ qua quyết toán / Order {OrderId} already in state {Status}, skipping settlement",
                        order.Id,
                        order.Status);

                    return new OrderSettlementResultDto { IsSuccess = true, FinalStatus = order.Status.ToString() };
                }

                // Bước 2: Cập nhật biến động số dư và trạng thái trong cùng Unit of Work (Atomicity)
                // Step 2: Update balance changes and status within the same Unit of Work
                order.MarkAsPaid(input.TransactionReference);

                var ledgerEntry = new FinancialLedger(
                    orderId: order.Id,
                    amount: order.TotalAmount,
                    type: LedgerEntryType.Credit,
                    recordedAt: DateTimeOffset.UtcNow);

                await _dbContext.FinancialLedgers.AddAsync(ledgerEntry, cancellationToken);

                // Bước 3: Ghi xuống cơ sở dữ liệu đúng 1 lần duy nhất trước khi commit (Unit of Work)
                // Step 3: Save changes exactly once before committing
                await _dbContext.SaveChangesAsync(cancellationToken);

                // Bước 4: Commit giao dịch bền vững (Durability)
                // Step 4: Commit transaction durably
                await transaction.CommitAsync(cancellationToken);

                _logger.LogInformation(
                    "Quyết toán thành công cho đơn hàng {OrderId} với mã giao dịch {Ref} / Successfully settled order {OrderId} with transaction ref {Ref}",
                    order.Id,
                    input.TransactionReference);

                return new OrderSettlementResultDto { IsSuccess = true, FinalStatus = order.Status.ToString() };
            }
            catch (Exception ex)
            {
                // Rollback an toàn khi có bất kỳ ngoại lệ nào phát sinh
                // Safely rollback upon any exception
                await transaction.RollbackAsync(cancellationToken);

                _logger.LogError(
                    ex,
                    "Lỗi trong giao dịch quyết toán đơn hàng {OrderId}, đã rollback an toàn / Error in settlement transaction for order {OrderId}, successfully rolled back",
                    input.OrderId);

                throw;
            }
        });
    }
}
```

---

### 29.4. Kiểm Soát Tương Tranh Lạc Quan (Optimistic Concurrency Control)

Để tránh hiện tượng hai tiến trình cùng sửa một bản ghi làm mất dữ liệu của nhau (*Lost Update*), **không dùng khóa bi quan (Pessimistic Lock / Table Lock)** kéo dài. Thay vào đó, áp dụng cơ chế **Optimistic Concurrency** thông qua trường `RowVersion` / `ConcurrencyToken`.

#### Cấu hình Entity:
```csharp
public class AccountBalance
{
    public Guid Id { get; set; }

    public decimal AvailableBalance { get; set; }

    // Trường chống xung đột tương tranh: Tự động cập nhật mỗi khi bản ghi thay đổi
    // Concurrency token: Automatically increments upon any record modification
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}
```

#### Bắt và xử lý lỗi xung đột:
```csharp
try
{
    account.AvailableBalance -= amount;
    await _dbContext.SaveChangesAsync(cancellationToken);
}
catch (DbUpdateConcurrencyException ex)
{
    // Bị xung đột tương tranh do người khác đã cập nhật trước: Log và xử lý retry hoặc thông báo
    // Concurrency conflict detected: Log and handle retry or notify user
    _logger.LogWarning(
        ex,
        "Xung đột tương tranh tài khoản {AccountId}, dữ liệu đã bị sửa đổi bởi giao dịch khác / Concurrency conflict on account {AccountId}, data modified by another transaction",
        accountId);

    throw new BusinessException(FinancialErrorCodes.ConcurrentModificationConflict);
}
```

---

### 29.5. Transactional Outbox Pattern — Nhất Quán Dữ Liệu Phân Tán

Khi một giao dịch nghiệp vụ trong DB cần phát sinh sự kiện gửi sang Message Broker (RabbitMQ, Kafka, Azure Service Bus) hoặc gửi Webhook sang đối tác:

- ❌ **CẤM**: Gửi message sang Broker ngay trong hoặc ngay sau DB Transaction mà không có cơ chế lưu trữ bảo đảm (Nếu Broker sập hoặc mạng lỗi, event bị mất vĩnh viễn dù DB đã commit).
- ✅ **BẮT BUỘC**: Lưu message vào bảng `OutboxMessages` **trong cùng Database Transaction** với dữ liệu nghiệp vụ. Một Background Worker riêng biệt sẽ đọc bảng Outbox và phát lại message với cơ chế retry (At-least-once delivery).

```csharp
// Đảm bảo cả hai thao tác cùng nằm trong 1 Transaction nguyên tố (Atomic)
await using var transaction = await _dbContext.Database.BeginTransactionAsync(cancellationToken);

// 1. Cập nhật thực thể nghiệp vụ
_dbContext.Orders.Update(order);

// 2. Thêm Outbox Message vào cùng DbContext
var outboxMessage = new OutboxMessage(
    eventType: "OrderPaidIntegrationEvent",
    payload: JsonSerializer.Serialize(new OrderPaidPayloadDto(order.Id, order.TotalAmount)),
    createdAt: DateTimeOffset.UtcNow);

await _dbContext.OutboxMessages.AddAsync(outboxMessage, cancellationToken);

// 3. SaveChanges 1 lần cho cả Entity và Outbox
await _dbContext.SaveChangesAsync(cancellationToken);
await transaction.CommitAsync(cancellationToken);
```

---

### 29.6. Checklist Tuân Thủ Rule 29

- [ ] **Atomicity**: Mọi thao tác ghi nhiều thực thể liên đới nằm trong cùng phạm vi Transaction (`BeginTransactionAsync` hoặc Unit of Work).
- [ ] **Unit of Work**: Chỉ gọi `SaveChangesAsync()` một lần duy nhất tại điểm kết thúc luồng ghi, không gọi rải rác sau mỗi câu lệnh.
- [ ] **Network Isolation**: Tuyệt đối **không** gọi Third-party HTTP API, gRPC, gửi Email/SMS hoặc thao tác mạng chậm bên trong Database Transaction.
- [ ] **Resiliency**: Sử dụng `strategy.ExecuteAsync()` khi `DbContext` bật `EnableRetryOnFailure`.
- [ ] **Isolation Level**: Khai báo mức cô lập tường minh khi mở transaction (`ReadCommitted` mặc định, `Snapshot` cho báo cáo, `Serializable` cho giao dịch tài chính cốt lõi).
- [ ] **Concurrency**: Các thực thể nhạy cảm (số dư, tồn kho, trạng thái đơn) có cấu hình Optimistic Concurrency (`[Timestamp]` / `RowVersion`) và bắt lỗi `DbUpdateConcurrencyException`.
- [ ] **Idempotency**: Mọi thao tác cập nhật từ Webhook/Callback đối tác có guard clause kiểm tra `IdempotencyKey` và trạng thái bản ghi để chống xử lý trùng lặp.
- [ ] **Distributed Consistency**: Sử dụng Transactional Outbox Pattern khi cần xuất bản sự kiện ra hệ thống bên ngoài sau khi commit dữ liệu nội bộ.

