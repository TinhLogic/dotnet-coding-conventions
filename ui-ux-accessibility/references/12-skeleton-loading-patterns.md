# Quy Chuẩn Thiết Kế & Lập Trình Skeleton Loading (Content-Aware Shimmer & A11y)

Skeleton Loading (Khung xương tải dữ liệu) là tiêu chuẩn vàng trong thiết kế giao diện hiện đại để thay thế cho Spinner xoay vòng toàn màn hình. Một hệ thống Skeleton chuẩn Enterprise phải đạt được 3 mục tiêu cốt lõi:
1. **Triệt tiêu giật nhảy giao diện (Zero Layout Shift - CLS = 0)**.
2. **Tuân thủ chuẩn tiếp cận WAI-ARIA (Screen Reader Friendly)**.
3. **Hiệu ứng chuyển động mượt mà (GPU-accelerated), tôn trọng `prefers-reduced-motion` và hỗ trợ Light/Dark mode**.

---

## 1. Triết Lý Khớp Hình Dạng Nội Dung (Content-Aware Skeleton)

Skeleton không phải là những khối chữ nhật vẽ tùy tiện. Mỗi khối Skeleton phải mô phỏng chính xác kích thước, khoảng cách và tỷ lệ của phần tử nội dung thật sắp xuất hiện:

| Thành Phần Thật | Cấu Trúc Khối Skeleton Tương Ứng | Quy Cách Kỹ Thuật |
| :--- | :--- | :--- |
| **Dòng chữ (Text)** | `.skeleton-text` | Chiều cao khớp với `font-size` hoặc `line-height` (thường là `1rem` hoặc `0.875rem`), bo góc nhẹ `4px`. Dòng cuối cùng của đoạn văn nên có chiều dài ngắn hơn (~60% – 75%) để tạo cảm giác tự nhiên. |
| **Tiêu đề (Heading)** | `.skeleton-heading` | Chiều cao lớn hơn (`1.5rem` – `2rem`), chiều rộng khoảng 40% – 60% chiều ngang container. |
| **Ảnh đại diện (Avatar)** | `.skeleton-avatar` | Hình tròn `border-radius: 9999px` hoặc bo góc vuông khớp chính xác `width` x `height` của avatar thật (`32x32`, `40x40`, `48x48`). |
| **Hình ảnh / Banner** | `.skeleton-media` | Cố định tỷ lệ khung hình (`aspect-ratio: 16/9` hoặc `aspect-ratio: 4/3`) để đảm bảo không bị giật layout khi ảnh thật tải xong. |
| **Nút bấm (Button)** | `.skeleton-btn` | Khớp chiều cao chuẩn của button (`36px` hoặc `40px`), bo góc đồng bộ với nút thật. |
| **Thẻ trạng thái (Badge)** | `.skeleton-badge` | Chiều cao `20px` – `24px`, chiều rộng `60px` – `80px`, bo viên thuốc (`border-radius: 9999px`). |

---

## 2. Chuẩn Tiếp Cận WAI-ARIA Cho Skeleton Loading

Screen Reader (trình đọc màn hình) cho người khiếm thị không thể đọc hiểu các thẻ `<div>` nhấp nháy vô nghĩa. Nếu không khai báo WAI-ARIA đúng cách, người dùng khiếm thị sẽ bị nghe đọc hàng chục thẻ div rác hoặc không hề biết trang đang tải dữ liệu.

### Quy tắc WAI-ARIA bắt buộc:
1. **Container bao bọc toàn bộ khối Skeleton**:
   - Bắt buộc khai báo `aria-busy="true"` khi đang tải và `aria-busy="false"` khi tải xong.
   - Thêm thuộc tính `aria-live="polite"` để trình đọc màn hình thông báo trạng thái cập nhật cho người dùng.
2. **Tất cả các khối Skeleton con**:
   - Bắt buộc gắn thuộc tính `aria-hidden="true"` để trình đọc màn hình bỏ qua hoàn toàn các khối hình học trang trí này.
