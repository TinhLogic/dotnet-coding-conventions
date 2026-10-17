# Phần XVI. Enforcement Bằng Công Cụ & Kiểm Thử (Tooling Enforcement & Testing)

Convention chỉ có hiệu lực khi **máy kiểm tra được**. Tài liệu này quy định cách mã hoá các rule thành `.editorconfig` và Roslyn analyzer chạy trong build/CI, và chuẩn viết test cho code .NET.

---

## 36. Convention phải được enforce bằng `.editorconfig`, analyzer và CI, không chỉ bằng văn bản

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Vi phạm bị chặn ngay trong IDE và lúc `dotnet build`, không chờ reviewer phát hiện. AI agent cũng bị chặn tự động vì build fail thì không thể báo xong task (Rule 21). Review tập trung vào nghiệp vụ thay vì soi dấu ngoặc. |
| **Rủi ro nếu vi phạm** | Rule chỉ nằm trên giấy sẽ trôi dần theo thời gian, mỗi team một kiểu. Checklist thủ công bị bỏ qua khi gấp deadline. |
| **Rủi ro khi áp dụng** | Bật `TreatWarningsAsErrors` cho solution cũ có thể làm build vỡ hàng trăm lỗi. Áp dụng theo giai đoạn: bật cho project mới trước, project cũ bật dần theo nhóm rule, và **không** được "giải quyết" bằng suppress hàng loạt (Rule 42). |

### 36.1. `Directory.Build.props` ở root solution

```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <!-- CS1591 (thiếu XML doc) để warning thường cho code nội bộ; public API của library thì bật lại thành error -->
    <!-- CS1591 (missing XML doc) stays a plain warning for internal code; re-enable as error for library public APIs -->
    <WarningsNotAsErrors>$(WarningsNotAsErrors);CS1591</WarningsNotAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <!-- Pin đúng version lấy từ Nexus nội bộ, không dùng version range -->
    <!-- Pin the exact version from the internal Nexus feed, never a version range -->
    <PackageReference Include="Meziantou.Analyzer" Version="<pinned-version>" PrivateAssets="all" />
    <PackageReference Include="Microsoft.VisualStudio.Threading.Analyzers" Version="<pinned-version>" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

Lưu ý: `<pinned-version>` là chỗ điền version cụ thể đã được duyệt trên Nexus. Không dùng `*` hay range. Nếu project dùng Central Package Management (`Directory.Packages.props`) thì khai version ở đó.

### 36.2. `.editorconfig`: mã hoá rule style thành severity

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4

# Rule 8: luôn có ngoặc nhọn
csharp_prefer_braces = true:error
dotnet_diagnostic.IDE0011.severity = error

# Rule 9: var khi kiểu rõ ràng, tường minh với built-in type
csharp_style_var_when_type_is_apparent = true:warning
csharp_style_var_for_built_in_types = false:warning
csharp_style_var_elsewhere = false:warning

# Rule 10 / 10.1: file-scoped namespace, using ngoài namespace
csharp_style_namespace_declarations = file_scoped:error
csharp_using_directive_placement = outside_namespace:error
dotnet_diagnostic.IDE0161.severity = error
dotnet_diagnostic.IDE0065.severity = error

# Rule 10.3: dùng keyword thay CLR type
dotnet_style_predefined_type_for_locals_parameters_members = true:error
dotnet_style_predefined_type_for_member_access = true:error
dotnet_diagnostic.IDE0049.severity = error

# Rule 10.3: static field tiền tố s_
dotnet_naming_rule.static_fields_s_prefix.symbols = static_fields
dotnet_naming_rule.static_fields_s_prefix.style = s_prefix_style
dotnet_naming_rule.static_fields_s_prefix.severity = error
dotnet_naming_symbols.static_fields.applicable_kinds = field
dotnet_naming_symbols.static_fields.applicable_accessibilities = private, internal, private_protected
dotnet_naming_symbols.static_fields.required_modifiers = static
dotnet_naming_style.s_prefix_style.required_prefix = s_
dotnet_naming_style.s_prefix_style.capitalization = camel_case

# Rule 10.3: hằng số PascalCase
dotnet_naming_rule.constants_pascal.symbols = constants
dotnet_naming_rule.constants_pascal.style = pascal_style
dotnet_naming_rule.constants_pascal.severity = error
dotnet_naming_symbols.constants.applicable_kinds = field, local
dotnet_naming_symbols.constants.required_modifiers = const
dotnet_naming_style.pascal_style.capitalization = pascal_case

# Rule 13: method async hậu tố Async
dotnet_naming_rule.async_methods_suffix.symbols = async_methods
dotnet_naming_rule.async_methods_suffix.style = async_suffix_style
dotnet_naming_rule.async_methods_suffix.severity = error
dotnet_naming_symbols.async_methods.applicable_kinds = method
dotnet_naming_symbols.async_methods.required_modifiers = async
dotnet_naming_style.async_suffix_style.required_suffix = Async
dotnet_naming_style.async_suffix_style.capitalization = pascal_case
dotnet_diagnostic.IDE1006.severity = error

# Thứ tự khai báo naming rule: rule cụ thể (const, static) phải đứng trước rule chung
```

