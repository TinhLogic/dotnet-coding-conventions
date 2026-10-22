# Phần XVIII. Quản Lý Bộ Nhớ & Tối Ưu Zero-Allocation (High Performance & Zero-Allocation Hot Path)

Tài liệu chi tiết về tối ưu bộ nhớ đệm (Memory Pooling: `ArrayPool`, `RecyclableMemoryStream`), thao tác dữ liệu zero-allocation bằng `Span<T>` / `ReadOnlySpan<T>`, `string.Create`, triệt tiêu Boxing và Closure allocations, và đo lường bằng `BenchmarkDotNet`.

---

## 43. Tối ưu bộ nhớ đệm và I/O lớn bằng Memory Pooling (`ArrayPool<T>`, `RecyclableMemoryStream`)

Trong các hệ thống xử lý khối lượng dữ liệu lớn hoặc thông lượng cao (High-throughput APIs, Payment Gateways, Data Streaming, File Upload/Download, JSON payloads kích thước lớn), việc liên tục khởi tạo các mảng byte (`byte[]`) hoặc `MemoryStream` tạm thời sẽ tạo ra áp lực khổng lồ lên Garbage Collector (GC), đặc biệt là phân mảnh trên **Large Object Heap (LOH)** khi kích thước vượt quá 85,000 bytes.

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Tái sử dụng mảng và buffer có sẵn từ bộ nhớ đệm; triệt tiêu việc cấp phát bộ nhớ mới trên Heap cho mỗi request; loại bỏ hiện tượng phân mảnh LOH; ngăn ngừa các đợt "Stop-the-world" (Gen 2 / Full GC pauses); giữ cho p99 latency của hệ thống ổn định và mượt mà dưới tải cao. |
| **Rủi ro nếu vi phạm** | `new byte[bufferSize]` hoặc `new MemoryStream()` liên tục khiến lượng rác sinh ra vượt tầm kiểm soát; LOH nhanh chóng cạn kiệt bộ nhớ ảo liên tục dẫn tới `OutOfMemoryException` (OOM) dù máy chủ vẫn còn RAM trống; GC Gen 2 chạy liên tục chiếm dụng 100% CPU. |
| **Rủi ro khi áp dụng sai** | Quên không trả buffer về pool (`ArrayPool.Return`) gây rò rỉ bộ nhớ (leak); tái sử dụng lại buffer sau khi đã trả về pool gây race condition và sai lệch dữ liệu trầm trọng; quên xóa dữ liệu nhạy cảm (`clearArray: true`) làm rò rỉ token hoặc PII sang request khác. |

### 43.1. Quy tắc sử dụng `ArrayPool<T>`

1. **Mượn từ pool dùng chung**: Khi cần mảng đệm tạm thời (đặc biệt là kích thước ≥ 1 KB, hoặc bất kỳ kích thước nào trong vòng lặp triệu lần), **bắt buộc** mượn từ `ArrayPool<T>.Shared.Rent(minimumLength)`.
2. **Kích thước thực tế vs Kích thước trả về**: Phương thức `Rent(minimumLength)` trả về mảng có chiều dài **lớn hơn hoặc bằng** `minimumLength`. **Tuyệt đối không** giả định `buffer.Length == minimumLength`. Bắt buộc chỉ thao tác trên phạm vi dữ liệu hợp lệ bằng cách cắt lát (`Slice` / `AsSpan`):
   ```csharp
   byte[] buffer = ArrayPool<byte>.Shared.Rent(requiredSize);
   try
   {
       // Chỉ thao tác trên slice đúng kích thước thực tế
       // Only operate on the slice matching the actual required size
       Span<byte> span = buffer.AsSpan(0, requiredSize);
       // ... xử lý trên span ...
   }
   finally
   {
       ArrayPool<byte>.Shared.Return(buffer);
   }
   ```
