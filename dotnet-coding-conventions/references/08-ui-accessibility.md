# Phần IX. Giao Diện Người Dùng & Dữ Liệu Bảng (UI, Data Grid & Accessibility)

> [!TIP]
> Quy chuẩn UI, UX & Web Accessibility đã được tách thành **Skill độc lập đa ngôn ngữ**: [`ui-ux-accessibility`](../../ui-ux-accessibility/SKILL.md). Skill này áp dụng cho mọi nền tảng frontend (Blazor, React, Vue, Angular, Svelte, HTML/CSS). Phần này trong .NET conventions giữ lại các hướng dẫn chi tiết dành riêng cho hệ sinh thái C# / Blazor.

Tài liệu chi tiết về tối ưu hóa nạp dữ liệu bảng (Server-Side Query & Virtualization), chuẩn tiếp cận Web WAI-ARIA, Clean Data Representation và quy chuẩn Biểu tượng (SVG Icons).

---

## 25. Quy chuẩn nạp dữ liệu vào Table / Data Grid (Google-Style Virtualization & Server-Side Query)

Mọi bảng dữ liệu (Table / Data Grid) trong hệ thống hiển thị danh sách từ cơ sở dữ liệu **bắt buộc** phải tuân theo quy chuẩn nạp dữ liệu hiện đại, tối ưu hiệu năng theo mô hình của Google (Google Cloud Console, Google BigQuery, Google Drive), tuyệt đối không được tải ồ ạt toàn bộ dữ liệu vào bộ nhớ.

### 25.1. Bắt buộc phân trang & lọc ở tầng Cơ sở dữ liệu (Server-Side Query)
- ❌ **CẤM**: Gọi `GetAllAsync()` lấy hàng nghìn hoặc hàng chục nghìn bản ghi về RAM của Web Server / Client rồi dùng LINQ `Where()`, `Skip()`, `Take()` in-memory.
- ✅ **BẮT BUỘC**: Đẩy toàn bộ điều kiện lọc (`search`, `filter`) và phân trang (`skip`, `take` hoặc `pageIndex`, `pageSize`) xuống SQL thông qua `IQueryable` của EF Core với `AsNoTracking()`, `Skip()`, `Take()` và `CountAsync()`.
- DTO kết quả trả về phải đóng gói bằng `PagedResultDto<T>` (chứa `TotalCount` và danh sách chỉ đọc `IReadOnlyList<T> Items`).

```csharp
// Repository / Service pattern bắt buộc cho bảng dữ liệu:
public async Task<PagedResultDto<WardDto>> GetWardsPagedAsync(
    int startIndex,
    int count,
    Guid? provinceId = null,
    string? search = null,
    CancellationToken cancellationToken = default)
{
    await using AppDbContext queryContext = await DbContextFactory.CreateDbContextAsync(cancellationToken);
    IQueryable<Ward> query = queryContext.Wards.AsNoTracking();

    if (provinceId.HasValue && provinceId.Value != Guid.Empty)
    {
        query = query.Where(w => w.ProvinceId == provinceId.Value);
    }

    if (!string.IsNullOrWhiteSpace(search))
    {
        string keyword = search.Trim();
        query = query.Where(w => EF.Functions.Like(w.Name, $"%{keyword}%") || EF.Functions.Like(w.Code, $"%{keyword}%"));
    }

    int totalCount = await query.CountAsync(cancellationToken);
    List<Ward> items = await query
        .OrderBy(w => w.Name)
        .Skip(startIndex)
        .Take(count)
        .ToListAsync(cancellationToken);

    return new PagedResultDto<WardDto>(totalCount, items.Select(MapToDto).ToList());
}
```

### 25.2. Ảo hóa kết xuất DOM (Virtual Scrolling O(1) Memory)
- Với các bảng dữ liệu danh mục hoặc danh sách lớn, **bắt buộc sử dụng cơ chế ảo hóa DOM** (Blazor `<Virtualize ItemsProvider="..." SpacerElement="tr">` hoặc container Virtual Scroll tương đương).
- **Nguyên tắc O(1) DOM**: Dù cơ sở dữ liệu có 10.000 hay 1.000.000 bản ghi, số lượng thẻ `<tr>` thực tế được kết xuất trong DOM chỉ giới hạn trong khung nhìn hiển thị (~25 đến 40 DOM nodes) để duy trì 60 FPS mượt mà và triệt tiêu nguy cơ rò rỉ hoặc nghẽn bộ nhớ trình duyệt.
- Bảng cuộn ảo hóa phải có container giới hạn chiều cao (`max-height: calc(100vh - ...)`), bật `overflow-y: auto`, và tiêu đề cột cố định (`thead th { position: sticky; top: 0; }`).

