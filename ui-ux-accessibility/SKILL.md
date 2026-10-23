---
name: ui-ux-accessibility
description: >-
  Mandatory skill for UI/UX design, frontend engineering, web accessibility (WAI-ARIA, WCAG 2.1 AA),
  high-performance data grids (virtual scrolling, server-side pagination), clean data representation, and SVG icon standards.
  Covers enterprise frontend architecture: Design Tokens, Dark/Light mode, High Contrast, Form Architecture (dirty state, unsaved changes guard),
  Power-User UX (Command Palette Cmd+K, Deep Linking, Optimistic UI), i18n & l10n (Intl.NumberFormat, Intl.DateTimeFormat),
  Error Boundaries, Telemetry & RUM, Frontend Security (XSS DOMPurify, CSP, secure token storage, PII masking), and Core Web Vitals (LCP, INP, CLS).
  Auto-triggers whenever working with: UI, UX, frontend, HTML, CSS, React, Next.js, Vue, Nuxt, Angular, Svelte, Blazor, Tailwind,
  accessibility, a11y, WAI-ARIA, WCAG, screen reader, table, data grid, virtual scroll, skeleton shimmer, debounce, badges, SVG icons,
  modal, dialog, dropdown, custom select, combobox, listbox, design tokens, dark mode, unsaved changes guard, command palette, i18n, l10n, Intl, error boundary,
  telemetry, XSS, DOMPurify, CSP, Core Web Vitals, LCP, INP, CLS.
  Tự động kích hoạt khi prompt tiếng Việt liên quan đến: thiết kế giao diện, làm UI/UX, bảng dữ liệu, table, data grid,
  danh sách dữ liệu, phân trang server-side, cuộn ảo hóa, virtual scroll, hiệu ứng skeleton, shimmer loading, tìm kiếm debounce,
  chuẩn tiếp cận trợ năng, accessibility, a11y, WAI-ARIA, đọc màn hình screen reader, hộp thoại modal, dialog, popup xác nhận,
  menu dropdown, custom dropdown, custom select, combobox tìm kiếm, select box, nút bấm icon, biểu tượng SVG, Lucide, Heroicons, gắn badge, thẻ trạng thái, hiển thị dữ liệu bảng, WCAG,
  design tokens, giao diện dark mode, tương phản cao, cảnh báo chưa lưu dữ liệu unsaved changes, form validation, bảng lệnh command palette Cmd+K,
  đồng bộ URL query parameters, deep linking, đa ngôn ngữ i18n, định dạng tiền tệ ngày giờ Intl, error boundary chống sập trang,
  bảo mật XSS, DOMPurify, CSP, che dữ liệu nhạy cảm PII, tối ưu hiệu năng web Core Web Vitals LCP INP CLS,
  skeleton loading, khung xương tải dữ liệu, content-aware skeleton, hiệu ứng shimmer wave, aria-busy skeleton.
  Agent MUST apply these rules across all web stacks and frontend frameworks.
---

# UI, UX & Enterprise Web Standards (WAI-ARIA / WCAG 2.1 AA)

Quy chuẩn bắt buộc cho mọi giao diện Web, thành phần Frontend và trải nghiệm người dùng (UX) được sinh ra hoặc chỉnh sửa bởi AI Agent trên mọi nền tảng kỹ thuật (**React, Vue, Angular, Blazor, Svelte, Next.js, Plain HTML/CSS/JS**).

Áp dụng mô hình **Progressive Disclosure**: file `SKILL.md` này tóm tắt 13 trụ cột cốt lõi, bảng tra cứu nhanh và checklist nghiệm thu. Hướng dẫn chi tiết kèm ví dụ đa ngôn ngữ nằm trong thư mục [references/](./references/).

---

## 📚 Danh Mục Quy Chuẩn & Tài Liệu Chi Tiết