3. **Bắt buộc dùng `try ... finally`**: Việc trả buffer về pool phải luôn nằm trong khối `finally` để đảm bảo mảng được hoàn trả kể cả khi có ngoại lệ phát sinh.
4. **Bảo mật dữ liệu nhạy cảm (`clearArray`)**:
   - Nếu buffer lưu trữ mật khẩu, mã PIN, Token, khóa mã hóa (secret/key), hoặc PII của khách hàng: **bắt buộc** truyền `clearArray: true` khi gọi `Return(buffer, clearArray: true)` để xóa trắng mảng trong RAM.
   - Với buffer đọc file, ảnh, payload công khai không nhạy cảm: giữ mặc định `clearArray: false` để tối đa hóa hiệu năng.
5. **Cấm giữ lại tham chiếu (No Retained Reference)**: Sau khi đã gọi `Return()`, tuyệt đối không đọc, ghi, hoặc lưu tham chiếu mảng vào bất kỳ biến instance/static nào.

### 43.2. Quy tắc sử dụng `RecyclableMemoryStream`

1. **Cấm `new MemoryStream()` trong Hot-Path**: `System.IO.MemoryStream` mở rộng kích thước mảng theo cấp số nhân khi ghi thêm dữ liệu, khiến các mảng cũ bị bỏ rơi trên Heap và mảng lớn rơi thẳng vào LOH.
2. **Sử dụng `Microsoft.IO.RecyclableMemoryStreamManager`**: Với các tác vụ nén/giải nén gzip, mã hóa crypto, serialize JSON stream, hoặc xuất báo cáo:
   - Đăng ký `RecyclableMemoryStreamManager` là **Singleton** trong DI container:
     ```csharp
     builder.Services.AddSingleton<RecyclableMemoryStreamManager>();
     ```
   - Tạo stream qua manager và bọc trong `using` / `await using`:
     ```csharp
     await using MemoryStream memoryStream = _streamManager.GetStream();
     ```

### 43.3. Ví dụ đối chiếu Code

❌ **SAI** — Cấp phát mảng lớn trên Heap và dùng MemoryStream thông thường:

```csharp
public async Task<byte[]> CompressDataAsync(byte[] sourceData, CancellationToken cancellationToken = default)
{
    // ❌ Cấp phát MemoryStream thông thường: tạo ra chuỗi mảng byte rác trên LOH khi phình to
    using MemoryStream outputStream = new MemoryStream();
    using (GZipStream gzipStream = new GZipStream(outputStream, CompressionMode.Compress, leaveOpen: true))
    {
        // ❌ Cấp phát buffer 64KB trực tiếp trên Heap cho mỗi lời gọi
        byte[] buffer = new byte[64 * 1024];
        int bytesRead;
        using MemoryStream inputStream = new MemoryStream(sourceData);
        while ((bytesRead = await inputStream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)) > 0)
        {
            await gzipStream.WriteAsync(buffer, 0, bytesRead, cancellationToken);
        }
    }

    return outputStream.ToArray(); // ❌ ToArray() nhân bản thêm một mảng byte mới
}
```

✅ **ĐÚNG** — Dùng `ArrayPool<byte>` và `RecyclableMemoryStreamManager`:

```csharp
public sealed class DataCompressionService : IDataCompressionService
{
    private const int BufferSize = 64 * 1024;
    private readonly RecyclableMemoryStreamManager _streamManager;

    public DataCompressionService(RecyclableMemoryStreamManager streamManager)
    {
        _streamManager = streamManager;
    }

    public async Task CompressToStreamAsync(
        ReadOnlyMemory<byte> sourceData,
        Stream destinationStream,
        CancellationToken cancellationToken = default)
    {
        // Mượn stream đệm có khả năng tái sử dụng bộ nhớ từ Pool
        // Rent a reusable memory stream from the pool
        await using MemoryStream tempStream = _streamManager.GetStream("DataCompressionService.Compress");

        await using (GZipStream gzipStream = new GZipStream(tempStream, CompressionLevel.Fastest, leaveOpen: true))
        {
            await gzipStream.WriteAsync(sourceData, cancellationToken);
        }

        tempStream.Position = 0;

        // Mượn buffer 64KB từ ArrayPool thay vì new byte[]
        // Rent a 64KB buffer from ArrayPool instead of new byte[]
        byte[] buffer = ArrayPool<byte>.Shared.Rent(BufferSize);
        try
        {
            int bytesRead;
            while ((bytesRead = await tempStream.ReadAsync(buffer.AsMemory(0, BufferSize), cancellationToken)) > 0)
            {
                await destinationStream.WriteAsync(buffer.AsMemory(0, bytesRead), cancellationToken);
            }
        }
        finally
        {
            // Trả buffer về pool an toàn trong khối finally
            // Safely return the rented buffer to the pool in finally block
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}
```

