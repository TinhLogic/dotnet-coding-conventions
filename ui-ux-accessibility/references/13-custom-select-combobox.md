# Quy Chuẩn Custom Select, Combobox & Dropdown Menu (Mô Hình Shadcn UI & Google Material 3)

Trong các ứng dụng Web hiện đại và hệ thống Enterprise, **thẻ `<select>` và `<option>` hiển thị mặc định của hệ điều hành (Native OS Dropdown) bị CẤM sử dụng trực tiếp trên giao diện người dùng** vì 3 hạn chế cốt lõi:
1. **Phá vỡ Design System & Dark Mode**: Khung popup `<option>` được vẽ bởi tầng hệ điều hành (Windows, macOS, Linux, iOS, Android) nên không thể tùy biến bo góc, đổ bóng, khoảng cách, font chữ hay đồng bộ hoàn toàn với bảng màu Dark/Light Tokens.
2. **Không đáp ứng nghiệp vụ dữ liệu lớn**: Thiếu ô tìm kiếm nhanh (Searchable), không hỗ trợ tải phân trang bất đồng bộ (Async Server-side), không thể ảo hóa DOM (Virtual Scroll) và không hiển thị được dữ liệu đa tầng (Avatar, Badge trạng thái, mã phụ SKU).
3. **Trải nghiệm thiếu nhất quán**: Giao diện và hành vi hiển thị bị biến dạng khác nhau trên từng trình duyệt và hệ điều hành.

Mọi thành phần chọn dữ liệu và menu thả xuống **bắt buộc phải sử dụng Custom Select / Combobox / Dropdown Menu** theo kiến trúc phân tầng của **shadcn/ui (Radix UI)** và **Google Material Design 3**.

---

## 1. Ma Trận Phân Loại 3 Tầng (Chuẩn Shadcn UI & Google)

Không gộp chung mọi menu xổ xuống vào một kiểu duy nhất. AI Agent bắt buộc phải chọn đúng 1 trong 3 biến thể sau tùy theo ngữ cảnh nghiệp vụ:

| Tiêu chí | 1. Custom Select (`Listbox`) | 2. Searchable Combobox (`Command`) | 3. Action Dropdown Menu (`Menu`) |
| :--- | :--- | :--- | :--- |
| **Ngữ cảnh áp dụng** | Chọn 1 giá trị trong Form với **danh sách ngắn, cố định (< 8 mục)**.<br>*(VD: Giới tính, Trạng thái, Vai trò, Số dòng/trang `10/20/50`)* | Chọn 1 hoặc nhiều giá trị với **danh sách từ 8 mục trở lên ($\ge 8$ mục) hoặc tải động từ API**.<br>*(VD: Khách hàng, Tỉnh/Thành, Sản phẩm, Nhân viên)* | Danh sách **hành động / câu lệnh**.<br>*(VD: Nút `...` cuối dòng bảng dữ liệu, Menu tài khoản cá nhân)* |
| **WAI-ARIA Role** | Trigger: `role="combobox"` + `aria-haspopup="listbox"`<br>Popup: `role="listbox"` > `role="option"` | Trigger: `role="combobox"` + `aria-expanded`<br>Search Input: `aria-autocomplete="list"`<br>Popup: `role="listbox"` > `role="option"` | Trigger: `aria-haspopup="menu"`<br>Popup: `role="menu"` > `role="menuitem"` |
| **Đặc điểm UI/UX bắt buộc** | - Nút bấm có icon `ChevronDown` xoay `180deg` khi mở.<br>- Mục đang chọn hiển thị icon dấu tích (`Check` SVG) và `aria-selected="true"`.<br>- Hỗ trợ gõ phím nhanh (Typeahead) để nhảy đến mục. | - Mở popup tự động đặt tiêu điểm (`autofocus`) vào ô tìm kiếm ở đỉnh.<br>- Có nút xóa nhanh (`Clear ×`) nếu trường không bắt buộc.<br>- Hỗ trợ Debounce 300ms khi tìm kiếm qua API.<br>- Có giao diện Empty State (*"Không tìm thấy kết quả"*) và Skeleton/Spinner khi đang tải. | - **Tuyệt đối không dùng để chọn giá trị Form**.<br>- Hỗ trợ vạch ngăn cách nhóm (`role="separator"`).<br>- Hiển thị phím tắt gợi ý bên phải (`⌘E`, `⌫`).<br>- Hành động nguy hiểm (Xóa/Khóa) tô màu cảnh báo đỏ (`danger`). |