| Phần | Trụ Cột Quy Chuẩn | Phạm Vi Áp Dụng | Tài Liệu Chi Tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Data Grid & Hiệu Năng Nạp Bảng** | Server-side query, ảo hóa DOM $O(1)$, Shimmer loading, Debounce 300ms | [01-data-grid-virtualization.md](./references/01-data-grid-virtualization.md) |
| **II** | **Chuẩn Tiếp Cận WAI-ARIA (WCAG 2.1 AA)** | Table, Modal/Dialog, Focus Trap, Dropdown, Screen Reader, Keyboard Nav | [02-wai-aria-accessibility.md](./references/02-wai-aria-accessibility.md) |
| **III** | **Trình Bày Dữ Liệu Sạch (Clean Data)** | Văn bản thuần túy, cấm emoji/icon trang trí trong data, chuẩn hóa Badge/Tag | [03-clean-data-representation.md](./references/03-clean-data-representation.md) |
| **IV** | **Quy Chuẩn Biểu Tượng (SVG Icons)** | Outline vector (Lucide/Heroicons), `stroke="currentColor"`, cỡ chuẩn, `aria-hidden` | [04-svg-icons-styling.md](./references/04-svg-icons-styling.md) |
| **V** | **Hệ Thống Thiết Kế & Theming** | Design Tokens, Dark/Light mode, High Contrast, CSS Logical Properties | [05-design-tokens-theming.md](./references/05-design-tokens-theming.md) |
| **VI** | **Kiến Trúc Biểu Mẫu & Xác Thực** | Unsaved Changes Guard, Dirty state, Inline A11y Validation, Async Debounce | [06-form-architecture-validation.md](./references/06-form-architecture-validation.md) |
| **VII** | **Trải Nghiệm Chuyên Nghiệp (Power-User UX)**| Command Palette (`Cmd+K`), Deep Linking URL query sync, Optimistic UI | [07-power-user-ux-deep-linking.md](./references/07-power-user-ux-deep-linking.md) |
| **VIII** | **Quốc Tế Hóa & Địa Phương Hóa (i18n & l10n)** | Native `Intl` API, Tiền tệ, Ngày giờ tương đối `<time>`, Pluralization, RTL | [08-i18n-l10n-formatting.md](./references/08-i18n-l10n-formatting.md) |
| **IX** | **Khả Năng Chịu Lỗi & Giám Sát** | Phân tầng Error Boundaries, Telemetry RUM, Nhận diện mất mạng Offline | [09-error-resilience-telemetry.md](./references/09-error-resilience-telemetry.md) |
| **X** | **Bảo Mật Tầng Giao Diện & Riêng Tư** | HttpOnly Cookie / RAM token, XSS DOMPurify, CSP Headers, Che giấu PII | [10-frontend-security-hardening.md](./references/10-frontend-security-hardening.md) |
| **XI** | **Tối Ưu Hiệu Năng & Core Web Vitals** | LCP < 2.5s, INP < 200ms, CLS < 0.1, Code-splitting, Zero-CLS Images | [11-core-web-vitals-performance.md](./references/11-core-web-vitals-performance.md) |
| **XII** | **Quy Chuẩn Skeleton Loading** | Content-Aware (CLS=0), WAI-ARIA Screen Reader, CSS Shimmer, Reduced Motion | [12-skeleton-loading-patterns.md](./references/12-skeleton-loading-patterns.md) |
| **XIII** | **Custom Select, Combobox & Dropdown Menu** | Cấm `<select>` mặc định, phân tầng 3 lớp (Shadcn/Google), Portal/Flip, Keyboard A11y | [13-custom-select-combobox.md](./references/13-custom-select-combobox.md) |

---

## ⚡ Cheatsheet: Nguyên Tắc Cốt Lõi Cần Nhớ

### 1. Bảng Dữ Liệu & Hiệu Năng
- ❌ **CẤM**: Tải toàn bộ dữ liệu về RAM client rồi dùng `.filter()`, `.slice()`, `.Where()`.
- ✅ **BẮT BUỘC Server-Side**: Đẩy `search`, `filter`, `sort`, `pagination` xuống Backend / SQL.
- ✅ **Ảo hóa DOM ($O(1)$ Memory)**: Dùng Virtual Scroll giữ số thẻ `<tr>` luôn ở mức 25–40 nodes trong viewport.
- ✅ **Shimmer Skeleton Loading**: Dùng Placeholder Shimmer (`.skeleton-box`), không dùng spinner che toàn màn hình.
- ✅ **Debounce 300ms**: Mọi ô tìm kiếm trực tiếp bắt buộc debounce tối thiểu 300ms trước khi gọi API.

