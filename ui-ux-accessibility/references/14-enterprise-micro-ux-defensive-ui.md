# Quy Chuẩn Micro-UX, Bố Cục Phòng Thủ & Luồng Tương Tác Chuẩn Big Tech (Google, Stripe, Linear, Apple)

Bên cạnh hiệu năng và trợ năng (WAI-ARIA), sự khác biệt cốt lõi giữa một ứng dụng "chạy được" và một sản phẩm Enterprise đẳng cấp thế giới (**Google Material Design 3, Google Cloud Console, Stripe Dashboard, Linear, Vercel, Apple HIG**) nằm ở **6 nguyên tắc thiết kế tương tác tinh tế (Micro-UX) và bố cục phòng thủ (Defensive UI)**.

Mọi giao diện được AI Agent sinh ra hoặc chỉnh sửa **bắt buộc phải tuân thủ 6 nguyên tắc này**.

---

## 1. Số Liệu Thẳng Hàng Tuyệt Đối (`Tabular Numbers`) & Ma Trận Căn Lề Bảng
*(Chuẩn Stripe Billing, Google Cloud Console & Bloomberg)*

### 1.1. Bắt Buộc `font-variant-numeric: tabular-nums` Cho Mọi Con Số
Với kiểu hiển thị chữ số mặc định (Proportional numerals), chữ số `1` có bề ngang hẹp hơn chữ số `8` hoặc `0`. Khi hiển thị trong bảng dữ liệu, cột tiền tệ, số lượng hoặc bộ đếm thời gian, các con số ở hàng trên và hàng dưới sẽ bị lệch hàng (зиг-zăc), gây mỏi mắt khi đối chiếu và làm rung giật layout khi số nhảy động.
- **Quy tắc bắt buộc**: Mọi thành phần hiển thị **số tiền, số lượng, tỷ lệ phần trăm, mã định danh số, ngày giờ, đồng hồ đếm ngược, thẻ KPI** bắt buộc phải áp dụng thuộc tính CSS `tabular-nums`:

```css
.tabular-nums,
.data-table td[data-type="numeric"],
.data-table th[data-type="numeric"],
.kpi-metric-value,
.price-tag,
time {
  font-variant-numeric: tabular-nums;
  font-feature-settings: "tnum" 1;
}
```

### 1.2. Ma Trận Căn Lề Bất Di Bất Dịch Trong Bảng Dữ Liệu (Data Grid Alignment)
Không bao giờ căn giữa (`text-align: center`) hoặc căn trái tùy tiện toàn bộ các cột của bảng dữ liệu. Tiêu đề cột (`<th>`) và ô dữ liệu (`<td>`) của cùng một cột **bắt buộc phải có cùng hướng căn lề** theo ma trận sau:

| Loại dữ liệu cột | Hướng căn lề (`<th>` & `<td>`) | CSS Class / Thuộc tính | Lý do UX & Khả năng quét mắt (Scannability) |
| :--- | :--- | :--- | :--- |
| **Văn bản tự nhiên**<br>*(Họ tên, Tên SP, Email, Địa chỉ, Ghi chú)* | **Căn trái** (`Start`) | `text-align: start;` | Mắt người đọc theo mẫu chữ F (F-Pattern) từ trái sang phải; căn trái tạo trục dọc cố định ở mép đầu dòng. |
| **Số lượng, Tiền tệ, Phần trăm, Dung lượng**<br>*(Đơn giá, Thành tiền, Tồn kho, VAT %)* | **Căn phải** (`End`) + `tabular-nums` | `text-align: end;`<br>`font-variant-numeric: tabular-nums;` | Giúp hàng đơn vị thẳng hàng đơn vị, hàng nghìn thẳng hàng nghìn, dấu thập phân thẳng trục dọc để so sánh độ lớn tức thì. |
| **Chuỗi độ dài cố định & Trạng thái**<br>*(Mã QG `VN`, Ngày `23/10/2026`, Badge trạng thái, STT)* | **Căn trái** hoặc **Căn giữa** đồng nhất | `text-align: start;` hoặc `text-align: center;` | Các chuỗi có số ký tự bằng nhau ở mọi hàng có thể căn giữa để tạo khoảng nghỉ thị giác giữa các cột chữ và cột số. |
| **Cột Hành Động (Actions)**<br>*(Nút Sửa, Menu `...`)* | **Căn phải** (`End`) + Ghim cố định (`Sticky right` nếu bảng rộng) | `text-align: end;` | Đưa cụm nút thao tác sát mép phải ngoài cùng của dòng, tránh bấm nhầm vào dữ liệu. |