---

## 2. Năm Nguyên Tắc Kiến Trúc Bắt Buộc

### 2.1. Ưu Tiên Thư Viện Headless UI Primitives (Khi Dùng Framework)
Khi dự án sử dụng các Frontend Framework hiện đại, **ưu tiên sử dụng các thư viện Headless UI đã đạt chuẩn WAI-ARIA** và tùy biến giao diện bằng CSS/Tailwind theo Design Tokens của dự án:
- **React / Next.js**: `@radix-ui/react-select`, `cmdk` (bộ đôi nền tảng của **shadcn/ui**), `react-aria-components` (Adobe), hoặc `@headlessui/react`.
- **Vue / Nuxt**: `radix-vue` (`shadcn-vue`), `reka-ui`, hoặc `@headlessui/vue`.
- **Angular**: `@angular/cdk/listbox`, `@angular/cdk/overlay`, Angular Material.
- **Blazor**: `MudSelect` / `MudAutocomplete` (MudBlazor), `FluentSelect` / `FluentCombobox` (FluentUI Blazor), hoặc Custom Component tuân thủ đầy đủ quy chuẩn bên dưới.

### 2.2. Chống Cắt Khung Nhìn (Anti-Clipping, Portal & Flip Placement)
Một lỗi phổ biến nhất của Custom Dropdown tự viết là bị cắt mất menu khi đặt bên trong ô bảng dữ liệu (`<td>`), thẻ Card hoặc `Modal` có thuộc tính `overflow: hidden` / `overflow: auto`.
- **Giải pháp bắt buộc**:
  1. Render khung popup thông qua **Portal** (đẩy ra ngoài thẻ `<body>`), sử dụng **CSS Popover API (`popover="auto"`)**, hoặc định vị `position: fixed` kết hợp tính toán tọa độ thực tế (`getBoundingClientRect()` / thư viện `@floating-ui/dom`).
  2. **Tự động đảo chiều (Collision Detection / Flip)**: Mặc định mở xổ xuống dưới (`bottom-start`), nhưng nếu khoảng cách từ nút đến đáy khung nhìn (viewport) `< 220px`, popup phải tự động bật ngược lên phía trên (`top-start`).
  3. Giới hạn chiều cao tối đa của danh sách (`max-height: 260px` hoặc `280px`) kèm thanh cuộn mượt (`overflow-y: auto`, `overscroll-behavior: contain`).

### 2.3. Bảo Toàn Tích Hợp Biểu Mẫu Gốc (Hidden Input Synchronization)
Custom Select xây dựng bằng `<button>` và `<ul>/<li>` sẽ không tự động gửi dữ liệu khi `<form>` submit hoặc khi gọi `new FormData(formElement)`.
- **Bắt buộc**: Bên trong mỗi component Custom Select / Combobox dùng cho Form phải có một thẻ `<input type="hidden" name="[ten_truong]" value="[gia_tri_chon]">` được đồng bộ giá trị tức thì.
- Khi giá trị thay đổi, phải kích hoạt sự kiện `change` / `input` để hệ thống **Unsaved Changes Guard (Dirty State)** và **Form Validation (`aria-invalid="true"`)** nhận diện chính xác.

