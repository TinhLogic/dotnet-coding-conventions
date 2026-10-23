# Tối Ưu Hiệu Năng & Chỉ Số Core Web Vitals (Web Performance)

Hiệu năng của ứng dụng không chỉ là tốc độ tải ban đầu mà là sự mượt mà và ổn định trong toàn bộ vòng đời tương tác của người dùng. Hệ thống Enterprise phải đáp ứng bộ tiêu chuẩn **Google Core Web Vitals**.

---

## 1. Bộ Ba Chỉ Số Core Web Vitals Chuẩn Doanh Nghiệp

| Chỉ Số | Mục Tiêu Chuẩn | Vấn Đề Gây Giảm Điểm Thường Gặp | Giải Pháp Kỹ Thuật |
| :--- | :--- | :--- | :--- |
| **LCP** *(Largest Contentful Paint)* | **< 2.5 giây** | Font chữ tải chậm, ảnh banner/hero quá nặng, bundle JS quá lớn chặn render | Tự host font với `font-display: swap`, nén ảnh sang WebP/AVIF, CDN Edge caching |
| **INP** *(Interaction to Next Paint)* | **< 200 ms** | Tác vụ JS nặng chiếm dụng Main Thread (Long Tasks > 50ms), re-render DOM diện rộng | Chia nhỏ tác vụ bằng `scheduler.yield()` hoặc `setTimeout(0)`, dùng Web Worker |
| **CLS** *(Cumulative Layout Shift)* | **< 0.1** | Ảnh và banner quảng cáo không khai báo kích thước, font bị nhảy chữ (FOUT) | Luôn khai báo `width`, `height` hoặc `aspect-ratio` trên ảnh và thẻ video |

---

## 2. Phân Tách Gói Mã & Tải Lười (Code Splitting & Lazy Loading)

Không bao giờ đóng gói (bundle) toàn bộ thư viện và màn hình của hệ thống vào một file JavaScript duy nhất:
1. **Route-based Splitting**: Mỗi trang/route chỉ tải bundle JavaScript của chính nó khi người dùng điều hướng tới.
2. **Component-based Lazy Loading**: Các thành phần nặng nhưng không hiển thị ngay lúc đầu (như Modal chỉnh sửa chi tiết, Trình soạn thảo văn bản Rich Text, Biểu đồ thống kê 3D, Trình đọc PDF) phải được import động (`React.lazy()`, dynamic `import()`):

```typescript
// Chỉ tải thư viện biểu đồ khi component được gắn vào màn hình
const HeavyChart = React.lazy(() => import('./components/AnalyticsHeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<div class="skeleton-chart" />}>
      <HeavyChart />
    </Suspense>
  );
}
```

---

## 3. Tối Ưu Hóa Hình Ảnh & Chống Giật Layout (Zero CLS)

- Mọi thẻ `<img>` hiển thị trên giao diện **bắt buộc phải có thuộc tính `width` và `height`** (hoặc CSS `aspect-ratio`):
  ```html
  <!-- ĐÚNG: Trình duyệt dành sẵn không gian trống ngay từ đầu, CLS = 0 -->
  <img src="product.webp" width="300" height="200" alt="Mô tả sản phẩm" loading="lazy" decoding="async" />
  ```
- Sử dụng thuộc tính `loading="lazy"` cho toàn bộ các hình ảnh nằm bên dưới khung nhìn màn hình đầu tiên (Below-the-fold).
- Ảnh nằm ở màn hình đầu tiên (Above-the-fold / Hero banner) thì **không dùng `loading="lazy"`** mà dùng `fetchpriority="high"` để tối ưu chỉ số LCP.

---

## 4. Phòng Chống Rò Rỉ Bộ Nhớ (Memory Leak Prevention)

Trong các ứng dụng Single Page Application (SPA), người dùng có thể giữ tab mở làm việc suốt 8–10 tiếng:
- Mọi sự kiện lắng nghe toàn cục (`window.addEventListener('resize', ...)` hoặc `document.addEventListener('keydown', ...)`): **Bắt buộc phải gỡ bỏ (`removeEventListener`) khi component bị unmount**.
- Mọi timer (`setInterval`, `setTimeout`): **Bắt buộc phải hủy (`clearInterval`, `clearTimeout`) khi component bị unmount**.
- Mọi request API đang chạy dang dở: **Bắt buộc truyền `AbortController.signal` và gọi `abort()` khi rời trang** để tránh cập nhật state vào component đã bị hủy.

