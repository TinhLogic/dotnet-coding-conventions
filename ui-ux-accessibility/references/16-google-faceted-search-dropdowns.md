# Quy Chuẩn Tìm Kiếm Phân Nhánh & Dropdown Dữ Liệu Lớn (Chuẩn Google Cloud Omnibox & Material 3)

Xử lý tìm kiếm và bộ chọn thả xuống (Dropdown) trên các trang dữ liệu lớn là bài toán tương tác phức tạp nhất trong kiến trúc Frontend. Nếu triển khai sai:
* Người dùng sẽ bị tràn ngập dữ liệu, không thể lọc chính xác thứ họ cần.
* Các truy vấn tìm kiếm gửi ồ ạt sẽ làm sập Database hoặc nghẽn mạng do Race Condition.
* Dropdown nhiều dữ liệu sẽ làm đơ DOM hoặc bị cắt cụt bởi các thẻ cha có `overflow: hidden`.
* Trên thiết bị di động, người dùng không thể chạm chính xác vào các dòng nhỏ của một menu lơ lửng.

Tài liệu này chuẩn hóa toàn bộ kiến trúc **Tìm kiếm phân nhánh (Faceted / Tokenized Search Bar)** và **Dropdown dữ liệu lớn (Big Data Combobox / Bottom Sheet)** theo mô hình của **Google Cloud Console, Gmail, Google Drive và Google Material Design 3**.

---

## 1. Kiến Trúc Thanh Tìm Kiếm Phân Nhánh (Faceted Tokenized Search Bar)

Trong các hệ thống quản trị dữ liệu lớn của Google, thanh tìm kiếm không phải là một ô nhập text đơn thuần (`<input type="text">`), mà là một **Omnibox đa năng** kết hợp giữa văn bản tự do và các **Filter Chips**.

```
+----------------------------------------------------------------------------------------------------+
|  🔍 [ Status : RUNNING  x ] [ Zone : us-central1-a  x ] [ Nhập thêm thuộc tính hoặc từ khóa... ]  |
+----------------------------------------------------------------------------------------------------+
|  Menu gợi ý 3 bước (Guided Auto-complete):                                                         |
|  1. Thuộc tính (Properties):  🏷️ status, 🌐 zone, 💻 machineType, 📦 name                          |
|  2. Toán tử (Operators):      : (bằng/chứa), = (chính xác), != (khác), > (lớn hơn), < (nhỏ hơn)    |
|  3. Giá trị (Values):         RUNNING, STOPPED, SUSPENDED...                                       |
+----------------------------------------------------------------------------------------------------+
```

### 1.1. Luồng Gợi Ý 3 Bước Tự Động (Guided Auto-complete Pipeline)
Khi người dùng đặt con trỏ vào ô tìm kiếm:
1. **Bước 1 (Chọn Thuộc tính - Field/Facet Key)**: Menu dropdown hiện danh sách các trường dữ liệu có thể lọc (ví dụ: `Tên`, `Trạng thái`, `Khu vực`, `Ngày tạo`).
2. **Bước 2 (Chọn Toán tử - Operator)**: Khi người dùng đã chọn một thuộc tính (ví dụ `status`), hệ thống gợi ý toán tử phù hợp với kiểu dữ liệu của trường đó:
   * Trường chuỗi / Enum: `:` (chứa/bằng), `!=` (khác).
   * Trường số / Ngày tháng: `=`, `!=`, `>`, `<`, `>=`, `<=`.
3. **Bước 3 (Chọn Giá trị - Value)**:
   * Nếu là Enum/Danh mục cố định: Dropdown hiện danh sách các giá trị có sẵn để click chọn.
   * Nếu là chuỗi tự do: Người dùng gõ text và nhấn `Enter`.
   * **Ngay lập tức**, tổ hợp `Thuộc tính + Toán tử + Giá trị` được nén lại thành một **Search Chip** độc lập.

### 1.2. Máy Trạng Thái Bàn Phím Cho Filter Chips (Keyboard State Machine)
AI Agent khi sinh code cho thanh tìm kiếm dạng Chip **bắt buộc phải cài đặt 4 hành vi phím**:

