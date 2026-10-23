# Enterprise AI Skills Collection

Tổng hợp các bộ quy chuẩn kỹ thuật và kỹ năng (**Skills**) dành cho AI Coding Assistant (Google Antigravity, Gemini CLI, Claude, Cursor, v.v.) và các đội ngũ phát triển phần mềm Enterprise.

---

## 📦 Danh Sách Skills

| Skill | Thư Mục | Ngôn Ngữ / Nền Tảng | Mô Tả Tóm Tắt |
| :--- | :--- | :--- | :--- |
| **`dotnet-coding-conventions`** | [`dotnet-coding-conventions/`](./dotnet-coding-conventions/SKILL.md) | C#, .NET, ASP.NET Core, Blazor | Quy chuẩn lập trình C#/.NET doanh nghiệp, Clean Architecture, SOLID, EF Core, Async/Await, DTO design, Testing. |
| **`ui-ux-accessibility`** | [`ui-ux-accessibility/`](./ui-ux-accessibility/SKILL.md) | Universal (React, Vue, Blazor, Angular, Svelte, HTML/CSS) | Chuẩn tiếp cận Web WAI-ARIA (WCAG 2.1 AA), ảo hóa $O(1)$ DOM, Skeleton Loading, Design Tokens, Form guard, Command Palette, Deep Linking, i18n, Error Boundaries, Bảo mật XSS/CSP, Core Web Vitals. |
| **`rfc9457-madr-standards`** | [`rfc9457-madr-standards/`](./rfc9457-madr-standards/SKILL.md) | Universal (REST APIs, Microservices, System Architecture) | Định dạng lỗi API chuẩn IETF RFC 9457 (Problem Details - thay thế RFC 7807), ghi nhận quyết định kiến trúc MADR 3.0, tài liệu kỹ thuật API và mô hình C4 cấu trúc hệ thống. |

---

## 🚀 Cách Cài Đặt & Sử Dụng

### 1. Dành cho Google Antigravity / Gemini CLI
Copy hoặc tạo symlink thư mục skill mong muốn vào thư mục cấu hình skills cá nhân (`~/.gemini/config/skills/` hoặc `~/.gemini/antigravity/skills/`):

```bash
# Cài đặt cả 3 skills vào cấu hình Antigravity/Gemini
mkdir -p ~/.gemini/config/skills
ln -s "$(pwd)/dotnet-coding-conventions" ~/.gemini/config/skills/dotnet-coding-conventions
ln -s "$(pwd)/ui-ux-accessibility" ~/.gemini/config/skills/ui-ux-accessibility
ln -s "$(pwd)/rfc9457-madr-standards" ~/.gemini/config/skills/rfc9457-madr-standards
```

### 2. Nguyên Tắc Thiết Kế Skill
- **Progressive Disclosure**: File `SKILL.md` đóng vai trò là mục lục tra cứu nhanh, checklist và quy tắc ngắn gọn để tiết kiệm context window của LLM.
- **Tài liệu tham khảo chuyên sâu (`references/`)**: Chứa phân tích chuyên sâu, ví dụ phản diện (Bad) và chính diện (Good) để AI đọc chi tiết khi cần giải quyết vấn đề phức tạp.