---

## 44. Thao tác dữ liệu không cấp phát Heap bằng `Span<T>`, `ReadOnlySpan<T>` và `string.Create`

Thao tác xử lý chuỗi và buffer (cắt chuỗi, tách chuỗi, so sánh, parse số/ngày tháng) là nguyên nhân hàng đầu sinh ra rác thế hệ 0 (Gen 0 GC) trong các ứng dụng C#. Sử dụng `Span<T>` và `ReadOnlySpan<T>` cho phép thao tác trực tiếp trên các vùng nhớ liên tục (Stack, Unmanaged memory, hoặc mảng có sẵn) với chi phí **0 byte allocation**.

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Cắt lát chuỗi (slicing) và parse dữ liệu không tạo bất kỳ string trung gian nào; giảm tải 80–99% khối lượng cấp phát bộ nhớ trong các tác vụ Text/Data processing; tăng thông lượng CPU nhờ cache locality tốt. |
| **Rủi ro nếu vi phạm** | `string.Substring()`, `string.Split()`, `string.Replace()` sinh ra hàng triệu object ngắn hạn; GC chạy dồn dập khiến CPU lãng phí thời gian thu gom thay vì phục vụ nghiệp vụ. |
| **Rủi ro khi áp dụng sai** | `Span<T>` là `ref struct`, không thể lưu vào field của class, không thể đóng gói vào closure, không thể await qua ranh giới async; dùng `stackalloc` không kiểm soát kích thước gây crash ứng dụng (`StackOverflowException`). |

### 44.1. Quy tắc phân tích chuỗi (String Parsing & Slicing)

1. **Thay thế `string.Substring()` bằng `AsSpan()` và `.Slice()`**:
   - Khi cần trích xuất một phần chuỗi để kiểm tra hoặc parse: **cấm** dùng `Substring()`. Dùng `input.AsSpan(start, length)`.
2. **Thay thế `string.Split()` bằng `MemoryExtensions.Split()` hoặc vòng lặp `IndexOf`**:
   - `string.Split(',')` tạo ra một mảng `string[]` và từng chuỗi phần tử mới trên Heap.
   - Trong hot path: sử dụng `MemoryExtensions.Split()` (.NET 9+) hoặc duyệt qua chuỗi bằng `ReadOnlySpan<char>` kết hợp `IndexOf(',')` để đọc từng đoạn mà không cấp phát Heap.
3. **Sử dụng `ISpanParsable<T>` thay vì convert qua string**:
   - Parse số, GUID, ngày tháng trực tiếp từ span:
     ```csharp
     ReadOnlySpan<char> span = "12345".AsSpan();
     int value = int.Parse(span, CultureInfo.InvariantCulture); // ✅ Zero-allocation
     ```

### 44.2. Quy tắc định dạng chuỗi (Formatting) trong Hot-Path

1. **Ghép chuỗi nhỏ cố định bằng `stackalloc` và `Span<char>`**:
   - Với các chuỗi ngắn có độ dài biết trước (như mã đối soát, mã đơn hàng, prefix định danh): dùng `stackalloc char[N]` để dựng chuỗi trực tiếp trên Stack.
