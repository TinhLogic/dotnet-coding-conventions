# Quy Chuẩn Responsive Cho Bảng & Trang Dữ Liệu Lớn (Chuẩn Google Cloud Console, Google Sheets & GA4)

Xử lý giao diện thích ứng (Responsive) cho các **bảng dữ liệu lớn (Big Data Tables / Data Grids)** và **trang quản trị dữ liệu phức tạp (Data-heavy Dashboards / Consoles)** là một trong những thử thách kỹ thuật và UX khó nhất trong kỹ thuật phần mềm doanh nghiệp.

Tài liệu này hệ thống hóa toàn bộ kiến trúc và quy chuẩn thiết kế của Google (**Material Design 3, Google Cloud Console, BigQuery Studio, Google Sheets, Google Analytics 4 và Gmail**) nhằm đảm bảo ứng dụng vừa hiển thị mượt mà trên mọi kích thước màn hình (Desktop, Tablet, Mobile), vừa giữ vững hiệu năng $O(1)$ và khả năng tiếp cận trợ năng (**WAI-ARIA / WCAG 2.1 AA**).

---

## 1. Triết Lý Thiết Kế Dữ Liệu Lớn Của Google (Google Big Data UX)

1. **Tuyệt đối không bóp méo dữ liệu (Never Squish Data)**: Khi bảng có từ 8–20+ cột, việc co hẹp chiều rộng các cột để "nhồi nhét" vừa màn hình mobile/tablet sẽ làm gãy chữ, tràn dòng và mất khả năng đọc. Chiều rộng cột dữ liệu phải có `min-width` tối thiểu để nội dung rõ ràng.
2. **Bảo toàn ngữ cảnh điều hướng (Context Preservation)**: Khi người dùng cuộn ngang/dọc trên tập dữ liệu hàng nghìn dòng, họ không bao giờ được phép mất dấu: *"Tôi đang xem dòng của ai/tài nguyên nào?"* (giải quyết bằng **Sticky Columns/Headers**) và *"Trang này có thể chia sẻ lại chính xác như đang thấy không?"* (giải quyết bằng **URL Deep Linking**).
3. **Tiết lộ tiệm tiến (Progressive Disclosure)**: Mặc định chỉ hiển thị các thông tin nhận diện cốt lõi (P0/P1). Chi tiết kỹ thuật sâu (P2) được đưa vào các lớp tương tác mở rộng (**Expandable Rows, Inspector Side Sheets, Bottom Sheets**).

---

## 2. Ma Trận Chiến Lược Responsive Theo Breakpoints

```
+----------------------------------------------------------------------------------------------------+
| Breakpoint       | Chiến Lược Bảng (Table Strategy)        | Chiến Lược Trang (Page Layout)        |
+----------------------------------------------------------------------------------------------------+
| Desktop (>1200px)| Wide Data Grid + Sticky Columns/Header  | Master-Detail dạng Side Sheet trượt   |
|                  | Scroll Shadows + Column Display Picker  | Thanh Filter Chips đầy đủ             |
+----------------------------------------------------------------------------------------------------+
| Tablet (768-1199)| Thu gọn ưu tiên (P0/P1 hiển thị,        | Side Sheet mở rộng 60-70% chiều rộng  |
|                  | P2 thu vào Accordion dòng) + Cuộn ngang | Filter Chips có thanh cuộn ngang      |
+----------------------------------------------------------------------------------------------------+
| Mobile (<768px)  | Table-to-Card Transformation            | Bottom Sheet toàn màn hình            |
|                  | Hoặc Master-Detail với Bottom Sheet     | Modal bộ lọc tập trung (Filter Modal) |
+----------------------------------------------------------------------------------------------------+
```

