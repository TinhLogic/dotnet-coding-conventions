# Trải Nghiệm Người Dùng Chuyên Nghiệp & Deep Linking (Power-User UX)

Các ứng dụng doanh nghiệp phục vụ người dùng thao tác hàng ngày (Kế toán, Vận hành, Quản trị viên, CSKH). Hệ thống phải tối ưu hóa năng suất xử lý (Productivity First) thông qua phím tắt, tìm kiếm nhanh và khả năng đồng bộ trạng thái sâu vào URL.

---

## 1. Bảng Lệnh Toàn Năng (Command Palette: `Cmd + K` / `Ctrl + K`)

Tương tự như Linear, Slack, GitHub hay VS Code, hệ thống Enterprise nên cung cấp một Command Palette mở bằng tổ hợp phím tắt `Cmd + K` (trên macOS) hoặc `Ctrl + K` (trên Windows/Linux).

### Chức năng chính của Command Palette:
1. **Điều hướng nhanh (Quick Navigation)**: Chuyển ngay đến trang cài đặt, hồ sơ, danh mục sản phẩm, đơn hàng mà không cần click qua nhiều cấp menu.
2. **Thực thi hành động tức thì (Quick Actions)**: *"Tạo đơn hàng mới"*, *"Xuất báo cáo PDF"*, *"Mời thành viên"*.
3. **Chuyển đổi ngữ cảnh (Context Switch)**: Đổi tổ chức/chi nhánh làm việc, đổi theme sáng/tối.
4. **Trợ năng bàn phím**: Phím `ArrowUp`/`ArrowDown` di chuyển, `Enter` thực thi, `Escape` đóng.

---

## 2. Đồng Bộ Trạng Thái Sâu Lên URL (Deep Linking & Query Sync)

Một sai lầm rất phổ biến của lập trình viên là lưu toàn bộ bộ lọc bảng, số trang, tab đang chọn vào bộ nhớ RAM của component (`useState` / local variable) mà không đồng bộ lên URL.

- ❌ **HẬU QUẢ**: Khi người dùng nhấn F5 làm mới trang, bấm nút Back của trình duyệt hoặc sao chép link gửi cho đồng nghiệp, toàn bộ màn hình bị reset về mặc định ban đầu.
- ✅ **BẮT BUỘC**: Mọi trạng thái hiển thị quan trọng phải được đồng bộ hai chiều (Bi-directional sync) với URL Query Parameters:
  - Ví dụ URL chuẩn: `/admin/orders?page=2&status=processing&search=INV-992&tab=shipping`

### Quy tắc triển khai:
1. Khi URL thay đổi (bao gồm cả khi người dùng bấm nút Back/Forward của trình duyệt), giao diện phải tự động khôi phục chính xác trạng thái lọc tương ứng.
2. Dùng `history.replaceState` (thay vì `pushState`) khi gõ tìm kiếm để không làm rác lịch sử duyệt web.
3. Chỉ `pushState` khi chuyển hẳn sang một trang mới hoặc chuyển tab nghiệp vụ chính.

---

## 3. Cập Nhật Giao Diện Lạc Quan (Optimistic UI Updates)

Người dùng không nên phải chờ đợi 500ms – 1s quay spinner chỉ để thấy một nút gạt trạng thái (Toggle Active/Inactive) hoặc nút đánh dấu yêu thích được cập nhật.

- **Nguyên tắc Optimistic Update**:
  1. Khi người dùng click: Ngay lập tức đổi màu/trạng thái của UI sang giá trị mới (0ms latency).
  2. Gửi request cập nhật ngầm xuống API server.
  3. **Nếu thành công**: Giữ nguyên trạng thái và hiển thị Toast nhẹ nếu cần.
  4. **Nếu thất bại (Network Error / 500 Server)**: Tự động **Rollback (hoàn tác)** giao diện về trạng thái cũ ban đầu và hiển thị Toast báo lỗi rõ ràng.

---

## 4. Hệ Thống Phím Tắt Toàn Cục (Global Keyboard Shortcuts)

Các phím tắt tiêu chuẩn cần hỗ trợ trong môi trường Enterprise:

| Phím Tắt | Hành Động Tiêu Chuẩn |
| :--- | :--- |
| `Cmd + K` / `Ctrl + K` | Mở Command Palette / Bảng tìm kiếm toàn năng |
| `/` | Đưa tiêu điểm ngay vào ô tìm kiếm chính trên trang (khi không ở trong input) |
| `Cmd + S` / `Ctrl + S` | Lưu form đang chỉnh sửa hiện tại |
| `Escape` | Đóng Modal, Dropdown, Command Palette hoặc hủy thao tác |
| `?` | Hiển thị bảng trợ giúp danh sách toàn bộ phím tắt của hệ thống |