| Phím bấm | Ngữ cảnh | Hành vi chuẩn Google |
| :--- | :--- | :--- |
| **`Backspace`** | Con trỏ đang ở ô nhập và ô nhập đang **RỖNG** | **Lần 1**: Đánh dấu chọn (Focus/Highlight) vào Chip cuối cùng.<br>**Lần 2**: Xóa Chip đó khỏi danh sách và kích hoạt lọc lại. |
| **`ArrowLeft` / `ArrowRight`** | Con trỏ ở ô nhập rỗng | Di chuyển tiêu điểm bàn phím qua lại giữa các Chip đang có để người dùng có thể xóa hoặc chỉnh sửa Chip bất kỳ. |
| **`Enter`** | Đang có gợi ý trong menu | Chọn gợi ý đang highlight; nếu đã đủ `Key:Value` thì đóng gói thành Chip và kích hoạt tìm kiếm. |
| **`Escape`** | Dropdown gợi ý đang mở | Đóng dropdown gợi ý, giữ nguyên các Chip hiện có. |
| **Phím tắt `/`** | Bất kỳ vị trí nào trên trang | Tự động đưa con trỏ chuột (`focus()`) vào thanh tìm kiếm chính. |
| **`Cmd + K` / `Ctrl + K`** | Toàn hệ thống | Mở Command Palette / Hộp thoại tìm kiếm tài nguyên toàn cục. |

### 1.3. Kỹ Thuật Tối Ưu Hiệu Năng Tầng Mạng (Network & Client Optimization)

1. **Debounce 300ms bắt buộc**: Mọi ký tự gõ vào ô tìm kiếm trước khi gửi request lấy danh sách gợi ý (auto-complete suggestions) bắt buộc phải qua hàm Debounce tối thiểu 300ms.
2. **Hủy yêu cầu cũ bằng `AbortController` (Chống Race Condition)**:
   * Khi người dùng gõ ký tự thứ 2, request đang bay (in-flight request) của ký tự thứ 1 phải bị hủy ngay lập tức bằng `abortController.abort()`.
   * Điều này ngăn chặn triệt để tình trạng: Request 1 về sau request 2 làm đè kết quả gợi ý cũ lên màn hình.
3. **LRU Client-Side Cache (Bộ nhớ đệm gợi ý)**: Lưu tạm 50 truy vấn gợi ý gần nhất trong RAM client. Nếu người dùng gõ lại từ vừa tìm, hiển thị ngay kết quả với độ trễ 0ms mà không gọi lại mạng.

---

## 2. Kiến Trúc Dropdown & Combobox Cho Dữ Liệu Lớn (Big Data Combobox)

Trong các ứng dụng dữ liệu lớn, một dropdown có thể chứa từ vài trăm đến hàng chục nghìn mục (ví dụ: Chọn Project trong GCP, chọn SKU hàng hóa, chọn Khách hàng).

### 2.1. Ma Trận Phân Cấp Dropdown
* **Cấp 1 (< 8 mục tĩnh)**: Dropdown đơn giản, không cần ô tìm kiếm, hiển thị toàn bộ mục, có icon `Check` ở mục đang chọn.
* **Cấp 2 ($\ge 8$ mục hoặc tải động)**: Searchable Combobox — có ô tìm kiếm tích hợp ở đỉnh, tự động `autofocus` khi mở, có nút xóa nhanh `Clear ×`.
* **Cấp 3 (> 100 mục hoặc danh sách lớn)**:
  * **Ảo hóa DOM bên trong Dropdown (Virtual Scrolling Dropdown)**: Áp dụng Virtual Scroll ngay bên trong khung cuộn của menu thả xuống. Dù danh sách có 10,000 mục, số lượng phần tử DOM thực tế được tạo ra chỉ từ **10 đến 15 phần tử** ($O(1)$ Memory).
  * **Phân trang dạng Con trỏ (Infinite Cursor Loading)**: Khi người dùng cuộn đến 80% chiều cao của menu, tự động gọi API lấy trang tiếp theo bằng `pageToken`.