### 25.3. Shimmer Skeleton Loading (Khung xương tải dữ liệu mượt mà)
- Không dùng spinner che khuất toàn bộ màn hình khi cuộn hoặc tải từng khối dữ liệu mới.
- Bắt buộc khai báo `<Placeholder>` với các dòng Skeleton Shimmer (`.skeleton-box` animation) tương ứng với kích thước và số lượng các cột của bảng để tạo trải nghiệm người dùng liền mạch như Google Drive / Google Cloud.

### 25.4. Debounced Search (Tìm kiếm giảm tải server)
- Mọi ô input tìm kiếm dữ liệu trực tiếp phải có cơ chế **Debounce (tối thiểu 300ms)** sử dụng `CancellationTokenSource` trước khi gửi truy vấn xuống backend/database, tránh việc spam request khi người dùng gõ từng ký tự.
- Component chứa bộ tìm kiếm phải implement `IDisposable` để huỷ bỏ `CancellationTokenSource` khi bị huỷ khỏi bộ nhớ.

### 25.5. Quy chuẩn nạp dữ liệu & Bộ lọc đa tiêu chí cho trang Catalog / Sản phẩm (Web-Admin)

Đối với các trang quản lý danh mục và sản phẩm (`Products.razor`, `Brands.razor`, `ProductTypes.razor`) trong phân hệ Web-Admin, việc nạp dữ liệu và xử lý bộ lọc bắt buộc phải tuân theo các quy tắc sau:

#### 1. Nạp Master Data danh mục song song bằng `Task.WhenAll`
- Khi trang khởi tạo, các dữ liệu danh mục phụ trợ dùng cho các ô lọc dropdown (như danh sách Thương hiệu `Brand`, Loại sản phẩm `Type`) phải được tải song song bằng `Task.WhenAll`, tuyệt đối không `await` tuần tự từng lời gọi gây tích lũy thời gian chờ mạng:

```csharp
// Nạp song song dữ liệu danh mục ban đầu
var typesTask = ProductTypeService.GetTypesAsync(cancellationToken);
var brandsTask = BrandService.GetBrandsAsync(cancellationToken);

await Task.WhenAll(typesTask, brandsTask);

_types = (await typesTask).OrderBy(t => t.Type).ToList();
_brands = (await brandsTask).OrderBy(b => b.Brand).ToList();
```

#### 2. Toàn bộ tiêu chí lọc phải đẩy xuống Database (Server-Side Filtering)
- ❌ **CẤM**: Tải một mẻ lớn (ví dụ 500 hoặc 1.000 sản phẩm) về mảng bộ nhớ RAM (`_products = GetCatalogItemsAsync(0, 500)`) rồi dùng `.Where()`, `.OrderBy()`, `.Skip()`, `.Take()` in-memory trong hàm `ItemsProvider`. Cách làm này sẽ crash trình duyệt hoặc gây nghẽn RAM máy chủ khi catalog đạt tới hàng chục nghìn SKU.
- ✅ **BẮT BUỘC**: Hàm `ItemsProvider` của Virtual Table chuyển tiếp trực tiếp các tham số phân trang (`request.StartIndex`, `request.Count`) và toàn bộ tiêu chí lọc xuống tầng Service/Repository SQL:

```csharp
// Blazor Web-Admin: Products.razor
private async ValueTask<ItemsProviderResult<CatalogItemDto>> LoadProductsVirtualAsync(ItemsProviderRequest request)
{
    // Đẩy toàn bộ filter parameters và token hủy xuống Server qua SQL
    var result = await ProductService.GetCatalogItemsPagedAsync(
        startIndex: request.StartIndex,
        count: request.Count,
        brandId: _selectedBrandId,
        typeId: _selectedTypeId,
        stockFilter: _stockFilter,
        search: _searchKeyword,
        sort: _selectedSort,
        cancellationToken: request.CancellationToken);

    _totalFilteredCount = result.TotalCount;
    StateHasChanged();

    return new ItemsProviderResult<CatalogItemDto>(result.Items, result.TotalCount);
}
```

