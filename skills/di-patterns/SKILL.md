---
name: di-patterns
description: '依賴注入模式。使用 IServiceCollection 擴展方法組織註冊、Lifetime 管理、Keyed Services、與三層架構整合。'
user-invocable: false
---

# 依賴注入模式

## 適用時機

- 組織 Program.cs 的 Service 註冊
- 決定 Service 的 Lifetime（Singleton / Scoped / Transient）
- 在測試中替換依賴
- 設計可重用的 DI 擴展

---

## Extension Method 組織（推薦）

```csharp
// ✅ 乾淨的 Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddDataAccess(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddBusinessServices()
    .AddEmailServices();

var app = builder.Build();
```

```csharp
// DataAccessServiceCollectionExtensions.cs
public static class DataAccessServiceCollectionExtensions
{
    public static IServiceCollection AddDataAccess(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));

        services.AddDbContextFactory<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();

        return services;
    }
}
```

**命名慣例：** `{Feature}ServiceCollectionExtensions.cs`，放在對應功能的旁邊。

---

## Lifetime 速查

| Lifetime | 適用場景 | 範例 |
|----------|---------|------|
| **Singleton** | 無狀態、執行緒安全、建立成本高 | Configuration, HttpClientFactory, Cache |
| **Scoped** | 每個 HTTP 請求一個實例 | DbContext, Repository, UnitOfWork, Service |
| **Transient** | 輕量、短命、每次注入新實例 | Validator, 短命 Helper |

---

## 常見錯誤

### Singleton 注入 Scoped（必死）

```csharp
// ❌ Singleton 捕獲了 Scoped 的 DbContext — 過時的 context！
public class CacheService  // Singleton
{
    private readonly IOrderRepository _repo;  // Scoped — 啟動時捕獲，之後都是同一個！
}

// ✅ 注入 IServiceProvider，每次操作建立 scope
public class CacheService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public async Task<OrderDto?> GetOrderAsync(Guid id)
    {
        using var scope = _scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        var order = await repo.GetByIdAsync(id);
        return order?.ToDto();
    }
}
```

### 背景服務忘記建立 Scope

```csharp
// ❌ BackgroundService 注入 Scoped 會 throw
public class BadWorker : BackgroundService
{
    private readonly IOrderService _service;  // Scoped — 無法注入到 Singleton！
}

// ✅ 每個工作單元建立 scope
public class GoodWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
        await service.ProcessPendingOrdersAsync(ct);
    }
}
```

---

## Keyed Services（.NET 8+）

```csharp
// 同一介面，不同實作
builder.Services.AddKeyedScoped<INotificationSender, EmailSender>("email");
builder.Services.AddKeyedScoped<INotificationSender, SmsSender>("sms");

// 注入
public class NotificationService(
    [FromKeyedServices("email")] INotificationSender emailSender,
    [FromKeyedServices("sms")] INotificationSender smsSender)
{
}
```

---

## 測試中替換依賴

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // 生產 Service 已透過 Add* 方法註冊
                // 只替換測試需要的部分
                services.RemoveAll<IEmailSender>();
                services.AddSingleton<IEmailSender, FakeEmailSender>();
            });
        });
    }
}
```
