# Trình Bày Dữ Liệu Trong Bảng (Clean Data Representation in Tables)

Giao diện bảng dữ liệu (Data Table / Grid) trong các hệ thống doanh nghiệp, trang quản trị (Admin Dashboard) và ứng dụng người dùng chuyên nghiệp phải tuân thủ triết lý **Clean Data Representation** — hiển thị dữ liệu chính xác, chuyên nghiệp, tối giản và dễ đọc. Tuyệt đối không trang trí vụn vặt làm rối mắt và phân tán sự chú ý của người dùng.

---

## 1. Cấm Lạm Dụng Icon / Emoji Trang Trí Trong Ô Dữ Liệu

- Dữ liệu dạng danh từ riêng, tên địa danh, tên thực thể (ví dụ: tên tỉnh/thành phố, tên quận/huyện, tên xã/phường, tên danh mục, tên thương hiệu, tên sản phẩm, tên khách hàng...) **bắt buộc phải hiển thị dưới dạng văn bản thuần túy (Plain text)**.
- **Tuyệt đối không chèn thêm icon, emoji cảm xúc hoặc hình ảnh trang trí** phía trước hoặc phía sau nội dung văn bản dữ liệu.
- Biểu tượng (Icon/Emoji) chỉ được phép sử dụng ở các vị trí có chức năng tương tác rõ ràng (nút Sửa, nút Xóa, nút Xem chi tiết) hoặc chỉ báo trạng thái chuẩn (`.status-dot`).

### So sánh Bad vs Good:

❌ **SAI** — Chèn emoji/icon trang trí tùy tiện vào dữ liệu:
```html
<!-- SAI: Emoji chèn vào tên tỉnh / thành phố -->
<td>🏛️ Thành phố Hà Nội</td>
<td>🌴 Tỉnh Khánh Hòa</td>

<!-- SAI: Emoji chèn vào tên danh mục hoặc thương hiệu -->
<td>🏷️ Thời trang nam</td>
<td>📦 Nike Official Store</td>
<td>👤 Nguyễn Văn B</td>
```

✅ **ĐÚNG** — Hiển thị dữ liệu văn bản sạch sẽ, chuẩn mực:
```html
<!-- ĐÚNG: Văn bản thuần túy, sạch sẽ và chuyên nghiệp -->
<td>Thành phố Hà Nội</td>
<td>Tỉnh Khánh Hòa</td>
<td>Thời trang nam</td>
<td>Nike Official Store</td>
<td>Nguyễn Văn B</td>
```

---

## 2. Dữ Liệu Text & Mã Định Danh Phải Hiển Thị Thuần Túy

- Các trường thông tin dạng chuỗi ký tự, mã hành chính, mã vùng, mã bưu chính, số điện thoại, email, địa chỉ, mã SKU sản phẩm... **phải hiển thị dưới dạng text thông thường** (plain text hoặc thẻ `<code>` mộc nếu là chuỗi code định danh kỹ thuật).
- **Tuyệt đối không bọc thẻ `.badge`, `.chip`, `.tag` xung quanh các dữ liệu văn bản hoặc mã định danh thông thường** (như mã hành chính trong trang quản lý Tỉnh/TP, Quận/Huyện, Xã/Phường). Việc lạm dụng badge cho mọi loại dữ liệu sẽ làm mất đi điểm nhấn thị giác quan trọng nhất của bảng: trạng thái vòng đời.
- Thẻ `.badge` hoặc `.status-badge` **chỉ dành riêng cho các giá trị thực sự mang tính trạng thái vòng đời (Lifecycle Status / State)** (như: Active, Inactive, Paid, Pending, Shipping, Cancelled, Draft, Published...) hoặc nhãn tổng hợp đếm số lượng có ngữ nghĩa rõ ràng.

### So sánh Bad vs Good:

❌ **SAI** — Lạm dụng badge bọc mã hành chính và text thông thường:
```html
<!-- SAI: Mã hành chính và số điện thoại bị bọc badge làm rối mắt -->
<td>
  <span class="badge" style="background: var(--color-surface-2); color: var(--color-ink-2);">
    KH-079
  </span>
</td>
<td>
  <span class="badge badge-secondary">0912345678</span>
</td>
```

✅ **ĐÚNG** — Hiển thị mã định danh và text dạng plain text / code gọn gàng:
```html
<!-- ĐÚNG: Hiển thị text trực tiếp hoặc code đơn giản -->
<td><code class="text-mono font-bold">KH-079</code></td>
<td>0912 345 678</td>

<!-- ĐÚNG: Thẻ badge chỉ dùng cho trạng thái vòng đời -->
<td>
  <span class="status-badge status-active">Đã thanh toán</span>
</td>
```

---

## 3. Quy Chuẩn Màu Sắc Trạng Thái (Status Badges)

Khi sử dụng Badge cho trạng thái vòng đời, màu sắc phải tuân thủ nghiêm ngặt quy ước ngữ nghĩa toàn cầu:

| Loại Trạng Thái | Ý Nghĩa Ngữ Nghĩa | Màu Sắc Khuyến Nghị | Ví Dụ Áp Dụng |
| :--- | :--- | :--- | :--- |
| **Success / Positive** | Thành công, hoạt động tốt | Xanh lá (`green`) | `Hoạt động`, `Đã thanh toán`, `Đã giao hàng` |
| **Warning / Pending** | Chờ xử lý, cảnh báo nhẹ | Vàng cam (`amber` / `orange`) | `Chờ duyệt`, `Đang giao dịch`, `Tồn kho thấp` |
| **Danger / Failed** | Thất bại, hủy, lỗi | Đỏ (`red`) | `Đã hủy`, `Thanh toán lỗi`, `Khóa tài khoản` |
| **Neutral / Inactive** | Bản nháp, ngưng hoạt động | Xám (`slate` / `gray`) | `Bản nháp`, `Ngưng hoạt động`, `Lưu trữ` |
| **Info / Progress** | Đang tiến hành, thông tin | Xanh dương (`blue` / `sky`) | `Đang vận chuyển`, `Đang xử lý` |