#### 3. Điều khiển kích hoạt nạp lại (Trigger Data Refresh)
- **Khi gõ ô tìm kiếm từ khóa (`_searchKeyword`)**: Bắt buộc Debounce tối thiểu **300ms** thông qua `CancellationTokenSource` trước khi gọi `await _tableComponent.RefreshDataAsync()`.
- **Khi thay đổi dropdown lọc (Brand, Type, Stock) hoặc Sắp xếp (Sort)**: Gọi ngay `await _tableComponent.RefreshDataAsync()` để Virtual Table tự động yêu cầu chunk dữ liệu mới từ vị trí đầu trang.
- **Sau khi Thêm / Sửa / Xóa thành công**: Đóng modal, hiển thị Toast thông báo và gọi `await _tableComponent.RefreshDataAsync()` để cập nhật lại danh sách mà không cần tải lại toàn bộ trang (Full Page Reload).

---

## 26. Chuẩn Tiếp Cận Web WAI-ARIA (Web Accessibility Initiative - Accessible Rich Internet Applications)

Mọi thành phần giao diện (đặc biệt là Table, Modal, Dropdown, Form Controls) **bắt buộc phải tuân thủ chuẩn tiếp cận WAI-ARIA (WCAG 2.1 AA)** để đảm bảo người khiếm thị, người dùng sử dụng trình đọc màn hình (Screen Reader như NVDA, JAWS, VoiceOver) hoặc điều hướng hoàn toàn bằng bàn phím (Keyboard navigation) có thể sử dụng đầy đủ chức năng.

### 26.1. Quy chuẩn WAI-ARIA cho Table / Data Grid
1. **Ngữ nghĩa chuẩn**:
   - Sử dụng thẻ ngữ nghĩa HTML5 `<table>`, `<thead>`, `<tbody>`, `<tfoot>`, `<tr>`, `<th>`, `<td>`.
   - Bảng phải có nhãn mô tả thông qua thẻ `<caption>` hoặc thuộc tính `aria-label` / `aria-labelledby` trên thẻ `<table>`.
2. **Tiêu đề cột (Column Headers)**:
   - Mọi thẻ `<th>` phải có thuộc tính `scope="col"` (hoặc `scope="row"` nếu là header của dòng dữ liệu).
3. **Trạng thái động (Live Region & Busy State)**:
   - Khi bảng đang tải dữ liệu (loading / fetching chunk): Thẻ table hoặc container chứa bảng phải có thuộc tính `aria-busy="true"`; khi tải xong cập nhật lại `aria-busy="false"`.
   - Khu vực thông báo kết quả (ví dụ *"Tổng số: 10,598 đơn vị"*): Bắt buộc gắn `aria-live="polite"` và `aria-atomic="true"` để trình đọc màn hình thông báo cho người dùng khi số lượng cập nhật mà không cắt ngang âm thanh hiện tại.
4. **Sắp xếp cột (Sorting)**:
   - Cột có tính năng sắp xếp dữ liệu bắt buộc khai báo `aria-sort="ascending"`, `aria-sort="descending"`, hoặc `aria-sort="none"`.

### 26.2. Quy chuẩn WAI-ARIA cho Modal / Dialog
- Thẻ bao ngoài của modal phải có `role="dialog"` (hoặc `role="alertdialog"` đối với hộp thoại cảnh báo nguy hiểm/xác nhận xóa).
- Bắt buộc có thuộc tính `aria-modal="true"` để ngăn screen reader duyệt ra các phần tử nền bên ngoài.
- Bắt buộc liên kết tiêu đề modal bằng `aria-labelledby="[modal_title_id]"` và mô tả bằng `aria-describedby="[modal_desc_id]"`.
- Nút đóng modal phải có `aria-label="Đóng"` (hoặc `aria-label="@Loc["Common_Close"]"`), tuyệt đối không để nút chỉ có ký tự `&times;` mà không có nhãn văn bản.
- **Keyboard Trap & Escape**: Bắt buộc hỗ trợ đóng modal khi người dùng nhấn phím `Escape` và giữ tiêu điểm bàn phím (Focus Trap) bên trong modal khi đang hiển thị.

### 26.3. Quy chuẩn WAI-ARIA cho Dropdown & Custom Form Controls
- Nút kích hoạt dropdown phải có `aria-haspopup="listbox"` (hoặc `aria-haspopup="menu"`) và `aria-expanded="true/false"`.
- Container danh sách lựa chọn phải có `role="listbox"`, mỗi mục lựa chọn bên trong có `role="option"`.
- Phần tử đang được chọn phải có `aria-selected="true"`.
- Mọi nút bấm chỉ có biểu tượng icon SVG (Icon-only buttons) bắt buộc phải có `aria-label` hoặc thẻ con ẩn `.sr-only` mô tả hành động.

