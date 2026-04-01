---
name: efcore-patterns
description: 'Entity Framework Core 最佳實踐。涵蓋 NoTracking 預設、Query Splitting、Migration 管理、與 UnitOfWork 整合、常見陷阱。'
user-invocable: false
---

# Entity Framework Core 模式

## 適用時機

- 設定新專案的 EF Core
- 最佳化查詢效能
- 管理 Migration
- 除錯 Change Tracking 問題

---

## 核心規則

1. **預設 NoTracking** — 讀取查詢不追蹤，寫入時明確啟用
2. **絕不手動編輯 Migration** — 只用 CLI 指令
3. **Repository 不呼叫 SaveChanges** — 由 UnitOfWork 管理
4. **讀取使用 Projection** — 只取需要的欄位
5. **永遠有 Row Limit** — 每個查詢都要分頁或 Take

---

## NoTracking 預設配置

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
    }
}
```

**讀取（不需特別處理）：**
```csharp
var orders = await _context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync(ct);  // 快速，無追蹤開銷
```

**寫入（需要明確處理）：**
```csharp
// 方式 1：使用 Update 明確標記
var order = await _context.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
order.Status = OrderStatus.Shipped;
_context.Orders.Update(order);  // 明確標記為修改
// SaveChanges 由 UnitOfWork 處理

// 方式 2：AsTracking 查詢
var order = await _context.Orders
    .AsTracking()
    .FirstOrDefaultAsync(o => o.Id == id, ct);
order.Status = OrderStatus.Shipped;
// Change Tracking 自動偵測變更
```

---

## 批量操作（EF Core 7+）

```csharp
// ✅ 快速：單一 SQL UPDATE
await _context.Orders
    .Where(o => o.ExpiresAt < DateTimeOffset.UtcNow)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, OrderStatus.Expired)
        .SetProperty(o => o.UpdatedAt, DateTimeOffset.UtcNow), ct);

// ✅ 快速：單一 SQL DELETE
await _context.Orders
    .Where(o => o.Status == OrderStatus.Cancelled && o.CreatedAt < cutoffDate)
    .ExecuteDeleteAsync(ct);

// ❌ 慢：載入所有 Entity 到記憶體
var expired = await _context.Orders.Where(...).ToListAsync();
foreach (var o in expired) { o.Status = OrderStatus.Expired; }
await _context.SaveChangesAsync();
```

---

## 避免 N+1 查詢

```csharp
// ❌ N+1 — 每次迴圈查詢一次
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    var items = order.Items;  // 每次觸發查詢！
}

// ✅ Eager Loading — 一次查詢
var orders = await _context.Orders
    .Include(o => o.Items)
    .ToListAsync();

// ✅ Projection — 只取需要的資料
var summaries = await _context.Orders
    .AsNoTracking()
    .Select(o => new OrderSummaryDto(
        o.Id, o.Total, o.Status, o.Items.Count))
    .ToListAsync();
```

---

## Query Splitting（避免笛卡兒爆炸）

```csharp
// ⚠ 危險：多個 Include 可能產生笛卡兒積
var product = await _context.Products
    .Include(p => p.Reviews)     // 100 筆
    .Include(p => p.Images)      // 20 筆
    .Include(p => p.Categories)  // 5 筆
    .FirstOrDefaultAsync(p => p.Id == id);
// 結果：100 × 20 × 5 = 10,000 行！

// ✅ Split Query — 多個查詢，無笛卡兒積
var product = await _context.Products
    .AsSplitQuery()
    .Include(p => p.Reviews)
    .Include(p => p.Images)
    .Include(p => p.Categories)
    .FirstOrDefaultAsync(p => p.Id == id);
// 結果：4 個查詢，約 125 行
```

---

## Migration 管理

```bash
# 建立 Migration
dotnet ef migrations add AddOrderTable \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Web

# 套用 Migration
dotnet ef database update \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Web

# 移除最新 Migration（尚未套用時）
dotnet ef migrations remove \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Web

# 產生 SQL Script（生產環境部署用）
dotnet ef migrations script --idempotent \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Web
```

**禁止：** 手動編輯 migration 檔案、手動刪除 migration 檔案、在 migration 間複製檔案。

---

## Entity Configuration

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Total)
            .HasPrecision(18, 2);

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.CreatedAt);
    }
}
```

---

## 常見陷阱

| 陷阱 | 修正 |
|------|------|
| NoTracking 下更新 Entity 沒反應 | 使用 `_context.Update(entity)` 或 `AsTracking()` |
| 迴圈內查詢（N+1） | 使用 `Include` 或一次查詢所有 |
| 無限制的 `ToListAsync()` | 加 `Take(limit)` 或分頁 |
| 應用層做 Join | 在 SQL/LINQ 中做 Join |
| 多個 Include 的笛卡兒爆炸 | 使用 `AsSplitQuery()` |
| 背景服務使用 Scoped DbContext | 使用 `IDbContextFactory<T>` |
