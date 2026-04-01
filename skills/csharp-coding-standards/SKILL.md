---
name: csharp-coding-standards
description: '現代 C# 程式碼標準。涵蓋 record 型別、pattern matching、nullable reference types、async/await、value object、錯誤處理等最佳實踐。'
user-invocable: false
---

# 現代 C# 程式碼標準

## 適用時機

- 寫新的 C# 程式碼或重構現有程式碼
- 設計 Service / Repository 的 API
- 實作 Domain Model
- 效能關鍵路徑最佳化

---

## 核心原則

1. **預設不可變** — 使用 `record` 和 `init` 屬性
2. **型別安全** — 啟用 Nullable Reference Types，使用 Value Object
3. **Pattern Matching** — 大量使用 `switch` 表達式
4. **全面 Async** — 所有 I/O 操作使用 async/await + CancellationToken
5. **組合優於繼承** — 避免 abstract base class，sealed by default

---

## 型別選擇速查

```csharp
// DTO — 跨層傳遞用，不可變
public record OrderDto(Guid OrderId, Guid CustomerId, decimal Total);

// Value Object — 值語意，效能好
public readonly record struct OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString();
}

public readonly record struct Money(decimal Amount, string Currency);

// Entity — EF Core 需要 class
public class Order
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public List<OrderItem> Items { get; set; } = [];
}

// 錯誤型別 — 明確，不依賴 Exception
public readonly record struct OrderError(string Code, string Message);

// 設定類別 — mutable（Options pattern 需要）
public class SmtpSettings
{
    public const string SectionName = "Smtp";
    public required string Host { get; init; }
    public int Port { get; init; } = 587;
}
```

---

## Async/Await 規則

```csharp
// ✅ 正確：接受 CancellationToken，async 後綴
public async Task<Order> GetOrderAsync(Guid orderId, CancellationToken cancellationToken = default)
{
    return await _repository.GetByIdAsync(orderId, cancellationToken);
}

// ✅ 平行獨立操作
public async Task<DashboardDto> LoadDashboardAsync(Guid userId, CancellationToken ct)
{
    var ordersTask = _orderService.GetRecentAsync(userId, ct);
    var statsTask = _statsService.GetAsync(userId, ct);

    await Task.WhenAll(ordersTask, statsTask);

    return new DashboardDto(
        Orders: await ordersTask,
        Stats: await statsTask);
}

// ❌ 禁止：阻塞 async 程式碼
var result = GetDataAsync().Result;    // 死鎖風險！
var result = GetDataAsync().Wait();    // 死鎖風險！
```

**關鍵規則：**
- 所有 async 方法接受 `CancellationToken cancellationToken = default`
- Library code 使用 `ConfigureAwait(false)`
- 永遠不要用 `.Result` 或 `.Wait()` 阻塞 async

---

## Nullable Reference Types

```csharp
// 專案設定
// <Nullable>enable</Nullable>

// 明確的 null 處理
public async Task<OrderDto?> GetOrderAsync(Guid orderId, CancellationToken ct)
{
    var order = await _repository.GetByIdAsync(orderId, ct);
    return order?.ToDto();  // 可能為 null 就標記 ?
}

// Guard Clause
public void ProcessOrder(Order? order)
{
    ArgumentNullException.ThrowIfNull(order);
    // 此處 order 已確定非 null
}

// Pattern Matching with null
public decimal GetDiscount(Customer? customer) => customer switch
{
    null => 0m,
    { IsVip: true } => 0.20m,
    { OrderCount: > 10 } => 0.10m,
    _ => 0.05m
};
```

---

## Result 型別（取代 Exception 處理預期錯誤）

```csharp
public readonly record struct Result<TValue, TError>
{
    private readonly TValue? _value;
    private readonly TError? _error;
    private readonly bool _isSuccess;

    private Result(TValue value) { _value = value; _error = default; _isSuccess = true; }
    private Result(TError error) { _value = default; _error = error; _isSuccess = false; }

    public bool IsSuccess => _isSuccess;
    public bool IsFailure => !_isSuccess;
    public TValue Value => _isSuccess ? _value! : throw new InvalidOperationException("失敗的結果沒有 Value");
    public TError Error => !_isSuccess ? _error! : throw new InvalidOperationException("成功的結果沒有 Error");

    public static Result<TValue, TError> Success(TValue value) => new(value);
    public static Result<TValue, TError> Failure(TError error) => new(error);

    public TResult Match<TResult>(
        Func<TValue, TResult> onSuccess,
        Func<TError, TResult> onFailure)
        => _isSuccess ? onSuccess(_value!) : onFailure(_error!);
}
```

**何時用 Result vs Exception：**
- **Result：** 預期錯誤（驗證失敗、商業規則、找不到資料）
- **Exception：** 非預期錯誤（網路故障、系統錯誤、程式 bug）

---

## Mapping（禁用 AutoMapper）

```csharp
// ✅ 明確的 mapping extension method
public static class OrderMappingExtensions
{
    public static OrderDto ToDto(this Order entity) => new(
        OrderId: entity.Id,
        CustomerId: entity.CustomerId,
        Total: entity.Total,
        Status: entity.Status.ToString(),
        Items: entity.Items.Select(i => i.ToDto()).ToList());

    public static OrderItemDto ToDto(this OrderItem entity) => new(
        ProductId: entity.ProductId,
        Quantity: entity.Quantity,
        UnitPrice: entity.UnitPrice);
}

// ❌ 禁止：AutoMapper、Mapster 等反射 mapping
// 原因：隱藏 mapping 邏輯、難以除錯、效能差、容易在重構時悄悄壞掉
```

---

## DO's 和 DON'Ts 速查

### DO
- ✅ DTO 使用 `record`
- ✅ Value Object 使用 `readonly record struct`
- ✅ 類別預設 `sealed`
- ✅ 啟用 Nullable Reference Types
- ✅ async 方法接受 CancellationToken
- ✅ 預期錯誤使用 Result 型別
- ✅ 明確的 mapping 方法

### DON'T
- ❌ 可以用 record 時不要用 mutable class
- ❌ 不要建立深層繼承
- ❌ 不要忽略 nullable warning
- ❌ 不要用 `.Result` / `.Wait()` 阻塞 async
- ❌ 不要用 AutoMapper / Mapster
- ❌ 不要忘記 CancellationToken
- ❌ 不要對預期的商業錯誤 throw Exception