### 26.4. Bảng tổng hợp WAI-ARIA bắt buộc

| Thành phần UI | Thuộc tính WAI-ARIA bắt buộc | Hành vi bàn phím (Keyboard) bắt buộc |
| :--- | :--- | :--- |
| **Table / Data Grid** | `aria-label`/`<caption>`, `th[scope="col"]`, `aria-busy`, `aria-live="polite"` | Tab điều hướng vào được các control hành động trong cell |
| **Modal Dialog** | `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, `aria-label` cho nút đóng | Phím `Escape` đóng modal, hỗ trợ Focus Trap |
| **Custom Dropdown** | `aria-haspopup`, `aria-expanded`, `role="listbox"`, `role="option"`, `aria-selected` | Phím `ArrowUp`/`ArrowDown` di chuyển, `Enter` chọn, `Escape` đóng |
| **Icon Button** | `aria-label="..."` hoặc class `.sr-only` | `Enter` / `Space` kích hoạt hành động |

---

## 27. Trình Bày Dữ Liệu Trong Bảng (Clean Data Representation in Tables)

Giao diện bảng dữ liệu (Data Table / Grid) trong hệ thống quản trị Enterprise phải tuân thủ triết lý **Clean Data Representation** — hiển thị dữ liệu chính xác, chuyên nghiệp, tối giản và dễ đọc. Tuyệt đối không trang trí vụn vặt làm rối mắt và phân tán sự chú ý của người dùng.

### 27.1. Cấm lạm dụng Icon / Emoji trang trí trong ô dữ liệu
- Dữ liệu dạng danh từ riêng, tên địa danh, tên thực thể (ví dụ: tên tỉnh/thành phố, tên quận/huyện, tên xã/phường, tên danh mục, tên thương hiệu, tên sản phẩm...) **bắt buộc phải hiển thị dưới dạng văn bản thuần túy (Plain text)**.
- **Tuyệt đối không chèn thêm icon, emoji cảm xúc hoặc hình ảnh trang trí** phía trước/sau nội dung văn bản.
- Biểu tượng (Icon/Emoji) chỉ được phép sử dụng ở các vị trí có chức năng tương tác rõ ràng (nút Thao tác, nút Hành động) hoặc chỉ báo trạng thái chuẩn (`.status-dot`).

❌ **SAI** — Chèn emoji/icon trang trí tùy tiện vào dữ liệu:
```html
<!-- SAI: Emoji chèn vào tên tỉnh / thành phố -->
<td>🏛️ Khánh Hòa</td>

<!-- SAI: Emoji chèn vào tên danh mục hoặc thương hiệu -->
<td>🏷️ Clothing</td>
<td>📦 Nike Official</td>
```

✅ **ĐÚNG** — Hiển thị dữ liệu văn bản sạch sẽ, chuẩn mực:
```html
<!-- ĐÚNG: Văn bản thuần túy, sạch sẽ và chuyên nghiệp -->
<td>Khánh Hòa</td>
<td>Clothing</td>
<td>Nike Official</td>
```

### 27.2. Dữ liệu Text / Mã định danh phải hiển thị thuần túy (Không bọc Badge / Tag không cần thiết)
- Các trường thông tin dạng chuỗi ký tự, mã hành chính, mã vùng, mã bưu chính, số điện thoại, email, địa chỉ... **phải hiển thị dưới dạng text thông thường** (plain text hoặc thẻ `<code>` mộc nếu là mã code định danh).
- **Tuyệt đối không bọc thẻ `.badge`, `.chip`, `.tag` xung quanh các dữ liệu văn bản hoặc mã định danh thông thường** (như mã hành chính trong trang quản lý Tỉnh/TP, Quận/Huyện, Xã/Phường).
- Thẻ `.badge` hoặc `.status-badge` **chỉ dành riêng cho các giá trị thực sự mang tính trạng thái vòng đời (Lifecycle Status / State)** (như: Active, Inactive, Paid, Pending, Shipping, Cancelled...) hoặc nhãn tổng hợp đếm số lượng có ngữ nghĩa rõ ràng.

❌ **SAI** — Lạm dụng badge bọc mã hành chính và text thông thường:
```html
<!-- SAI: Mã hành chính bị bọc badge trang trí không đúng ngữ nghĩa -->
<td>
    <span class="badge" style="background: var(--color-surface-2); color: var(--color-ink-2);">
        KH
    </span>
