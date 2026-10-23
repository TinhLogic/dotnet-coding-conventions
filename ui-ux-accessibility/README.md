# UI, UX & Enterprise Web Standards (WAI-ARIA / WCAG 2.1 AA)

[![W3C WCAG 2.1 AA](https://img.shields.io/badge/W3C-WCAG%202.1%20AA-green.svg)](https://www.w3.org/WAI/standards-guidelines/wcag/)
[![WAI-ARIA](https://img.shields.io/badge/WAI--ARIA-1.2-blue.svg)](https://www.w3.org/TR/wai-aria-1.2/)
[![Core Web Vitals](https://img.shields.io/badge/Core%20Web%20Vitals-LCP%20%7C%20INP%20%7C%20CLS-yellowgreen.svg)](https://web.dev/explore/learn-core-web-vitals)
[![Web Standards](https://img.shields.io/badge/Web-HTML5%20%7C%20CSS3%20%7C%20SVG-orange.svg)](https://developer.mozilla.org/)

> **Ngôn ngữ / Languages**: [Tiếng Việt](#ti%E1%BA%BFng-vi%E1%BB%87t) | [English](#english)

---

# Tiếng Việt

Bộ quy chuẩn toàn diện về thiết kế giao diện người dùng (UI), trải nghiệm người dùng (UX), khả năng tiếp cận web (**WAI-ARIA / WCAG 2.1 AA**) và kiến trúc Frontend doanh nghiệp dành cho mọi ứng dụng web hiện đại. Bộ skill này được thiết kế độc lập, áp dụng trên mọi ngôn ngữ và framework frontend (**React, Vue, Angular, Blazor, Svelte, Next.js, Plain HTML/CSS**).

## 🎯 11 Trụ Cột Doanh Nghiệp

| Phần | Trụ Cột Quy Chuẩn | Phạm Vi Áp Dụng | Tài Liệu Chi Tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Data Grid & Hiệu Năng Nạp Bảng** | Server-side query, ảo hóa DOM $O(1)$, Shimmer loading, Debounce 300ms | [`01-data-grid-virtualization.md`](./references/01-data-grid-virtualization.md) |
| **II** | **Chuẩn Tiếp Cận WAI-ARIA (WCAG 2.1 AA)** | Table, Modal/Dialog, Focus Trap, Dropdown, Screen Reader, Keyboard Nav | [`02-wai-aria-accessibility.md`](./references/02-wai-aria-accessibility.md) |
| **III** | **Trình Bày Dữ Liệu Sạch (Clean Data)** | Văn bản thuần túy, cấm emoji/icon trang trí trong data, chuẩn hóa Badge/Tag | [`03-clean-data-representation.md`](./references/03-clean-data-representation.md) |
| **IV** | **Quy Chuẩn Biểu Tượng (SVG Icons)** | Outline vector (Lucide/Heroicons), `stroke="currentColor"`, cỡ chuẩn, `aria-hidden` | [`04-svg-icons-styling.md`](./references/04-svg-icons-styling.md) |
| **V** | **Hệ Thống Thiết Kế & Theming** | Design Tokens, Dark/Light mode, High Contrast, CSS Logical Properties | [`05-design-tokens-theming.md`](./references/05-design-tokens-theming.md) |
| **VI** | **Kiến Trúc Biểu Mẫu & Xác Thực** | Unsaved Changes Guard, Dirty state, Inline A11y Validation, Async Debounce | [`06-form-architecture-validation.md`](./references/06-form-architecture-validation.md) |
| **VII** | **Trải Nghiệm Chuyên Nghiệp (Power-User UX)**| Command Palette (`Cmd+K`), Deep Linking URL query sync, Optimistic UI | [`07-power-user-ux-deep-linking.md`](./references/07-power-user-ux-deep-linking.md) |
| **VIII** | **Quốc Tế Hóa & Địa Phương Hóa (i18n & l10n)** | Native `Intl` API, Tiền tệ, Ngày giờ tương đối `<time>`, Pluralization, RTL | [`08-i18n-l10n-formatting.md`](./references/08-i18n-l10n-formatting.md) |
| **IX** | **Khả Năng Chịu Lỗi & Giám Sát** | Phân tầng Error Boundaries, Telemetry RUM, Nhận diện mất mạng Offline | [`09-error-resilience-telemetry.md`](./references/09-error-resilience-telemetry.md) |
| **X** | **Bảo Mật Tầng Giao Diện & Riêng Tư** | HttpOnly Cookie / RAM token, XSS DOMPurify, CSP Headers, Che giấu PII | [`10-frontend-security-hardening.md`](./references/10-frontend-security-hardening.md) |
| **XI** | **Tối Ưu Hiệu Năng & Core Web Vitals** | LCP < 2.5s, INP < 200ms, CLS < 0.1, Code-splitting, Zero-CLS Images | [`11-core-web-vitals-performance.md`](./references/11-core-web-vitals-performance.md) |
| **XII** | **Quy Chuẩn Skeleton Loading** | Khớp hình dạng nội dung (CLS=0), WAI-ARIA Screen Reader, CSS Shimmer, Reduced motion | [`12-skeleton-loading-patterns.md`](./references/12-skeleton-loading-patterns.md) |

Bản tóm tắt, cheatsheet và checklist nghiệm thu nhanh nằm tại [`SKILL.md`](./SKILL.md).

---

# English

Enterprise UI, UX, Web Accessibility (**WAI-ARIA / WCAG 2.1 AA**), and Frontend Architecture standards for modern web applications. Designed as a universal, cross-framework skill for AI Coding Assistants and engineering teams across **React, Vue, Angular, Blazor, Svelte, Next.js, and Plain HTML/CSS**.

## 🎯 12 Core Enterprise Pillars

1. **High-Performance Data Grids**: $O(1)$ DOM memory via virtual scrolling, server-side pagination, shimmer skeletons, and 300ms debouncing.
2. **Web Accessibility (WCAG 2.1 AA)**: Semantic HTML5, full keyboard navigation, screen reader live regions, and modal focus traps.
3. **Clean Data Representation**: Zero distracting emojis in data cells, strict plain text formatting, and reserved lifecycle badges.
4. **SVG Icon Standards**: Outline vector icons (Lucide / Heroicons), `currentColor` inheritance, and `aria-hidden` support.
5. **Design Tokens & Theming**: Centralized CSS variables, Dark/Light modes, High Contrast, and CSS Logical Properties (LTR/RTL).
6. **Form Architecture & Validation**: Unsaved changes guards (`beforeunload`), accessible inline error linking, and non-blocking debounced async checks.
7. **Power-User UX & Deep Linking**: Command Palette (`Cmd + K`), full URL query synchronization, and optimistic UI rollbacks.
8. **Internationalization (i18n & l10n)**: Native `Intl` APIs for currency, numbers, relative time `<time>`, and pluralization.
9. **Error Resilience & Telemetry**: Isolated Error Boundaries preventing White Screen of Death, RUM telemetry, and offline detection banners.
10. **Frontend Security**: Safe token storage in HttpOnly cookies / RAM, XSS sanitization via `DOMPurify`, strict CSP headers, and PII masking.
11. **Core Web Vitals**: LCP < 2.5s, INP < 200ms, CLS < 0.1, route-based code-splitting, zero-layout-shift image sizing, and memory leak prevention.
12. **Content-Aware Skeleton Loading**: Exact proportion matching (Zero CLS), `aria-busy` and `aria-hidden` a11y, GPU shimmer wave, and `prefers-reduced-motion` support.
10. **Frontend Security**: Safe token storage in HttpOnly cookies / RAM, XSS sanitization via `DOMPurify`, strict CSP headers, and PII masking.
11. **Core Web Vitals**: LCP < 2.5s, INP < 200ms, CLS < 0.1, route-based code-splitting, zero-layout-shift image sizing, and memory leak prevention.