### 2. Tiếp Cận Trợ Năng (WAI-ARIA WCAG 2.1 AA)
- ✅ **Table**: Dùng thẻ ngữ nghĩa HTML5, `th[scope="col"]`, `aria-label`/`caption`, `aria-busy` khi nạp và `aria-live="polite"` cho tổng số dòng.
- ✅ **Modal**: Container mang `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, bẫy tiêu điểm **Focus Trap**, phím `Escape` đóng và nút đóng có `aria-label="Đóng"`.
- ✅ **Custom Select / Dropdown**: ❌ **CẤM dùng `<select>` mặc định**; bắt buộc dùng Custom Select/Combobox có `role="combobox"`, `aria-haspopup="listbox"`, `aria-expanded`, `role="listbox"`, `role="option"`, `aria-selected`.
- ✅ **Nút icon-only**: Bắt buộc có thuộc tính `aria-label="..."` hoặc thẻ con `.sr-only`.

### 3. Trình Bày Dữ Liệu Sạch (Clean Data)
- ❌ **CẤM Emoji / Icon trang trí trong ô dữ liệu**: Tên người, địa danh, danh mục, thương hiệu phải hiển thị dạng Plain Text.
- ❌ **CẤM bọc Badge quanh mã số & text thông thường**: Thẻ `.badge` chỉ dành riêng cho **trạng thái vòng đời** (Active, Inactive, Paid, Pending, Cancelled...).

### 4. Biểu Tượng SVG Đồng Nhất
- ✅ Nét vẽ viền đơn **Outline Vector** (chuẩn Lucide / Heroicons).
- ✅ Tự động kế thừa màu chữ qua `stroke="currentColor"` và `fill="none"`, không hardcode mã hex.
- ✅ Icon trang trí có `aria-hidden="true"`, thêm `flex-shrink: 0` chống méo mó.

### 5. Design Tokens & Theming
- ✅ Tập trung thông số màu sắc, khoảng cách, bo góc, z-index qua CSS Custom Properties (`:root`, `[data-theme="dark"]`).
- ✅ Hỗ trợ tự động `prefers-color-scheme`, `prefers-contrast: more` và `prefers-reduced-motion: reduce`.
- ✅ Dùng CSS Logical Properties (`margin-inline-start`, `padding-inline-end`) để tự động hỗ trợ LTR và RTL.

### 6. Quản Lý Form & Chống Mất Dữ Liệu
- ✅ Bắt buộc hiển thị cảnh báo **Unsaved Changes Guard** (`beforeunload`) khi form ở trạng thái dirty mà người dùng rời trang.
- ✅ Lỗi xác thực input phải gắn `aria-invalid="true"` và liên kết với dòng báo lỗi bằng `aria-describedby`.
- ✅ Async validation (kiểm tra trùng lặp) phải có debounce 400–500ms hoặc kích hoạt khi `blur`.

### 7. Power-User UX & Deep Linking
- ✅ Cung cấp **Command Palette (`Cmd + K` / `Ctrl + K`)** cho tìm kiếm và thao tác nhanh bằng bàn phím.
- ✅ Toàn bộ bộ lọc bảng, số trang, search keyword phải được **đồng bộ 2 chiều lên URL Query Parameters**.
- ✅ **Optimistic UI**: Cập nhật giao diện tức thì khi bấm toggle/status và tự động Rollback nếu server trả lỗi.

### 8. Đa Ngôn Ngữ & Định Dạng Bản Địa (i18n & l10n)
- ✅ Dùng chuẩn trình duyệt `Intl.NumberFormat` cho tiền tệ và số; `Intl.DateTimeFormat` cho ngày tháng.
- ✅ Hiển thị thời gian tương đối trong thẻ `<time datetime="[ISO_UTC]"` kèm tooltip ngày giờ tuyệt đối.
- ✅ Xử lý số ít/số nhiều qua `Intl.PluralRules`, bao bọc văn bản hỗn hợp bằng `<bdi>`.

### 9. Chịu Lỗi & Giám Sát (Resilience & Telemetry)
- ✅ Phân tầng **Error Boundary**: Cô lập lỗi runtime tại từng widget con, không để sập toàn bộ ứng dụng (No White Screen of Death).
- ✅ Tích hợp Real User Monitoring (RUM) bắt unhandled rejection và breadcrumbs (đã khử dữ liệu riêng tư PII).
- ✅ Lắng nghe sự kiện `offline`/`online` để cảnh báo mất mạng và tạm hoãn gửi request.

### 10. Bảo Mật Tầng Giao Diện
- ❌ **CẤM** lưu trữ token nhạy cảm trong `localStorage`. Dùng **HttpOnly Secure Cookie** hoặc lưu Access Token trong RAM.
- ✅ Lọc sạch mọi dữ liệu HTML nhập từ người dùng bằng **`DOMPurify`** trước khi gán vào UI để chống XSS.
- ✅ Cấu hình Header **Content-Security-Policy (CSP)** và mặc định che giấu dữ liệu cá nhân nhạy cảm (PII Masking).

### 11. Tối Ưu Hiệu Năng & Core Web Vitals
- 🎯 **LCP < 2.5s** (Tự host font với `font-display: swap`, CDN caching).
- 🎯 **INP < 200ms** (Không block Main Thread > 50ms, chia nhỏ long tasks).
- 🎯 **CLS < 0.1** (Mọi thẻ `<img>` bắt buộc có `width`, `height` hoặc `aspect-ratio`).
- ✅ Route-based code splitting, lazy load các modal và thư viện đồ thị nặng, dọn sạch event listener/timers khi unmount.

### 12. Quy Chuẩn Skeleton Loading
- ✅ **Content-Aware Shape (CLS = 0)**: Kích thước các khối skeleton (text, avatar, media, button) phải khớp 100% kích thước nội dung thật sắp tải.
- ✅ **Chuẩn Trợ Năng WAI-ARIA**: Container bao bọc phải có `aria-busy="true"` và `aria-live="polite"`. Các khối skeleton con bên trong bắt buộc gắn `aria-hidden="true"` và có thẻ thông báo ẩn `.sr-only`.
- ✅ **CSS Shimmer Wave & Reduced Motion**: Hiệu ứng quét sáng mượt mà bằng `transform: translateX()`; bắt buộc tắt animation quét sáng khi người dùng kích hoạt `prefers-reduced-motion: reduce`.
- ❌ **CẤM**: Chớp nháy skeleton khi thời gian tải quá ngắn (< 150ms); cấm để skeleton chạy mãi mãi mà không có timeout (8–10s) fallback sang nút "Thử lại".

### 13. Quy Chuẩn Custom Select, Combobox & Dropdown Menu (Mô Hình Shadcn & Google)
- ❌ **CẤM dùng `<select>` / `<option>` mặc định của hệ thống**: Bắt buộc thay thế 100% bằng Custom Select / Combobox đồng bộ với Design Tokens và Light/Dark mode.
- ✅ **Ma trận phân loại 3 tầng**:
  - **Danh sách ngắn (< 8 mục tĩnh)**: Dùng **Custom Select (`role="listbox"`)** có icon `ChevronDown` xoay `180deg` khi mở và dấu tích `Check` SVG ở mục đang chọn.
  - **Danh sách dài ($\ge 8$ mục hoặc tải từ API)**: Bắt buộc dùng **Searchable Combobox (`Popover` + Search Input)** tự động `autofocus` ô tìm kiếm khi mở, hỗ trợ Debounce 300ms, Empty State và nút xóa nhanh (`Clear ×`).
  - **Menu hành động (Nút `...`, User menu)**: Dùng **Action Dropdown Menu (`role="menu"` > `role="menuitem"`)**, tách biệt hoàn toàn với Form Select.
- ✅ **Chống cắt khung nhìn & Bảo toàn Form**: Dùng Portal / Popover API / Flip placement để không bị cắt bởi `overflow: hidden`; luôn kèm `<input type="hidden">` để đồng bộ `FormData` & Dirty State; hỗ trợ đầy đủ bàn phím (`ArrowUp/Down`, `Home/End`, `Enter`, `Escape` trả focus) và tự đóng khi click ra ngoài (`Click-outside`).

---

## 📋 Checklist Nghiệm Thu Chuẩn Enterprise (Audit Checklist)

- [ ] Bảng dữ liệu có tải bằng Server-side pagination & filter không?
- [ ] Bảng có áp dụng cơ chế Virtual Scroll $O(1)$ DOM khi danh sách lớn không?
- [ ] Có hiệu ứng Shimmer Skeleton loading thay vì spinner toàn màn hình không?
- [ ] Ô tìm kiếm trực tiếp có Debounce tối thiểu 300ms không?
- [ ] Thẻ `<th>` có `scope="col"`, `<table>` có `aria-label`/`caption`, và bảng nạp có `aria-busy` không?
- [ ] Modal có `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, hỗ trợ `Escape` và Focus Trap không?
- [ ] Dữ liệu trong ô bảng có hiển thị Plain Text sạch sẽ (không emoji, không badge tùy tiện) không?
- [ ] Icon có phải là Outline SVG với `stroke="currentColor"` và `aria-hidden="true"` không?
- [ ] Biến CSS màu sắc và khoảng cách có được quản lý tập trung qua Design Tokens không?
- [ ] Có hỗ trợ Light/Dark mode và tôn trọng `prefers-reduced-motion` không?
- [ ] Form có cơ chế cảnh báo chưa lưu (Unsaved Changes Guard) và `aria-invalid` khi có lỗi không?
- [ ] Trạng thái lọc bảng và trang có được đồng bộ lên URL Query parameters không?
- [ ] Ngày tháng, tiền tệ có được format qua native `Intl` API không?
- [ ] Các widget có được bọc trong Error Boundary kèm nút Thử lại không?
- [ ] Token xác thực có được bảo vệ an toàn (tránh `localStorage`) và chuỗi HTML có được lọc qua `DOMPurify` không?
- [ ] Ảnh có kích thước cố định để đảm bảo chỉ số CLS < 0.1 không?
- [ ] Skeleton loading có khớp kích thước thực tế (Zero CLS), khai báo `aria-busy="true"` và tôn trọng `prefers-reduced-motion` không?
- [ ] Đã loại bỏ hoàn toàn thẻ `<select>` mặc định và thay bằng Custom Select (< 8 mục) hoặc Searchable Combobox ($\ge 8$ mục) chuẩn WAI-ARIA, hỗ trợ bàn phím, chống bị cắt khung nhìn và đồng bộ `<input type="hidden">` chưa?
