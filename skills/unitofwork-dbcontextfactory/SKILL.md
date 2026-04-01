---
name: unitofwork-dbcontextfactory
description: 'UnitOfWork + IDbContextFactory 模式。暴露 Repository 屬性的 UnitOfWork、IDbContextFactory 懶載入、多工場景、背景服務。'
user-invocable: false
---

# UnitOfWork + IDbContextFactory 模式

## 適用時機

- 建立資料存取層
- 需要跨多個 Repository 的交易一致性
- 背景服務或多工場景需要獨立 DbContext
- 新增 Repository 時須同步更新 IUnitOfWork + UnitOfWork

---

## 核心模式

### IUnitOfWork — 暴露 Repository 屬性

```csharp
public interface IUnitOfWork : IDisposable, IAsyncDisposable
{
    IOrderRepository OrderRepository { get; }
    IProductRepository ProductRepository { get; }
    ICustomerRepository CustomerRepository { get; }
    // 新增 Repository 時在此加入

    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

### UnitOfWork — IDbContextFactory + 懶載入 ??=

```csharp
public sealed class UnitOfWork(
    IDbContextFactory<ApplicationDbContext> factory) : IUnitOfWork
{
    private readonly ApplicationDbContext _context = factory.CreateDbContext();

    private IOrderRepository? _orderRepository;
    private IProductRepository? _productRepository;
    private ICustomerRepository? _customerRepository;

    public IOrderRepository OrderRepository
        => _orderRepository ??= new OrderRepository(_context);

    public IProductRepository ProductRepository
        => _productRepository ??= new ProductRepository(_context);

    public ICustomerRepository CustomerRepository
        => _customerRepository ??= new CustomerRepository(_context);

    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await _context.SaveChangesAsync(ct).ConfigureAwait(false);

    public void Dispose() => _context.Dispose();

    public async ValueTask DisposeAsync() => await _context.DisposeAsync();
}
```

**關鍵設計：**
- `factory.CreateDbContext()` 在 UnitOfWork 建構時建立 DbContext
- Repository 使用 `??=` 懶載入（首次存取時才建立）
- 所有 Repository 共享同一個 DbContext（同一個交易）
- `SaveChangesAsync` 一次性提交所有變更

---

## DI 註冊

```csharp
public static class DataServiceExtensions
{
    public static IServiceCollection AddDataServices(
        this IServiceCollection services, IConfiguration config)
    {
        var connectionString = config.GetConnectionString("DefaultConnection")!;

        // DbContextFactory（讓 UnitOfWork 和多工場景都能用）
        services.AddDbContextFactory<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));

        // Identity
        services.AddIdentity<AppUser, IdentityRole>()
            .AddEntityFrameworkStores<ApplicationDbContext>()
            .AddDefaultTokenProviders();

        // UnitOfWork — Scoped
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        return services;
    }
}
```

**注意：** 不個別註冊 Repository。Repository 透過 UnitOfWork 懶載入存取。

---

## 使用模式

### 模式 1：標準 Service 操作

```csharp
public sealed class OrderService(
    IUnitOfWork uow,
    IMapper mapper,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<ServiceResult<OrderDetailViewModel>> CreateOrderAsync(
        CreateOrderViewModel model, CancellationToken ct = default)
    {
        logger.LogInformation("建立訂單，客戶：{CustomerId}", model.CustomerId);

        var product = await uow.ProductRepository
            .GetByIdAsync(model.ProductId, ct).ConfigureAwait(false);

        if (product is null)
            return ServiceResult<OrderDetailViewModel>.Failure("商品不存在");

        var order = mapper.Map<Order>(model);
        await uow.OrderRepository.AddAsync(order, ct).ConfigureAwait(false);
        await uow.SaveChangesAsync(ct).ConfigureAwait(false);

        logger.LogInformation("訂單建立成功：{OrderId}", order.Id);
        return ServiceResult<OrderDetailViewModel>.Success(
            mapper.Map<OrderDetailViewModel>(order));
    }
}
```

### 模式 2：平行查詢用 IDbContextFactory

```csharp
// 後台 Dashboard — 多個統計同時查詢
public sealed class DashboardService(
    IDbContextFactory<ApplicationDbContext> dbFactory,
    ILogger<DashboardService> logger) : IDashboardService
{
    public async Task<DashboardViewModel> GetDashboardAsync(CancellationToken ct = default)
    {
        logger.LogInformation("載入後台儀表板");

        // 每個 Task 各自建立 DbContext — 執行緒安全
        var totalUsersTask = CountAsync(ctx => ctx.Users.CountAsync(ct));
        var totalRevenueTask = CountAsync(ctx =>
            ctx.Orders.Where(o => o.Status == OrderStatus.Completed)
                .SumAsync(o => o.Total, ct));
        var recentOrdersTask = QueryAsync(ctx =>
            ctx.Orders.AsNoTracking()
                .OrderByDescending(o => o.CreatedAt)
                .Take(10).ToListAsync(ct));

        await Task.WhenAll(totalUsersTask, totalRevenueTask, recentOrdersTask);

        return new DashboardViewModel
        {
            TotalUsers = await totalUsersTask,
            TotalRevenue = await totalRevenueTask,
            RecentOrders = await recentOrdersTask
        };
    }

    private async Task<T> CountAsync<T>(
        Func<ApplicationDbContext, Task<T>> query)
    {
        await using var ctx = await dbFactory.CreateDbContextAsync();
        return await query(ctx).ConfigureAwait(false);
    }

    private async Task<T> QueryAsync<T>(
        Func<ApplicationDbContext, Task<T>> query)
    {
        await using var ctx = await dbFactory.CreateDbContextAsync();
        return await query(ctx).ConfigureAwait(false);
    }
}
```

### 模式 3：背景服務

```csharp
public sealed class DailyMaintenanceService(
    IDbContextFactory<ApplicationDbContext> dbFactory,
    ILogger<DailyMaintenanceService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var timer = new PeriodicTimer(TimeSpan.FromHours(24));

        while (await timer.WaitForNextTickAsync(ct))
        {
            logger.LogInformation("開始每日維護作業");

            await using var context = await dbFactory.CreateDbContextAsync(ct);

            var expiredCount = await context.Orders
                .Where(o => o.Status == OrderStatus.Pending
                         && o.CreatedAt < DateTime.UtcNow.AddDays(-7))
                .ExecuteUpdateAsync(
                    s => s.SetProperty(o => o.Status, OrderStatus.Expired), ct);

            logger.LogInformation("每日維護完成，過期訂單：{Count}", expiredCount);
        }
    }
}
```

---

## 新增 Repository 的 Checklist

新增一個 Entity 的 Repository 時，必須同步更新：

1. `Data/Repositories/Interfaces/I{Entity}Repository.cs` — 建立介面
2. `Data/Repositories/Implementation/{Entity}Repository.cs` — 建立實作
3. `Data/UnitOfWork/IUnitOfWork.cs` — 加入 `I{Entity}Repository {Entity}Repository { get; }`
4. `Data/UnitOfWork/UnitOfWork.cs` — 加入 private field + 懶載入屬性
5. `dotnet build` — 確認編譯通過

---

## 何時用 UnitOfWork vs IDbContextFactory

| 場景 | 方式 |
|------|------|
| Controller → Service（標準 CRUD） | `IUnitOfWork`（Scoped，共享 DbContext） |
| Dashboard 平行查詢 | `IDbContextFactory`（各自建立 DbContext + Task.WhenAll） |
| BackgroundService | `IDbContextFactory`（每次操作建立新 DbContext） |
| Parallel.ForEachAsync | `IDbContextFactory`（每個 task 獨立 DbContext） |
| 需要交易一致性的多步操作 | `IUnitOfWork`（同一個 DbContext） |

**鐵律：** DbContext 不是執行緒安全的。多工場景必須每個 task 各自建立 DbContext。