3. **Thông báo văn bản ẩn cho Trợ năng (Screen Reader Only)**:
   - Đặt một thẻ thông báo văn bản sử dụng class `.sr-only` bên trong container để Screen Reader đọc rõ ràng ngữ cảnh:
   ```html
   <span class="sr-only">Đang nạp dữ liệu chi tiết người dùng, vui lòng chờ...</span>
   ```
4. **Khi dữ liệu tải thành công**:
   - Gỡ bỏ khối Skeleton, chuyển `aria-busy="false"` và hiển thị nội dung thật.

---

## 3. Kỹ Thuật CSS Shimmer Animation (Tối Ưu GPU & Chống Động Kinh)

### 3.1. Biến CSS và Hiệu ứng Quét Ánh Sáng (Shimmer Wave)
Sử dụng CSS Custom Properties để tự động thích ứng giữa Light Mode và Dark Mode:

```css
:root {
  --skeleton-base: #e2e8f0;
  --skeleton-highlight: #f1f5f9;
}

[data-theme="dark"] {
  --skeleton-base: #1e293b;
  --skeleton-highlight: #334155;
}

/* Base Skeleton Class */
.skeleton {
  position: relative;
  overflow: hidden;
  background-color: var(--skeleton-base);
  border-radius: var(--radius-md, 4px);
}

/* Hiệu ứng tia sáng Shimmer quét ngang mượt mà */
.skeleton::after {
  content: "";
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  transform: translateX(-100%);
  background-image: linear-gradient(
    90deg,
    rgba(255, 255, 255, 0) 0,
    rgba(255, 255, 255, 0.4) 50%,
    rgba(255, 255, 255, 0) 100%
  );
  animation: shimmer 1.6s infinite;
}

[data-theme="dark"] .skeleton::after {
  background-image: linear-gradient(
    90deg,
    rgba(255, 255, 255, 0) 0,
    rgba(255, 255, 255, 0.08) 50%,
    rgba(255, 255, 255, 0) 100%
  );
}

@keyframes shimmer {
  100% {
    transform: translateX(100%);
  }
}
```

### 3.2. Bắt buộc tôn trọng `prefers-reduced-motion`
Với những người dùng bị rối loạn tiền đình, hiệu ứng quét sáng lặp đi lặp lại có thể gây chóng mặt. Khi người dùng bật tính năng giảm chuyển động trong hệ điều hành, **bắt buộc tắt hiệu ứng quét sáng (shimmer)** và chuyển sang hiệu ứng làm mờ tĩnh nhẹ (gentle pulse) hoặc tĩnh hoàn toàn:

```css
@media (prefers-reduced-motion: reduce) {
  .skeleton::after {
    display: none !important;
    animation: none !important;
  }

  .skeleton {
    animation: pulse-subtle 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }
}

@keyframes pulse-subtle {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.6; }
}
```

---

## 4. Các Mẫu Skeleton Phổ Biến (Ready-to-Use Patterns)

### Pattern 1: User Profile / Header Skeleton
```html
<div class="user-card-skeleton" aria-busy="true" aria-live="polite">
  <span class="sr-only">Đang nạp thông tin tài khoản...</span>
  
  <div class="skeleton skeleton-avatar" style="width: 48px; height: 48px; border-radius: 9999px;" aria-hidden="true"></div>
  
  <div class="user-info-skeleton">
    <div class="skeleton" style="width: 140px; height: 16px; margin-bottom: 8px;" aria-hidden="true"></div>
    <div class="skeleton" style="width: 200px; height: 12px;" aria-hidden="true"></div>
  </div>
</div>
```

