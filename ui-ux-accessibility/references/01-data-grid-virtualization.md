# Quy Chuẩn Nạp Dữ Liệu Bảng & Ảo Hóa DOM (Data Grid & Virtualization)

Mọi bảng dữ liệu (Table / Data Grid) trong hệ thống hiển thị danh sách từ cơ sở dữ liệu **bắt buộc** phải tuân theo quy chuẩn nạp dữ liệu hiện đại, tối ưu hiệu năng theo mô hình của Google (Google Cloud Console, Google Drive, BigQuery), tuyệt đối không tải ồ ạt toàn bộ dữ liệu vào bộ nhớ RAM của trình duyệt hoặc máy chủ.

> [!TIP]
> Xem hướng dẫn chuyên sâu toàn diện về thiết kế Responsive cho bảng nhiều cột, Sticky Columns/Headers, Scroll Shadows, Master-Detail Side Sheet và phân trang Keyset Cursor (`pageToken`) theo chuẩn Google tại: [`15-responsive-big-data-tables-pages.md`](./15-responsive-big-data-tables-pages.md).

---

## 1. Bắt Buộc Phân Trang & Lọc Tại Tầng Cơ Sở Dữ Liệu (Server-Side Query)

- ❌ **CẤM**: Tải toàn bộ tập dữ liệu (hàng nghìn đến hàng chục nghìn bản ghi) về RAM trình duyệt hoặc RAM backend rồi dùng các hàm in-memory (`.filter()`, `.slice()`, `.Where()`, `.Skip()`, `.Take()`).
- ✅ **BẮT BUỘC**: Đẩy toàn bộ điều kiện lọc (`search`, `filter`), sắp xếp (`sort`) và phân trang (`skip`, `take` hoặc `pageIndex`, `pageSize`) xuống tầng cơ sở dữ liệu thông qua câu truy vấn SQL tối ưu (`LIMIT / OFFSET` hoặc Keyset pagination).
- Dữ liệu trả về cho frontend phải đóng gói dạng Paged Result (bao gồm `totalCount` và danh sách mảng dữ liệu `items` của trang hiện tại).

### Ví dụ TypeScript / React & Node.js:
```typescript
// Backend (Express / Fastify / NestJS / Prisma)
async function getPagedCustomers(params: {
  skip: number;
  take: number;
  search?: string;
  status?: string;
}) {
  const where: Prisma.CustomerWhereInput = {
    ...(params.status && { status: params.status }),
    ...(params.search && {
      name: { contains: params.search, mode: 'insensitive' }
    }),
  };

  const [totalCount, items] = await Promise.all([
    prisma.customer.count({ where }),
    prisma.customer.findMany({
      where,
      skip: params.skip,
      take: params.take,
      orderBy: { createdAt: 'desc' }
    })
  ]);

  return { totalCount, items };
}
```

### Ví dụ C# / ASP.NET Core & Blazor:
```csharp
// Backend / Service Repository
public async Task<PagedResultDto<CustomerDto>> GetPagedCustomersAsync(
    int startIndex,
    int count,
    string? search,
    string? status,
    CancellationToken cancellationToken = default)
{
    await using AppDbContext db = await DbFactory.CreateDbContextAsync(cancellationToken);
    IQueryable<Customer> query = db.Customers.AsNoTracking();

    if (!string.IsNullOrWhiteSpace(status))
        query = query.Where(c => c.Status == status);

    if (!string.IsNullOrWhiteSpace(search))
        query = query.Where(c => EF.Functions.Like(c.Name, $"%{search.Trim()}%"));

    int totalCount = await query.CountAsync(cancellationToken);
    List<CustomerDto> items = await query
        .OrderByDescending(c => c.CreatedAt)
        .Skip(startIndex)
        .Take(count)
        .Select(c => MapToDto(c))
        .ToListAsync(cancellationToken);

    return new PagedResultDto<CustomerDto>(totalCount, items);
}
```

---

## 2. Ảo Hóa Kết Xuất DOM (Virtual Scrolling O(1) Memory)

Khi hiển thị danh sách lớn (hàng trăm đến hàng trăm nghìn dòng), việc gắn toàn bộ các phần tử `<tr>` vào DOM cây HTML sẽ gây nghẽn RAM trình duyệt, giảm tốc độ cuộn xuống dưới 15 FPS và tăng nguy cơ crash tab.