2. **Dựng chuỗi phức tạp bằng `string.Create`**:
   - Thay vì nối chuỗi qua `+`, `string.Format`, hay `StringBuilder` (vốn vẫn cấp phát bộ nhớ nội bộ), dùng `string.Create` để cấp phát đúng duy nhất 1 lần cho chuỗi kết quả:
     ```csharp
     string orderCode = string.Create(16, (partnerId, sequenceNumber), static (span, state) =>
     {
         // Dựng chuỗi trực tiếp trên span đích của string trước khi bàn giao
         // Populate directly into the destination span of the string before hand-off
         state.partnerId.AsSpan().CopyTo(span[..4]);
         span[4] = '-';
         state.sequenceNumber.TryFormat(span[5..], out _, "D11");
     });
     ```
3. **Ngưỡng an toàn khi dùng `stackalloc` (Stackalloc Threshold Rule)**:
   - **Tuyệt đối cấm** dùng `stackalloc` với kích thước do người dùng nhập từ bên ngoài hoặc kích thước biến thiên không giới hạn (dẫn tới nguy cơ bảo mật và `StackOverflowException` không thể catch).
   - **Ngưỡng tối đa**: Chỉ dùng `stackalloc` khi kích thước tối đa ≤ 256 bytes (hoặc 128 chars). Nếu kích thước vượt quá hoặc chưa xác định, bắt buộc fallback về `ArrayPool<T>.Shared`.
     ```csharp
     Span<char> buffer = length <= 128
         ? stackalloc char[length]
         : (rented = ArrayPool<char>.Shared.Rent(length)).AsSpan(0, length);
     ```

### 44.3. Ví dụ đối chiếu Code

❌ **SAI** — Sinh hàng loạt chuỗi rác khi parse dữ liệu đối soát:

```csharp
public (string PartnerCode, long OrderId, decimal Amount) ParseReconciliationLine(string line)
{
    // ❌ string.Split tạo ra mảng string[] và 3 string con mới trên Heap
    string[] parts = line.Split('|');
    if (parts.Length != 3)
    {
        throw new FormatException("Invalid reconciliation line format");
    }

    // ❌ Substring tạo thêm chuỗi rác, decimal.Parse từ string
    string partner = parts[0].Trim().Substring(0, 4);
    long orderId = long.Parse(parts[1].Trim());
    decimal amount = decimal.Parse(parts[2].Trim());

    return (partner, orderId, amount);
}
```

✅ **ĐÚNG** — Zero-Allocation hoàn toàn với `ReadOnlySpan<char>`:

```csharp
public static (string PartnerCode, long OrderId, decimal Amount) ParseReconciliationLine(ReadOnlySpan<char> line)
{
    // Tìm dấu phân cách đầu tiên bằng Span IndexOf (không tạo string rác)
    // Find first delimiter via Span IndexOf (no garbage strings created)
    int firstPipe = line.IndexOf('|');
    if (firstPipe < 0)
    {
        throw new FormatException("Invalid reconciliation line format: missing first delimiter");
    }

    ReadOnlySpan<char> partnerPart = line[..firstPipe].Trim();
    ReadOnlySpan<char> remaining = line[(firstPipe + 1)..];

    int secondPipe = remaining.IndexOf('|');
    if (secondPipe < 0)
    {
        throw new FormatException("Invalid reconciliation line format: missing second delimiter");
    }

    ReadOnlySpan<char> orderPart = remaining[..secondPipe].Trim();
    ReadOnlySpan<char> amountPart = remaining[(secondPipe + 1)..].Trim();

    // Parse trực tiếp từ ReadOnlySpan<char> không qua string trung gian
    // Parse directly from ReadOnlySpan<char> with zero intermediate strings
    if (!long.TryParse(orderPart, NumberStyles.Integer, CultureInfo.InvariantCulture, out long orderId))
    {
        throw new FormatException("Invalid OrderId format in line");
    }

    if (!decimal.TryParse(amountPart, NumberStyles.Number, CultureInfo.InvariantCulture, out decimal amount))
    {
        throw new FormatException("Invalid Amount format in line");
    }

    // Chỉ cấp phát đúng 1 chuỗi kết quả cần lưu trữ
    // Allocate only the single string needed for long-term storage
    string partnerCode = new string(partnerPart[..Math.Min(4, partnerPart.Length)]);

    return (partnerCode, orderId, amount);
}
```