---

## 2. Kiến Trúc Thao Tác Nguy Hiểm: "Undo Toast" vs "Confirmation AlertDialog"
*(Chuẩn Gmail, Google Drive, Linear & GitHub)*

Việc bật hộp thoại *"Bạn có chắc chắn muốn...?"* cho mọi thao tác xóa/lưu trữ sẽ gây hội chứng **nhờn cảnh báo (Confirmation Fatigue)** — người dùng bấm *Đồng ý* theo phản xạ vô thức rồi mới nhận ra mình bấm nhầm. Hệ thống bắt buộc phân tách thành **2 cấp độ xử lý**:

### 2.1. Cấp Độ 1: Thao Tác Có Thể Khôi Phục (Soft Delete, Archive, Đổi Trạng Thái, Bỏ Ghim)
- ❌ **CẤM**: Bật Modal/Popup xác nhận trước khi thực hiện.
- ✅ **BẮT BUỘC (Optimistic Action + Undo Toast)**:
  1. Cập nhật giao diện ngay lập tức (ẩn dòng vừa xóa mềm hoặc chuyển trạng thái tức thì).
  2. Hiển thị một thanh **Toast / Snackbar** ở góc dưới màn hình (`role="status"`, `aria-live="polite"`) trong **5 – 8 giây** kèm nút hành động **`"Hoàn tác (Undo)"`**.
  3. **Tạm dừng bộ đếm (`Pause on Hover / Focus`)**: Khi người dùng di chuột vào hoặc focus bàn phím vào Toast, đồng hồ tự đóng phải tạm dừng để người dùng kịp đọc và bấm nút `"Hoàn tác"`.
  4. **Giới hạn xếp chồng**: Tối đa hiển thị **3 Toast** cùng lúc (hoặc xếp chồng dạng thẻ thu gọn như `Sonner`), tuyệt đối không để danh sách Toast che kín nửa màn hình.

```html
<!-- Cấu trúc Undo Snackbar / Toast chuẩn Gmail & Linear -->
<div class="toast-region" role="region" aria-label="Thông báo hệ thống" aria-live="polite">
  <div class="toast-card" data-toast-item onmouseenter="pauseTimer()" onmouseleave="resumeTimer()">
    <span class="toast-message">Đã chuyển đơn hàng <strong>#ORD-8942</strong> vào thùng rác.</span>
    <div class="toast-actions">
      <button type="button" class="toast-undo-btn" onclick="undoDelete('ORD-8942')">Hoàn tác</button>
      <button type="button" class="toast-close-btn" aria-label="Đóng thông báo" onclick="dismissToast()">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
          <path stroke-linecap="round" stroke-linejoin="round" d="M18 6 6 18M6 6l12 12"/>
        </svg>
      </button>
    </div>
  </div>
</div>
```

