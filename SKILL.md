---
name: dotnet-coding-conventions
description: >-
  Mandatory skill for writing, reviewing, refactoring, or generating C#, .NET, ASP.NET Core, and Blazor code.
  Auto-triggers whenever the prompt involves: C#, csharp, .NET, dotnet, Blazor, ASP.NET, EF Core, Entity Framework,
  LINQ, DTO, Controller, Service, Repository, CancellationToken, or any .cs file.
  Tự động kích hoạt khi prompt tiếng Việt liên quan đến: viết code C#, tạo DTO, tạo controller, review code .NET,
  refactor C#, xử lý async C#, tối ưu bảng Blazor, WAI-ARIA.
  Agent MUST read this SKILL.md before generating or editing C# code.
---

# .NET & C# Enterprise Coding Conventions

Quy chuẩn bắt buộc cho mọi code C#/.NET và Blazor được sinh ra hoặc chỉnh sửa bởi AI Agent.
AI Agent **phải tuân thủ 100%**, không tự ý bỏ qua. Nếu code hiện tại vi phạm, đề xuất refactor thay vì bắt chước cái sai.

Skill này áp dụng mô hình **Progressive Disclosure**: file `SKILL.md` này tóm tắt nhanh nguyên tắc cốt lõi và checklist. Chi tiết chuyên sâu cho từng mảng được chia tách trong thư mục [references/](./references/).

---

## 📚 Danh Mục Quy Tắc & Tài Liệu Chi Tiết

| Phần | Nhóm quy tắc | Các Rule | Tài liệu chi tiết |
| :--- | :--- | :--- | :--- |
| **I** | **Thiết Kế & Cấu Trúc DTO** | Rule 1 - 4 | [01-dto-design.md](./references/01-dto-design.md) |
| **II** | **Coding Style & Định Dạng C#** | Rule 5 - 10.4 | [02-csharp-style.md](./references/02-csharp-style.md) |
| **III & IV** | **JSON & Bất Đồng Bộ (Async)** | Rule 11 - 14 | [03-json-async.md](./references/03-json-async.md) |
| **V** | **API Controller & Swagger** | Rule 15 - 16 | [04-api-controller.md](./references/04-api-controller.md) |
| **VI** | **Kiến Trúc & Tích Hợp Đối Tác** | Rule 17 - 19 | [05-architecture-business.md](./references/05-architecture-business.md) |
| **VII** | **Quy Trình Git & Build** | Rule 20 - 23 | [06-workflow-git-build.md](./references/06-workflow-git-build.md) |
| **VIII** | **Design Pattern & Quản Lý Độ Phức Tạp** | Rule 24 | [07-design-patterns.md](./references/07-design-patterns.md) |
| **IX** | **Giao Diện UI & Accessibility** | Rule 25 - 28 | [08-ui-accessibility.md](./references/08-ui-accessibility.md) |

---

## ⚡ Cheatsheet: Nguyên Tắc Cốt Lõi Cần Nhớ

### 1. DTO & Model ([Chi tiết](./references/01-dto-design.md))
- **Mỗi class một file riêng**, tên file trùng tên class. Không dùng nested class container (Rule 1).
- **Hậu tố `Dto`** bắt buộc cho mọi DTO (Rule 2).
- **Tên DTO = Danh từ**: `PaymentCreationRequestDto` thay vì `CreatePaymentDto` (Rule 3).
- **Biên DTO**: `InputDto` vào → `OutputDto` ra (với caller); `RequestDto` đi → `ResponseDto` về (với đối tác).
- **Không expose internal fields** (signature, config, internal requestId) trong `InputDto` từ client (Rule 4).

