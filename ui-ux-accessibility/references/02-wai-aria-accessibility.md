# Chuẩn Tiếp Cận Web WAI-ARIA (Web Accessibility Initiative - WCAG 2.1 AA)

Mọi thành phần giao diện (đặc biệt là Table, Modal, Dropdown, Form Controls) **bắt buộc phải tuân thủ chuẩn tiếp cận WAI-ARIA (WCAG 2.1 AA)** để đảm bảo người khiếm thị, người dùng sử dụng trình đọc màn hình (Screen Reader như NVDA, JAWS, VoiceOver) hoặc người dùng điều hướng hoàn toàn bằng bàn phím (Keyboard navigation) có thể tiếp cận và thao tác đầy đủ mọi chức năng.

---

## 1. Quy Chuẩn WAI-ARIA Cho Table / Data Grid

### 1.1. Cấu Trúc Ngữ Nghĩa Chuẩn (Semantic HTML5)
- Sử dụng đúng hệ thống thẻ ngữ nghĩa của HTML5: `<table>`, `<thead>`, `<tbody>`, `<tfoot>`, `<tr>`, `<th>`, `<td>`. Tránh dựng bảng bằng các thẻ `<div>` lồng nhau trừ khi có khai báo đầy đủ các role ARIA (`role="table"`, `role="row"`, `role="cell"`...).
- Bảng bắt buộc phải có tên/nhãn mô tả chức năng thông qua:
  - Thẻ `<caption>Bảng danh sách đơn hàng</caption>` đặt ngay sau thẻ `<table>`.
  - Hoặc thuộc tính `aria-label="Danh sách người dùng"` trên thẻ `<table>`.
  - Hoặc `aria-labelledby="[id_tieu_de]"` liên kết với tiêu đề phần tử ngoài.

### 1.2. Tiêu Đề Cột (Column Headers)
- Mọi thẻ `<th>` bắt buộc phải có thuộc tính `scope="col"` (nếu là tiêu đề cột) hoặc `scope="row"` (nếu là tiêu đề của dòng dữ liệu).
- Khi cột có chức năng sắp xếp (Sorting):
  - Khai báo thuộc tính `aria-sort="ascending"` (tăng dần), `aria-sort="descending"` (giảm dần), hoặc `aria-sort="none"` (chưa sắp xếp).

### 1.3. Trạng Thái Động (Live Region & Busy State)
- **Khi bảng đang tải dữ liệu**: Thẻ `<table>` hoặc container bao quanh phải có `aria-busy="true"`. Khi tải xong, cập nhật lại thành `aria-busy="false"`.
- **Khu vực hiển thị tổng số kết quả** (ví dụ: *"Hiển thị 1 - 20 trên tổng số 1,450 đơn hàng"*):
  - Bắt buộc gắn `aria-live="polite"` và `aria-atomic="true"` để trình đọc màn hình đọc thông báo cập nhật cho người dùng mà không ngắt quãng âm thanh đang phát.

```html
<!-- Cấu trúc HTML Data Table chuẩn WCAG 2.1 AA -->
<div class="table-container" aria-busy="false">
  <table class="data-table" aria-label="Danh sách tài khoản">
    <thead>
      <tr>
        <th scope="col" aria-sort="ascending">Họ và tên</th>
        <th scope="col">Email</th>
        <th scope="col">Trạng thái</th>
        <th scope="col" class="text-right">Hành động</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Nguyễn Văn A</td>
        <td>a.nguyen@example.com</td>
        <td><span class="status-badge status-active">Hoạt động</span></td>
        <td class="text-right">
          <button type="button" class="btn-icon" aria-label="Chỉnh sửa tài khoản Nguyễn Văn A">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
              <!-- icon edit -->
            </svg>
          </button>
        </td>
      </tr>
    </tbody>
  </table>
  <div class="table-footer" aria-live="polite" aria-atomic="true">
    <span>Tổng số: 1,450 tài khoản</span>
  </div>
</div>
```

---

## 2. Quy Chuẩn WAI-ARIA Cho Modal / Dialog

- **Role & Thuộc tính cơ bản**:
  - Container chính của Modal bắt buộc có `role="dialog"` (hoặc `role="alertdialog"` đối với hộp thoại xác nhận hành động nguy hiểm như Xóa vĩnh viễn dữ liệu).
  - Thuộc tính `aria-modal="true"` bắt buộc phải có để báo hiệu cho trình đọc màn hình biết rằng nội dung nền bên ngoài đang bị vô hiệu hóa.
- **Tiêu đề và mô tả**:
  - `aria-labelledby="modal-title-id"`: Trỏ đến thẻ tiêu đề của modal (`<h2 id="modal-title-id">...</h2>`).
  - `aria-describedby="modal-desc-id"`: Trỏ đến đoạn văn bản giải thích hoặc hướng dẫn bên dưới tiêu đề (nếu có).