### 2.2. Cấp Độ 2: Thao Tác Hủy Diệt Không Thể Khôi Phục (Hard Delete, Xóa Vĩnh Viễn, Hủy Dự Án, Thu Hồi API Key)
- ✅ **BẮT BUỘC dùng `role="alertdialog"`**:
  1. Tiêu đề nói rõ hành động và tên đối tượng bị tác động (VD: *"Xóa vĩnh viễn khách hàng Công ty TNHH ABC?"* thay vì chỉ ghi *"Xác nhận"*).
  2. Nội dung cảnh báo nêu rõ hậu quả không thể hoàn tác.
  3. **Focus mặc định (`autofocus`) bắt buộc đặt vào nút `"Hủy"` (Cancel)**, tuyệt đối không đặt focus mặc định vào nút `"Xóa"` để phòng người dùng vô tình gõ phím `Enter` hoặc `Space`.
  4. Nút hành động chính phải dùng nhãn động từ cụ thể (*"Xóa vĩnh viễn"*, *"Thu hồi khóa"*) kèm màu cảnh báo đỏ (`--color-danger: #dc2626`), không dùng chữ *"OK"* hay *"Đồng ý"* mơ hồ.
  5. **Cơ chế Type-to-Confirm (Dành cho tài nguyên tối quan trọng như Xóa Database / Workspace)**: Yêu cầu người dùng gõ lại chính xác mã/tên tài nguyên vào ô `<input>` thì nút `"Xóa vĩnh viễn"` mới chuyển từ `disabled` sang khả dụng.

---

## 3. Thiết Kế Trạng Thái Trống Hướng Hành Động (Actionable Empty States)
*(Chuẩn Google Workspace, Stripe & Shopify Polaris)*

- ❌ **CẤM**: Hiển thị một vùng trắng trơn hoặc một dòng chữ cụt lủn *"Không có dữ liệu"* / *"No data"* / *"Danh sách trống"*.
- ✅ **BẮT BUỘC phân biệt rõ 3 ngữ cảnh Empty State** với cấu trúc 4 phần (**Minh họa SVG tối giản + Tiêu đề rõ nguyên nhân + Mô tả hướng dẫn + Nút hành động CTA**):

| Ngữ cảnh Empty State | Nguyên nhân | Nội dung hiển thị & Nút hành động bắt buộc (CTA) |
| :--- | :--- | :--- |
| **1. Chưa có dữ liệu gốc**<br>*(First-time / Zero Data)* | Tài khoản mới hoặc phân hệ chưa từng tạo bản ghi nào. | - **Tiêu đề**: *"Chưa có hóa đơn nào được tạo"*<br>- **Mô tả**: Giải thích ngắn gọn lợi ích hoặc bước bắt đầu.<br>- **CTA chính**: Nút Primary **`"+ Tạo hóa đơn đầu tiên"`** hoặc **`"Nhập từ Excel (Import)"`**. |
| **2. Không khớp bộ lọc / tìm kiếm**<br>*(No Search / Filter Results)* | Người dùng đang gõ từ khóa hoặc chọn bộ lọc không có bản ghi thỏa mãn. | - **Tiêu đề**: *"Không tìm thấy kết quả cho 'Nguyễn Văn Z'"*<br>- **Mô tả**: Gợi ý kiểm tra lại chính tả hoặc nới lỏng điều kiện lọc.<br>- **CTA chính**: Nút Secondary **`"Xóa toàn bộ bộ lọc (Reset filters)"`** để khôi phục danh sách chỉ bằng 1 click. |
| **3. Lỗi tải dữ liệu cục bộ**<br>*(Fetch Error / Permission)* | Mất kết nối API hoặc người dùng chưa được cấp quyền xem. | - **Tiêu đề**: *"Không thể nạp danh sách giao dịch"*<br>- **Mô tả**: Thông báo lỗi thân thiện kèm mã tham chiếu (`traceId`).<br>- **CTA chính**: Nút **`"Thử lại (Retry)"`**. |