### 2.4. Điều Hướng Bàn Phím Toàn Diện (Keyboard-First Navigation)
Mọi Custom Select và Combobox bắt buộc phản hồi đầy đủ các phím sau:
- `Enter`, `Space`, hoặc `ArrowDown` (khi đang focus ở nút Trigger và menu đang đóng): Mở menu và đưa tiêu điểm (active highlight) vào mục đang được chọn (hoặc mục đầu tiên).
- `ArrowDown` / `ArrowUp` (khi menu đang mở): Di chuyển vạch sáng lựa chọn xuống/lên giữa các mục (bỏ qua các mục bị `aria-disabled="true"`).
- `Home` / `End`: Nhảy nhanh tới mục đầu tiên hoặc mục cuối cùng.
- `Enter` (hoặc `Space` trong Select không có ô search): Chốt chọn mục đang highlight, đóng menu và **trả focus (`element.focus()`) về lại nút Trigger**.
- `Escape`: Đóng menu ngay lập tức mà không thay đổi giá trị, **trả focus về lại nút Trigger**.
- `Tab`: Đóng menu và cho phép trình duyệt chuyển focus tự nhiên sang ô nhập liệu kế tiếp.

### 2.5. Đóng Độc Quyền (Exclusive Open), Hiệu Ứng GPU & Tương Thích Mobile
- **Click-Outside & Exclusive Open**: Bấm chuột ra ngoài vùng component (`pointerdown` ngoài container) hoặc bấm mở một Custom Dropdown khác trên cùng trang phải lập tức đóng menu đang mở.
- **Hiệu ứng chuyển động mượt (Micro-interaction)**:
  - Menu mở ra với hiệu ứng `opacity: 0 -> 1` và `transform: scale(0.96) translateY(-4px) -> scale(1) translateY(0)` trong `140ms cubic-bezier(0.16, 1, 0.3, 1)`.
  - Icon mũi tên `ChevronDown` xoay mượt `transform: rotate(180deg)` trong `150ms`.
  - Tôn trọng `prefers-reduced-motion: reduce` (tắt toàn bộ transition khi người dùng bật chế độ giảm chuyển động).
- **Thích ứng thiết bị di động (Mobile Responsive)**:
  - Trên màn hình hẹp (`< 640px`), với các **Searchable Combobox** nhiều dữ liệu, ưu tiên hiển thị khung chọn dưới dạng **Bottom Sheet** (ngăn kéo trượt từ đáy màn hình) hoặc giữ cố định thanh tìm kiếm ở đỉnh popup để bàn phím ảo không che khuất danh sách kết quả.

---

## 3. Mẫu Code Chuẩn Production (HTML / CSS / JS Vanilla)

### 3.1. CSS Design Tokens Chuẩn Shadcn / Google (Dùng Chung Cho Select & Combobox)