### 36.3. Bảng map Rule → Diagnostic ID

| Rule | Nội dung | Diagnostic |
| --- | --- | --- |
| 8 | Ngoặc nhọn bắt buộc | `IDE0011` |
| 9 | Quy tắc `var` | `IDE0007`, `IDE0008` |
| 10 / 10.1 | File-scoped namespace, vị trí `using` | `IDE0161`, `IDE0065` |
| 10.3 | Keyword thay CLR type, naming | `IDE0049`, `IDE1006` |
| 10.4 | Sealed internal class | `CA1852` |
| 13 | Hậu tố `Async` cho mọi method trả `Task` | `IDE1006` (method `async`), `VSTHRD200` |
| 14 | Forward / đặt cuối `CancellationToken` | `CA2016`, `CA1068` |
| 26 | Giữ stacktrace, không bắt `Exception` chung | `CA2200`, `CA1031` |
| 27 | Nullable | Toàn bộ nhóm `nullable` (`CS86xx`), bật qua `WarningsAsErrors` |
| 28 | Sync-over-async, `async void`, `ValueTask`, `ConfigureAwait` | `VSTHRD002`, `VSTHRD100`, `CA2012`, `CA2007` (chỉ bật cho library) |
| 31 | Message template log, `[LoggerMessage]` | `CA2254`, `CA1848` |
| 32 | SQL injection, random không an toàn, tắt verify cert | `CA2100`, `CA5394`, `CA5359`, `CA5400` |
| 35 | Culture, `StringComparison` | `CA1304`, `CA1305`, `CA1307`, `CA1309`, `CA1310`, `CA1311` |
| — | Dispose đúng | `CA2000`, `CA1816` |

### 36.4. CI bắt buộc

```bash
dotnet restore
dotnet format --verify-no-changes --no-restore   # Fail nếu code chưa format đúng .editorconfig
dotnet build --no-restore -warnaserror           # Fail nếu còn warning
dotnet test --no-build --collect:"XPlat Code Coverage"
```

- Pipeline `.gitlab-ci.yml` chạy các bước trên trước SonarQube / Semgrep / Gitleaks.
- MR không được merge khi job lint/build/test đỏ.
- Rule chưa có analyzer (Rule 1–4, 17, 19, 24, 25...) vẫn kiểm bằng checklist và review (Rule 41).

---

## 37. Code mới phải có test tự động theo chuẩn thống nhất

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Refactor (Rule 24) và nâng cấp .NET an toàn vì test bắt được hồi quy. Test là tài liệu sống mô tả hành vi nghiệp vụ. AI agent có tín hiệu khách quan để biết code đúng hay sai, thay vì chỉ "build xanh". |
| **Rủi ro nếu vi phạm** | Bug nghiệp vụ (sai tiền, sai trạng thái) chỉ lộ ra trên production. Test giả (mock chính đối tượng cần test, assert không có gì) cho cảm giác an toàn sai. Dùng EF InMemory provider để test logic transaction/constraint sẽ pass trong khi DB thật fail. |

### 37.1. Công cụ chuẩn

| Mục đích | Công cụ |
| --- | --- |
| Framework | xUnit |
| Assertion | Shouldly (hoặc FluentAssertions nếu repo đã dùng) |
| Test double | NSubstitute |
| Integration DB / Redis / Kafka | Testcontainers (`Testcontainers.MsSql`, `Testcontainers.PostgreSql`...) |
| Integration HTTP API | `WebApplicationFactory<Program>` (ABP: `AbpIntegratedTest<TModule>`) |
| HTTP đối tác | Stub `HttpMessageHandler` hoặc WireMock.Net |
| Thời gian | `FakeTimeProvider` (Rule 35) |
| Validator | `FluentValidation.TestHelper` |
| Coverage | coverlet |