```html
<!-- Mẫu Empty State Khi Không Khớp Bộ Lọc (Chuẩn Stripe / Linear) -->
<div class="empty-state-container" role="status" aria-live="polite">
  <div class="empty-state-icon" aria-hidden="true">
    <svg width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
      <circle cx="11" cy="11" r="8" />
      <path stroke-linecap="round" d="m21 21-4.3-4.3M8 11h6" />
    </svg>
  </div>
  <h3 class="empty-state-title">Không tìm thấy đơn hàng phù hợp</h3>
  <p class="empty-state-desc">
    Không có đơn hàng nào khớp với từ khóa <strong>"DH-9999"</strong> và bộ lọc trạng thái hiện tại.
  </p>
  <div class="empty-state-actions">
    <button type="button" class="btn btn-secondary" onclick="resetAllFilters()">
      Xóa bộ lọc & tìm kiếm
    </button>
  </div>
</div>
```

---

## 4. Phân Cấp Nút Bấm (Button Hierarchy) & Chống Bấm Đúp Gửi Lặp (Idempotent Button Lock)
*(Chuẩn Stripe Checkout & Google Material 3)*

### 4.1. Quy Tắc "1 Primary CTA" Trên Mỗi Vùng Nhìn (Hick's Law)
- Trong một biểu mẫu, một thanh công cụ (Toolbar), một Card hoặc một hộp thoại Modal, **chỉ được phép có duy nhất 1 nút mang cấp độ `Primary` (màu nền đặc nổi bật nhất)**.
- Các hành động phụ đi kèm bắt buộc phải hạ cấp thị giác rõ rệt theo 4 tầng:
  1. **Primary (`btn-primary`)**: Hành động mục tiêu quan trọng nhất của màn hình (VD: *"Lưu thay đổi"*, *"Tạo đơn hàng"*).
  2. **Secondary (`btn-secondary` / `Outline` / `Tonal`)**: Hành động phổ biến thứ hai (VD: *"Xuất Excel"*, *"Lưu bản nháp"*).
  3. **Ghost / Tertiary (`btn-ghost`)**: Hành động bổ trợ hoặc nút *"Hủy bỏ (Cancel)"*, *"Đặt lại"*.
  4. **Destructive (`btn-danger`)**: Chỉ dùng cho hành động xóa vĩnh viễn / hủy bỏ nguy hiểm.
- **Thứ tự đặt nút trong Modal / Form Footer**:
  - Nút `Hủy (Cancel)` nằm bên trái, nút `Chính (Primary CTA)` nằm ngoài cùng bên phải (theo luồng đọc kết thúc ở góc dưới phải).

### 4.2. Khóa Nút Tức Thì Khi Submit (Double-Submit Prevention & Zero-Shift Spinner)
Khi người dùng bấm nút gửi Form (`Submit`, `Thanh toán`, `Phê duyệt`):
- ❌ **CẤM**: Để nút bấm vẫn có thể click tiếp trong lúc chờ API phản hồi (gây lỗi tạo 2 bản ghi trùng lặp hoặc trừ tiền 2 lần).
- ❌ **CẤM**: Thay đổi nội dung nút làm chiều rộng nút bị co rút hoặc giãn ra gây giật các nút đứng cạnh.
- ✅ **BẮT BUỘC**:
  1. Chuyển ngay nút sang trạng thái `disabled`, `aria-disabled="true"`, `aria-busy="true"` ngay tại mili-giây đầu tiên khi kích hoạt sự kiện submit.
  2. Hiển thị `Inline Spinner` SVG xoay mượt nhưng **giữ nguyên kích thước nút** (dùng kỹ thuật ẩn chữ bằng `visibility: hidden` và đặt spinner ở `position: absolute` chính giữa nút).