### 2. C# Style, Comments & Logic ([Chi tiết](./references/02-csharp-style.md))
- **Property PascalCase**; dùng `[JsonPropertyName("...")]` để map JSON (Rule 5).
- **Ngăn cách một dòng trống** giữa các property trong DTO, Options, Entity (Rule 6).
- **XML Documentation & Comments song ngữ**: Tiếng Việt trước (có dấu), tiếng Anh sau (bọc trong `<para>` nếu là XML doc) (Rule 7, 7.1).
- **Comment giải thích "VÌ SAO"**, không mô tả lại "LÀM GÌ"; dead code phải xoá sạch, không comment lại (Rule 7.1).
- **`TODO(<JIRA>):`**: Mọi TODO/FIXME bắt buộc kèm mã Jira ticket (Rule 7.1).
- **Khối `if`/`else` luôn có `{ }`** kể cả 1 dòng (Rule 8).
- **Invert `if` (Guard Clauses & Early Return)**: Luôn loại bỏ edge-case trước để luồng chính (Happy path) ở cấp ngoài cùng (Rule 8.1).
- **Quy tắc `var`**: Dùng `var` khi kiểu rõ ở vế phải; khai báo kiểu tường minh khi gọi method hoặc kiểu primitive (Rule 9).
- **File-scoped namespace** (`namespace App;`) cho .NET 6+ (Rule 10). Toàn bộ `using` đặt ở đầu file ngoài namespace (Rule 10.1).
- **Từ khóa nguyên thủy & Định danh chuẩn .NET**: Dùng `string`, `int`, `bool` thay cho CLR types; ưu tiên `int` có dấu; static field có tiền tố `s_`, thread static `t_`; hằng số (`const`) PascalCase (cấm `UPPER_SNAKE_CASE`); enum số ít (flags số nhiều); generic prefix `T` (Rule 10.3).
- **Sealed by default**: Mọi concrete class (Service, Strategy, Resolver, Repository, Validator...) không chủ đích kế thừa bắt buộc đánh dấu `sealed` (hoặc `internal sealed`) để bảo vệ đóng gói và kích hoạt tối ưu JIT Devirtualization; dùng `sealed override` khi ngắt chuỗi override (Rule 10.4).

### 3. JSON & Async ([Chi tiết](./references/03-json-async.md))
- Dùng **`System.Text.Json`** thay cho `Newtonsoft.Json` (Rule 11).
- Property nullable trong DTO có `[JsonIgnore(Condition = WhenWritingNull)]` (Rule 12).
- Mọi hàm trả `Task`/`ValueTask` kết thúc bằng **`Async`** (Rule 13).
- Luôn nhận **`CancellationToken cancellationToken = default`** ở cuối và forward xuống tầng dưới (Rule 14).

### 4. API & Controller ([Chi tiết](./references/04-api-controller.md))
- Route RESTful: danh từ số nhiều, kebab-case, không động từ trong URL (Rule 15).
- Mọi action controller có **`[SwaggerOperation]`** và **`[SwaggerResponse]`**, không lộ URL đối tác/secret trong Swagger (Rule 16).

### 5. Kiến Trúc & Đối Tác ([Chi tiết](./references/05-architecture-business.md))
- Validation input bắt buộc dùng **`FluentValidation`** (`AbstractValidator<T>`), không validate trong Service (Rule 17).
- HTTP API bắt buộc dùng **`IHttpClientFactory`**, không dùng RestSharp hay `new HttpClient()` thủ công (Rule 18).
- Tích hợp đối tác: Orchestrator + Functional Pipeline + Dynamic Proxy Interceptor (Auth/Headers) + GoF Decorator (Cache, Input shaping) (Rule 19).

### 6. Workflow & C# Hiện Đại ([Chi tiết](./references/06-workflow-git-build.md))
- Commit chuẩn Commitizen: `feat(PH-1289): ...`, scope bắt buộc là mã Jira ticket (Rule 20).
- Bắt buộc chạy **`dotnet build` đạt 0 lỗi** trước khi kết thúc task (Rule 21).
- Cú pháp C# theo phiên bản: Collection Expressions `[...]`, Primary Constructors, `System.Threading.Lock`, `required` (Rule 22).
- Thư viện .NET 8+ bật `Directory.Build.props` tập trung gom `artifacts/` (Rule 23).

### 7. GoF Design Patterns & SOLID ([Chi tiết](./references/07-design-patterns.md))
- Ngưỡng phức tạp: ≥ 4 nhánh phân loại cùng enum, chuỗi lặp ở ≥ 2 method, method > 80 dòng, lồng > 3 cấp → **bắt buộc tái cấu trúc bằng GoF Pattern** (Strategy, Factory, Decorator, Adapter...) (Rule 24).
- **ISP**: Tách interface theo nghiệp vụ, không tạo interface béo buộc viết stub ném `NotSupportedException` (Rule 24.5a).
- **LSP**: Implementation phải tuân thủ trọn vẹn hợp đồng (Rule 24.5b).