### 2.2. Chống Cắt Khung Nhìn (Anti-Clipping Bằng Portal & Auto-Flipping)
* ❌ **CẤM**: Render khung Dropdown bên trong thẻ `<td>` hoặc container của Table vì sẽ bị `overflow: hidden / auto` cắt cụt mất phần menu thả xuống.
* ✅ **BẮT BUỘC**:
  1. Sử dụng **Portal** (React Portal / Vue Teleport / Angular CDK Overlay) hoặc **CSS Popover API (`popover="auto"`)** để gắn menu trực tiếp vào `<body>`.
  2. Định vị menu theo tọa độ tuyệt đối thông qua `getBoundingClientRect()` của nút kích hoạt (Trigger).
  3. **Tự động đảo chiều (Auto-flipping)**: Nếu khoảng cách từ nút đến đáy màn hình `< 260px`, menu phải tự động đảo chiều hiển thị lên phía trên nút bấm (`placement: top-start`).

### 2.3. Multi-Select Dropdown & Cơ Chế Thu Gọn Thẻ (Tag Condensation)
Khi người dùng chọn nhiều mục trong một Dropdown:
- ❌ **CẤM**: Nhồi nhét toàn bộ 10 Badge vào ô chọn khiến ô bị phình to làm vỡ chiều cao hàng của bảng.
- ✅ **BẮT BUỘC (Quy tắc nén thẻ - Badge Condensation)**:
  * Chọn 1 mục: Hiển thị tên mục (VD: `Vùng: us-central1-a`).
  * Chọn 2 mục: Hiển thị cả 2 mục (VD: `us-central1-a, us-east1-b`).
  * Chọn $\ge 3$ mục: Hiển thị mục đầu tiên kèm Badge đếm số lượng còn lại:
    $$\text{Vùng: us-central1-a, +2 khác}$$
  * Có nút *"Chọn tất cả (Select All)"* và *"Xóa bộ lọc (Clear All)"* ở hàng đầu tiên của danh sách.

---

## 3. Kiến Trúc Thích Ứng Thiết Bị Di Động (Mobile-Adaptive Architecture)

Giao diện Desktop sử dụng chuột có độ chính xác cao (pixel-precision), trong khi Mobile sử dụng ngón tay với vùng chạm lớn (touch target $\ge 48\text{px}$). Do đó, Google áp dụng quy tắc **chuyển đổi hình thái**:

```
+----------------------------------------------------------------------------------------------------+
|  TRÊN DESKTOP (>= 768px)                     |  TRÊN MOBILE (< 768px)                              |
+----------------------------------------------------------------------------------------------------+
|  - Dropdown: Menu lơ lửng dưới nút bấm       |  - Dropdown: Biến thành MODAL BOTTOM SHEET          |
|  - Chiều cao dòng: 32px - 36px               |  - Chiều cao dòng: Tối thiểu 48px                   |
|  - Search: Nằm trong thanh công cụ           |  - Search: Biến thành FULL-SCREEN SEARCH OVERLAY    |
|  - Thao tác: Click chuột, phím tắt           |  - Thao tác: Vuốt ngón tay (Swipe down to dismiss)  |
+----------------------------------------------------------------------------------------------------+
```

### 3.1. Dropdown Biến Thành Modal Bottom Sheet Trên Mobile
Khi người dùng nhấn vào bất kỳ Dropdown / Bộ lọc nào trên màn hình `< 768px`:
1. Menu không mở lơ lửng mà một **Bottom Sheet** trượt từ mép dưới màn hình lên (chiếm từ 60% đến 90% chiều cao màn hình).
2. Có thanh tay cầm vuốt nhẹ (**Drag Handle / Scrim**) ở đỉnh để người dùng có thể vuốt xuống để đóng.
3. Ô tìm kiếm được gắn cố định (`position: sticky; top: 0`) ở đỉnh Bottom Sheet để khi bàn phím ảo bật lên, ô tìm kiếm không bị đẩy ra ngoài tầm nhìn.
4. Mỗi dòng lựa chọn có chiều cao tối thiểu **48px** kèm Checkbox to rõ.
5. Ở đáy Bottom Sheet có một thanh hành động cố định gồm 2 nút: **"Thiết lập lại (Reset)"** và **"Áp dụng (Apply)"**.

