# .NET & C# Enterprise Coding Conventions

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-6.0%20%7C%207.0%20%7C%208.0%20%7C%209.0-purple.svg)](https://dotnet.microsoft.com/)
[![C#](https://img.shields.io/badge/C%23-10%20%7C%2011%20%7C%2012%20%7C%2013-blue.svg)](https://learn.microsoft.com/en-us/dotnet/csharp/)

> **Ngôn ngữ / Languages**: [Tiếng Việt](#ti%E1%BA%BFng-vi%E1%BB%87t) | [English](#english)

---

# Tiếng Việt

Bộ quy chuẩn lập trình bắt buộc dành cho mọi mã nguồn C# / .NET, ASP.NET Core và Blazor. Dự án này được thiết kế theo chuẩn **Skill** cho AI Coding Assistant (Google Antigravity, Gemini CLI, Claude, Cursor, v.v.) và cẩm nang quy chuẩn cho các đội ngũ phát triển phần mềm Enterprise.

## 🎯 Mục Tiêu
- **Nhất quán & Tiêu chuẩn hóa**: Đảm bảo toàn bộ mã nguồn tuân theo kiến trúc Clean Architecture, SOLID và GoF Design Patterns.
- **Thân thiện với AI Agent**: Cung cấp bộ quy tắc rõ ràng, có checklist nghiệm thu cụ thể để AI không tự ý viết code cẩu thả hoặc áp dụng sai pattern.
- **Progressive Disclosure**: Bản tóm tắt nằm tại [`SKILL.md`](./SKILL.md), trong khi hướng dẫn chuyên sâu kèm ví dụ phản diện (Bad) và chính diện (Good) được phân tách chi tiết trong thư mục [`references/`](./references/).

---

## 📚 Cấu Trúc Tài Liệu Quy Chuẩn

| Phần | Nhóm Quy Tắc | Các Rule | Tài Liệu Chi Tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Thiết Kế & Cấu Trúc DTO** | Rule 1 - 4 | [`01-dto-design.md`](./references/01-dto-design.md) |
| **II** | **Coding Style & Định Dạng C#** | Rule 5 - 10.4 | [`02-csharp-style.md`](./references/02-csharp-style.md) |
| **III & IV** | **JSON & Bất Đồng Bộ (Async)** | Rule 11 - 14 | [`03-json-async.md`](./references/03-json-async.md) |
| **V** | **API Controller & Swagger** | Rule 15 - 16 | [`04-api-controller.md`](./references/04-api-controller.md) |
| **VI** | **Kiến Trúc & Tích Hợp Đối Tác** | Rule 17 - 19 | [`05-architecture-business.md`](./references/05-architecture-business.md) |
| **VII** | **Quy Trình Git & Build** | Rule 20 - 23 | [`06-workflow-git-build.md`](./references/06-workflow-git-build.md) |
| **VIII** | **Design Pattern & Quản Lý Độ Phức Tạp** | Rule 24 | [`07-design-patterns.md`](./references/07-design-patterns.md) |
| **IX** | **Giao Diện UI & Accessibility** | Rule 25 - 28 | [`08-ui-accessibility.md`](./references/08-ui-accessibility.md) |

---

## ⚡ Cheatsheet: Nguyên Tắc Cốt Lõi

1. **DTO & Model**:
   - Mỗi DTO một file riêng, đặt tên trùng với tên file.
   - Hậu tố bắt buộc là `Dto` và tên phải là Danh từ (`PaymentCreationRequestDto`, không dùng `CreatePaymentDto`).
   - Phân biệt rõ: `InputDto`/`OutputDto` với Caller nội bộ; `RequestDto`/`ResponseDto` với Third-party API.
   - Không để lộ các trường nội bộ (signature, secret, internalId) ra ngoài public `InputDto`.

2. **C# Style & Clean Code**:
   - Sử dụng PascalCase cho thuộc tính; cách nhau 1 dòng trống; dùng `[JsonPropertyName("...")]` để map JSON.
   - Dùng từ khóa nguyên thủy (`string`, `int`, `bool`) thay cho CLR types; ưu tiên `int` có dấu; static field có tiền tố `s_`, thread static `t_`; hằng số (`const`) PascalCase (cấm `UPPER_SNAKE_CASE`); enum số ít (flags số nhiều); generic tiền tố `T`.
   - "Sealed by default": Mọi concrete class (Service, Strategy, Repository, Validator...) không chủ đích kế thừa bắt buộc đánh dấu `sealed` (hoặc `internal sealed`) để bảo vệ đóng gói và kích hoạt JIT Devirtualization.
   - XML Doc và comment song ngữ: Tiếng Việt trước, tiếng Anh sau; comment giải thích **TẠI SAO**, không tả lại code; xóa hẳn dead code.
   - Mọi `TODO`/`FIXME` bắt buộc phải kèm mã Jira (ví dụ: `// TODO(PROJ-123): ...`).
   - Khối `if`/`else` luôn có ngoặc nhọn `{ }`. Áp dụng Guard Clauses (Early Return) để phẳng hóa luồng xử lý.
   - Sử dụng file-scoped namespace (`namespace App;`) cho .NET 6+; đặt `using` ở đầu file bên ngoài namespace.

3. **JSON & Bất Đồng Bộ (Async)**:
   - Dùng `System.Text.Json` thay cho `Newtonsoft.Json`.
   - Thuộc tính nullable có `[JsonIgnore(Condition = WhenWritingNull)]`.
   - Mọi phương thức trả về `Task`/`ValueTask` bắt buộc có hậu tố `Async` và nhận `CancellationToken cancellationToken = default` ở cuối tham số.

4. **API Controller & Swagger**:
   - Đường dẫn chuẩn RESTful (danh từ số nhiều, kebab-case, không gắn động từ vào URI).
   - Khai báo đầy đủ `[SwaggerOperation]` và `[SwaggerResponse]`. Tuyệt đối không để lộ URL nội bộ/đối tác.

5. **Kiến Trúc & Tích Hợp**:
   - Validate dữ liệu đầu vào bằng `FluentValidation` (`AbstractValidator<T>`), không validate thủ công trong Service.
   - Giao tiếp HTTP bắt buộc dùng `IHttpClientFactory`.
   - Tích hợp đối tác theo mô hình Orchestrator + Functional Pipeline + Dynamic Proxy Interceptor + GoF Decorator.

6. **GoF Design Patterns**:
   - Khi logic vượt ngưỡng phức tạp (≥ 4 nhánh `switch`/`if` cùng enum, phương thức > 80 dòng, lồng sâu > 3 cấp) phải tái cấu trúc bằng GoF Pattern (Strategy, Factory, Decorator, Adapter).
   - Tuân thủ nghiêm ngặt nguyên lý SOLID (đặc biệt là ISP và LSP).

7. **UI, Accessibility & SVG**:
   - Phân trang & lọc phía SQL qua EF Core (`AsNoTracking()`, `Skip()`, `Take()`, `CountAsync()`).
   - Tuân thủ chuẩn tiếp cận WAI-ARIA (`aria-label`, `aria-busy`, `role="dialog"` cho modal).
   - Dữ liệu hiển thị trong bảng là văn bản thuần, không gắn emoji trang trí. Icon dùng SVG vector outline (Lucide / Heroicons).

---

## 📖 Tài Liệu Tham Chiếu Chính Thức (Official Microsoft References)

Các quy chuẩn cốt lõi trong tài liệu này kế thừa trực tiếp và chuẩn hóa dựa trên các hướng dẫn chính thức từ Microsoft Learn nhằm đảm bảo tính chuẩn mực và tránh tranh cãi kỹ thuật:
- [.NET Common C# Code Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions): Quy ước định dạng mã nguồn C#, bố cục mã, cú pháp và thực hành chuẩn của .NET Runtime & Docs team.
- [C# Identifier Naming Rules & Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/identifier-names): Quy chuẩn đặt tên định danh (PascalCase, camelCase, tiền tố `s_` static fields, hằng số `const`, Enum flags, generic type parameters).
- [Abstract and Sealed Classes and Class Members](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/abstract-and-sealed-classes-and-class-members): Nguyên lý kế thừa, thiết kế abstraction và thực hành "Sealed by default" tối ưu JIT Devirtualization.

---

## 🛠 Hướng Dẫn Cài Đặt Cho AI Coding Assistant

### 1. Cài đặt toàn cục cho Google Antigravity / Gemini CLI:
Tạo symlink từ thư mục skill toàn cục của Antigravity trỏ vào thư mục repository này:
```bash
ln -s ~/Documents/Github/dotnet-coding-conventions ~/.gemini/config/skills/dotnet-coding-conventions
```

### 2. Sử dụng trong dự án cụ thể (Project Scope):
Sao chép hoặc tạo symlink vào thư mục `.agents/skills/` hoặc `.gemini/skills/` của project mục tiêu:
```bash
mkdir -p .agents/skills
ln -s ~/Documents/Github/dotnet-coding-conventions .agents/skills/dotnet-coding-conventions
```

---

<br/>

# English

Mandatory enterprise coding standards and guidelines for all C# / .NET, ASP.NET Core, and Blazor codebases. This repository is architected as an **AI Agent Skill** (for Google Antigravity, Gemini CLI, Claude, Cursor, etc.) and an enterprise handbook for engineering teams.

## 🎯 Objectives
- **Consistency & Standardization**: Ensure all code strictly complies with Clean Architecture, SOLID principles, and GoF Design Patterns.
- **AI Agent Alignment**: Provide explicit rules, anti-patterns, and an end-of-task verification checklist to prevent AI hallucinations or anti-pattern code generation.
- **Progressive Disclosure**: Core principles and summary checklists reside in [`SKILL.md`](./SKILL.md), while detailed explanations and comprehensive Good/Bad examples are organized in the [`references/`](./references/) folder.

---

## 📚 Documentation & Reference Directory

| Part | Topic Group | Rules | Detailed Document |
| :--- | :--- | :--- | :--- |
| **I** | **DTO Design & Structure** | Rule 1 - 4 | [`01-dto-design.md`](./references/01-dto-design.md) |
| **II** | **C# Coding Style & Formatting** | Rule 5 - 10.4 | [`02-csharp-style.md`](./references/02-csharp-style.md) |
| **III & IV** | **JSON & Asynchronous Programming** | Rule 11 - 14 | [`03-json-async.md`](./references/03-json-async.md) |
| **V** | **API Controller & Swagger** | Rule 15 - 16 | [`04-api-controller.md`](./references/04-api-controller.md) |
| **VI** | **Architecture & Partner Integration** | Rule 17 - 19 | [`05-architecture-business.md`](./references/05-architecture-business.md) |
| **VII** | **Git Workflow & Modern Build** | Rule 20 - 23 | [`06-workflow-git-build.md`](./references/06-workflow-git-build.md) |
| **VIII** | **Design Patterns & Complexity Management** | Rule 24 | [`07-design-patterns.md`](./references/07-design-patterns.md) |
| **IX** | **UI Components & Accessibility** | Rule 25 - 28 | [`08-ui-accessibility.md`](./references/08-ui-accessibility.md) |

---

## ⚡ Core Cheatsheet Principles

1. **DTO & Models**:
   - Single class per file; file name matches class name.
   - Mandatory `Dto` suffix; class name must be a noun phrase (`PaymentCreationRequestDto`, never `CreatePaymentDto`).
   - Strict boundary naming: `InputDto`/`OutputDto` for internal callers; `RequestDto`/`ResponseDto` for external partners.
   - Never expose internal fields (signatures, secrets, internal IDs) on public `InputDto`.

2. **C# Style & Clean Code**:
   - PascalCase properties; separated by one blank line; mapped with `[JsonPropertyName("...")]`.
   - Use language keywords (`string`, `int`, `bool`) over CLR types; prefer signed `int`; static fields prefix `s_`, thread static `t_`; PascalCase constants (no `UPPER_SNAKE_CASE`); singular enums (plural for `[Flags]`); generic prefix `T`.
   - "Sealed by default": Concrete classes (Services, Strategies, Repositories, Validators...) must be marked `sealed` (or `internal sealed`) unless explicitly designed for inheritance to preserve encapsulation and enable JIT Devirtualization.
   - Bilingual XML docs and comments: Vietnamese first, English second. Comments must answer **WHY**, not restate what code does. Remove dead code completely.
   - All `TODO`/`FIXME` items require an issue/ticket ID (e.g., `// TODO(PROJ-123): ...`).
   - Mandatory braces `{ }` for all `if`/`else` statements. Use Guard Clauses (Early Return) to flatten execution flow.
   - Use file-scoped namespaces (`namespace App;`) for .NET 6+; place all `using` directives at top of file outside namespaces.

3. **JSON & Asynchronous (Async)**:
   - Always use `System.Text.Json` over `Newtonsoft.Json`.
   - Nullable DTO properties must include `[JsonIgnore(Condition = WhenWritingNull)]`.
   - All methods returning `Task`/`ValueTask` must have the `Async` suffix and accept `CancellationToken cancellationToken = default` as the trailing parameter.

4. **API Controllers & Swagger**:
   - RESTful URLs (plural nouns, kebab-case, no verbs in path).
   - Document all actions with `[SwaggerOperation]` and `[SwaggerResponse]`. Never leak private or partner endpoints.

5. **Architecture & Integrations**:
   - Request input validation must use `FluentValidation` (`AbstractValidator<T>`), never inline in domain services.
   - HTTP communications must use `IHttpClientFactory`.
   - Third-party integrations follow the Orchestrator + Functional Pipeline + Dynamic Proxy Interceptor + GoF Decorator architecture.

6. **GoF Design Patterns**:
   - Refactor using GoF Patterns (Strategy, Factory, Decorator, Adapter) when complexity thresholds are met (≥ 4 enum branches, methods > 80 lines, nesting > 3 levels).
   - Strictly uphold SOLID principles (notably ISP and LSP).

7. **UI, Accessibility & Icons**:
   - Server-side pagination & filtering via SQL using EF Core (`AsNoTracking()`, `Skip()`, `Take()`, `CountAsync()`).
   - Adhere to WAI-ARIA accessibility (`aria-label`, `aria-busy`, `role="dialog"` for modals).
   - Clean data representation: plain text in table cells, no decorative emojis. Use outline vector SVG icons (Lucide / Heroicons).

---

## 📖 Official Microsoft References

The conventions codified in this repository directly inherit and align with official Microsoft Learn architectural and language guidelines to maintain consistency and eliminate technical ambiguity:
- [.NET Common C# Code Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions): C# source code formatting, layout, modern language constructs, and runtime conventions used by the .NET Runtime and Docs teams.
- [C# Identifier Naming Rules & Conventions](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/identifier-names): Identifier naming standards (PascalCase, camelCase, `s_` static fields, `const` naming, Enum flags, generic type parameters).
- [Abstract and Sealed Classes and Class Members](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/abstract-and-sealed-classes-and-class-members): OOP inheritance principles, virtual members, and "Sealed by default" practices for JIT Devirtualization.

---

## 🛠 Setup & AI Assistant Integration

### 1. Global Setup for Antigravity / Gemini CLI:
Create a symlink from the global skill directory pointing to this repository:
```bash
ln -s ~/Documents/Github/dotnet-coding-conventions ~/.gemini/config/skills/dotnet-coding-conventions
```

### 2. Workspace / Repository Scope:
Copy or symlink into the target project's `.agents/skills/` or `.gemini/skills/` directory:
```bash
mkdir -p .agents/skills
ln -s ~/Documents/Github/dotnet-coding-conventions .agents/skills/dotnet-coding-conventions
```

---

## 📄 License
This project is licensed under the [MIT License](LICENSE).