### 8. UI, Data Table & Biểu Tượng ([Chi tiết](./references/08-ui-accessibility.md))
- Bảng dữ liệu phân trang & lọc ở SQL qua EF Core `AsNoTracking()`, `Skip()`, `Take()`, `CountAsync()`; nạp master data song song qua `Task.WhenAll`; toàn bộ tiêu chí lọc trang Catalog/Sản phẩm đẩy xuống SQL (Rule 25, 25.5).
- Ảo hóa kết xuất DOM (`<Virtualize>`) cho danh sách lớn, skeleton shimmer placeholder, debounce tìm kiếm ≥ 300ms (Rule 25).
- Tiếp cận **WAI-ARIA**: `caption`/`aria-label`, `th[scope="col"]`, `aria-busy`, `aria-live="polite"`, modal có `role="dialog"` và phím `Escape`, icon button có `aria-label` (Rule 26).
- **Clean Data Representation**: Dữ liệu ô bảng là text thuần túy; cấm emoji/icon trang trí; text/mã định danh không bọc `.badge` (badge chỉ dùng cho status) (Rule 27).
- **Icon Style & SVG Usage**: Vector outline icon (Lucide / Heroicons outline), `stroke="currentColor"`, `fill="none"`, `stroke-width="2"` (hoặc `2.5` cho nút bấm nhỏ), bo góc tròn `round`; không dùng Icon Font / PNG bitmap; icon trang trí có `aria-hidden="true"`, icon-only button có `aria-label` (Rule 28).

---

## ✅ Checklist Trước Khi Kết Thúc Task