### 2.1. Desktop (> 1200px): Data Grid Tối Ưu Với Sticky & Scroll Shadows
- **Ghim cột định danh bên trái (`Sticky Left`)**: Cột Tên tài nguyên / Checkbox / Mã định danh bắt buộc giữ cố định `position: sticky; left: 0; z-index: 2;` để khi cuộn ngang 20 cột, người dùng luôn biết dữ liệu thuộc về bản ghi nào.
- **Ghim cột hành động bên phải (`Sticky Right`)**: Cột nút bấm sửa/xóa/menu `...` ghim ở mép phải (`right: 0`).
- **Ghim tiêu đề bảng (`Sticky Header`)**: Thẻ `<thead> th` ghim ở đỉnh container cuộn (`top: 0; z-index: 3;`).
- **Hiệu ứng Scroll Shadows tự động (CSS Pure Gradient)**: Bắt buộc áp dụng hiệu ứng đổ bóng ở mép cột ghim khi nội dung bị tràn để báo hiệu cho người dùng biết còn dữ liệu có thể cuộn ngang:

```css
/* Container bao ngoài cho phép cuộn ngang có Scroll Shadows tự động */
.g-table-container {
  width: 100%;
  overflow-x: auto;
  border: 1px solid var(--border-color, #e0e0e0);
  border-radius: 8px;
  background-color: var(--surface-bg, #ffffff);

  /* Hiệu ứng bóng cuộn mép trái & mép phải không cần JavaScript */
  background-image: 
    linear-gradient(to right, white 30%, rgba(255, 255, 255, 0)),
    linear-gradient(to right, rgba(255, 255, 255, 0), white 70%),
    radial-gradient(farthest-side at 0 50%, rgba(0, 0, 0, 0.15), rgba(0, 0, 0, 0)),
    radial-gradient(farthest-side at 100% 50%, rgba(0, 0, 0, 0.15), rgba(0, 0, 0, 0));
  background-position: 0 0, 100% 0, 0 0, 100% 0;
  background-repeat: no-repeat;
  background-color: white;
  background-size: 40px 100%, 40px 100%, 14px 100%, 14px 100%;
  background-attachment: local, local, scroll, scroll;
}

/* Cột ghim bên trái */
.g-table th.col-sticky-start,
.g-table td.col-sticky-start {
  position: sticky;
  left: 0;
  background-color: inherit;
  z-index: 2;
  box-shadow: 2px 0 4px -2px rgba(0, 0, 0, 0.12);
}

/* Hàng tiêu đề ghim trên đỉnh */
.g-table thead th {
  position: sticky;
  top: 0;
  background-color: #f8f9fa;
  z-index: 3;
}
```

### 2.2. Tablet (768px – 1199px): Phân Cấp Cột Ưu Tiên (P0, P1, P2)
Chia toàn bộ cột của bảng thành 3 nhóm:
* **P0 (Cốt lõi)**: Tên đối tượng, Trạng thái (Badge), Hành động chính. *(Luôn hiển thị)*.
* **P1 (Quan trọng trung bình)**: Chỉ số KPI, Ngày cập nhật gần nhất. *(Hiển thị nếu còn chỗ)*.
* **P2 (Chi tiết phụ)**: Mã vùng, IP nội bộ, Người tạo, Ngày tạo, Mô tả...

Khi màn hình co lại, các cột **P2** tự động ẩn (`display: none`) và chuyển vào:
1. Nút bấm mở rộng dòng (**Expandable Row Accordion**); hoặc
2. Menu hành động ba chấm `...` cuối dòng.

### 2.3. Mobile (< 768px): Chuyển Đổi Hình Thái Sang Card (Table-to-Card)
Trên màn hình điện thoại dưới 768px, **bỏ qua hoàn toàn cấu trúc `<table>` truyền thống** bằng kỹ thuật CSS Table-to-Card:

```css
@media (max-width: 767px) {
  /* Biến bảng thành danh sách Cards độc lập */
  .g-responsive-table table,
  .g-responsive-table tbody,
  .g-responsive-table tr,
  .g-responsive-table td {
    display: block;
    width: 100%;
  }

  /* Ẩn tiêu đề thead trên mobile */
  .g-responsive-table thead {
    display: none;
  }

  /* Mỗi dòng dữ liệu trở thành một Card nổi */
  .g-responsive-table tr {
    margin-bottom: 12px;
    padding: 16px;
    background: #ffffff;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  }

  /* Hiển thị nhãn cột qua data-label */
  .g-responsive-table td {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 0;
    border-bottom: 1px solid #f1f3f4;
    text-align: end;
  }

  .g-responsive-table td:last-child {
    border-bottom: none;
    padding-bottom: 0;
  }

  /* Tự động lấy tên cột từ thuộc tính data-label */
  .g-responsive-table td::before {
    content: attr(data-label);
    font-weight: 500;
    color: #5f6368;
    text-align: start;
    padding-right: 12px;
  }
}
```

---

## 3. Kỹ Thuật Render & Tối Ưu Bộ Nhớ DOM (DOM & Memory Optimization)

### 3.1. Ảo Hóa DOM (DOM Virtualization / Windowing $O(1)$)
*Áp dụng trong: Google Cloud Console, BigQuery Logs, GA4 Report Explorer.*

- **Vấn đề**: Bảng hiển thị 10,000 dòng `<tr>` tiêu tốn > 200MB RAM, khiến thao tác cuộn tụt xuống 10 FPS và làm đơ Main Thread của trình duyệt.
- **Quy tắc bắt buộc**: 
  - Số lượng thẻ `<tr>` được kết xuất thực tế trong DOM chỉ duy trì ở mức **25 đến 40 phần tử** (đủ lấp đầy khung nhìn + 5 dòng đệm trên và dưới).
  - Vị trí cuộn được tính toán để tái sử dụng lại các DOM node sẵn có bằng CSS `transform: translateY(...)`.
  - Công cụ tiêu chuẩn: `@tanstack/react-virtual` (React), `vue-virtual-scroller` (Vue), `<Virtualize>` (Blazor), `@angular/cdk/scrolling` (Angular).

### 3.2. Canvas Rendering Engine (Chuẩn Google Sheets)
*Áp dụng khi tập dữ liệu vượt quá giới hạn của DOM (> 100,000 ô tính tương tác).*

- Google Sheets đã chuyển đổi toàn bộ giao diện từ HTML DOM/SVG sang **HTML5 Canvas kết hợp WebGL**:
  - Không sinh ra bất kỳ thẻ `<div>` hay `<td>` nào cho từng ô dữ liệu.
  - Toàn bộ lưới được vẽ trực tiếp lên một phần tử `<canvas>` duy nhất dựa trên tọa độ pixel tính toán từ WebAssembly (Wasm).
  - Loại bỏ hoàn toàn chi phí tính toán lại bố cục (**Layout Reflow**) và vẽ lại (**Repaint**) của trình duyệt, đạt tốc độ phản hồi 60 FPS bất kể kích thước bảng.

### 3.3. Tách Biệt Luồng Xử Lý Nặng Ra Web Workers (Off-Main-Thread Processing)
- Các thao tác lọc (filter), sắp xếp (sort) hoặc tổng hợp dữ liệu mảng lớn (> 50,000 dòng) ở client bắt buộc phải đẩy vào **Web Worker**.
- Giữ Main Thread luôn rảnh rỗi nhằm đảm bảo chỉ số **INP (Interaction to Next Paint) luôn < 200ms**.

---

## 4. Kiến Trúc Responsive Cho Toàn Bộ Trang Dữ Liệu Lớn (Data-Heavy Pages)

