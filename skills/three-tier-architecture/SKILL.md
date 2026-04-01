---
name: three-tier-architecture
description: 'C# .NET 10 MVC 四層架構規範（Web / Service / Data / Library）。定義各層責任、依賴方向、檔案結構、禁止事項、日誌規範。設計功能或審查架構時自動載入。'
user-invocable: false
---

# 四層架構規範

## 適用時機

- 建立新功能（確認程式碼放在正確的層）
- 審查程式碼（檢查架構邊界）
- 重構（跨層移動程式碼）

---

## 架構概覽

```
┌─────────────────────────────────────┐
│   {Project}.Web（展示層）            │
│   Controllers, Areas/Admin,         │
│   Views, Infrastructure,            │
│   Middleware, ViewComponent          │
├─────────────────────────────────────┤
│   {Project}.Service（商業邏輯層）    │
│   Services, ViewModels, Mapper,     │
│   Constants                         │
├─────────────────────────────────────┤
│   {Project}.Data（資料存取層）       │
│   Entities, Repositories,           │
│   UnitOfWork, DbContext, Migrations │
├─────────────────────────────────────┤
│   {Project}.Library（共用工具層）    │
│   Helpers, Enums, Exceptions        │
│   （無框架相依）                     │
└─────────────────────────────────────┘
```

**依賴方向：**
```
Web → Service → Data → Library（所有層都可引用 Library，Library 不引用其他層）
```

---

## 各層職責

### Web（展示層）

```csharp
[Area("Admin")]
[Authorize(Roles = "Admin")]
public sealed class OrderController(IOrderService orderService) : Controller
{
    [HttpPost, ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(
        CreateOrderViewModel model, CancellationToken ct)
    {
        if (!ModelState.IsValid)
            return View(model);

        var result = await orderService.CreateOrderAsync(model, ct);

        if (result.IsSuccess)
        {
            TempData["Success"] = "訂單建立成功";
            return RedirectToAction(nameof(Index));
        }

        TempData["Error"] = result.ErrorMessage;
        return View(model);
    }
}
```

**允許：**
- 注入 Service interface（`IOrderService`、`IAuthService`）
- ModelState 驗證、TempData 訊息、RedirectToAction
- IFormFile 處理（放 `Infrastructure/` 或 Controller 內的 private method）

**禁止：**
- ❌ 注入 Repository 或 DbContext
- ❌ 包含商業邏輯
- ❌ 直接操作 Entity
- ❌ 直接用 UserManager / SignInManager（透過 IAuthService）

---

### Service（商業邏輯層）

```csharp
public sealed class OrderService(
    IUnitOfWork uow,
    IMapper mapper,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<ServiceResult<OrderDetailViewModel>> CreateOrderAsync(
        CreateOrderViewModel model, CancellationToken ct = default)
    {
        logger.LogInformation("開始建立訂單，客戶：{CustomerId}", model.CustomerId);

        // 商業驗證
        var product = await uow.ProductRepository.GetByIdAsync(model.ProductId, ct)
            .ConfigureAwait(false);

        if (product is null)
        {
            logger.LogWarning("商品不存在：{ProductId}", model.ProductId);
            return ServiceResult<OrderDetailViewModel>.Failure("商品不存在");
        }

        if (product.Stock < model.Quantity)
        {
            logger.LogWarning("庫存不足，商品：{ProductId}，需求：{Requested}，剩餘：{Stock}",
                model.ProductId, model.Quantity, product.Stock);
            return ServiceResult<OrderDetailViewModel>.Failure("庫存不足");
        }

        // Entity 建立
        var order = mapper.Map<Order>(model);
        await uow.OrderRepository.AddAsync(order, ct).ConfigureAwait(false);
        await uow.SaveChangesAsync(ct).ConfigureAwait(false);

        logger.LogInformation("訂單建立成功：{OrderId}", order.Id);
        return ServiceResult<OrderDetailViewModel>.Success(mapper.Map<OrderDetailViewModel>(order));
    }
}
```

