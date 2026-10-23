# Kiến Trúc Biểu Mẫu & Xác Thực Dữ Liệu (Form Architecture & Validation)

Trong các ứng dụng Enterprise, biểu mẫu nhập liệu (Forms) chiếm đến 60–70% thời gian thao tác của người dùng. Một hệ thống form chuẩn mực phải đảm bảo: chống mất dữ liệu của người dùng, xác thực trực quan (A11y-compliant), không bị giật lag khi gõ phím và hỗ trợ kiểm tra tính duy nhất bất đồng bộ.

---

## 1. Cơ Chế Chống Mất Dữ Liệu (Unsaved Changes Guard)

Khi người dùng đã bắt đầu chỉnh sửa dữ liệu trong form (trạng thái **Dirty State**), nếu họ vô tình:
1. Bấm nút Back / Forward trên trình duyệt.
2. Đóng tab hoặc tắt trình duyệt.
3. Bấm chuyển trang trong SPA router.

Hệ thống **bắt buộc phải hiển thị cảnh báo xác nhận** trước khi cho phép rời đi.

### Ví dụ JavaScript / TypeScript (Native & SPA Guard):
```typescript
// 1. Chặn đóng tab / tải lại trang trình duyệt
window.addEventListener('beforeunload', (event) => {
  if (isFormDirty) {
    event.preventDefault();
    event.returnValue = ''; // Kích hoạt dialog xác nhận mặc định của trình duyệt
  }
});

// 2. Chặn chuyển Route trong SPA (React Router v6 / Next.js / Vue Router)
// Hiển thị Custom Modal xác nhận: "Bạn có thay đổi chưa lưu. Bạn có chắc chắn muốn rời đi?"
```

---

## 2. Xác Thực Trực Quan Chuẩn Trợ Năng (Accessible Inline Validation)

Khi một ô nhập liệu có lỗi xác thực:
- Thẻ `<input>`, `<textarea>` hoặc nút Trigger của **Custom Select / Combobox** (`role="combobox"`) bắt buộc phải có thuộc tính `aria-invalid="true"`.
- Thẻ hiển thị thông báo lỗi phải có `id` duy nhất và được liên kết với ô nhập thông qua `aria-describedby="[error_id]"`.
- Không chỉ dựa vào màu đỏ để báo lỗi (vi phạm WCAG cho người mù màu); bắt buộc có nhãn văn bản giải thích rõ nguyên nhân và cách khắc phục.

```html
<!-- Cấu trúc Form Field chuẩn WAI-ARIA -->
<div class="form-field">
  <label for="txt-email" class="form-label">
    Email công ty <span class="required" aria-hidden="true">*</span>
  </label>
  
  <input 
    type="email" 
    id="txt-email" 
    name="email"
    class="input-control input-error" 
    aria-invalid="true" 
    aria-describedby="email-error-msg email-hint-msg"
    aria-required="true"
    value="invalid-email@" />

  <span id="email-hint-msg" class="hint-text">Dùng email có đuôi @company.com</span>
  
  <div id="email-error-msg" class="error-message" role="alert">
    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
      <circle cx="12" cy="12" r="10" />
      <line x1="12" y1="8" x2="12" y2="12" />
      <line x1="12" y1="16" x2="12.01" y2="16" />
    </svg>
    <span>Email không đúng định dạng chuẩn (ví dụ: ten@congty.com).</span>
  </div>
</div>
```

---

## 3. Xác Thực Bất Đồng Bộ Giảm Tải (Debounced Async Validation)

Khi cần kiểm tra trùng lặp (ví dụ: kiểm tra mã nhân viên, mã sản phẩm SKU, tên miền hoặc số điện thoại đã tồn tại trong database):
- ❌ **CẤM**: Gửi request kiểm tra ngay sau mỗi ký tự gõ.
- ✅ **BẮT BUỘC**: Áp dụng Debounce tối thiểu **400ms – 500ms** sau khi người dùng ngừng gõ (hoặc chỉ kiểm tra khi sự kiện `onBlur` xảy ra).
- Hiển thị chỉ báo đang kiểm tra nhỏ gọn (Inline spinner hoặc text *"Đang kiểm tra tính khả dụng..."*) kèm `aria-busy="true"`.

---

## 4. Tối Ưu Hiệu Năng Form Lớn (Tránh Re-render Toàn Cây DOM)

Trong các form doanh nghiệp phức tạp (50 – 100 trường nhập liệu, bảng nhập chi tiết hợp đồng):
- Không lưu toàn bộ trạng thái của form vào một biến state duy nhất ở component cha khiến mỗi lần người dùng gõ 1 phím là toàn bộ 100 component con bị re-render lại.
- **Giải pháp**:
  - **React**: Sử dụng `react-hook-form` (Uncontrolled inputs) để cô lập việc render tại từng ô input độc lập.
  - **Blazor**: Dùng `EditForm` đi kèm `InputText` với cơ chế binding hợp lý, tránh gọi `StateHasChanged()` trên toàn trang khi thay đổi field con.
  - **Vue**: Sử dụng `v-model.lazy` hoặc form context biệt lập.

---

## 5. Quy Chuẩn Chọn Dữ Liệu Trong Biểu Mẫu (Cấm Native `<select>`, Dùng Custom Select / Combobox)

- ❌ **CẤM sử dụng thẻ `<select>` / `<option>` mặc định của hệ thống** trên giao diện biểu mẫu.
- ✅ **BẮT BUỘC phân tầng theo số lượng dữ liệu** (chi tiết tại [13-custom-select-combobox.md](./13-custom-select-combobox.md)):
  - **Dưới 8 mục cố định (< 8 items)**: Dùng **Custom Select (`role="listbox"`)** có icon `ChevronDown` xoay khi mở, dấu tích `Check` ở mục đang chọn và đồng bộ với `<input type="hidden">`.
  - **Từ 8 mục trở lên ($\ge 8$ items) hoặc dữ liệu động từ Server**: Dùng **Searchable Combobox (`Popover` + Search Input)** tự động focus vào ô tìm kiếm khi mở, hỗ trợ Debounce 300ms và nút xóa nhanh (`Clear ×`).

---

## 6. Phân Cấp Nút Bấm (1 Primary CTA) & Chống Bấm Đúp Gửi Lặp (Double-Submit Lock)

Theo chuẩn Stripe Checkout & Google Material 3 (chi tiết tại [14-enterprise-micro-ux-defensive-ui.md](./14-enterprise-micro-ux-defensive-ui.md)):
- **Chỉ 1 nút Primary CTA trên mỗi Form / Modal**: Các nút phụ như *"Hủy"*, *"Lưu nháp"*, *"Đặt lại"* bắt buộc hạ cấp xuống `Secondary` hoặc `Ghost`. Đặt nút *"Hủy"* bên trái, nút *"Lưu / Gửi"* (Primary) ngoài cùng bên phải.
- **Khóa nút tức thì khi Submit (Idempotent Button Lock)**: Ngay khi bấm Submit, chuyển nút sang `disabled` (`aria-disabled="true"`, `aria-busy="true"`), hiển thị `Inline Spinner` nhưng **giữ nguyên chiều rộng nút** (`visibility: hidden` trên nhãn chữ + `position: absolute` cho spinner) để chống gửi trùng request và không làm co giật layout.