---

## 45. Triệt tiêu Boxing, Hidden Allocations và Closure trong Hot-Path; đo lường bằng BenchmarkDotNet

Những chi tiết nhỏ trong cú pháp C# có thể âm thầm sinh mã cấp phát bộ nhớ ngầm (Boxing của Value Types, Compiler-generated Closure Class cho lambda, Delegate allocations). Trên các luồng chạy triệu lần/giây hoặc các bộ lọc Middleware, những chi phí ẩn này tích tụ làm suy giảm nghiêm trọng thông lượng hệ thống.

### Vì sao

| | |
| --- | --- |
| **Lợi ích** | Loại bỏ 100% các chi phí ẩn không cần thiết; giảm chu kỳ CPU tiêu tốn cho việc sao chép từ Stack sang Heap; tăng độ ổn định của throughput khi tải tăng vọt. |
| **Rủi ro nếu vi phạm** | Các hàm xử lý ngỡ như rất nhẹ nhưng lại tạo ra hàng triệu instance của DisplayClass closure hoặc Boxed Structs; gây khó hiểu cho đội ngũ khi profiling vì không thấy toán tử `new` nào trong code. |
| **Rủi ro khi áp dụng sai** | Quá ám ảnh việc tối ưu (Premature Optimization) trên các luồng nghiệp vụ thông thường (chỉ chạy vài lần/ngày), khiến code trở nên khó đọc, phức tạp và khó bảo trì. **Chỉ áp dụng quy tắc này cho các Hot-Path được định nghĩa rõ ràng.** |

### 45.1. Định nghĩa "Hot-Path" cần áp dụng Rule này

Không phải mọi dòng code đều cần triệt tiêu allocation tới mức cực đoan. Hot-Path được định nghĩa là:
- Bộ lọc Middleware trong ASP.NET Core (chạy qua mọi request).
- Vòng lặp xử lý Message Consumer từ Kafka / RabbitMQ (xử lý hàng nghìn tin nhắn/giây).
- Logic mã hóa, tính toán Chữ ký (Checksum / Signature) cho cổng thanh toán.
- Vòng lặp parse file dữ liệu lớn (đối soát, import sao kê ngân hàng).
- Các thuật toán tính toán tài chính / pricing trong luồng lõi.

### 45.2. Triệt tiêu Boxing (Value Types, Structs, Enums)

1. **Cấm gọi `.ToString()` trên Enum trong Hot-Path**:
   - `myEnum.ToString()` dùng reflection ngầm và cấp phát một string mới trên Heap.
   - Giải pháp: Sử dụng `nameof(MyEnum.Value)` nếu cố định, hoặc cache mảng chuỗi theo index enum, hoặc sử dụng Source Generator (như thư viện `NetEscapades.EnumGenerators`).
2. **Ép kiểu Struct về Interface gây Boxing**:
   - Khi truyền `struct` vào một tham số interface (ví dụ `IDisposable`, `IComparable`):
     ```csharp
     // ❌ Gây boxing struct lên Heap:
     public void LogItem(IEntity entity) { }

     // ✅ Dùng Generic Constraint để giữ struct trên Stack:
     public void LogItem<T>(T entity) where T : struct, IEntity { }
     ```
3. **Interpolated String & Boxing**:
   - Tránh `string.Format("Code: {0}, Amount: {1}", statusEnum, amountInt)` vì `object` params ép mọi struct/enum vào boxing. Luôn dùng cấu trúc Source Generator `[LoggerMessage]` (Rule 31) hoặc `string.Create`.

### 45.3. Triệt tiêu Closure & Delegate Allocations