```
+---------------------------------------------------------------------------------------------------+
| Thanh Lọc Thích Ứng (Adaptive Filter Bar): Search Input + Filter Chips (Horizontal Scroll on Mobile)|
+---------------------------------------------------------------------------------------------------+
| KPI Widget 1 (Skeleton Isolated) | KPI Widget 2 (Skeleton Isolated) | KPI Widget 3 (Isolated)     |
+---------------------------------------------------------------------------------------------------+
| DANH SÁCH BẢNG CHÍNH (MASTER TABLE)                    | SIDE SHEET / CHI TIẾT (INSPECTOR DRAWER) |
| [x] instance-prod-1 | Running | us-central1 | 14.2% CPU| - Thông tin cấu hình chi tiết (Tabs)     |
| [ ] instance-prod-2 | Stopped | europe-west1| 0.0%  CPU| - Biểu đồ CPU/RAM trực tiếp              |
| [ ] instance-prod-3 | Running | asia-east1  | 62.1% CPU| - Nhật ký Logs gần nhất                  |
|                                                        | [Đóng X] [Chỉnh Sửa] [Khởi Động Lại]     |
| <- Trang trước | Trang sau (Keyset: pageToken) ->      | (Trên mobile: Biến thành Bottom Sheet)   |
+---------------------------------------------------------------------------------------------------+
```

### 4.1. Pattern Master-Detail Qua Side Sheet (Bảng Điều Khiển Inspector)
*Chuẩn Google Cloud VM Instances, Gmail, Google Drive.*

- **Tuyệt đối cấm**: Chuyển hướng sang một URL mới chỉ để xem thông tin chi tiết của 1 dòng rồi bắt người dùng bấm nút *"Quay lại (Back)"* và nạp lại toàn bộ bảng.
- **Quy tắc chuẩn**:
  - Khi click vào một dòng trong bảng, một **Side Sheet (Ngăn trượt phải)** mở ra, chiếm 35% – 50% màn hình Desktop.
  - Bảng dữ liệu tự động co lại nhưng vẫn giữ nguyên vị trí cuộn và danh sách dòng đã chọn.
  - Trên **Mobile (< 768px)**: Side Sheet tự động chuyển thành **Bottom Sheet** (kéo từ đáy lên) hoặc **Full-screen Dialog**.

### 4.2. Thanh Lọc Thích Ứng (Adaptive Filter Chips)
- Trên **Desktop**: Thanh tìm kiếm tích hợp các Filter Chips trực quan (`Status: RUNNING`, `Zone: us-central1-a`).
- Trên **Tablet/Mobile**: 
  - Danh sách Chips cho phép cuộn ngang (`overflow-x: auto; flex-wrap: nowrap;`).
  - Có nút *"Tất cả bộ lọc (All Filters)"* với badge đếm số lượng đang chọn (ví dụ: `Bộ lọc (3)`). Bấm vào sẽ mở Bottom Sheet tập trung để thao tác chọn tiêu chí bằng ngón tay cái dễ dàng.
- > [!TIP]
  > Xem quy chuẩn chuyên sâu về xây dựng thanh tìm kiếm Omnibox dạng Chip (Tokenized Filter Bar), gợi ý 3 bước (`key:operator:value`), hủy request bằng `AbortController`, và ảo hóa Dropdown tại: [`16-google-faceted-search-dropdowns.md`](./16-google-faceted-search-dropdowns.md).

### 4.3. Phân Trang Bằng Con Trỏ (Keyset / Cursor Pagination - `pageToken`)
- ❌ **CẤM**: Dùng `OFFSET / LIMIT` trên các bảng dữ liệu hàng trăm nghìn đến hàng triệu dòng (gây Full-Table Scan ở cơ sở dữ liệu và làm sụt giảm nghiêm trọng tốc độ khi đến trang lớn).
- ✅ **BẮT BUỘC**: Dùng Keyset / Cursor Pagination với `pageToken` theo chuẩn Google Cloud API:

```typescript
// Định dạng Response chuẩn Keyset Pagination
interface PaginatedResponse<T> {
  items: T[];
  nextPageToken?: string | null;
  prevPageToken?: string | null;
  estimatedTotalCount?: number;
}
```

