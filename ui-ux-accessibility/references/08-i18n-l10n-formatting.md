# Quốc Tế Hóa & Định Dạng Bản Địa Hóa (i18n & l10n)

Trong các hệ thống Enterprise đa quốc gia hoặc phục vụ nhiều vùng miền, việc hiển thị ngày tháng, tiền tệ, số lượng và văn bản phải tuân theo chuẩn địa phương hóa (Localization - l10n) của người dùng thay vì hardcode chuỗi định dạng thủ công.

---

## 1. Định Dạng Tiền Tệ & Con Số Bằng `Intl.NumberFormat`

Tuyệt đối không tự viết hàm format tiền tệ bằng regex hoặc ghép chuỗi thủ công (ví dụ: `price + " VND"` hay `price + " USD"`). Bắt buộc sử dụng API chuẩn của trình duyệt `Intl.NumberFormat` (hoặc `CultureInfo` trong .NET):

```typescript
// Tiền tệ Việt Nam
const formatVND = new Intl.NumberFormat('vi-VN', {
  style: 'currency',
  currency: 'VND'
});
console.log(formatVND.format(1500000)); // "1.500.000 ₫"

// Tiền tệ Mỹ
const formatUSD = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD'
});
console.log(formatUSD.format(1500.5)); // "$1,500.50"

// Phần trăm (Percentage)
const formatPercent = new Intl.NumberFormat('vi-VN', {
  style: 'percent',
  minimumFractionDigits: 1
});
console.log(formatPercent.format(0.125)); // "12,5%"
```

---

## 2. Định Dạng Ngày Giờ & Thời Gian Tương Đối (Relative Time)

### 2.1. Định dạng ngày giờ chuẩn
Sử dụng `Intl.DateTimeFormat` để tự động sắp xếp thứ tự ngày, tháng, năm theo thói quen của từng quốc gia (người Mỹ dùng `MM/DD/YYYY`, người Việt dùng `DD/MM/YYYY`, chuẩn ISO dùng `YYYY-MM-DD`):

```typescript
const formatDate = new Intl.DateTimeFormat('vi-VN', {
  year: 'numeric',
  month: '2-digit',
  day: '2-digit',
  hour: '2-digit',
  minute: '2-digit'
});
console.log(formatDate.format(new Date())); // "14:30 23/10/2026"
```

### 2.2. Thời gian tương đối kèm Tooltip tuyệt đối
- Khi hiển thị thời gian trong bảng hoặc danh sách thông báo (ví dụ: *"5 phút trước"*, *"hôm qua"*), dùng `Intl.RelativeTimeFormat`.
- **Nguyên tắc Enterprise**: Luôn bọc phần tử hiển thị thời gian tương đối trong thẻ HTML5 `<time>` với thuộc tính `datetime="[ISO_UTC_STRING]"` và tooltip (hoặc thuộc tính `title`) hiển thị chính xác ngày giờ tuyệt đối theo giờ UTC / múi giờ địa phương:

```html
<time datetime="2026-10-23T07:30:00Z" title="23/10/2026 14:30:00 (GMT+7)">
  10 phút trước
</time>
```

---

## 3. Quy Tắc Số Ít / Số Nhiều (Pluralization)

Nhiều ngôn ngữ có quy tắc chia số ít / số nhiều phức tạp (tiếng Anh có `1 item` và `2 items`; tiếng Ả Rập có tới 6 dạng số: zero, one, two, few, many, other; tiếng Việt không thay đổi danh từ).

- Không ghép chuỗi dạng: `count + " sản phẩm"` hoặc `count + " item(s)"`.
- Bắt buộc dùng `Intl.PluralRules` hoặc các thư viện i18n chuyên dụng (như `i18next`, `react-intl`, Fluent) để cung cấp đúng chuỗi ngôn ngữ theo từng ngữ cảnh số lượng.

```typescript
const enPlural = new Intl.PluralRules('en-US');
function getItemsLabel(count: number): string {
  const rule = enPlural.select(count);
  return rule === 'one' ? `${count} item` : `${count} items`;
}
```

---

## 4. Xử Lý Văn Bản Hai Chiều (Bi-directional Text & RTL)

Khi hiển thị dữ liệu kết hợp giữa tiếng Anh và tiếng Ả Rập / Do Thái (ví dụ: tên công ty tiếng Anh xen kẽ trong văn bản RTL):
- Sử dụng thẻ `<bdi>` (Bidirectional Isolation) xung quanh dữ liệu động do người dùng nhập để ngăn hiện tượng xáo trộn thứ tự đọc của văn bản bao quanh:
  ```html
  <p>Khách hàng: <bdi>ABC International Ltd.</bdi> đã đặt hàng thành công.</p>
  ```