```css
/* Kỹ thuật giữ nguyên chiều rộng Button khi bật Loading Spinner (Zero Layout Shift) */
.btn-submit {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.btn-submit[data-loading="true"] {
  pointer-events: none;
  opacity: 0.85;
}

.btn-submit[data-loading="true"] .btn-label {
  visibility: hidden; /* Giữ nguyên diện tích chiều rộng của chữ nhưng ẩn đi */
}

.btn-submit .btn-spinner {
  position: absolute;
  display: none;
  animation: spin 0.75s linear infinite;
}

.btn-submit[data-loading="true"] .btn-spinner {
  display: inline-flex;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

---

## 5. Ngăn Kéo Chi Tiết (Slide-over Drawer / Sheet) & Cấm Modal Chồng Modal
*(Chuẩn Google Cloud Console, Linear & Vercel)*

### 5.1. Nghiêm Cấm "Modal Chồng Lên Modal" (No Nested Modals)
- ❌ **CẤM TUYỆT ĐỐI**: Mở một hộp thoại Modal thứ 2 đè lên trên một hộp thoại Modal thứ 1 đang mở (trừ trường hợp duy nhất là hộp thoại `alertdialog` nhỏ xác nhận *"Bạn có muốn hủy các thay đổi chưa lưu?"* khi bấm đóng Modal 1).
- *Lý do*: Hai lớp nền tối (`backdrop`) chồng nhau gây rối loạn lịch sử phím `Escape`, làm hỏng `Focus Trap` của trình đọc màn hình và che khuất hoàn toàn ngữ cảnh trên màn hình nhỏ.

### 5.2. Sử Dụng Slide-over Drawer / Side Sheet Cho Luồng Xem Nhanh & Form Nhiều Trường
Khi người dùng thao tác trên bảng dữ liệu lớn và muốn **Xem chi tiết bản ghi**, **Chỉnh sửa thông tin đầy đủ**, hoặc **Mở bộ lọc nâng cao (Advanced Filters)**:
- Thay vì dùng Modal giữa màn hình (bị giới hạn chiều cao) hoặc chuyển hẳn sang trang mới (làm mất trạng thái cuộn và bộ lọc của bảng):
- ✅ **BẮT BUỘC ưu tiên dùng Side Sheet / Drawer** (ngăn kéo trượt ra từ cạnh phải màn hình, `inset-inline-end: 0`, chiều cao `100dvh`, chiều rộng `420px – 640px`):
  - Có Header cố định ở đỉnh (`position: sticky; top: 0`), vùng nội dung cuộn độc lập ở giữa (`overflow-y: auto`), và Footer chứa nút `Lưu / Hủy` ghim cố định ở đáy (`position: sticky; bottom: 0`).
  - Vẫn tuân thủ đầy đủ `role="dialog"`, `aria-modal="true"`, `Focus Trap` và nhấn `Escape` để đóng giống như Modal.

---

## 6. Định Luật Fitts (Hit Area), Mật Độ Không Gian & Bố Cục Phòng Thủ (Defensive CSS)
*(Chuẩn Apple Human Interface Guidelines & Google Material 3)*

### 6.1. Vùng Chạm Tối Thiểu (Minimum Target Size - WCAG 2.2 / Material 3)
- Mọi phần tử tương tác (Nút bấm icon-only, Checkbox, Radio, Nút đóng `×`, Nút phân trang) dù kích thước biểu tượng đồ họa chỉ là `16x16px` hoặc `20x20px`, **vùng diện tích nhận cú nhấp chuột / chạm ngón tay thực tế bắt buộc đạt tối thiểu**:
  - **Desktop**: $\ge 36 \times 36\text{px}$ (khuyến nghị $40 \times 40\text{px}$).
  - **Mobile / Touch**: $\ge 44 \times 44\text{px}$ (Apple HIG) đến $48 \times 48\text{px}$ (Google Material 3).
- Khoảng cách tối thiểu giữa 2 nút hành động đặt cạnh nhau (đặc biệt là nút `"Sửa"` và nút `"Xóa"`) phải $\ge 8\text{px}$ (`gap: 0.5rem`) để ngăn chặn bấm nhầm.

```css
/* Mở rộng vùng bấm (Hit Area) lên 44x44px cho nút icon nhỏ mà không làm lệch bố cục */
.btn-icon-compact {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2rem;  /* 32px hiển thị */
  height: 2rem;
}