### 4.4. Đồng Bộ Hai Chiều Lên URL Query Parameters (Deep Linking)
Toàn bộ trạng thái của trang dữ liệu lớn **bắt buộc** phải được phản ánh lên URL:
- Từ khóa tìm kiếm: `?search=prod-api`
- Bộ lọc: `?status=RUNNING&zone=us-central1-a`
- Con trỏ trang: `?pageToken=Cg0IARABGAEiBAgBEAI`
- ID dòng đang mở Side Sheet: `?selectedId=vm-982134`
- Thứ tự sắp xếp: `?sort=cpu:desc`

*Lợi ích: Khi người dùng xoay ngang màn hình thiết bị, reload trang hoặc chia sẻ link cho đồng nghiệp, trang mở ra với chính xác 100% ngữ cảnh đang làm việc.*

### 4.5. Phân Tách Widget Độc Lập & Cô Lập Skeleton (Independent Streaming)
- Trong một trang dashboard phức tạp (gồm 4 KPI Cards, 2 Biểu đồ thống kê, 1 Bảng dữ liệu lớn):
  - Mỗi Widget phải tự gọi API độc lập và quản lý trạng thái tải riêng.
  - Sử dụng **Content-Aware Skeleton Shimmer** riêng cho từng widget.
  - Một truy vấn API chậm ở biểu đồ thống kê **tuyệt đối không được làm nghẽn** việc hiển thị bảng dữ liệu chính.

---

## 5. Chuẩn Tiếp Cận Trợ Năng (WAI-ARIA Cho Responsive Data Table)

1. **Khung cuộn ngang độc lập**: Bắt buộc có `tabindex="0"`, `role="region"`, và `aria-label="Tên bảng dữ liệu"` để người dùng bàn phím có thể focus vào container và dùng phím mũi tên `ArrowLeft` / `ArrowRight` để cuộn ngang.
2. **Tiêu đề cột**: Thẻ `<th>` bắt buộc có thuộc tính `scope="col"`.
3. **Chống đọc trùng lặp trên Mobile**: Khi áp dụng Table-to-Card với `data-label`, ẩn tiêu đề gốc bằng `aria-hidden="true"` nếu cần để Screen Reader không đọc lặp 2 lần nhãn cột.
4. **Thông báo trạng thái bảng**: Khai báo vùng `aria-live="polite"` để Screen Reader đọc to thông báo: *"Đang hiển thị 50 trên 1.420 kết quả, lọc theo trạng thái Running"* sau khi tải xong.

---

## 6. Checklist Nghiệm Thu Bảng & Trang Dữ Liệu Lớn (Big Data Audit)

- [ ] Bảng có chiều rộng cột hợp lý, không bị bóp méo chữ trên màn hình nhỏ không?
- [ ] Cột định danh chính (Tên/Mã) có được ghim `position: sticky; left: 0` khi cuộn ngang không?
- [ ] Có hiệu ứng Scroll Shadows báo hiệu trực quan khi bảng tràn ngang không?
- [ ] Bảng có tự động chuyển sang chế độ Accordion (Tablet) hoặc Table-to-Card / Bottom Sheet (Mobile) không?
- [ ] Đã ảo hóa DOM (Virtual Scrolling $O(1)$) để giữ số dòng DOM $\le 40$ khi danh sách lớn chưa?
- [ ] Đã thay thế Offset Pagination bằng Keyset Cursor Pagination (`pageToken`) chưa?
- [ ] Bấm vào dòng để xem chi tiết có mở Side Sheet / Inspector Drawer (giữ nguyên vị trí cuộn) thay vì chuyển trang không?
- [ ] Toàn bộ bộ lọc, từ khóa tìm kiếm, con trỏ trang và ID dòng đang chọn đã được đồng bộ 2 chiều lên URL Query Params chưa?
- [ ] Container cuộn ngang của bảng có thuộc tính `role="region"`, `tabindex="0"` và `aria-label` cho người dùng bàn phím chưa?
- [ ] Các con số, tiền tệ, tỷ lệ % có áp dụng `tabular-nums` và căn phải đồng nhất không?