- [ ] **DTO**: Mỗi DTO nằm trong file `.cs` riêng, tên file = tên class (Rule 1)
- [ ] **DTO**: Mọi DTO có hậu tố `Dto` (Rule 2)
- [ ] **DTO**: Tên class DTO là danh từ/cụm danh từ, không bắt đầu bằng động từ (Rule 3)
- [ ] **DTO**: DTO nhận từ caller (InputDto) không chứa field internal như requestId, signature, config...; map sang RequestDto mới inject (Rule 4)
- [ ] **Style**: Property trong DTO/class dùng PascalCase, mapping JSON bằng `[JsonPropertyName]` (Rule 5)
- [ ] **Style**: Property trong DTO/Options/Entity cách nhau 1 dòng trống (Rule 6)
- [ ] **Style**: Options/DTO từ tài liệu đối tác có XML comment song ngữ (tiếng Việt có dấu trước, tiếng Anh trong `<para>` sau) mô tả ý nghĩa field (Rule 7)
- [ ] **Comment**: Comment giải thích "vì sao", không mô tả lại code; có comment ở magic number, workaround đối tác, rule/guard bị tắt có chủ đích, regex phức tạp và `catch` cố ý swallow (Rule 7.1 §1, §3)
- [ ] **Comment**: XML doc đủ `<summary>`/`<param>`/`<returns>`/`<exception>` cho public member; implementation/override/decorator dùng `<inheritdoc />`, không copy lại summary của interface (Rule 7.1 §2)
- [ ] **Comment**: **Mọi** comment đều song ngữ — tiếng Việt trước, tiếng Anh sau; `//` tách 2 dòng, XML doc để dòng tiếng Anh trong `<para>`; mã ticket ghi ở **cả hai** dòng, riêng marker `TODO`/`FIXME` chỉ ghi một lần ở dòng tiếng Việt (Rule 7.1 §4)
- [ ] **Comment**: Không để dead code bị comment, không changelog/tên người/ngày sửa, không banner hay `#region` che method dài, không trộn 2 ngôn ngữ trong cùng 1 dòng (Rule 7.1 §5)
- [ ] **Comment**: Không đưa secret/key/token/signature thật hay PII khách hàng vào comment (Rule 7.1 §5)
- [ ] **Comment**: Mọi `TODO`/`FIXME` có mã Jira (`// TODO(PH-1289): ...`) và được liệt kê trong phần tổng kết task (Rule 7.1 §6)
- [ ] **Style**: Mọi block `if`/`else if`/`else` đều có `{ }` kể cả 1 dòng (Rule 8)
- [ ] **Style**: Ưu tiên Invert `if` (Guard Clauses / Early Return) để làm phẳng luồng xử lý, tránh lồng if sâu (Rule 8.1)
- [ ] **Style**: Dùng `var` khi kiểu rõ ở vế phải; dùng kiểu tường minh khi gọi method hoặc kiểu primitive (Rule 9)
- [ ] **Style**: Dùng file-scoped namespace cho .NET 6+ (C# 10+), block-scoped namespace `{ }` cho .NET 5 trở xuống (Rule 10)
- [ ] **Style**: Luôn đặt chỉ thị `using` ở đầu file, bên ngoài khai báo namespace (Rule 10.1)
- [ ] **Style**: Tuân thủ thụt lề 4 spaces, ngoặc nhọn Allman, 1 lệnh/dòng, ngắt dòng trước toán tử nhị phân và XML comment cho public member (Rule 10.2)
- [ ] **Style**: Dùng từ khóa C# nguyên thủy (`string`, `int`, `bool`), ưu tiên `int` có dấu, static field dùng `s_`, thread static `t_`, hằng số PascalCase, enum số ít (flags số nhiều), generic tiền tố `T` (Rule 10.3)
- [ ] **Architecture/Style**: Mọi concrete class (Service, Strategy, Resolver, Repository, Validator...) không dùng làm base class phải đánh dấu `sealed` (hoặc `internal sealed`) (Rule 10.4)
- [ ] **JSON**: Dùng `System.Text.Json`, không thêm `Newtonsoft.Json` cho code mới (Rule 11)
- [ ] **JSON**: Property nullable trong DTO gửi đi/trả về có `[JsonIgnore(Condition = WhenWritingNull)]` hoặc global config (Rule 12)
- [ ] **Async**: Mọi method trả `Task` có hậu tố `Async` (Rule 13)
- [ ] **Async**: Mọi method async mới có `CancellationToken cancellationToken = default` ở cuối và được forward (Rule 14)
- [ ] **Controller**: Route controller theo chuẩn RESTful (danh từ số nhiều, kebab-case, không động từ) (Rule 15)
- [ ] **Controller**: Mọi action có `[SwaggerOperation]` với Summary, Description, OperationId, Tags và `[SwaggerResponse]` (Rule 16)
- [ ] **Architecture**: Validation input/request dùng FluentValidation riêng (AbstractValidator<T>), không validate trong Business Service (Rule 17)
- [ ] **Architecture**: Gọi HTTP API chỉ dùng `IHttpClientFactory`, không dùng `RestSharp` hoặc thư viện bên thứ 3 (Rule 18)
- [ ] **Architecture**: Tích hợp API đối tác mới tuân thủ đúng mô hình Orchestrator + Functional Pipeline + Decorator + Dynamic Proxy Interceptor (Rule 19)
- [ ] **Workflow**: Commit message đúng Commitizen format, scope = mã Jira ticket, subject ≤ 50 ký tự (Rule 20)
- [ ] **Workflow**: Chạy `dotnet build` thành công (0 errors) trước khi báo task hoàn thành (Rule 21)
- [ ] **C# Syntax**: Ưu tiên sử dụng cú pháp mới nhất tương ứng với phiên bản C#/.NET của project (Collection Expressions `[...]`, Primary Constructor, `required`, `Lock`, `params Span<T>`) (Rule 22)
- [ ] **Build Output**: Lib project mới target .NET 8+ đã bật output build tập trung (`ArtifactsPath`/`UseArtifactsOutput` trong `Directory.Build.props` ở root solution) (Rule 23)
- [ ] **Design Pattern**: Code chạm ngưỡng phức tạp ở Rule 24.1 (≥ 4 nhánh phân loại cùng enum, chuỗi lặp ở ≥ 2 method, method > 80 dòng, lồng > 3 cấp, > 5 tham số, ≥ 2 cờ `bool`) đã được tái cấu trúc bằng pattern GoF đúng triệu chứng (Rule 24)
- [ ] **SOLID**: Interface tách theo nghiệp vụ (ISP, Rule 24.5a); Implementation tuân thủ trọn hợp đồng, không `NotSupportedException` hay stub giả dối (LSP, Rule 24.5b)
- [ ] **Data Table**: Tải dữ liệu bảng bằng Server-Side Pagination/Filter qua SQL (`Skip`, `Take`, `CountAsync`), không tải toàn bộ tập dữ liệu về RAM; nạp master data song song qua `Task.WhenAll`; áp dụng Virtualization (`<Virtualize>`); có Shimmer Skeleton loading và debounce tìm kiếm (Rule 25, 25.5)
- [ ] **Accessibility (WAI-ARIA)**: Bảng dữ liệu có `caption`/`aria-label`, `th[scope="col"]`, `aria-busy`, `aria-live="polite"`; Modal có `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, nút đóng có `aria-label`, phím `Escape`; Custom dropdown có `aria-expanded`, `role="listbox"`, `role="option"`; Icon button có `aria-label` (Rule 26)
- [ ] **Table Data Formatting**: Dữ liệu hiển thị trong bảng không chèn emoji/icon trang trí (hiển thị text thuần túy); text thông thường, mã hành chính, mã định danh không bọc thẻ `.badge`/`.chip`/`.tag` (Rule 27)
- [ ] **Icon Style & SVG Usage**: Dùng vector outline icon (Lucide / Heroicons outline), `stroke="currentColor"`, `fill="none"`, `stroke-width="2"` (hoặc `2.5` cho nút bấm nhỏ), bo góc tròn `round`; không dùng Icon Font / PNG bitmap / solid icon bừa bãi; icon trang trí có `aria-hidden="true"`, icon-only button có `aria-label` (Rule 28)