- **Nút đóng Modal**:
  - Phải có nhãn văn bản tường minh bằng `aria-label="Đóng"` hoặc `aria-label="Close"`.
  - ❌ **CẤM**: Để nút đóng chỉ chứa ký tự `&times;` hoặc `×` mà không có nhãn văn bản.
- **Bẫy tiêu điểm (Focus Trap) & Phím Escape**:
  - Khi Modal mở ra, tiêu điểm bàn phím (Focus) tự động chuyển vào phần tử tương tác đầu tiên bên trong modal (thường là ô input đầu tiên hoặc nút đóng).
  - Nhấn phím `Tab` hoặc `Shift + Tab` chỉ được di chuyển luẩn quẩn bên trong Modal, không được lọt ra ngoài nền trang.
  - Nhấn phím `Escape` (`Esc`) phải đóng modal ngay lập tức và trả tiêu điểm về phần tử đã kích hoạt modal trước đó.

```html
<!-- Cấu trúc Modal chuẩn WAI-ARIA -->
<div class="modal-backdrop">
  <div role="dialog" 
       aria-modal="true" 
       aria-labelledby="create-user-title" 
       aria-describedby="create-user-desc" 
       class="modal-dialog">
    <div class="modal-header">
      <h2 id="create-user-title">Thêm người dùng mới</h2>
      <button type="button" class="btn-close" aria-label="Đóng hộp thoại" onclick="closeModal()">
        <svg width="20" height="20" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2" aria-hidden="true">
          <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
        </svg>
      </button>
    </div>
    <div class="modal-body" id="create-user-desc">
      Điền đầy đủ các thông tin cá nhân dưới đây để kích hoạt tài khoản hệ thống.
      <!-- Form inputs -->
    </div>
  </div>
</div>
```

---

## 3. Quy Chuẩn WAI-ARIA Cho Dropdown & Custom Form Controls

Khi xây dựng các menu thả xuống tùy biến (Custom Select / Dropdown):
- **Nút kích hoạt (Trigger Button)**:
  - Bắt buộc có thuộc tính `aria-haspopup="listbox"` (nếu là chọn giá trị) hoặc `aria-haspopup="menu"` (nếu là menu chức năng).
  - Có thuộc tính `aria-expanded="false"` khi đóng và `aria-expanded="true"` khi mở.
- **Danh sách lựa chọn (Option List Container)**:
  - Container danh sách phải mang thuộc tính `role="listbox"`.
  - Mỗi mục lựa chọn bên trong mang `role="option"`.
  - Mục hiện đang được chọn phải có thuộc tính `aria-selected="true"`, các mục còn lại là `aria-selected="false"`.
- **Hành vi bàn phím (Keyboard Navigation)**:
  - `ArrowDown` / `ArrowUp`: Di chuyển lựa chọn giữa các item.
  - `Enter` hoặc `Space`: Chọn item đang focus và đóng dropdown.
  - `Escape`: Đóng dropdown và trả focus về nút kích hoạt ban đầu.

---

## 4. Nút Bấm Chỉ Có Biểu Tượng (Icon-Only Buttons)

Mọi nút bấm chỉ có biểu tượng icon SVG (không hiển thị chữ cho mắt thường):
- Bắt buộc phải có thuộc tính `aria-label="..."` mô tả chính xác hành động (ví dụ: `aria-label="Xóa đơn hàng #1234"`).
- Hoặc bên trong nút phải chứa một thẻ văn bản ẩn bằng CSS chuyên dụng cho trợ năng:
  ```html
  <button type="button" class="icon-btn">
    <svg aria-hidden="true">...</svg>
    <span class="sr-only">Tải lại bảng dữ liệu</span>
  </button>
  ```
- CSS `.sr-only` chuẩn:
  ```css
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }
  ```

---

## 5. Bảng Tổng Hợp Thuộc Tính & Hành Vi Bàn Phím

| Thành phần UI | Thuộc tính WAI-ARIA bắt buộc | Hành vi bàn phím bắt buộc |
| :--- | :--- | :--- |
| **Table / Data Grid** | `aria-label` / `<caption>`, `th[scope="col"]`, `aria-busy`, `aria-live="polite"` | Phím `Tab` điều hướng lần lượt qua các nút hành động trong ô |
| **Modal / Dialog** | `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, `aria-describedby` | Phím `Escape` đóng modal, hỗ trợ Focus Trap bên trong |
| **Custom Dropdown** | `aria-haspopup`, `aria-expanded`, `role="listbox"`, `role="option"`, `aria-selected` | `ArrowUp`/`ArrowDown` chuyển mục, `Enter` chọn, `Escape` đóng |
| **Icon-only Button** | `aria-label="..."` hoặc class `.sr-only` | `Enter` hoặc `Space` kích hoạt hành động |
