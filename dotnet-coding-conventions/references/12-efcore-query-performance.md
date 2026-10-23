# Phần XIII. Hiệu Năng Truy Vấn EF Core (EF Core Query Performance)

Bổ sung cho Rule 25 (ghi dữ liệu ACID): các quy tắc cho **đường đọc** để tránh N+1, load thừa dữ liệu, cartesian explosion và query không phân trang.

---

## 30. Query EF Core phải chỉ lấy đúng dữ liệu cần, trong số round-trip tối thiểu

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Giảm tải DB và network. Latency ổn định khi dữ liệu tăng. Tránh sự cố "chạy tốt ở dev, sập ở production" khi bảng có hàng triệu dòng. |
| **Rủi ro nếu vi phạm** | N+1 biến 1 request thành hàng trăm query. `ToList()` sớm kéo cả bảng vào RAM gây OOM pod. API trả danh sách không phân trang có thể bị dùng làm DoS. `Include` nhiều collection gây cartesian explosion (số dòng nhân lên theo cấp số nhân). |
| **Rủi ro khi áp dụng** | Projection DTO nghĩa là không có change tracking: **không** dùng kết quả projection hoặc `AsNoTracking` để ghi ngược (Rule 25.3). |

### 30.1. Quy tắc

1. **Query chỉ đọc dùng `AsNoTracking()`** (hoặc `AsNoTrackingWithIdentityResolution()` khi cần gộp entity trùng). Với ABP read-only repository, dùng `IReadOnlyRepository` hoặc `GetQueryableAsync()` + `AsNoTracking()`.
2. **Projection sang DTO bằng `Select`** (hoặc `ProjectTo` của AutoMapper/Mapster) cho API đọc, thay vì load entity đầy đủ rồi map trong memory.
3. **Lọc, sắp xếp, phân trang, đếm ở DB.** Mọi `Where`, `OrderBy`, `Skip/Take`, `Count`, `Any` phải đứng **trước** `ToListAsync()`. Cấm `ToList()` / `AsEnumerable()` rồi mới lọc.
4. **Cấm N+1**: không query trong vòng lặp `foreach`. Gom bằng `Include`, join, `Select` lồng, hoặc một query `Where(x => ids.Contains(x.Id))`.
5. **`Include` từ 2 collection trở lên** trên cùng một query phải dùng `AsSplitQuery()` (hoặc cấu hình `UseQuerySplittingBehavior(SplitQuery)` mặc định) để tránh cartesian explosion.
6. **API trả danh sách bắt buộc phân trang** với `MaxPageSize` (ví dụ 100). Bảng lớn hoặc infinite scroll dùng **keyset pagination** (`WHERE Id > @lastId ORDER BY Id`) thay cho `Skip` sâu.
7. **Kiểm tra tồn tại dùng `AnyAsync`**, không dùng `CountAsync() > 0` hay `FirstOrDefaultAsync() != null`.
8. **Cập nhật / xoá hàng loạt** dùng `ExecuteUpdateAsync` / `ExecuteDeleteAsync` (EF Core 7+), không load từng entity rồi sửa.
9. **Raw SQL** dùng `FromSql` / `FromSqlInterpolated` / `SqlQuery<T>` (tham số hoá tự động). Cấm `FromSqlRaw` nối chuỗi input (Rule 32).
10. **Không trả `IQueryable` ra khỏi tầng repository / service**: query phải được định hình và thực thi trong tầng data access, tầng trên chỉ nhận kết quả đã materialize.
11. **Query hot path** (gọi hàng nghìn lần/giây) cân nhắc `EF.CompileAsyncQuery`; mọi query nên gắn `TagWith("<Service>.<Method>")` để truy vết trong slow query log.
12. **Bật cảnh báo** trong môi trường dev: `ConfigureWarnings(w => w.Throw(RelationalEventId.MultipleCollectionIncludeWarning))` và log query chậm.

❌ **SAI** — N+1, load thừa, lọc trên RAM, không phân trang:

```csharp
List<Order> orders = await _dbContext.Orders.ToListAsync(cancellationToken); // ❌ Kéo cả bảng

List<OrderSummaryOutputDto> result = [];

foreach (Order order in orders.Where(x => x.ShopCode == shopCode))           // ❌ Lọc trên RAM
{
    Customer customer = await _dbContext.Customers
        .SingleAsync(x => x.Id == order.CustomerId, cancellationToken);      // ❌ N+1: một query mỗi đơn

    result.Add(new OrderSummaryOutputDto { OrderCode = order.OrderCode, CustomerName = customer.FullName });
}
```

✅ **ĐÚNG** — Một query, projection, phân trang ở DB:

```csharp
int pageSize = Math.Min(input.PageSize, OrderQueryLimits.MaxPageSize);

List<OrderSummaryOutputDto> items = await _dbContext.Orders
    .AsNoTracking()
    .TagWith("OrderQueryService.GetPagedAsync")
    .Where(x => x.ShopCode == input.ShopCode)
    .OrderByDescending(x => x.CreatedAt)
    .ThenByDescending(x => x.Id)
    .Skip((input.Page - 1) * pageSize)
    .Take(pageSize)
    .Select(x => new OrderSummaryOutputDto
    {
        OrderCode = x.OrderCode,
        CustomerName = x.Customer.FullName,
        TotalAmount = x.TotalAmount
    })
    .ToListAsync(cancellationToken);
```

✅ **ĐÚNG** — Keyset pagination cho bảng lớn:

```csharp
// Keyset dùng index (CreatedAt, Id) nên tốc độ không đổi dù đang ở trang thứ 10.000, khác với Skip
// Keyset uses the (CreatedAt, Id) index so speed stays flat even at page 10,000, unlike Skip
List<OrderSummaryOutputDto> items = await _dbContext.Orders
    .AsNoTracking()
    .Where(x => x.ShopCode == input.ShopCode
        && (x.CreatedAt < input.CursorCreatedAt
            || (x.CreatedAt == input.CursorCreatedAt && x.Id < input.CursorId)))
    .OrderByDescending(x => x.CreatedAt)
    .ThenByDescending(x => x.Id)
    .Take(pageSize)
    .Select(x => new OrderSummaryOutputDto { /* ... */ })
    .ToListAsync(cancellationToken);
```

✅ **ĐÚNG** — Split query khi include nhiều collection:

```csharp
Order? order = await _dbContext.Orders
    .AsNoTracking()
    .Include(x => x.Details)
    .Include(x => x.Payments)
    .AsSplitQuery()
    .SingleOrDefaultAsync(x => x.OrderCode == orderCode, cancellationToken);
```

### 30.2. Anti-patterns (KHÔNG làm)

- Query trong vòng lặp (N+1).
- `ToList()` / `AsEnumerable()` trước `Where` / `OrderBy` / `Skip` / `Take`.
- `CountAsync() > 0` để kiểm tra tồn tại.
- API danh sách không có giới hạn `PageSize`.
- `Skip` sâu (trang hàng nghìn) trên bảng lớn.
- `Include` nhiều collection không `AsSplitQuery`.
- Load entity đầy đủ chỉ để lấy 2–3 cột.
- Trả `IQueryable` ra controller.
- `FromSqlRaw($"... {input} ...")` nối chuỗi.
- Gọi method C# không dịch được sang SQL trong `Where` (EF Core 3+ ném lỗi, đừng "fix" bằng `AsEnumerable()`).