</td>
<td>
    <span class="badge">79</span>
</td>
```

✅ **ĐÚNG** — Hiển thị mã định danh và text dạng plain text / code gọn gàng:
```html
<!-- ĐÚNG: Hiển thị text trực tiếp hoặc code đơn giản -->
<td style="font-weight: 600; color: var(--color-ink);">KH</td>
<!-- hoặc -->
<td><code style="font-weight: 700;">79</code></td>
```

---

## 28. Quy chuẩn Thiết Kế & Sử Dụng Biểu Tượng (Icon Style & SVG Guidelines)

Giao diện hệ thống (cả Web Customer và Web Admin) **bắt buộc tuân thủ phong cách biểu tượng đồng nhất, hiện đại, tối giản và tối ưu hiệu năng**, dựa trên bộ biểu tượng nét vẽ véc-tơ (**Outline Vector Icons** theo chuẩn **Lucide Icons** và **Heroicons Outline**). Tuyệt đối không pha trộn các phong cách đối nghịch hoặc tải thư viện font cồng kềnh.

### 28.1. Phong cách thiết kế đồng nhất (Icon Family & Visual Language)
- **Chuẩn nét vẽ (Stroke / Outline Style)**: Sử dụng các icon có nét viền đơn mảnh, đầu mút và góc uốn bo tròn mềm mại (`stroke-linecap="round"`, `stroke-linejoin="round"`), hình học tối giản, hiện đại và không đổ bóng.
- **Kế thừa màu chữ tự động (`stroke="currentColor"`)**: Tuyệt đối **không hardcode mã màu hex cố định** vào thuộc tính `stroke` hoặc `fill` của icon (trừ các logo đối tác thương hiệu đặc thù). Bắt buộc dùng `stroke="currentColor"` và `fill="none"` để icon tự động kế thừa màu chữ của phần tử cha, hỗ trợ chuyển đổi mượt mà giữa **Light / Dark theme** và các trạng thái hover (`:hover`), active (`:active`), disabled.
- ❌ **CẤM**:
  - **Không dùng Icon Font** (FontAwesome, Material Icons webfont...): Gây hiện tượng chớp văn bản vô hình (FOIT), tăng dung lượng tải trang và khó căn chỉnh pixel-perfect.
  - **Không dùng icon dạng ảnh bitmap** (PNG, JPEG, GIF) cho các nút bấm và điều hướng chức năng. Ngoại lệ duy nhất là logo thương hiệu / phương thức thanh toán đối tác bên ngoài (như VietQR, COD).
  - **Không tùy tiện dùng icon đặc khối (Filled / Solid)** xen lẫn với Outline icons, ngoại trừ các trường hợp có ý nghĩa ngữ nghĩa đặc biệt (như số sao đánh giá Rating đã chọn, dấu tích checkbox đã chọn).
  - **Không dùng emoji cảm xúc** làm icon chức năng hoặc trang trí tiêu đề bảng / dữ liệu (tuân thủ Rule 27).

### 28.2. Quy chuẩn kỹ thuật SVG (Inline SVG Attributes)
Mọi thẻ `<svg>` nhúng trực tiếp trong Razor component bắt buộc phải khai báo đầy đủ các thuộc tính chuẩn sau:

```html
<svg width="[size]" height="[size]" viewBox="0 0 24 24" fill="none" 
     stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" 
     aria-hidden="true">
    <!-- Path vector -->
