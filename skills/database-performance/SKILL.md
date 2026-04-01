---
name: database-performance
description: '資料庫效能模式。讀寫分離、避免 N+1、AsNoTracking、Row Limit、禁止應用層 Join。'
user-invocable: false
---

# 資料庫效能模式

## 適用時機

- 設計資料存取層
- 最佳化慢查詢
- 避免常見效能陷阱

---

## 核心規則

1. **讀寫分離** — 讀取和寫入用不同的型別
2. **批次思維** — 避免 N+1 查詢
3. **只取需要的** — 使用 Projection，不要 SELECT *
4. **永遠有 Row Limit** — 每個查詢都要 Take / 分頁
5. **SQL 做 Join** — 禁止在應用層做 Join
6. **讀取用 NoTracking** — 追蹤很貴

---

## 讀寫分離（CQRS 簡化版）

```csharp
// 讀取 — 回傳 DTO，輕量
public interface IOrderReadStore
{
    Task<OrderDetailDto?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<OrderSummaryDto>> GetByCustomerAsync(
        Guid customerId, int page, int pageSize, CancellationToken ct = default);
}

// 寫入 — 操作 Entity
public interface IOrderWriteStore
{
    Task AddAsync(Order order, CancellationToken ct = default);
    void Update(Order order);
    void Remove(Order order);
}
```

---

## 分頁模式

```csharp
public async Task<PagedResult<OrderSummaryDto>> GetOrdersPagedAsync(
    Guid customerId, int page, int pageSize, CancellationToken ct)
{
    var query = _context.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId)
        .OrderByDescending(o => o.CreatedAt);

    var totalCount = await query.CountAsync(ct);

    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)  // 永遠有 limit！
        .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status, o.CreatedAt))
        .ToListAsync(ct);

    return new PagedResult<OrderSummaryDto>(items, totalCount, page, pageSize);
}

public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}
```

---

## 反模式速查

| 反模式 | 修正 |
|--------|------|
| 無 Row Limit 的查詢 | 加 `Take(limit)` 或分頁 |
| SELECT *（載入所有欄位） | 使用 `Select()` 只取需要的 |
| 迴圈內查詢（N+1） | 使用 `Include` 或一次查詢所有 |
| 應用層做 Join | 在 LINQ/SQL 中做 Join |
| 笛卡兒爆炸（多個 Include） | 使用 `AsSplitQuery()` |
| 讀取時追蹤 Entity | 使用 `AsNoTracking()` |
| Generic Repository | 目的導向的 Repository interface |