### 3.2. Search Bar Biến Thành Full-Screen Search View Trên Mobile
* Khi người dùng chạm vào ô Search trên điện thoại:
  1. Trình duyệt lập tức mở chế độ **Toàn màn hình (Full-Screen Overlay)**.
  2. Bàn phím ảo tự động kích hoạt.
  3. Có nút mũi tên quay lại `←` ở góc trái để đóng chế độ tìm kiếm.
  4. Hiển thị danh sách các tìm kiếm gần đây (Recent Searches) và các Filter Chips gợi ý nhanh ngay bên dưới ô nhập.

---

## 4. Chuẩn Tiếp Cận Trợ Năng (WAI-ARIA 1.2 Combobox Pattern)

Để đảm bảo người dùng khiếm thị dùng Screen Reader và người dùng bàn phím có thể thao tác 100% không cần chuột, bắt buộc phải triển khai cấu trúc ngữ nghĩa sau:

```html
<!-- Container bao ngoài cho Combobox -->
<div class="g-combobox" role="group" aria-label="Bộ lọc tài nguyên">

  <!-- Ô nhập tìm kiếm kiêm Trigger điều khiển -->
  <input 
    type="text" 
    role="combobox"
    aria-expanded="false" 
    aria-haspopup="listbox" 
    aria-autocomplete="list" 
    aria-controls="filter-listbox-popup" 
    aria-activedescendant=""
    placeholder="Lọc tài nguyên hoặc nhấn '/'..."
  />

  <!-- Khung danh sách popup (Gắn qua Portal ở <body>) -->
  <ul 
    id="filter-listbox-popup" 
    role="listbox" 
    aria-label="Danh sách gợi ý lọc"
    aria-multiselectable="true"
    tabindex="-1"
  >
    <!-- Các mục lựa chọn -->
    <li id="opt-status" role="option" aria-selected="false" class="combobox-item">
      <span class="item-icon" aria-hidden="true">🏷️</span>
      <span class="item-label">status</span>
    </li>
    <li id="opt-zone" role="option" aria-selected="true" class="combobox-item is-active">
      <span class="item-icon" aria-hidden="true">🌐</span>
      <span class="item-label">zone</span>
    </li>
  </ul>
</div>
```

### Các thuộc tính ARIA bắt buộc giải thích:
* `role="combobox"`: Khai báo với Screen Reader đây là ô nhập liệu kết hợp danh sách gợi ý.
* `aria-expanded="true/false"`: Báo trạng thái danh sách popup đang mở hay đóng.
* `aria-autocomplete="list"`: Báo cho người dùng biết khi gõ text sẽ có danh sách gợi ý hiện ra bên dưới.
* `aria-activedescendant="[ID_CUA_ITEM]"`: Khi người dùng bấm phím mũi tên `ArrowDown` / `ArrowUp`, thuộc tính này nhận ID của mục đang được vạch sáng (highlight). Screen Reader sẽ đọc to mục đó lên mà **không cần di chuyển con trỏ chuột ra khỏi ô input**.
* `aria-selected="true/false"`: Báo trạng thái mục đã được chọn hay chưa.

---

## 5. Mẫu Code Chuẩn Doanh Nghiệp (TypeScript / React Implementation)

Dưới đây là mã nguồn chuẩn hóa của Hook tìm kiếm kết hợp **Debounce + AbortController** và Component **Filter Chips Bar**:

### 5.1. Hook Tìm Kiếm An Toàn (Debounced Async Search Hook)
```typescript
import { useState, useEffect, useRef } from 'react';

export function useDebouncedSearch<T>(
  query: string,
  fetchFn: (searchTerm: string, signal: AbortSignal) => Promise<T[]>,
  delay: number = 300
) {
  const [data, setData] = useState<T[]>([]);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<Error | null>(null);
  const abortControllerRef = useRef<AbortController | null>(null);

  useEffect(() => {
    // Hủy request cũ đang bay nếu người dùng gõ tiếp
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    if (!query.trim()) {
      setData([]);
      setLoading(false);
      return;
    }

    const handler = setTimeout(async () => {
      const controller = new AbortController();
      abortControllerRef.current = controller;
      setLoading(true);
      setError(null);

      try {
        const results = await fetchFn(query, controller.signal);
        setData(results);
      } catch (err: any) {
        if (err.name !== 'AbortError') {
          setError(err);
        }
      } finally {
        setLoading(false);
      }
    }, delay);

    return () => {
      clearTimeout(handler);
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, [query, delay, fetchFn]);

  return { data, loading, error };
}
```

### 5.2. Component Thanh Tìm Kiếm Dạng Chip (Tokenized Chips Bar)
```tsx
import React, { useState, useRef, KeyboardEvent } from 'react';

export interface FilterChip {
  id: string;
  field: string;
  operator: string;
  value: string;
}

interface FilterBarProps {
  chips: FilterChip[];
  onAddChip: (chip: FilterChip) => void;
  onRemoveChip: (chipId: string) => void;
  onSearchChange: (text: string) => void;
}

export const TokenizedFilterBar: React.FC<FilterBarProps> = ({
  chips,
  onAddChip,
  onRemoveChip,
  onSearchChange,
}) => {
  const [inputValue, setInputValue] = useState('');
  const [focusedChipIndex, setFocusedChipIndex] = useState<number | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    // Xử lý phím Backspace khi ô nhập đang trống
    if (e.key === 'Backspace' && inputValue === '') {
      if (chips.length === 0) return;

      if (focusedChipIndex === null) {
        // Lần 1: Highlight chip cuối cùng
        setFocusedChipIndex(chips.length - 1);
      } else {
        // Lần 2: Xóa chip đang được highlight
        onRemoveChip(chips[focusedChipIndex].id);
        setFocusedChipIndex(null);
      }
    } else {
      setFocusedChipIndex(null);
    }
  };

  return (
    <div 
      className="g-filter-bar" 
      onClick={() => inputRef.current?.focus()}
      role="search"
    >
      <span className="search-icon" aria-hidden="true">🔍</span>

      {/* Render danh sách Filter Chips */}
      {chips.map((chip, index) => (
        <span 
          key={chip.id} 
          className={`filter-chip ${focusedChipIndex === index ? 'is-focused' : ''}`}
        >
          <span className="chip-key">{chip.field}</span>
          <span className="chip-operator">{chip.operator}</span>
          <span className="chip-value">{chip.value}</span>
          <button 
            type="button" 
            className="chip-remove-btn"
            aria-label={`Xóa bộ lọc ${chip.field}`}
            onClick={(e) => {
              e.stopPropagation();
              onRemoveChip(chip.id);
            }}
          >
            ×
          </button>
        </span>
      ))}

      {/* Ô nhập từ khóa chính */}
      <input
        ref={inputRef}
        type="text"
        className="filter-input"
        value={inputValue}
        placeholder={chips.length === 0 ? "Lọc theo thuộc tính hoặc từ khóa (nhấn '/' để tìm)..." : "Nhập thêm bộ lọc..."}
        onChange={(e) => {
          setInputValue(e.target.value);
          onSearchChange(e.target.value);
        }}
        onKeyDown={handleKeyDown}
      />
    </div>
  );
};
```