1. **Bắt buộc dùng `static lambda` hoặc `static local function` trong Hot-Path**:
   - Thêm từ khóa `static` trước lambda: `static (x) => ...`. Compiler sẽ phát hiện và báo lỗi ngay lập tức (`CS8829`) nếu lambda vô tình capture (chụp) biến từ scope bên ngoài.
   - Khi lambda capture biến ngoài, compiler buộc phải tạo một class ẩn `DisplayClass` trên Heap cho mỗi lần chạy.
2. **Truyền State qua tham số `factoryArgument`**:
   - Tận dụng các API có sẵn tham số `state`:
     ```csharp
     // ❌ SAI: Lambda capture biến "prefix" từ bên ngoài → cấp phát closure class
     string result = _cache.GetOrAdd(key, k => prefix + "_" + k);

     // ✅ ĐÚNG: Dùng static lambda và truyền "prefix" qua state parameter
     string result = _cache.GetOrAdd(key, static (k, state) => state + "_" + k, prefix);
     ```
3. **Cấm dùng LINQ trong Hot-Path**:
   - Các extension method của LINQ (`.Where()`, `.Select()`, `.Any()`, `.First()`) luôn cấp phát các đối tượng iterator, enumerator và delegate instances.
   - Trong hot-path: chuyển toàn bộ LINQ thành vòng lặp `for` hoặc `foreach` truyền thống trên mảng hoặc `Span`.

### 45.4. Tiêu chuẩn Định lượng Bắt buộc bằng BenchmarkDotNet

Mọi tuyên bố "tối ưu hiệu năng" hoặc "zero-allocation" trong Merge Request phải được **chứng minh bằng số liệu thực tế qua BenchmarkDotNet**:

1. Class benchmark phải có attribute `[MemoryDiagnoser]`.
2. Báo cáo kết quả phải chứng minh rõ:
   - Cột **`Allocated`**: giảm về **`0 B`** (hoặc giảm tối thiểu > 90%).
   - Cột **`Gen0` / `Gen1` / `Gen2`**: đạt **`0.0000`** ở tải cao.

```csharp
[MemoryDiagnoser]
public class SignatureBenchmark
{
    private readonly byte[] _data = new byte[1024];

    [Benchmark(Baseline = true)]
    public string LegacySignature()
    {
        // Code cũ dùng string concat & LINQ: Allocated = 2.4 KB, Gen0 = 0.58
        return LegacySigner.ComputeHash(_data);
    }

    [Benchmark]
    public string ZeroAllocSignature()
    {
        // Code mới dùng Span & ArrayPool: Allocated = 0 B, Gen0 = 0.0000
        return OptimizedSigner.ComputeHash(_data);
    }
}
```

### 45.5. Anti-patterns tổng hợp (KHÔNG làm)

- Dùng `stackalloc` với kích thước do caller truyền vào mà không chặn trần an toàn (nguy cơ crash do StackOverflow).
- Gọi `string.Split()` hoặc chuỗi lồng `Substring()` trong các vòng lặp xử lý triệu bản ghi.
- Quên hoàn trả mảng mượn từ `ArrayPool` về pool trong khối `finally`.
- Dùng lại buffer sau khi đã gọi `ArrayPool.Return()`.
- Tối ưu hóa quá sớm (Premature Optimization) trên các luồng CRUD đơn giản chỉ chạy 1 lần/ngày.
- Khẳng định "code này chạy nhanh hơn/ít tốn RAM hơn" trong PR mà không có kết quả đo đạc từ `BenchmarkDotNet`.

---

## 📚 Tài Liệu Tham Chiếu Microsoft (Official References)

- [Memory Management and Garbage Collection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/memory)
- [Memory-efficient data structures with Span&lt;T&gt; and Memory&lt;T&gt;](https://learn.microsoft.com/en-us/dotnet/standard/memory-and-spans/)
- [Use ArrayPool to avoid repetitive allocations](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
- [Writing high-performance C# and .NET code (Performance Guidelines)](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/)

