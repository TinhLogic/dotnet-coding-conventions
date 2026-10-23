# Khả Năng Chịu Lỗi & Giám Sát Người Dùng (Error Resilience & Telemetry)

Trong một hệ sinh thái Enterprise, phần mềm không bao giờ được phép sụp đổ hoàn toàn khiến người dùng thấy một trang trắng vô nghĩa (**White Screen of Death**). Ứng dụng phải có cơ chế chịu lỗi phân tầng (Fault Isolation) và tự động ghi nhận nhật ký lỗi cho đội ngũ kỹ thuật.

---

## 1. Cơ Chế Cô Lập Lỗi Phân Tầng (Hierarchical Error Boundaries)

Khi một thành phần UI con (như Biểu đồ doanh thu, Widget thời tiết, Bảng phụ) gặp lỗi JavaScript runtime hoặc ném Exception:
- ❌ **CẤM**: Để lỗi lan truyền lên root làm crash toàn bộ trang web.
- ✅ **BẮT BUỘC**: Bọc các khối chức năng quan trọng bằng các **Error Boundary** riêng biệt:
  - Nếu Widget Biểu đồ crash: Chỉ vùng hiển thị biểu đồ chuyển sang trạng thái Fallback (hiển thị thông báo *"Không thể tải biểu đồ vào lúc này"* kèm nút **Thử lại / Retry**).
  - Thanh menu chính (Sidebar), Header điều hướng và các dữ liệu khác trên màn hình **vẫn phải tiếp tục hoạt động bình thường**.

```tsx
// Cấu trúc React Error Boundary Fallback mẫu
<ErrorBoundary fallback={({ error, resetErrorBoundary }) => (
  <div class="error-fallback-card" role="alert">
    <h4>Đã xảy ra lỗi khi nạp thành phần này</h4>
    <p class="error-hint">Vui lòng bấm thử lại hoặc liên hệ quản trị viên.</p>
    <button type="button" class="btn btn-outline-sm" onClick={resetErrorBoundary}>
      Thử lại
    </button>
  </div>
)}>
  <AnalyticsChart />
</ErrorBoundary>
```

---

## 2. Giám Sát Thời Gian Thực (Real User Monitoring - RUM & Telemetry)

Để phát hiện và khắc phục lỗi trước khi người dùng kịp gửi yêu cầu hỗ trợ, frontend phải tích hợp cơ chế Telemetry:
1. **Bắt lỗi tự động (Unhandled Rejection & Global Error)**: Lắng nghe `window.onerror` và `window.onunhandledrejection` để gửi thông tin lỗi về hệ thống giám sát (Sentry, Datadog RUM, New Relic, OpenTelemetry).
2. **Ghi lại dấu vết thao tác (Breadcrumbs)**: Tự động ghi lại 5–10 hành động gần nhất của người dùng (URL đã truy cập, nút đã click, API vừa gọi) để đội ngũ kỹ sư dễ dàng tái hiện lỗi.
3. **Bảo vệ quyền riêng tư (PII Scrubbing)**: Bắt buộc cấu hình bộ lọc để loại bỏ toàn bộ mật khẩu, số thẻ ngân hàng, token xác thực trước khi gửi dữ liệu Telemetry lên server.

---

## 3. Nhận Diện Trạng Thái Mất Mạng (Offline & Network Loss Handling)

Mất kết nối Internet tạm thời là sự cố thường xuyên xảy ra ở phía người dùng.
- Lắng nghe các sự kiện mạng toàn cục:
  ```typescript
  window.addEventListener('offline', () => {
    showOfflineBanner("Mất kết nối mạng. Các thao tác đang được tạm hoãn...");
  });

  window.addEventListener('online', () => {
    hideOfflineBanner();
    showToast("Đã khôi phục kết nối Internet. Đang đồng bộ dữ liệu...");
    triggerPendingSync();
  });
  ```
- Khi mất mạng, vô hiệu hóa các nút gửi biểu mẫu (`disabled`) hoặc chuyển các thao tác vào hàng đợi cục bộ (Local Queue) để tự động gửi lại khi có mạng trở lại.