```css
:root {
  --select-bg: #ffffff;
  --select-fg: #0f172a;
  --select-muted: #64748b;
  --select-border: #cbd5e1;
  --select-border-focus: #2563eb;
  --select-ring: rgba(37, 99, 235, 0.2);
  --select-popover-bg: #ffffff;
  --select-popover-shadow: 0 10px 25px -5px rgba(15, 23, 42, 0.12), 0 8px 10px -6px rgba(15, 23, 42, 0.08);
  --select-item-hover: #f1f5f9;
  --select-item-selected-bg: #eff6ff;
  --select-item-selected-fg: #1d4ed8;
  --select-radius: 0.5rem; /* 8px */
}

[data-theme="dark"] {
  --select-bg: #0f172a;
  --select-fg: #f8fafc;
  --select-muted: #94a3b8;
  --select-border: #334155;
  --select-border-focus: #3b82f6;
  --select-ring: rgba(59, 130, 246, 0.25);
  --select-popover-bg: #1e293b;
  --select-popover-shadow: 0 12px 28px -4px rgba(0, 0, 0, 0.5);
  --select-item-hover: #334155;
  --select-item-selected-bg: rgba(59, 130, 246, 0.18);
  --select-item-selected-fg: #60a5fa;
}

.custom-select-wrapper {
  position: relative;
  width: 100%;
  font-family: inherit;
}

/* Trigger Button (Thay thế thẻ <select> gốc) */
.custom-select-trigger {
   display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  width: 100%;
  min-height: 2.5rem; /* 40px chuẩn chạm cảm ứng */
  padding: 0.5rem 0.75rem;
  background-color: var(--select-bg);
  color: var(--select-fg);
  border: 1px solid var(--select-border);
  border-radius: var(--select-radius);
  font-size: 0.875rem;
  line-height: 1.25rem;
  text-align: left;
  cursor: pointer;
  transition: border-color 150ms ease, box-shadow 150ms ease;
}

.custom-select-trigger:hover:not(:disabled) {
  border-color: var(--select-muted);
}

.custom-select-trigger:focus-visible,
.custom-select-trigger[aria-expanded="true"] {
  outline: none;
  border-color: var(--select-border-focus);
  box-shadow: 0 0 0 3px var(--select-ring);
}

.custom-select-trigger[aria-invalid="true"] {
  border-color: #ef4444;
}

.custom-select-trigger:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.custom-select-placeholder {
  color: var(--select-muted);
}

.custom-select-chevron {
  flex-shrink: 0;
  color: var(--select-muted);
  transition: transform 150ms cubic-bezier(0.16, 1, 0.3, 1);
}

.custom-select-trigger[aria-expanded="true"] .custom-select-chevron {
  transform: rotate(180deg);
}

/* Khung Popover Danh Sách */
.custom-select-popover {
  position: absolute;
  top: calc(100% + 0.375rem);
  inset-inline-start: 0;
  z-index: 50;
  width: 100%;
  min-width: 12rem;
  background-color: var(--select-popover-bg);
  border: 1px solid var(--select-border);
  border-radius: var(--select-radius);
  box-shadow: var(--select-popover-shadow);
  padding: 0.25rem;
  opacity: 0;
  visibility: hidden;
  transform: scale(0.96) translateY(-4px);
  transform-origin: top center;
  transition: opacity 140ms cubic-bezier(0.16, 1, 0.3, 1),
              transform 140ms cubic-bezier(0.16, 1, 0.3, 1),
              visibility 140ms;
}

.custom-select-popover[data-open="true"] {
  opacity: 1;
  visibility: visible;
  transform: scale(1) translateY(0);
}

/* Lật ngược lên trên khi sát đáy màn hình */
.custom-select-popover[data-placement="top"] {
  top: auto;
  bottom: calc(100% + 0.375rem);
  transform-origin: bottom center;
}

/* Ô Tìm Kiếm Bên Trong Combobox */
.combobox-search-box {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.375rem 0.5rem;
  border-bottom: 1px solid var(--select-border);
  margin-bottom: 0.25rem;
}

.combobox-search-input {
  width: 100%;
  border: none;
  background: transparent;
  color: var(--select-fg);
  font-size: 0.875rem;
  outline: none;
}

/* Danh sách Option */
.custom-select-listbox {
  list-style: none;
  margin: 0;
  padding: 0;
  max-height: 16rem; /* 256px */
  overflow-y: auto;
  overscroll-behavior: contain;
}

.custom-select-option {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  padding: 0.5rem 0.625rem;
  border-radius: calc(var(--select-radius) - 2px);
  font-size: 0.875rem;
  color: var(--select-fg);
  cursor: pointer;
  user-select: none;
}

.custom-select-option:hover,
.custom-select-option[data-highlighted="true"] {
  background-color: var(--select-item-hover);
}

.custom-select-option[aria-selected="true"] {
  background-color: var(--select-item-selected-bg);
  color: var(--select-item-selected-fg);
  font-weight: 500;
}

.custom-select-check {
  flex-shrink: 0;
  opacity: 0;
  color: currentColor;
}

.custom-select-option[aria-selected="true"] .custom-select-check {
  opacity: 1;
}

@media (prefers-reduced-motion: reduce) {
  .custom-select-popover,
  .custom-select-chevron,
  .custom-select-trigger {
    transition: none !important;
  }
}
```

### 3.2. Cấu Trúc HTML & JS Cho Custom Select (< 8 Mục)