**允許：**
- 注入 `IUnitOfWork`、`IMapper`、`ILogger<T>`
- 商業規則驗證
- 透過 UnitOfWork 存取 Repository
- 呼叫 `uow.SaveChangesAsync()`
- Entity ↔ ViewModel 透過 Mapper 轉換

**禁止：**
- ❌ 直接注入 DbContext
- ❌ 回傳 Entity（必須轉 ViewModel）
- ❌ 處理 HTTP 邏輯（StatusCode, ModelState）
- ❌ 方法無日誌

---

### Data（資料存取層）

```csharp
// Entity + Configuration 同檔
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
    public Customer Customer { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.Property(o => o.Total).HasPrecision(10, 2);
        builder.HasQueryFilter(o => !o.IsDeleted);
        builder.HasMany(o => o.Items).WithOne().HasForeignKey(i => i.OrderId);
    }
}
```

```csharp
// Repository — 不呼叫 SaveChanges
public sealed class OrderRepository(ApplicationDbContext context) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct = default)
        => await context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct)
            .ConfigureAwait(false);

    public async Task AddAsync(Order order, CancellationToken ct = default)
        => await context.Orders.AddAsync(order, ct).ConfigureAwait(false);

    public void Update(Order order) => context.Orders.Update(order);
    public void Remove(Order order) => context.Orders.Remove(order);
    // Remove 會被 DbContext 攔截為軟刪除
}
```

```csharp
// UnitOfWork — 暴露所有 Repository + SaveChanges
public sealed class UnitOfWork(
    IDbContextFactory<ApplicationDbContext> factory) : IUnitOfWork
{
    private readonly ApplicationDbContext _context = factory.CreateDbContext();
    private IOrderRepository? _orderRepository;
    private IProductRepository? _productRepository;

    public IOrderRepository OrderRepository
        => _orderRepository ??= new OrderRepository(_context);

    public IProductRepository ProductRepository
        => _productRepository ??= new ProductRepository(_context);

    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await _context.SaveChangesAsync(ct).ConfigureAwait(false);

    public void Dispose() => _context.Dispose();
    public async ValueTask DisposeAsync() => await _context.DisposeAsync();
}
```

**允許：**
- DbContext 操作（LINQ, Include, Projection）
- Fluent API Configuration（與 Entity 同檔）
- Migration

**禁止：**
- ❌ 包含商業邏輯
- ❌ Repository 呼叫 SaveChanges
- ❌ 直接被 Controller 使用

---

### Library（共用工具層）

```csharp
// ServiceResult<T> — 統一 Service 回傳
public class ServiceResult<T>
{
    public bool IsSuccess { get; }
    public T? Data { get; }
    public string? ErrorMessage { get; }

    public static ServiceResult<T> Success(T data) => new() { ... };
    public static ServiceResult<T> Failure(string message) => new() { ... };
}

// PagedResult<T> — 分頁結果
public class PagedResult<T>
{
    public IReadOnlyList<T> Items { get; }
    public int TotalCount { get; }
    public int Page { get; }
    public int PageSize { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}
```

**規則：** 無框架相依（不引用 ASP.NET Core、EF Core）。只放 Helper、Enum、Exception。

---

## 檔案結構