### 37.2. Quy tắc

1. **Tên test**: `MethodName_Scenario_ExpectedResult`. Tên class test: `{ClassUnderTest}Tests`. Project test: `{Project}.Tests` / `{Project}.IntegrationTests`.
2. **Cấu trúc AAA** (Arrange / Act / Assert), cách nhau một dòng trống. Mỗi test kiểm **một hành vi**. Không có `if` / `for` trong test.
3. **Chỉ mock phụ thuộc bên ngoài** (repository, HTTP client, queue, clock). **Không** mock đối tượng đang test, không mock DTO / value object.
4. **Test độc lập**: không phụ thuộc thứ tự, không chia sẻ state tĩnh, mỗi integration test tự dọn dữ liệu (hoặc dùng transaction rollback / database riêng).
5. **Không dùng EF Core InMemory provider** cho logic phụ thuộc transaction, constraint, concurrency, raw SQL (Rule 25, 30). Dùng Testcontainers với đúng loại DB production.
6. **Không `Thread.Sleep` / `Task.Delay`** để chờ trong test. Dùng `FakeTimeProvider.Advance(...)` hoặc cơ chế đồng bộ rõ ràng.
7. **Bắt buộc có test cho**:
   - Mọi validator FluentValidation (Rule 17): mỗi rule một case pass và một case fail.
   - Mọi Strategy / Handler (Rule 24): mỗi implementation có test riêng.
   - Thứ tự Decorator (Rule 19 §3.3): test cache hit/miss và input đã chuẩn hoá trước khi vào cache.
   - Luồng tiền và trạng thái: happy path, lỗi đối tác, timeout, xung đột concurrency, trùng idempotency key.
   - Bug fix: một test tái hiện bug, fail trước khi sửa.
8. **Coverage** không phải mục tiêu tự thân. Ngưỡng tối thiểu do repo quy định trong CI; ưu tiên cover nhánh nghiệp vụ và error path.

✅ **ĐÚNG** — Unit test:

```csharp
public sealed class PaymentServiceTests
{
    private readonly IPartnerApiClient _partnerClient = Substitute.For<IPartnerApiClient>();
    private readonly FakeTimeProvider _timeProvider = new(new DateTimeOffset(2026, 10, 16, 3, 0, 0, TimeSpan.Zero));

    [Fact]
    public async Task CreatePaymentAsync_PartnerTimesOut_KeepsPaymentPending()
    {
        _partnerClient
            .CreateAsync(Arg.Any<PaymentCreationRequestDto>(), Arg.Any<CancellationToken>())
            .ThrowsAsync(new TimeoutRejectedException());
        PaymentService sut = CreateSut();

        PaymentCreationOutputDto result = await sut.CreatePaymentAsync(PaymentInputs.Valid(), CancellationToken.None);

        result.Status.ShouldBe(PaymentStatus.Pending);
    }
}
```

✅ **ĐÚNG** — Test validator:

```csharp
[Fact]
public void Validate_AmountIsZero_HasErrorForAmount()
{
    var validator = new MoMoAIOPaymentCreationInputDtoValidator();

    TestValidationResult<MoMoAIOPaymentCreationInputDto> result = validator.TestValidate(PaymentInputs.Valid(amount: 0));

    result.ShouldHaveValidationErrorFor(x => x.Amount);
}
```

✅ **ĐÚNG** — Integration test với DB thật:

```csharp
public sealed class WalletRepositoryTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();

    public Task InitializeAsync() => _sqlContainer.StartAsync();

    public Task DisposeAsync() => _sqlContainer.DisposeAsync().AsTask();

    [Fact]
    public async Task DebitAsync_TwoConcurrentDebitsExceedBalance_OnlyOneSucceeds()
    {
        // ... seed ví 100, chạy song song hai lệnh trừ 80 trên hai DbContext riêng

        // ... assert đúng một lệnh thành công và số dư cuối là 20
    }
}
```

### 37.3. Anti-patterns (KHÔNG làm)

- Test không có assert, hoặc chỉ assert `NotNull`.
- Mock chính class đang test; mock `DbContext` bằng tay để test query.
- Test phụ thuộc thứ tự chạy hoặc dữ liệu có sẵn trên môi trường dùng chung.
- EF InMemory cho logic transaction / constraint.
- `Thread.Sleep` trong test; test gọi API đối tác thật.
- Test chạm dữ liệu production.
- Commit code mới không kèm test với lý do "sẽ bổ sung sau" mà không có ticket (Rule 7.1 §6).