```html
<div class="form-field">
  <label id="lbl-role" class="form-label">Vai trò hệ thống <span aria-hidden="true">*</span></label>

  <div class="custom-select-wrapper" data-custom-select>
    <!-- Hidden input đảm bảo FormData và Form Validation hoạt động như <select> gốc -->
    <input type="hidden" name="role" value="editor" data-select-input />

    <!-- Trigger Button chuẩn WAI-ARIA -->
    <button
      type="button"
      class="custom-select-trigger"
      role="combobox"
      aria-labelledby="lbl-role"
      aria-haspopup="listbox"
      aria-expanded="false"
      aria-controls="listbox-role"
      data-select-trigger>
      <span data-select-value>Biên tập viên (Editor)</span>
      <svg class="custom-select-chevron" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
        <path stroke-linecap="round" stroke-linejoin="round" d="m6 9 6 6 6-6" />
      </svg>
    </button>

    <!-- Khung Popover chứa danh sách lựa chọn -->
    <div class="custom-select-popover" data-open="false" data-select-popover>
      <ul id="listbox-role" class="custom-select-listbox" role="listbox" tabindex="-1" aria-labelledby="lbl-role">
        <li class="custom-select-option" role="option" data-value="admin" aria-selected="false">
          <span>Quản trị viên (Admin)</span>
          <svg class="custom-select-check" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
            <polyline points="20 6 9 17 4 12" />
          </svg>
        </li>
        <li class="custom-select-option" role="option" data-value="editor" aria-selected="true" data-highlighted="true">
          <span>Biên tập viên (Editor)</span>
          <svg class="custom-select-check" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
            <polyline points="20 6 9 17 4 12" />
          </svg>
        </li>
        <li class="custom-select-option" role="option" data-value="viewer" aria-selected="false">
          <span>Người xem (Viewer)</span>
          <svg class="custom-select-check" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
            <polyline points="20 6 9 17 4 12" />
          </svg>
        </li>
      </ul>
    </div>
  </div>
</div>
```

### 3.3. Logic JavaScript Điều Khiển (Keyboard Nav, Flip Placement & Click-Outside)

```javascript
function initCustomSelect(wrapper) {
  const trigger = wrapper.querySelector('[data-select-trigger]');
  const popover = wrapper.querySelector('[data-select-popover]');
  const valueDisplay = wrapper.querySelector('[data-select-value]');
  const hiddenInput = wrapper.querySelector('[data-select-input]');
  const options = Array.from(wrapper.querySelectorAll('[role="option"]'));
  let activeIndex = Math.max(0, options.findIndex(o => o.getAttribute('aria-selected') === 'true'));

  function setHighlight(index) {
    activeIndex = (index + options.length) % options.length;
    options.forEach((opt, i) => {
      opt.dataset.highlighted = i === activeIndex ? 'true' : 'false';
      if (i === activeIndex) opt.scrollIntoView({ block: 'nearest' });
    });
  }

  function toggleMenu(open) {
    const isOpen = open ?? trigger.getAttribute('aria-expanded') !== 'true';
    if (isOpen) {
      // Đóng mọi Custom Select khác đang mở trên trang
      document.querySelectorAll('[data-custom-select]').forEach(other => {
        if (other !== wrapper) {
          other.querySelector('[data-select-trigger]')?.setAttribute('aria-expanded', 'false');
          other.querySelector('[data-select-popover]')?.setAttribute('data-open', 'false');
        }
      });
      // Kiểm tra khoảng trống phía dưới để tự động lật lên trên (Flip placement)
      const rect = trigger.getBoundingClientRect();
      const spaceBelow = window.innerHeight - rect.bottom;
      popover.dataset.placement = spaceBelow < 220 && rect.top > 220 ? 'top' : 'bottom';
      setHighlight(Math.max(0, options.findIndex(o => o.getAttribute('aria-selected') === 'true')));
    }
    trigger.setAttribute('aria-expanded', String(isOpen));
    popover.setAttribute('data-open', String(isOpen));
  }

  function selectOption(index) {
    const chosen = options[index];
    if (!chosen) return;
    options.forEach((opt, i) => opt.setAttribute('aria-selected', String(i === index)));
    valueDisplay.textContent = chosen.innerText.trim();
    valueDisplay.classList.remove('custom-select-placeholder');
    if (hiddenInput) {
      hiddenInput.value = chosen.dataset.value || '';
      hiddenInput.dispatchEvent(new Event('change', { bubbles: true }));
    }
    toggleMenu(false);
    trigger.focus(); // Trả tiêu điểm về nút Trigger theo chuẩn WAI-ARIA
  }

  trigger.addEventListener('click', () => toggleMenu());

  trigger.addEventListener('keydown', (e) => {
    const isOpen = trigger.getAttribute('aria-expanded') === 'true';
    if (['ArrowDown', 'ArrowUp', 'Enter', ' '].includes(e.key) && !isOpen) {
      e.preventDefault();
      toggleMenu(true);
      return;
    }
    if (!isOpen) return;
    switch (e.key) {
      case 'ArrowDown': e.preventDefault(); setHighlight(activeIndex + 1); break;
      case 'ArrowUp':   e.preventDefault(); setHighlight(activeIndex - 1); break;
      case 'Home':      e.preventDefault(); setHighlight(0); break;
      case 'End':       e.preventDefault(); setHighlight(options.length - 1); break;
      case 'Enter':
      case ' ':         e.preventDefault(); selectOption(activeIndex); break;
      case 'Escape':    e.preventDefault(); toggleMenu(false); trigger.focus(); break;
      case 'Tab':       toggleMenu(false); break;
    }
  });

  options.forEach((opt, idx) => {
    opt.addEventListener('mouseenter', () => setHighlight(idx));
    opt.addEventListener('click', () => selectOption(idx));
  });

  document.addEventListener('pointerdown', (e) => {
    if (!wrapper.contains(e.target) && trigger.getAttribute('aria-expanded') === 'true') {
      toggleMenu(false);
    }
  });
}
```