```
src/
  {Project}.Web/
    Controllers/                  # 前台 Controller
    Areas/Admin/Controllers/      # 後台 Controller（[Area("Admin")]）
    Views/                        # Razor Views
    Infrastructure/
      Extensions/                 # DataServiceExtensions, ApplicationServiceExtensions
      Middleware/                  # GlobalExceptionMiddleware 等
      ViewComponents/             # CartBadgeViewComponent 等
    wwwroot/css/, js/, images/
    Program.cs

  {Project}.Service/
    Services/Interfaces/          # I{Feature}Service
    Services/Implementation/      # {Feature}Service（primary ctor）
    ViewModels/{Feature}/         # Create{X}ViewModel, {X}ListViewModel
    Mapper/                       # MapperProfile.cs 或 MapsterConfig.cs
    Constants/                    # CacheKeys, DisplayConstants

  {Project}.Data/
    Entities/                     # Entity + IEntityTypeConfiguration（同檔）
    Repositories/Interfaces/
    Repositories/Implementation/
    UnitOfWork/                   # IUnitOfWork + UnitOfWork
    ApplicationDbContext.cs       # 軟刪除攔截 + Global Query Filter
    DbInitializer.cs              # 種子資料 + Admin 帳號
    Migrations/

  {Project}.Library/
    Helpers/                      # ServiceResult<T>, PagedResult<T>
    Enums/
    Exceptions/                   # NotFoundException, BusinessException

tests/
  {Project}.Tests/
    Unit/Services/
    Integration/
    E2E/
    Fixtures/
```

---

## 跨層資料流

```
HTTP Request
  ↓
Controller（ViewModel 驗證 ModelState）
  ↓
Service（ViewModel → Entity via Mapper，商業邏輯 + 日誌）
  ↓
Repository（Entity ↔ DB）+ UnitOfWork（SaveChanges）
  ↓
Service（Entity → ViewModel via Mapper，回傳 ServiceResult<T>）
  ↓
Controller（ServiceResult → TempData + Redirect 或 View）
  ↓
HTTP Response
```

---

## 軟刪除模式

```csharp
// ISoftDeletable 介面
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

// DbContext 攔截 — SaveChangesAsync 自動轉為軟刪除
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    foreach (var entry in ChangeTracker.Entries<ISoftDeletable>()
        .Where(e => e.State == EntityState.Deleted))
    {
        entry.State = EntityState.Modified;
        entry.Entity.IsDeleted = true;
        entry.Entity.DeletedAt = DateTime.UtcNow;
    }
    return await base.SaveChangesAsync(ct);
}

// Global Query Filter
builder.HasQueryFilter(o => !o.IsDeleted);

// 後台查詢已刪除資料
var allOrders = await context.Orders.IgnoreQueryFilters().ToListAsync(ct);
```

---

## 日誌規範（每個 Service 方法必須有日誌）

```csharp
public async Task<ServiceResult<T>> SomeMethodAsync(...)
{
    _logger.LogInformation("開始 XXX，參數：{ParamName}", paramValue);

    // ... 商業邏輯 ...

    if (失敗)
    {
        _logger.LogWarning("XXX 失敗，原因：{Reason}", reason);
        return ServiceResult<T>.Failure(reason);
    }

    _logger.LogInformation("XXX 成功，結果：{ResultId}", result.Id);
    return ServiceResult<T>.Success(result);
}
```

- 使用 `{PropertyName}` 結構化佔位符（不用字串插值 `$""`）
- **禁止記錄 PII**（Email、密碼、姓名）
- Info: 業務事件 / Warning: 可恢復異常 / Error: 系統錯誤（附 Exception）

---

## 常見違規與修正

| 違規 | 修正 |
|------|------|
| Controller 注入 `IOrderRepository` | 改為注入 `IOrderService` |
| Service 回傳 `Order`（Entity） | 改為回傳 `ServiceResult<OrderDetailViewModel>` |
| Repository 包含商業邏輯 | 移到 Service 層 |
| Controller 呼叫 `uow.SaveChanges()` | 移到 Service 層 |
| Service 方法無日誌 | 加入 LogInformation / LogWarning |
| 日誌使用字串插值 `$"訂單 {id}"` | 改用佔位符 `"訂單：{OrderId}", id` |
| Entity 直接傳到 View | 透過 Mapper 轉為 ViewModel |
| Service 用 `new DbContext()` | 透過 UnitOfWork 存取 |
