# Quy Chuẩn Thiết Kế & Sử Dụng Biểu Tượng (Icon Style & SVG Guidelines)

Giao diện người dùng trên mọi nền tảng web **bắt buộc tuân thủ phong cách biểu tượng đồng nhất, hiện đại, tối giản và tối ưu hiệu năng**, dựa trên bộ biểu tượng nét vẽ véc-tơ (**Outline Vector Icons** theo chuẩn **Lucide Icons** hoặc **Heroicons Outline**). Tuyệt đối không pha trộn các phong cách đối nghịch hoặc tải các thư viện font cồng kềnh.

---

## 1. Phong Cách Thiết Kế Đồng Nhất (Visual Language)

- **Chuẩn nét vẽ (Stroke / Outline Style)**: Sử dụng icon có nét viền đơn mảnh, đầu mút và góc uốn bo tròn mềm mại (`stroke-linecap="round"`, `stroke-linejoin="round"`), hình học tối giản, hiện đại và không đổ bóng.
- **Kế thừa màu chữ tự động (`stroke="currentColor"`)**: Tuyệt đối **không hardcode mã màu hex cố định** vào thuộc tính `stroke` hoặc `fill` của icon (trừ các logo thương hiệu đối tác đặc thù). Bắt buộc dùng `stroke="currentColor"` và `fill="none"` để icon tự động kế thừa màu chữ (`color`) của phần tử cha, hỗ trợ chuyển đổi mượt mà giữa **Light / Dark theme** và các trạng thái hover (`:hover`), active (`:active`), disabled.
- ❌ **CẤM**:
  - **Không dùng Icon Font** (FontAwesome, Material Icons webfont...): Gây hiện tượng chớp văn bản vô hình (Flash of Invisible Text - FOIT), tăng dung lượng tải trang và rất khó căn chỉnh pixel-perfect.
  - **Không dùng icon dạng ảnh bitmap** (PNG, JPEG, GIF) cho các nút bấm và điều hướng chức năng. Ngoại lệ duy nhất là logo thương hiệu / phương thức thanh toán đối tác bên ngoài (như Visa, VietQR).
  - **Không tùy tiện dùng icon đặc khối (Filled / Solid)** xen lẫn với Outline icons, ngoại trừ các trường hợp có ý nghĩa ngữ nghĩa đặc biệt (như số sao Rating đã chọn, dấu tích checkbox đã chọn).
  - **Không dùng emoji cảm xúc** làm icon chức năng hoặc trang trí tiêu đề bảng / dữ liệu.

---

## 2. Quy Chuẩn Kỹ Thuật SVG (Inline SVG Attributes)

Mọi thẻ `<svg>` nhúng trực tiếp trong HTML / JSX / Vue / Svelte / Razor component bắt buộc phải khai báo đầy đủ các thuộc tính chuẩn sau:

```html
<svg width="[size]" height="[size]" viewBox="0 0 24 24" fill="none" 
     stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" 
     aria-hidden="true">
    <!-- Path vector -->
</svg>
```

- **`viewBox`**: Luôn sử dụng hệ tọa độ chuẩn của icon (thường là `0 0 24 24`, hoặc `0 0 20 20` tùy bộ icon gốc).
- **`fill="none"`**: Đảm bảo không bị tô mảng đen ngầm định của SVG.
- **`stroke="currentColor"`**: Kế thừa màu từ CSS `color`.
- **`stroke-width`**:
  - `2.0`: Độ dày chuẩn cho Navigation Sidebar, Dropdown Menu, Toolbar thông thường.
  - `2.5`: Sử dụng cho icon trong các nút bấm kích thước nhỏ (`.btn-sm`, button header) để nét vẽ sắc nét, rõ ràng hơn.
  - `1.5` - `1.8`: Sử dụng khi hiển thị icon kích thước lớn (như Empty state, Hero banner).
- **`stroke-linecap="round"` & `stroke-linejoin="round"`**: Giữ các đầu nối và góc bo tròn đồng bộ.

---

## 3. Bảng Kích Thước Chuẩn (Standard Icon Sizes)

| Ngữ cảnh / Vị trí | Kích thước (`width` x `height`) | Độ dày `stroke-width` | Ví dụ áp dụng |
| :--- | :--- | :--- | :--- |
| **Nút hành động Header / Nút chính** | `16x16` (hoặc `18x18`) | `2.5` | Nút Thêm mới (+), Lưu, Xuất dữ liệu |
| **Thanh điều hướng Sidebar (`.nav-icon`)** | `18x18` (viewBox 24) | `2.0` | Menu Dashboard, Orders, Products, Customers |
| **Menu người dùng / Dropdown subpanel** | `18x18` hoặc `20x20` | `2.0` | UserMenu, Settings, ThemeSelector |
| **Caret / Chevron / Mũi tên lướt** | `14x14` đến `16x16` | `2.0` | Mũi tên xổ xuống, mũi tên chuyển trang |
| **Empty State / Khung thông báo rỗng** | `40x40` đến `48x48` | `1.5` - `2.0` | Biểu tượng không có dữ liệu trong Card/Table |

---

## 4. Chuẩn Tiếp Cận WAI-ARIA Cho Icon (Accessibility)

- **Icon mang tính trang trí (Đi kèm nhãn chữ)**:
  - Bắt buộc gắn thuộc tính `aria-hidden="true"` vào thẻ `<svg>` (hoặc container `<span class="icon-wrap">` bao bọc nó) để trình đọc màn hình bỏ qua, không đọc rác mã véc-tơ.
- **Icon-only Button / Link (Nút bấm chỉ có icon, không có chữ)**:
  - Thẻ `<button>` hoặc `<a>` bắt buộc phải có thuộc tính `aria-label="..."` mô tả rõ hành động (ví dụ: `aria-label="Đóng hộp thoại"` hoặc `aria-label="Chỉnh sửa sản phẩm"`).
  - Thẻ `<svg>` bên trong vẫn phải có `aria-hidden="true"`.

---

## 5. Bố Cục CSS & Căn Chỉnh (Layout & Alignment)

- Thêm `flex-shrink: 0` trên icon trong CSS (`.btn svg`, `.button svg`) để icon không bao giờ bị méo mó, co cụm khi text quá dài hoặc co giãn màn hình.
- Căn giữa trục dọc mượt mà với text bằng Flexbox: `display: inline-flex; align-items: center; gap: 6px;` (hoặc `vertical-align: middle;`).
- Với các icon tương tác tròn (như nút Giỏ hàng, Tài khoản, Theme toggle): Dùng class `.icon-button` hoặc `.icon-btn` với `border-radius: 9999px` để tạo hiệu ứng hover nền mềm mại không viền (ghost style).

---

## 6. Ví Dụ So Sánh Trực Quan

❌ **SAI** — Dùng FontAwesome/icon font, hardcode màu, thiếu `aria-hidden`:
```html
<!-- SAI: Thư viện font cồng kềnh, hardcode màu đỏ, thiếu trợ năng -->
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
<button type="button" class="btn btn-primary btn-sm">
    <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.5" aria-hidden="true">
        <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
    </svg>
    <span>Thêm thương hiệu</span>
</button>

<!-- ĐÚNG: Nút icon-only có aria-label trên button, svg ẩn khỏi screen reader -->
<button type="button" class="icon-btn" aria-label="Đóng bảng điều hướng">
    <svg width="20" height="20" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <path d="M6 18L18 6M6 6l12 12" />
    </svg>
</button>
```

