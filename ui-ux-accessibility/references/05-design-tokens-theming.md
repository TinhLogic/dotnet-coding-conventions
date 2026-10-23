# Hệ Thống Thiết Kế & Theming (Design Tokens & Semantic Themes)

Một website doanh nghiệp (Enterprise Web Application) hiện đại không được phép phân tán các giá trị giao diện (màu sắc, lề, khoảng cách, kích cỡ chữ) dưới dạng hardcode (như `style="margin: 12px; color: #333;"`). Toàn bộ giao diện phải vận hành dựa trên hệ thống **Design Tokens** tập trung qua biến CSS (CSS Custom Properties).

---

## 1. Hệ Thống Design Tokens (CSS Variables)

Toàn bộ thông số nền tảng của hệ thống thiết kế (Design System) phải được định nghĩa tại root (`:root`) với tính nhất quán cao:

```css
:root {
  /* 1. Palette màu ngữ nghĩa (Semantic Color Tokens) */
  --color-bg-app: #f8fafc;
  --color-bg-surface: #ffffff;
  --color-bg-surface-elevated: #ffffff;
  --color-border: #e2e8f0;
  --color-border-subtle: #f1f5f9;

  --color-text-primary: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #94a3b8;

  --color-brand-primary: #2563eb;
  --color-brand-primary-hover: #1d4ed8;
  --color-brand-text: #ffffff;

  --color-status-success: #16a34a;
  --color-status-warning: #d97706;
  --color-status-danger: #dc2626;
  --color-status-info: #0284c7;

  /* 2. Khoảng cách (Spacing Scale) */
  --space-1: 0.25rem; /* 4px */
  --space-2: 0.5rem;  /* 8px */
  --space-3: 0.75rem; /* 12px */
  --space-4: 1rem;    /* 16px */
  --space-6: 1.5rem;  /* 24px */
  --space-8: 2rem;    /* 32px */

  /* 3. Bo góc (Border Radii) */
  --radius-sm: 0.25rem; /* 4px */
  --radius-md: 0.375rem; /* 6px */
  --radius-lg: 0.5rem;   /* 8px */
  --radius-pill: 9999px;

  /* 4. Thứ tự xếp lớp (Z-Index Hierarchy Scale) */
  --z-base: 0;
  --z-sticky: 100;
  --z-dropdown: 200;
  --z-modal-backdrop: 500;
  --z-modal: 600;
  --z-toast: 1000;
  --z-tooltip: 1100;

  /* 5. Bóng đổ (Elevation / Shadows) */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
}
```

---

## 2. Hỗ Trợ Chế Độ Sáng / Tối (Light & Dark Theme)

Hệ thống phải hỗ trợ chuyển đổi giao diện mượt mà dựa trên thuộc tính `data-theme="dark"` hoặc tự động theo cấu hình hệ điều hành qua `prefers-color-scheme`.

```css
/* Tự động theo OS hoặc khi người dùng kích hoạt dark mode */
[data-theme="dark"] {
  --color-bg-app: #0b0f19;
  --color-bg-surface: #111827;
  --color-bg-surface-elevated: #1f2937;
  --color-border: #374151;
  --color-border-subtle: #1f2937;

  --color-text-primary: #f9fafb;
  --color-text-secondary: #9ca3af;
  --color-text-muted: #6b7280;

  --color-brand-primary: #3b82f6;
  --color-brand-primary-hover: #60a5fa;

  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.5);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.5);
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-bg-app: #0b0f19;
    --color-bg-surface: #111827;
    --color-bg-surface-elevated: #1f2937;
    --color-border: #374151;
    --color-text-primary: #f9fafb;
    --color-text-secondary: #9ca3af;
  }
}
```

---

## 3. Chế Độ Tương Phản Cao (High Contrast Mode)

Đối với người dùng suy giảm thị lực kích hoạt tùy chọn trợ năng trong hệ điều hành, CSS phải tự động tăng độ tương phản của viền và chữ:

```css
@media (prefers-contrast: more) {
  :root {
    --color-border: #000000;
    --color-text-primary: #000000;
    --color-text-secondary: #000000;
    --color-brand-primary: #0000cc;
  }

  [data-theme="dark"] {
    --color-border: #ffffff;
    --color-text-primary: #ffffff;
    --color-text-secondary: #ffffff;
    --color-brand-primary: #66b2ff;
  }

  button, input, select, textarea {
    border-width: 2px !important;
  }
}
```

---

## 4. Tôn Trọng Giảm Chuyển Động (`prefers-reduced-motion`)

Những người dùng bị hội chứng rối loạn tiền đình (Vestibular disorders) có thể bị chóng mặt hoặc buồn nôn khi thấy các hiệu ứng chuyển động lớn hoặc rung lắc. Bắt buộc tắt hoặc giảm tối đa animation khi người dùng kích hoạt tùy chọn này:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

---

## 5. Đa Hướng Giao Diện (CSS Logical Properties cho LTR & RTL)

Khi ứng dụng mở rộng ra thị trường toàn cầu (hỗ trợ tiếng Ả Rập, tiếng Do Thái viết từ phải sang trái - RTL), không dùng `margin-left`, `padding-right`. Bắt buộc sử dụng các thuộc tính logic:

| Thuộc tính truyền thống (Cấm dùng) | Thuộc tính CSS Logical (Bắt buộc dùng) | Ý nghĩa |
| :--- | :--- | :--- |
| `margin-left: 16px;` | `margin-inline-start: 16px;` | Cách lề phía bắt đầu dòng văn bản |
| `padding-right: 12px;` | `padding-inline-end: 12px;` | Đệm phía kết thúc dòng văn bản |
| `border-left: 1px solid;` | `border-inline-start: 1px solid;` | Đường viền phía đầu văn bản |
| `left: 0; right: 0;` | `inset-inline: 0;` | Neo dính 2 bên ngang |
| `text-align: left;` | `text-align: start;` | Căn về phía đầu đọc |

