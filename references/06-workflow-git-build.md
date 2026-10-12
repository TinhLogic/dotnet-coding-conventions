# Phần VII. Quy Trình Phát Triển & Build (Workflow & Git)

Tài liệu chi tiết về Commit message theo chuẩn Commitizen (scope = mã Jira ticket), quy trình `dotnet build` 0 lỗi, quy chuẩn C# Version Alignment (.NET 8/9/10+) và cấu hình build artifacts tập trung.

---

## 20. Commit message theo chuẩn Commitizen, scope = mã Jira ticket

Mọi commit phải tuân thủ [Conventional Commits](https://www.conventionalcommits.org/) với format Commitizen. **Scope bắt buộc là mã Jira ticket** (viết hoa, ví dụ `PH-1289`).

Format:

```
<type>(<jira-ticket>): <mô tả ngắn gọn>

[optional body]

[optional footer]
```

Các `type` hợp lệ:

| type       | Dùng khi                                         |
| ---------- | ------------------------------------------------ |
| `feat`     | Thêm tính năng mới                               |
| `fix`      | Sửa bug                                          |
| `refactor` | Refactor code không thay đổi hành vi             |
| `docs`     | Chỉ sửa documentation                            |
| `style`    | Sửa format, whitespace, không thay đổi logic     |
| `test`     | Thêm hoặc sửa test                               |
| `chore`    | Công việc maintenance (dependency, CI config...) |
| `perf`     | Cải thiện performance                            |

✅ **ĐÚNG**:

```
feat(PH-1289): integrate MoMo AIO on EDC device

- Add MoMoAIOService with create payment flow
- Add appsettings MoMoAIO section
- Add controller endpoint POST /momo-aio/payments
```

```
fix(FI-12345): handle null serialNumber
```

```
refactor(PH-1289): split MoMoAIO DTOs
```

Chi tiết bị cắt khỏi subject để giữ 50 ký tự thì đẩy xuống body, đừng rút gọn đến mức mất nghĩa:

```
fix(FI-12345): handle null serialNumber

MoMo AIO trả serialNumber = null khi thiết bị chưa đăng ký, gây NullReferenceException lúc build request.
```

❌ **SAI**:

```
update momo code              ← không có type, scope, mô tả quá chung
feat: add momo aio            ← thiếu scope (mã Jira)
feat(momo): add aio payment   ← scope phải là mã Jira, không phải tên feature
FEAT(PH-1289): Add payment    ← type phải lowercase
```

**Quy tắc bắt buộc**:
- Nếu **không có mã Jira ticket**, AI phải **hỏi user** trước khi commit. Không được tự đặt scope bừa hoặc bỏ trống scope.
- Một commit có thể liên quan nhiều ticket → dùng ticket chính làm scope, liệt kê ticket phụ trong footer: `Refs: PH-1290, PH-1291`
- Dòng đầu (subject) tối đa **50 ký tự** tính cả `type(scope): `, viết thường chữ cái đầu mô tả, không chấm câu cuối.
- Body wrap ở **100 ký tự** mỗi dòng.

Giới hạn 50 ký tự là ngưỡng `git log --oneline`, `git shortlog` và cột commit list của GitLab/GitHub hiển thị trọn vẹn không bị cắt. Commitizen cũng cảnh báo ở mốc này (*Your string is 50 characters long*). Subject dài hơn thì phần cuối — thường là thông tin phân biệt commit — bị ẩn đúng ở chỗ người review cần nhìn nhất.

Vượt 50 ký tự thì **rút ngắn subject và đẩy chi tiết xuống body**, không cắt bớt scope hay bỏ type.

---

## 21. Bắt buộc chạy `dotnet build` và fix tất cả lỗi trước khi kết thúc task

Sau khi hoàn thành code, AI agent **bắt buộc** phải chạy `dotnet build` trên solution hoặc project liên quan và **fix tất cả build errors** trước khi báo task hoàn thành.

### Quy trình

1. Chạy `dotnet build` trên solution/project vừa sửa.
2. Nếu có **error** (CS####): phải fix ngay, không được bỏ qua.
3. Nếu có **warning** mới do code vừa tạo: ưu tiên fix, trừ khi warning đến từ code cũ không liên quan.
4. Sau khi fix, chạy lại `dotnet build` để xác nhận **0 errors**.
5. Lặp lại cho đến khi build thành công.

### Quy tắc

- **Không** được báo task hoàn thành nếu build còn error.
- **Không** được suppress warning bằng `#pragma warning disable` trừ khi có lý do chính đáng và được ghi chú rõ ràng (ví dụ: false positive từ analyzer).
- Nếu error đến từ dependency chưa có (missing NuGet package), phải thông báo user và hướng dẫn cài đặt, không im lặng bỏ qua.
- Nếu error không thể fix được do hạn chế context (ví dụ: thiếu file từ project khác), phải **liệt kê rõ** danh sách errors còn lại và lý do chưa fix trong phần tổng kết.

### Ví dụ workflow

```bash
# Bước 1: Build solution
dotnet build src/YourApp.sln

# Bước 2: Nếu có lỗi, đọc output và fix
# Error CS0246: The type or namespace name 'MoMoAIOPaymentCreationInputDto' could not be found
# → Thiếu using hoặc file chưa được thêm vào project

# Bước 3: Fix xong, build lại
dotnet build src/YourApp.sln
# Build succeeded. 0 Error(s).
```

### Xử lý các lỗi thường gặp

| Error                                 | Nguyên nhân thường gặp                   | Cách fix                             |
| ------------------------------------- | ---------------------------------------- | ------------------------------------ |
| CS0246 (type not found)               | Thiếu `using`, file chưa add vào project | Thêm `using` hoặc kiểm tra namespace |
| CS0103 (name does not exist)          | Biến chưa khai báo hoặc typo             | Khai báo biến hoặc sửa tên           |
| CS0161 (not all paths return)         | Thiếu `return` trong branch              | Thêm return statement                |
| CS1061 (does not contain definition)  | Gọi method/property không tồn tại        | Kiểm tra tên, thêm method nếu thiếu  |
| CS0535 (does not implement interface) | Thiếu implement method từ interface      | Implement đầy đủ interface members   |
| CS8602 (dereference of nullable)      | Nullable reference warning               | Thêm null check hoặc `!` operator    |
| CS0029 (cannot convert type)          | Sai kiểu dữ liệu                         | Cast hoặc sửa kiểu cho đúng          |

### Lưu ý quan trọng

- Nếu solution có nhiều project, chỉ cần build project/solution mà code vừa thay đổi thuộc về — không cần build toàn bộ monorepo.
- Nếu build mất quá lâu (>2 phút), có thể build từng project riêng lẻ: `dotnet build src/ProjectName/ProjectName.csproj`
- **Không** chạy `dotnet run` hoặc `dotnet test` — chỉ cần `dotnet build` để verify compile thành công. Test và run là bước riêng.

---

## 22. Ưu tiên cú pháp hiện đại theo phiên bản C# (.NET Version Alignment)

AI Agent **bắt buộc** phải nhận biết phiên bản C#/.NET của project (`<TargetFramework>` hoặc `<LangVersion>` trong `.csproj`) và **ưu tiên sử dụng cú pháp hiện đại tương ứng với phiên bản đó**, tuyệt đối tránh dùng các cú pháp cũ/lỗi thời khi project đã ở phiên bản .NET mới hơn.

### Chi tiết các tính năng mới theo từng phiên bản C#

#### 1. C# 12 (.NET 8)
* **Primary Constructors cho mọi Class & Struct**: Không còn giới hạn ở `record`. Cho phép khai báo tham số constructor trực tiếp trên khai báo `class`/`struct`. Các constructor phụ khác phải gọi `: this(...)`.
* **Collection Expressions `[...]`**: Cú pháp thống nhất và ngắn gọn để tạo mảng (`T[]`), `List<T>`, `Span<T>`, `ReadOnlySpan<T>` và các kiểu collection có sẵn mà không cần `new List<T> { ... }`.
* **Spread Operator `..` trong Collection Expressions**: Ghép, gộp các collection trực tiếp vào collection expression (ví dụ: `int[] combined = [..a, ..b, 100];`).
* **Inline Arrays**: Cung cấp struct kích thước cố định, liên tục trong bộ nhớ thông qua attribute `[System.Runtime.CompilerServices.InlineArray(N)]` phục vụ tối ưu hiệu năng cao.
* **Optional / Default Parameters trong Lambda Expressions**: Cho phép lambda có tham số mặc định và `params` (ví dụ: `var add = (int x, int y = 1) => x + y;`).
* **`ref readonly` Parameters**: Cho phép truyền tham chiếu chỉ đọc một cách rõ ràng (tương tự `in` nhưng không ép buộc caller phải ngầm định tạo biến tạm, bảo vệ caller khỏi silent copies).
* **Alias Any Type (`using` alias mở rộng)**: Cho phép dùng chỉ thị `using Alias = ...` cho mọi kiểu dữ liệu (Tuple types, Pointer types, Array types, Generic types) thay vì chỉ cho named types.
* **`[Experimental]` Attribute**: Đánh dấu các tính năng, API đang trong giai đoạn thử nghiệm để compiler phát cảnh báo/lỗi nếu sử dụng mà không bật cờ tương ứng.
* **Interceptors (Preview)**: Cho phép compiler / source generator chuyển hướng lệnh gọi phương thức cụ thể sang đoạn mã khác khi biên dịch.

#### 2. C# 13 (.NET 9)
* **`params Collections`**: Từ khóa `params` không còn giới hạn ở `params T[]` (mảng) mà hỗ trợ trực tiếp các kiểu tập hợp nhận dạng được: `ReadOnlySpan<T>`, `Span<T>`, `IEnumerable<T>`, `IReadOnlyList<T>`, `List<T>`, v.v. (giúp loại bỏ cấp phát mảng trên Heap khi truyền span).
* **Kiểu `System.Threading.Lock` mới**: Runtime cung cấp kiểu lock chuyên biệt hiệu năng cao `System.Threading.Lock`. Câu lệnh `lock (_lock) { ... }` tự động nhận diện và sinh mã thông qua phương thức `EnterScope()` trả về ref struct thay vì sử dụng `Monitor`.
* **Ký tự thoát Escape `\e`**: Thêm ký tự escape sequence `\e` cho ký tự Escape (`U+001B`), thay thế cách viết dễ gây nhầm lẫn `\x1b` hoặc dài dòng `\u001b`.
* **Method Group Natural Type Improvements**: Cải tiến thuật toán suy luận kiểu tự nhiên (natural type) cho method groups khi truyền vào delegate/overload resolution.
* **Implicit Index Access trong Object Initializers**: Cho phép toán tử index đảo ngược `^` hoạt động bên trong object initializer của collection (ví dụ: `buffer = { [^1] = 0, [^2] = 1 }`).
* **Hỗ trợ `ref` Locals & `unsafe` Contexts trong Iterators (`yield`) và Async Methods**: Loại bỏ các hạn chế cũ, cho phép khai báo biến `ref` hoặc khối `unsafe` trong method async hoặc method có `yield return` (khi không vượt qua ranh giới `await`/`yield`).
* **`ref struct` Types Implement Interfaces**: Cho phép `ref struct` implement các interface (ví dụ `IDisposable`, `IEquatable<T>`), mở rộng khả năng trừu tượng hóa hiệu năng cao.
* **`allows ref struct` Anti-constraint trong Generics**: Cho phép generic type parameters chấp nhận `ref struct` thông qua cú pháp `where T : allows ref struct`.
* **Partial Properties & Partial Indexers**: Cho phép khai báo `partial` trên properties và indexers (rất hữu ích cho Source Generators).
* **`[OverloadResolutionPriority]` Attribute**: Cho phép thư viện chỉ định mức độ ưu tiên của overload khi có nhiều method phù hợp.

#### 3. C# 14 & C# 15+ (.NET 10+)
* **Extension Members / Explicit Extensions (`extension` block)**: Cú pháp mở rộng toàn diện mới, cho phép khai báo:
  - Extension Properties (Instance & Static)
  - Static Extension Methods
  - Extension Operators (nạp chồng toán tử cho kiểu hiện có)
  - Extension Indexers
* **`field` Keyword trong Backing Properties**: Truy cập trực tiếp backing field ngầm định trong accessor `get`/`set` của auto-property mà không cần phải tự khai báo `private string _field;` thủ công (ví dụ: `public string Name { get; set => field = value ?? "default"; }`).
* **Null-conditional Assignment (`?.` và `?[]` ở vế trái phép gán)**: Gán giá trị chỉ khi đối tượng khác null (ví dụ: `customer?.Address = newAddress;`).
* **`nameof` Hỗ trợ Unbound Generic Types**: Cho phép gọi `nameof(List<>)` thay vì phải chỉ định kiểu cụ thể `nameof(List<int>)`.
* **Cải tiến chuyển đổi ngầm định cho `Span<T>` và `ReadOnlySpan<T>`**: Linh hoạt hơn khi chuyển đổi giữa mảng và span.
* **Modifiers trên Simple Lambda Parameters**: Cho phép thêm modifier (`ref`, `in`, `out`) trên tham số lambda không cần khai báo kiểu (ví dụ: `(ref x) => x++`).
* **Partial Events & Partial Constructors**: Bổ sung `partial` cho event và constructor.
* **User-defined Compound Assignment Operators**: Cho phép định nghĩa các toán tử gán phức hợp (`+=`, `-=`, ...).

### Bảng đối chiếu: Cú pháp NÊN DÙNG vs. Cần TRÁNH theo phiên bản

| Phiên bản .NET       | Cú pháp CŨ (Cần tránh) ❌                                                       | Cú pháp MỚI (Bắt buộc / Ưu tiên) ✅                                                       |
| :------------------- | :----------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------- |
| **.NET 8+ (C# 12+)** | `new string[] { "A", "B" }`<br>`new List<int> { 1, 2, 3 }`                     | `string[] arr = ["A", "B"];`<br>`List<int> list = [1, 2, 3];` *(Collection Expressions)* |
| **.NET 8+ (C# 12+)** | `listA.Concat(listB).ToArray()` *(khi gộp mảng/list)*                          | `[..listA, ..listB]` *(Spread Operator)*                                                 |
| **.NET 8+ (C# 12+)** | Khai báo `private readonly ILogger _logger;` và gán trong constructor dài dòng | `public class MyService(ILogger<MyService> logger)` *(Primary Constructor)*              |
| **.NET 9+ (C# 13+)** | `private readonly object _lock = new();`<br>`lock (_lock) { ... }`             | `private readonly System.Threading.Lock _lock = new();`<br>`lock (_lock) { ... }`        |
| **.NET 9+ (C# 13+)** | `public void Log(params string[] msgs)` *(gây heap allocation mảng)*           | `public void Log(params ReadOnlySpan<string> msgs)` *(Zero-allocation)*                  |
| **.NET 7+ (C# 11+)** | Constructor 10 tham số chỉ để gán các DTO properties                           | `public required string Prop { get; init; }`                                             |
| **.NET 7+ (C# 11+)** | Chuỗi JSON nối chuỗi `\"key\": \"value\"`                                      | Raw String Literal `"""{"key": "value"}"""`                                              |
| **.NET 6+ (C# 10+)** | `namespace Company.App { class Foo { } }` *(lồng ngoặc)*                       | `namespace Company.App;` *(File-scoped namespace)*                                       |

---

## 23. Library project .NET 8+ bắt buộc bật output build tập trung (`bin/` và `obj/` gom về một chỗ)

Khi **tạo mới** một library project (Class Library / project đóng gói NuGet) target **`net8.0` trở lên**, **bắt buộc** phải bật cấu hình **Artifacts output layout** để toàn bộ output build (`bin`, `obj`, `publish`, `package`) của mọi project trong solution được gom tập trung về **một thư mục gốc duy nhất**, thay vì rải rác `bin/` + `obj/` trong từng project folder.

### Lợi ích:
- **Dễ dọn dẹp**: xoá 1 thư mục `artifacts/` là sạch toàn bộ build cache, không phải đi tìm và xoá hàng chục cặp `bin`/`obj`.
- **Dễ cấu hình CI/CD & `.gitignore`**: chỉ cần ignore/collect 1 path duy nhất, không cần pattern `**/bin/**`, `**/obj/**`.
- **Tooling dự đoán được vị trí output**: Jenkins/GitLab CI, script `build-nuget.sh`, docker `COPY` đều trỏ vào path cố định.
- **Tránh lẫn output giữa các TFM**: layout tự tách theo project + configuration + target framework.

### Cấu hình bắt buộc

Tạo (hoặc cập nhật) file **`Directory.Build.props`** tại **thư mục gốc của solution** — ngang cấp file `.sln`:

```xml
<Project>

  <PropertyGroup>
    <!-- Gom toàn bộ output build (bin/obj/publish/package) của mọi project về thư mục artifacts tập trung -->
    <ArtifactsPath>$(MSBuildThisFileDirectory)artifacts</ArtifactsPath>
  </PropertyGroup>

</Project>
```

Hoặc dùng thư mục mặc định (`artifacts` nằm cạnh `Directory.Build.props`):

```xml
<Project>

  <PropertyGroup>
    <UseArtifactsOutput>true</UseArtifactsOutput>
  </PropertyGroup>

</Project>
```

Có thể sinh file tự động bằng CLI:

```bash
dotnet new buildprops --use-artifacts
```

### Cấu trúc thư mục kết quả

```
📁 artifacts
    └──📂 <Loại output>      (bin | obj | publish | package)
        └──📂 <Tên project>
            └──📂 <Pivot>    (configuration + TFM + RID)
```

| Path                                          | Ý nghĩa                                         |
| --------------------------------------------- | ----------------------------------------------- |
| `artifacts/bin/ICT.FRTTMO.X.Domain/debug`     | Output build (thay cho `bin/Debug/`)            |
| `artifacts/obj/ICT.FRTTMO.X.Domain/debug`     | File trung gian (thay cho `obj/Debug/`)         |
| `artifacts/bin/ICT.FRTTMO.X.Domain/debug_net8.0` | Output của project multi-target                 |
| `artifacts/package/release`                   | Nơi sinh file `.nupkg` (khi `GeneratePackageOnBuild`) |
| `artifacts/publish/ICT.FRTTMO.X.Host/release_linux-x64` | Output `dotnet publish`                         |

### ⚠️ Quy tắc bắt buộc (dễ làm sai)

1. **Property `ArtifactsPath` / `UseArtifactsOutput` PHẢI đặt trong `Directory.Build.props`**, **KHÔNG** đặt trong `.csproj` hay trong file props được import thủ công. Lý do: đường dẫn output được MSBuild tính toán rất sớm (trong `Microsoft.Common.props`), trước khi thân project được evaluate — đặt sai vị trí thì property **không có tác dụng**, build vẫn ra `bin/obj` cũ.
2. **KHÔNG** tắt/override artifacts layout ở từng project con (set `UseArtifactsOutput=false` trong một `.csproj` nằm dưới `Directory.Build.props` đã bật) — cấu hình lệch cấp gây warning `MSB5029` và làm mọi lệnh `dotnet` thất bại.
3. **Xoá các override output path thủ công cũ** trong `.csproj` khi đã bật artifacts: `<OutputPath>`, `<BaseOutputPath>`, `<BaseIntermediateOutputPath>`, `<PackageOutputPath>`.
4. **Cập nhật `.gitignore`**: thêm `artifacts/`.
5. **Cập nhật script build / CI / Dockerfile** nếu đang hardcode đường dẫn cũ.
6. **Áp dụng cho project mới**. Với solution cũ, **không tự ý** bật artifacts layout vì sẽ phá pipeline CI/CD hiện tại — phải **hỏi user** trước.
7. **Điều kiện áp dụng**: tính năng do .NET SDK 8+ cung cấp, không phụ thuộc TFM.