### 5.3. CSS Bố Cục Chuẩn Google Cloud Cho Filter Bar & Bottom Sheet
```css
/* Container thanh lọc dạng Chip */
.g-filter-bar {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  padding: 6px 12px;
  background-color: var(--surface-bg, #ffffff);
  border: 1px solid var(--border-color, #dadce0);
  border-radius: 8px;
  min-height: 44px;
  cursor: text;
  transition: border-color 0.15s, box-shadow 0.15s;
}

.g-filter-bar:focus-within {
  border-color: #1a73e8;
  box-shadow: 0 1px 2px rgba(60, 64, 67, 0.3), 0 1px 3px 1px rgba(60, 64, 67, 0.15);
}

/* Thẻ Filter Chip */
.filter-chip {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  background-color: #e8f0fe;
  color: #1967d2;
  border: 1px solid #d2e3fc;
  border-radius: 16px;
  font-size: 0.8125rem;
  font-weight: 500;
  user-select: none;
}

.filter-chip.is-focused {
  outline: 2px solid #1a73e8;
  background-color: #d2e3fc;
}

.chip-operator {
  color: #5f6368;
}

.chip-remove-btn {
  background: none;
  border: none;
  color: #1967d2;
  font-size: 14px;
  cursor: pointer;
  padding: 0 2px;
  display: flex;
  align-items: center;
}

/* Ô nhập liệu co giãn tự nhiên */
.filter-input {
  flex: 1;
  min-width: 140px;
  border: none;
  outline: none;
  font-size: 0.875rem;
  background: transparent;
  color: inherit;
}

/* Biến đổi trên Mobile (< 768px): Modal Bottom Sheet */
@media (max-width: 767px) {
  .g-dropdown-popup {
    position: fixed !important;
    top: auto !important;
    bottom: 0 !important;
    left: 0 !important;
    right: 0 !important;
    width: 100% !important;
    max-height: 85vh;
    border-radius: 16px 16px 0 0;
    box-shadow: 0 -4px 20px rgba(0, 0, 0, 0.2);
    animation: slideUp 0.25s cubic-bezier(0.16, 1, 0.3, 1);
    z-index: 9999;
  }

  .g-dropdown-item {
    min-height: 48px; /* Đảm bảo vùng chạm tối thiểu chuẩn WCAG */
    font-size: 1rem;
    padding: 12px 16px;
  }
}

@keyframes slideUp {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}
```

---

## 6. Checklist Nghiệm Thu Dành Cho AI Agent (AI Execution Checklist)

Trước khi xác nhận hoàn thành bất kỳ chức năng Tìm kiếm (Search) hoặc Bộ chọn (Dropdown) nào, AI Agent **bắt buộc phải đối chiếu danh sách kiểm tra sau**:

- [ ] **Thanh tìm kiếm có hỗ trợ Chip (Tokenized) không?** Các bộ lọc đã chọn có được đóng gói thành các Chip trực quan có nút xóa `×` không?
- [ ] **Đã xử lý phím `Backspace` trên Chip chưa?** Bấm `Backspace` khi ô nhập trống có highlight và xóa Chip liền trước không?
- [ ] **Đã có Debounce tối thiểu 300ms chưa?** Có đảm bảo không bắn request liên tục sau từng phím gõ không?
- [ ] **Đã dùng `AbortController` chưa?** Request tìm kiếm cũ có bị hủy tự động khi người dùng gõ ký tự mới không?
- [ ] **Dropdown có bị cắt bởi `overflow: hidden` không?** Đã dùng Portal / Floating Overlay để đẩy danh sách ra thẻ gốc `<body>` chưa?
- [ ] **Dropdown $\ge 8$ mục có ô tìm kiếm nhanh không?** Ô tìm kiếm có tự động `autofocus` khi mở menu không?
- [ ] **Dropdown nhiều mục (> 100 mục) có áp dụng ảo hóa DOM (Virtual Scroll) không?**
- [ ] **Multi-select có áp dụng cơ chế nén thẻ (Badge Condensation) không?** (Ví dụ: `Vùng: US, EU (+2 khác)` thay vì làm vỡ layout).
- [ ] **Trên Mobile (< 768px), Dropdown đã chuyển sang Modal Bottom Sheet chưa?** Vùng chạm của mỗi dòng có đạt tối thiểu 48px không?
- [ ] **Toàn bộ bộ lọc và từ khóa đã đồng bộ 2 chiều lên URL Query Params chưa?**
- [ ] **Đã khai báo đầy đủ thuộc tính WAI-ARIA 1.2 Combobox chưa?** (`role="combobox"`, `aria-expanded`, `aria-autocomplete="list"`, `aria-controls`, `aria-activedescendant`).
