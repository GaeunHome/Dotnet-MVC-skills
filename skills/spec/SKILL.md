---
name: spec
description: '在寫任何實作前定義介面契約。當 C# interface 或 method signature 尚未存在、公開 API 或回傳型別正在改變、或你無法立即說出 3 個以上邊界案例時使用。輸出：行為描述 + C# interface + 不變量清單。'
argument-hint: [功能描述]
disable-model-invocation: true
allowed-tools: Read, Grep, Glob,
  Bash(git log:*), Bash(git diff:*), Bash(git show:*),
  Skill
---

# /spec — 介面契約定義

## 適用性判斷（Spec Gate）

在載入此 skill 前回答三個問題：

| 問題 | YES → | NO → |
|------|-------|------|
| 這是 bug fix 或不影響公開介面的內部修改？ | 跳過 spec → `/testing` | 繼續 |
| 此功能的 C# interface / method signature 已存在？ | 跳過 spec → `/testing` | 繼續 |
| 現在可以立刻說出 3 個以上邊界案例？ | 跳過 spec → `/testing` | **載入 spec** |

---

## Spec 輸出：三個步驟

### 步驟 1 — 行為描述

用 1–3 句自然語言描述。使用 Given/When/Then 思維結構：

```
Given [起始條件或上下文]，
When  [動作或輸入]，
Then  [預期結果]。
```

規則：
- 使用商業語言，不用實作語言
- 涵蓋 happy path 和至少一個失敗案例
- 如果你無法不提及內部細節（例如「呼叫資料庫」）就無法描述，代表邊界不清楚 — 先釐清再繼續

**範例：**
```
Given 一個有效的訂單建立請求，
When  使用者提交訂單，
Then  系統建立訂單並回傳訂單編號，
  且庫存扣除對應數量，
  且若商品庫存不足則回傳明確錯誤。
```

---

### 步驟 2 — 影響範圍與 Entity 設計

定義受影響的四層檔案和 Entity 欄位。**越詳細 Claude 越不會猜錯。**

```
Entity: Wishlist
├── Id              int         PK
├── UserId          string      FK → AppUser
├── TripId          int         FK → Trip
├── CreatedAt       DateTime    UTC
├── IsDeleted       bool        軟刪除
└── DeletedAt       DateTime?

Fluent API:
├── HasQueryFilter(w => !w.IsDeleted)
└── HasIndex(w => new { w.UserId, w.TripId }).IsUnique()
```

影響範圍表：

| 層 | 異動項目 | 新增/修改 |
|----|----------|----------|
| Data | Entity + Configuration + Migration | 新增 |
| Data | Repository Interface + Implementation | 新增 |
| Data | IUnitOfWork + UnitOfWork | 修改 |
| Service | ViewModel + Mapper + Service | 新增 |
| Web | Controller + Views | 新增 |

規則：
- Entity 欄位寫清楚 — Claude 才不用猜型別和關聯
- Fluent API 約束寫清楚 — 特別是 Unique Index、DeleteBehavior
- 影響範圍表列出「新增」和「修改」— Claude 才知道要動哪些現有檔案

---

### 步驟 3 — 不變量清單

列出實作必須始終遵守的約束。這些將成為 TDD 的邊界測試案例。

格式：

```
✓ CustomerId 必須是有效的 GUID — 拒絕 Guid.Empty
✓ Items 不得為空集合 — 至少一個品項
✓ Quantity 必須 > 0 — 拒絕 0 和負數
✓ 當庫存不足時回傳 INSUFFICIENT_STOCK 錯誤（不是 Exception）
✓ 訂單總金額等於所有品項小計的加總
✓ 成功建立後，UnitOfWork 必須 Commit
⚠ AI 約束：不要在 Service 層直接操作 DbContext，必須透過 Repository + UnitOfWork
⚠ AI 約束：DTO 和 Entity 之間必須有明確的 mapping，不要共用型別
```

圖例：
- `✓` — 必須為真，將成為測試案例
- `⚠` — AI 執行約束 — 實作不得違反此邊界

---

## 連接 Spec 與 TDD

Spec 完成後，不變量直接對應第一批失敗測試：

| 不變量 | 測試名稱 |
|--------|---------|
| Items 不得為空集合 | `CreateOrder_EmptyItems_ReturnsValidationError` |
| Quantity 必須 > 0 | `CreateOrder_ZeroQuantity_ReturnsValidationError` |
| 庫存不足回傳錯誤 | `CreateOrder_InsufficientStock_ReturnsStockError` |

交棒給 `/testing`：介面是 TDD 循環的輸入。
