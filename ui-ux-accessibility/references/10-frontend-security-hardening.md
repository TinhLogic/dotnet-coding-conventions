# Bảo Mật Tầng Giao Diện & Dữ Liệu Riêng Tư (Frontend Security & Hardening)

Bảo mật phía Frontend là tuyến phòng thủ theo chiều sâu (Defense-in-Depth). Không bao giờ tin tưởng mù quáng vào dữ liệu phía Client và không bao giờ để lộ các lỗ hổng khiến kẻ tấn công có thể chiếm phiên người dùng hoặc đánh cắp dữ liệu doanh nghiệp.

---

## 1. Lưu Trữ Token Xác Thực An Toàn (Secure Token Handling)

- ❌ **CẤM**: Lưu trữ Refresh Token hoặc Access Token có thời hạn dài trong `localStorage` hoặc `sessionStorage`. Mọi mã độc JavaScript được tiêm vào trang thông qua tấn công XSS (Cross-Site Scripting) đều có thể đọc sạch dữ liệu trong Storage này chỉ bằng 1 dòng lệnh (`localStorage.getItem('token')`).
- ✅ **BẮT BUỘC**:
  - Đối với ứng dụng Web có backend phục vụ trực tiếp: Sử dụng **HttpOnly, Secure, SameSite=Strict/Lax Cookie** để lưu Refresh Token (JavaScript hoàn toàn không thể truy cập vào cookie này).
  - Đối với Access Token (thời hạn ngắn: 5–15 phút): Lưu trữ trong **bộ nhớ RAM của ứng dụng** (in-memory variable / state). Khi F5 làm mới trang, frontend tự động gọi API `/auth/refresh` bằng cookie HttpOnly để lấy lại Access Token mới vào RAM.

---

## 2. Phòng Chống Tấn Công XSS Khi Render Rich Text / Markdown

Khi ứng dụng cho phép người dùng nhập văn bản định dạng phong phú (Rich Text Editor, ghi chú Markdown, bình luận):
- ❌ **CẤM**: Gán trực tiếp chuỗi HTML thô vào DOM (`dangerouslySetInnerHTML`, `v-html`, `innerHtml`).
- ✅ **BẮT BUỘC**: Sử dụng thư viện lọc HTML an toàn (Sanitization) như **`DOMPurify`** để loại bỏ mọi thẻ `<script>`, thuộc tính `onerror=`, `onload=`, `javascript:` trước khi kết xuất:

```typescript
import DOMPurify from 'dompurify';

// Lọc sạch trước khi gán vào UI
const cleanHtml = DOMPurify.sanitize(userProvidedHtml, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'li', 'ol', 'code', 'pre', 'h1', 'h2', 'h3'],
  ALLOWED_ATTR: ['href', 'title', 'target', 'rel']
});
```
- Mọi thẻ `<a>` hướng ra liên kết bên ngoài phải tự động gắn thêm thuộc tính `rel="noopener noreferrer"`.

---

## 3. Chính Sách Bảo Mật Nội Dung (Content Security Policy - CSP)

Máy chủ Web phục vụ Frontend (Nginx, Caddy, Cloudflare, Azure Front Door) bắt buộc phải cấu hình HTTP Header CSP chặt chẽ:
```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; frame-ancestors 'none';
```
- **`frame-ancestors 'none'`** (hoặc `X-Frame-Options: DENY`): Ngăn chặn hoàn toàn việc nhúng website của bạn vào các thẻ `<iframe>` ở trang web lạ nhằm tấn công Clickjacking.

---

## 4. Che Giấu Dữ Liệu Cá Nhân Nhạy Cảm (PII Masking)

Các dữ liệu nhạy cảm (Số thẻ tín dụng, số CCCD/CMND, số tài khoản ngân hàng, mật khẩu, số điện thoại):
- Bắt buộc mặc định che giấu trên giao diện:
  - Thẻ ngân hàng: `•••• •••• •••• 5678`
  - CCCD: `••••••••1234`
  - Số điện thoại: `0912 ••• 789`
- Chỉ hiển thị đầy đủ khi người dùng có thẩm quyền và chủ động bấm nút "Xem" (Icon con mắt kèm xác nhận mật khẩu hoặc mã PIN nếu là dữ liệu tối mật).