### Pattern 2: Table / Data Grid Skeleton
Khi cuộn hoặc tải dữ liệu bảng, mỗi hàng skeleton phải khớp số cột của bảng thật:
```html
<table class="data-table" aria-busy="true" aria-label="Bảng dữ liệu đang tải">
  <thead>
    <tr>
      <th scope="col">Mã HĐ</th>
      <th scope="col">Khách hàng</th>
      <th scope="col">Số tiền</th>
      <th scope="col">Trạng thái</th>
    </tr>
  </thead>
  <tbody>
    <!-- Render 5 hàng skeleton mẫu -->
    <tr class="skeleton-row" aria-hidden="true">
      <td><div class="skeleton" style="width: 60px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 160px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 80px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 70px; height: 22px; border-radius: 9999px;"></div></td>
    </tr>
    <tr class="skeleton-row" aria-hidden="true">
      <td><div class="skeleton" style="width: 60px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 140px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 90px; height: 14px;"></div></td>
      <td><div class="skeleton" style="width: 70px; height: 22px; border-radius: 9999px;"></div></td>
    </tr>
  </tbody>
</table>
```

### Pattern 3: Product / Media Card Skeleton
```html
<div class="product-card-skeleton" aria-busy="true">
  <span class="sr-only">Đang nạp danh sách sản phẩm...</span>
  
  <!-- Khung ảnh đúng tỷ lệ 4:3 -->
  <div class="skeleton" style="aspect-ratio: 4/3; width: 100%;" aria-hidden="true"></div>
  
  <div class="card-content" style="padding: 16px;">
    <!-- Tên sản phẩm 2 dòng -->
    <div class="skeleton" style="width: 90%; height: 16px; margin-bottom: 6px;" aria-hidden="true"></div>
    <div class="skeleton" style="width: 60%; height: 16px; margin-bottom: 12px;" aria-hidden="true"></div>
    
    <!-- Giá và Nút mua -->
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <div class="skeleton" style="width: 80px; height: 20px;" aria-hidden="true"></div>
      <div class="skeleton" style="width: 90px; height: 32px; border-radius: 4px;" aria-hidden="true"></div>
    </div>
  </div>
</div>
```

---

## 5. Quy Tắc Vàng & Những Sai Lầm Phổ Biến (Dos and Don'ts)

| Thực Hành Tốt (DO) | Sai Lầm Cần Tránh (DON'T) |
| :--- | :--- |
| ✅ **Độ trễ tối thiểu (Debounce Display)**: Chỉ hiển thị Skeleton khi thời gian tải dự kiến > **150ms** – **200ms**. | ❌ **Không nháy Skeleton**: Tải quá nhanh (< 100ms) mà vẫn chớp Skeleton sẽ gây giật nháy mắt khó chịu (*Flickering*). |
| ✅ **Đúng tỷ lệ (Matching Proportions)**: Giữ kích thước Skeleton bằng 100% kích thước nội dung thật sắp render. | ❌ **Không làm lệch kích thước**: Skeleton cao 100px nhưng khi nội dung thật tải về lại cao 300px làm toàn bộ trang bị giật nhảy (tăng chỉ số CLS). |
| ✅ **Timeout & Error Fallback**: Nếu sau **8–10 giây** dữ liệu chưa tải xong, tự động dừng Skeleton và hiển thị thông báo lỗi mạng kèm nút **"Thử lại"**. | ❌ **Không chạy vô tận**: Để Skeleton chạy mãi mãi khi API backend bị lỗi 500 hoặc rớt mạng khiến người dùng chờ đợi vô vọng. |
| ✅ **Màu sắc trang nhã**: Màu nền của Skeleton chỉ nên chênh lệch nhẹ (subtle contrast) so với nền trang. | ❌ **Không dùng màu quá gắt**: Dùng màu quá đen hoặc quá chói làm người dùng lầm tưởng đó là một khối nội dung thật bị lỗi. |
| ✅ **Kết hợp WAI-ARIA**: Luôn có `aria-busy="true"` trên container và `aria-hidden="true"` trên các phần tử con. | ❌ **Không bỏ quên Screen Reader**: Để screen reader duyệt qua hàng tá thẻ div rác của skeleton. |