</svg>
```

- **`viewBox`**: Luôn sử dụng hệ tọa độ chuẩn của icon (thường là `0 0 24 24`, hoặc `0 0 20 20` / `0 0 16 16` tùy theo bộ icon gốc).
- **`fill="none"`**: Đảm bảo không bị tô mảng đen ngầm định của SVG.
- **`stroke="currentColor"`**: Kế thừa màu từ CSS `color`.
- **`stroke-width`**:
  - `2.0`: Độ dày chuẩn cho Navigation Sidebar, Dropdown Menu, Toolbar thông thường.
  - `2.5`: Sử dụng cho icon trong các nút bấm kích thước nhỏ (`.btn-sm`, button header) để nét vẽ sắc nét, rõ ràng hơn.
  - `1.5` - `1.8`: Sử dụng khi hiển thị icon kích thước lớn (như Empty state, Hero banner).
- **`stroke-linecap="round"` & `stroke-linejoin="round"`**: Giữ các đầu nối và góc bo tròn đồng bộ.

### 28.3. Bảng kích thước chuẩn (Standard Icon Sizes)

| Ngữ cảnh / Vị trí | Kích thước (`width` x `height`) | Độ dày `stroke-width` | Ví dụ áp dụng |
| :--- | :--- | :--- | :--- |
| **Nút hành động Header / Nút chính** | `16x16` (hoặc `18x18`) | `2.5` | Nút Thêm mới (+), Lưu, Xuất dữ liệu |
| **Thanh điều hướng Sidebar (`.nav-icon-svg`)** | `18x18` (viewBox 24) | `2.0` | Menu Dashboard, Orders, Products, Brands |
| **Menu người dùng / Dropdown subpanel** | `18x18` hoặc `20x20` | `2.0` | UserMenu, CultureSelector, ThemeSelector |
| **Caret / Chevron / Mũi tên lướt** | `14x14` đến `16x16` | `2.0` | Mũi tên xổ xuống, mũi tên chuyển trang |
| **Empty State / Khung thông báo rỗng** | `40x40` đến `48x48` | `1.5` - `2.0` | Biểu tượng không có dữ liệu trong Card/Table |

### 28.4. Chuẩn Tiếp Cận WAI-ARIA cho Icon (Accessibility)
- **Icon mang tính trang trí (Đi kèm nhãn chữ)**:
  - Bắt buộc gắn thuộc tính `aria-hidden="true"` vào thẻ `<svg>` (hoặc container `<span class="...-icon">` bao bọc nó) để trình đọc màn hình bỏ qua, không đọc rác mã véc-tơ.
- **Icon-only Button / Link (Nút bấm chỉ có icon, không có chữ)**:
  - Thẻ `<button>` hoặc `<a>` bắt buộc phải có thuộc tính `aria-label="..."` mô tả rõ hành động (ví dụ: `aria-label="@Loc["Common_Close"]"`).
  - Thẻ `<svg>` bên trong vẫn phải có `aria-hidden="true"`.
- **Icon tải từ file ngoài (`wwwroot/icons/*.svg`) qua thẻ `<img>`**:
  - Phải có `role="presentation"` hoặc `alt=""` nếu đi kèm chữ; nếu là icon hành động độc lập phải có `alt="..."` rõ ràng.

### 28.5. Bố cục CSS và Căn chỉnh (Layout & Alignment)
- Thêm `flex-shrink: 0` trên icon trong CSS (`.btn svg`, `.button svg`) để icon không bao giờ bị méo mó, co cụm khi text quá dài hoặc co giãn màn hình.
- Căn giữa trục dọc mượt mà với text bằng Flexbox: `display: inline-flex; align-items: center; gap: 6px;` (hoặc `vertical-align: middle;`).
- Với các icon tương tác tròn (như nút Giỏ hàng, Tài khoản, Theme toggle): Dùng class `.icon-button` hoặc `.icon-btn` với `border-radius: var(--radius-pill)` để tạo hiệu ứng hover nền mềm mại không viền (ghost style).

### 28.6. Ví dụ so sánh

❌ **SAI** — Dùng FontAwesome/icon font, hardcode màu, thiếu aria-hidden:
```html
<!-- SAI: Thư viện font cồng kềnh, không đồng bộ, hardcode màu đỏ, thiếu trợ năng -->
<button class="btn btn-primary">
    <i class="fa fa-plus" style="color: #ff0000;"></i>
    <span>Thêm mới</span>
</button>

<!-- SAI: Nút chỉ có icon mà không có aria-label cho screen reader -->
<button type="button" class="btn-close">
    <svg width="20" height="20"><path d="..." /></svg>
</button>
```

✅ **ĐÚNG** — Nét Outline stroke chuẩn mực, `stroke="currentColor"`, hỗ trợ WAI-ARIA đầy đủ:
```html
<!-- ĐÚNG: Nút có text + icon trang trí (kế thừa màu chữ, nét 2.5, aria-hidden="true") -->
<button type="button" @onclick="OpenCreateModal" class="btn btn-primary btn-sm">
    <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5" aria-hidden="true">
        <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
    </svg>
    <span>@Loc["AdminBrands_AddBrand"]</span>
</button>

<!-- ĐÚNG: Nút icon-only có aria-label trên button, svg ẩn khỏi screen reader -->
<button type="button" class="sidebar-close-btn" aria-label="@Loc["Common_Close"]">
    <svg width="20" height="20" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <path d="M6 18L18 6M6 6l12 12" />
    </svg>
</button>
```