- **Nguyên tắc O(1) DOM**: Dù cơ sở dữ liệu có 10.000 hay 1.000.000 bản ghi, số lượng thẻ `<tr>` thực tế được kết xuất trong DOM chỉ giới hạn trong khung nhìn hiển thị (~25 đến 40 DOM nodes) để duy trì **60 FPS** mượt mà.
- Container cuộn ảo hóa phải có:
  - Chiều cao giới hạn: `max-height: calc(100vh - 220px);` hoặc chiều cao cố định.
  - Bật cuộn dọc: `overflow-y: auto;`.
  - Tiêu đề cố định (Sticky Header): `thead th { position: sticky; top: 0; z-index: 2; }`.

### Thư viện & Công cụ đề xuất:
- **React**: `@tanstack/react-virtual`
- **Vue**: `vue-virtual-scroller` / `@tanstack/vue-virtual`
- **Blazor**: Component tích hợp sẵn `<Virtualize ItemsProvider="..." SpacerElement="tr">`
- **Angular**: `@angular/cdk/scrolling` (`CdkVirtualScrollViewport`)

---

## 3. Shimmer Skeleton Loading (Khung Xương Tải Dữ Liệu)

> [!TIP]
> Xem hướng dẫn chuyên sâu toàn diện về thiết kế Skeleton cho mọi thành phần (Card, Table, Form, Profile, Screen Reader A11y) tại: [`12-skeleton-loading-patterns.md`](./12-skeleton-loading-patterns.md).

- ❌ **CẤM**: Dùng spinner xoay che khuất toàn bộ màn hình khi cuộn hoặc nạp từng khối dữ liệu mới, gây giật lag thị giác.
- ✅ **BẮT BUỘC**: Khai báo vùng giữ chỗ (Placeholder) với các dòng Skeleton Shimmer (`.skeleton-box`) tương ứng với kích thước và số cột của bảng để tạo cảm giác phản hồi tức thì.

```css
/* CSS Hiệu ứng Skeleton Shimmer chuẩn mực */
.skeleton-box {
  display: inline-block;
  height: 1rem;
  width: 100%;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 4px;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

---

## 4. Debounced Search (Tìm Kiếm Giảm Tải Máy Chủ)

- Mọi ô input tìm kiếm dữ liệu trực tiếp bắt buộc có cơ chế **Debounce (tối thiểu 300ms)** trước khi gửi truy vấn xuống backend/database, tránh việc gửi request liên tục sau mỗi ký tự gõ.
- Hủy bỏ các request đang dang dở (In-flight request) khi có ký tự mới thông qua `AbortController` (JS/TS) hoặc `CancellationTokenSource` (C#).

### Ví dụ React Hook:
```typescript
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number = 300): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}
```

---

## 5. Nạp Song Song Dữ Liệu Danh Mục Phụ Trợ (Parallel Master Data)

Khi khởi tạo trang có nhiều dropdown lọc (ví dụ: Danh mục, Thương hiệu, Trạng thái), bắt buộc tải song song bằng `Promise.all` hoặc `Task.WhenAll`, tuyệt đối không `await` tuần tự từng request gây tích lũy thời gian chờ mạng.

```typescript
// Nạp song song bằng Promise.all
const [categories, brands, statuses] = await Promise.all([
  categoryApi.getAll(),
  brandApi.getAll(),
  statusApi.getAll()
]);
```

---

## 6. Điều Khiển Kích Hoạt Nạp Lại Dữ Liệu (Trigger Data Refresh)

- **Khi gõ ô tìm kiếm từ khóa**: Sau khi debounce đủ 300ms, tự động yêu cầu trang/khối dữ liệu mới từ vị trí đầu tiên (`skip = 0`).
- **Khi thay đổi bộ lọc hoặc thứ tự sắp xếp**: Reset về đầu trang và kích hoạt nạp lại ngay lập tức.
- **Sau khi Thêm / Sửa / Xóa thành công**: Đóng modal, hiển thị thông báo Toast thành công và kích hoạt refresh dữ liệu tại chỗ, **tuyệt đối không tải lại toàn bộ trang (Full Page Reload)**.