---

## 4. Quy Tắc Vàng & Các Lỗi Nghiêm Cấm (Dos and Don'ts)

| Thực Hành Chuẩn Enterprise (DO) | Hành Vi Bị Nghiêm Cấm (DON'T) |
| :--- | :--- |
| ✅ **Dùng Custom Select / Combobox đồng bộ Design Tokens**: Tùy biến 100% giao diện nút bấm và khung danh sách theo Light/Dark mode. | ❌ **CẤM dùng `<select>` mặc định của hệ thống**: Không để trình duyệt hiển thị popup `<option>` thô kệch của OS trên giao diện người dùng. |
| ✅ **Tự động chuyển sang Searchable Combobox khi $\ge 8$ mục**: Luôn có ô tìm kiếm bên trong popup khi danh sách có từ 8 lựa chọn trở lên. | ❌ **CẤM bắt người dùng cuộn danh sách 20–50 mục mà không có ô tìm kiếm**: Gây ức chế trải nghiệm thao tác (UX kém). |
| ✅ **Đồng bộ `<input type="hidden">` hoặc Form Controller**: Giữ nguyên khả năng submit form, reset form và theo dõi Dirty State. | ❌ **CẤM chỉ dùng `<div>` suông không gắn với Form State**: Làm mất dữ liệu khi submit hoặc hỏng tính năng kiểm tra lỗi. |
| ✅ **Đầy đủ Keyboard Navigation & Focus Return**: Dùng được 100% bằng `ArrowUp/Down`, `Enter`, `Escape` và trả focus về nút bấm khi đóng. | ❌ **CẤM chỉ bắt sự kiện `onClick` bằng chuột**: Vi phạm nghiêm trọng chuẩn tiếp cận WCAG 2.1 AA cho người dùng bàn phím & Screen Reader. |
| ✅ **Chống cắt khung nhìn (Flip / Portal)**: Kiểm tra khoảng trống màn hình và sử dụng Portal khi đặt dropdown trong Table/Modal. | ❌ **CẤM để popup bị cắt khuất bởi `overflow: hidden`** của thẻ cha hoặc tràn ra khỏi mép dưới màn hình. |