.btn-icon-compact::before {
  content: "";
  position: absolute;
  inset: -6px; /* Mở rộng vùng nhận click thêm 6px mỗi chiều -> đạt 44x44px */
}
```

### 6.2. Bố Cục Phòng Thủ Chống Vỡ Giao Diện (Defensive CSS & Text Truncation)
Dữ liệu thực tế từ người dùng không bao giờ hoàn hảo như bản thiết kế (tên dài 120 ký tự, chuỗi URL/Token dài không có khoảng trắng, tiêu đề 1 từ hoặc 20 từ). Giao diện bắt buộc phải có lớp phòng thủ:
1. **Chống tràn Flexbox / Grid (`min-width: 0`)**:
   - Mọi phần tử con chứa văn bản nằm trong `display: flex` hoặc `display: grid` bắt buộc phải có `min-width: 0` để thuộc tính cắt chữ (`text-overflow: ellipsis`) hoạt động và không đẩy vỡ cột cha.
2. **Cắt chữ có kiểm soát kèm Tooltip (`Line Clamping`)**:
   - Với ô dữ liệu đơn dòng (Tên, Email, Địa chỉ trong bảng): Dùng `.truncate` (`white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 260px`) kèm thuộc tính `title="[nội_dung_đầy_đủ]"` hoặc Tooltip tùy biến.
   - Với thẻ Card (Tên sản phẩm, Mô tả ngắn): Khóa cố định tối đa 2 dòng bằng `-webkit-line-clamp: 2` để các Card trên cùng một hàng luôn cao bằng nhau.
3. **Ngắt từ an toàn cho chuỗi dài không khoảng trắng (URL, Hash, Email)**:
   - Áp dụng `overflow-wrap: anywhere;` hoặc `word-break: break-word;` cho các khung hiển thị nội dung chi tiết, bình luận, log hệ thống để chuỗi URL dài không làm xuất hiện thanh cuộn ngang ngoài ý muốn.

---

## 7. Bảng Tổng Hợp Kiểm Tra Nhanh (Dos and Don'ts)

| Chuẩn Thiết Kế Big Tech (DO) | Sai Lầm Phổ Biến Cần Cấm (DON'T) |
| :--- | :--- |
| ✅ **Cột số/tiền tệ căn phải + `tabular-nums`**: Các chữ số và dấu phẩy thập phân thẳng tắp theo trục dọc. | ❌ **Căn giữa hoặc căn trái cột tiền tệ / dùng font chữ số lệch**: Làm người dùng không thể so sánh nhanh độ lớn các con số. |
| ✅ **Xóa mềm dùng Undo Toast (5–8s), xóa vĩnh viễn dùng AlertDialog (focus mặc định ở nút Hủy)**. | ❌ **Lạm dụng Popup "Bạn có chắc muốn xóa?" cho mọi thao tác nhỏ**, hoặc đặt focus mặc định vào nút Xóa đỏ. |
| ✅ **Empty State có phân loại & kèm nút CTA hành động** (*"+ Tạo mới"* hoặc *"Xóa bộ lọc"*). | ❌ **Chỉ hiện dòng chữ khô khan "Không có dữ liệu" (No data)** bỏ mặc người dùng không biết làm gì tiếp. |
| ✅ **Chỉ 1 nút Primary CTA mỗi vùng nhìn + Khóa nút & giữ nguyên độ rộng khi đang Submit**. | ❌ **Đặt 3–4 nút màu đậm cạnh nhau** hoặc để người dùng bấm đúp nút Lưu tạo ra 2 bản ghi trùng lặp. |
| ✅ **Dùng Side Sheet / Drawer cho xem chi tiết & sửa nhanh trên bảng; cấm Modal chồng Modal**. | ❌ **Mở Modal thứ 2 đè lên Modal thứ 1** gây rối loạn Focus Trap và che kín màn hình. |
| ✅ **Đảm bảo vùng chạm $\ge 40\text{px}$ và phòng thủ `min-width: 0` + `ellipsis` cho văn bản dài**. | ❌ **Để tên khách hàng hoặc đường link dài làm vỡ layout Card / đẩy lệch cột bảng**. |

