---
name: concurrency
description: '.NET 多工與並行模式。從 async/await 到 Channel 到 Parallel.ForEachAsync，選擇正確的並行抽象。'
user-invocable: false
---

# .NET 多工模式

## 適用時機

- 決定如何處理並行操作
- 需要平行處理 CPU 密集工作
- 設計 Producer/Consumer 模式
- 避免死鎖和競爭條件

---

## 決策樹

```
你要做什麼？
│
├─► 等待 I/O（HTTP、資料庫、檔案）？
│   └─► async/await
│
├─► 平行處理集合（CPU 密集）？
│   └─► Parallel.ForEachAsync
│
├─► Producer/Consumer 工作佇列？
│   └─► System.Threading.Channels
│
├─► 多個獨立的 async 操作同時執行？
│   └─► Task.WhenAll
│
├─► 定期執行工作？
│   └─► PeriodicTimer
│
└─► 以上都不符合？
    └─► 問自己：「真的需要共享可變狀態嗎？」
        ├─► 可以避免 → 重新設計（不可變、訊息傳遞）
        └─► 真的避不開 → Channel 或 ConcurrentDictionary
```

---

## Level 1：async/await（預設選擇）

```csharp
// 簡單 I/O
public async Task<OrderDto?> GetOrderAsync(Guid id, CancellationToken ct)
{
    var order = await _repository.GetByIdAsync(id, ct);
    return order?.ToDto();
}

// 平行執行獨立操作
public async Task<DashboardDto> LoadDashboardAsync(Guid userId, CancellationToken ct)
{
    var ordersTask = _orderService.GetRecentAsync(userId, ct);
    var statsTask = _statsService.GetAsync(userId, ct);
    var notificationsTask = _notificationService.GetUnreadAsync(userId, ct);

    await Task.WhenAll(ordersTask, statsTask, notificationsTask);

    return new DashboardDto(
        Orders: await ordersTask,
        Stats: await statsTask,
        Notifications: await notificationsTask);
}
```

---

## Level 2：Parallel.ForEachAsync（CPU 密集平行）

```csharp
public async Task ProcessOrdersAsync(
    IReadOnlyList<Guid> orderIds,
    CancellationToken ct)
{
    await Parallel.ForEachAsync(
        orderIds,
        new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        },
        async (orderId, token) =>
        {
            // 每個 task 用獨立 DbContext！
            await using var context = await _dbFactory.CreateDbContextAsync(token);
            await ProcessSingleOrderAsync(context, orderId, token);
        });
}
```

---

## Level 3：Channel（Producer/Consumer）

```csharp
public sealed class OrderProcessingChannel
{
    private readonly Channel<OrderMessage> _channel;

    public OrderProcessingChannel()
    {
        _channel = Channel.CreateBounded<OrderMessage>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait  // 背壓：滿了就等待
        });
    }

    // Producer（Controller 或 Service 呼叫）
    public async ValueTask EnqueueAsync(OrderMessage message, CancellationToken ct)
    {
        await _channel.Writer.WriteAsync(message, ct);
    }

    // Consumer（BackgroundService 處理）
    public IAsyncEnumerable<OrderMessage> ReadAllAsync(CancellationToken ct)
    {
        return _channel.Reader.ReadAllAsync(ct);
    }
}

// 背景消費者
public sealed class OrderProcessingWorker : BackgroundService
{
    private readonly OrderProcessingChannel _channel;
    private readonly IServiceScopeFactory _scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var message in _channel.ReadAllAsync(ct))
        {
            using var scope = _scopeFactory.CreateScope();
            var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
            await service.ProcessAsync(message, ct);
        }
    }
}
```

---

## 禁止事項

```csharp
// ❌ 手動建立 Thread
var thread = new Thread(() => ProcessOrders());
thread.Start();
// ✅ 使用 Task.Run 或更好的抽象

// ❌ 阻塞 async（死鎖風險）
var result = GetDataAsync().Result;
// ✅ await GetDataAsync()

// ❌ 共享可變狀態無保護
var results = new List<Result>();
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    results.Add(await ProcessAsync(item, ct));  // 競爭條件！
});
// ✅ 使用 ConcurrentBag<Result>

// ❌ 多個 Task 共用同一個 DbContext
// ✅ 每個 Task 透過 DbContextFactory 建立獨立 context
```

---

## MVC 常見多工場景

| 場景 | 方案 |
|------|------|
| Dashboard 載入多個資料來源 | `Task.WhenAll` 平行查詢 |
| 批次處理匯入資料 | `Parallel.ForEachAsync` + DbContextFactory |
| 背景處理訂單 | `Channel<T>` + BackgroundService |
| 定期清理過期資料 | `PeriodicTimer` + BackgroundService + DbContextFactory |
| 傳送通知給多人 | `Parallel.ForEachAsync`（有限並行度） |
