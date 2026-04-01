---
name: principles
description: '強制執行 C# .NET 10 MVC 專案的程式碼約束：方法長度限制、DI 原則、SOLID、三層架構邊界。由 /write、/fix、/review、/refactor 在設計介面或檢查架構時自動載入。'
user-invocable: false
---

# C# 程式碼原則

## 適用時機

- 設計新功能或模組
- 架構決策
- 定義介面 / API 邊界
- 決定依賴方向

---

## SOLID 快速檢查表

| 違反跡象 | 原則 |
|---------|------|
| 類別有 2 個以上不相關的修改原因 | SRP（單一責任） |
| 新增功能需要修改現有類別 | OCP（開放封閉） |
| `is` / `as` 型別判斷散佈在多處呼叫點 | LSP（里氏替換） |
| 介面有實作者留空或 throw 的方法 | ISP（介面隔離） |
| 在 Service 內部 `new Repository()` 而非建構子注入 | DIP（依賴反轉） |

如果都沒出現，SOLID 滿足 — 繼續。

---

## 方法長度限制

- C# 方法：**建議 ≤ 40 行**（不含測試）
- Controller Action：**建議 ≤ 20 行**（只做：驗證 → 呼叫 Service → 回傳結果）
- 協調型方法（orchestration）：允許到 60 行，前提是拆解會讓順序邏輯更難懂

當方法接近上限，提取 helper 處理不同的責任。

---

## 命名慣例

| 項目 | 慣例 | 範例 |
|------|------|------|
| 類別 / Record | PascalCase | `OrderService`, `OrderDto` |
| 介面 | I + PascalCase | `IOrderService`, `IUnitOfWork` |
| Public 方法 | PascalCase | `CreateOrderAsync` |
| Private 欄位 | _camelCase | `_orderRepository` |
| 參數 / 區域變數 | camelCase | `orderId`, `cancellationToken` |
| 常數 | PascalCase | `MaxRetryCount` |
| Async 方法 | 後綴 Async | `GetOrderAsync` |

---

## 依賴方向（三層架構）

```
Controller → Service → Repository → DbContext
     ↓           ↓          ↓
  ViewModel     DTO       Entity
```

**嚴格規則：**
- Controller 只依賴 Service interface
- Service 只依賴 Repository interface + UnitOfWork interface
- Repository 只依賴 DbContext / DbContextFactory
- **禁止反向依賴**
- **禁止跨層直接引用**（Controller 不得直接用 Repository）

---

## 型別選擇

| 場景 | 型別 |
|------|------|
| DTO（跨層傳遞） | `record` |
| Value Object（OrderId, Money） | `readonly record struct` |
| Entity（資料庫對應） | `class`（EF Core 需要） |
| Service | `sealed class` + interface |
| 設定 | `class` with `init` properties |
| 錯誤 | `readonly record struct` |

---

## Claude 常見錯誤

**1. 投機性泛化**
Claude 傾向加「以防萬一」的抽象、額外設定選項、假設性擴展點。只為當前需求設計。三行相似的程式碼 > 一個過早的 helper。

**2. 過度使用 Generic Repository**
Claude 預設建立 `IRepository<T>`。在三層架構中，使用**目的導向的 Repository interface**（`IOrderRepository`），才能優化每個查詢。

**3. Entity 洩漏到 Presentation Layer**
Claude 常直接把 Entity 回傳給 Controller。**必須透過 DTO 轉換**。

**4. 忘記 CancellationToken**
Claude 寫 async 方法時常忘記接受 `CancellationToken`。所有 async 方法都必須接受。

---

---

## 日誌規範

- Service 注入 `ILogger<T>`，使用 primary constructor
- **每個 Service 方法的入口和出口都要有日誌**
- 使用 `{PropertyName}` 結構化佔位符，**禁止** `$""` 字串插值
- **禁止記錄 PII**（Email、密碼、完整姓名）
- Info: 業務事件 / Warning: 可恢復異常 / Error: 系統錯誤（附 Exception）

```csharp
// ✅ 正確
_logger.LogInformation("訂單建立成功：{OrderId}，金額：{Total}", order.Id, order.Total);

// ❌ 錯誤：字串插值
_logger.LogInformation($"訂單建立成功：{order.Id}");

// ❌ 錯誤：記錄 PII
_logger.LogInformation("使用者登入：{Email}", user.Email);

// ✅ 正確：用 UserId 取代 Email
_logger.LogInformation("使用者登入：{UserId}", user.Id);
```

---

## 完成檢查表

| 檢查項目 | 通過條件 |
|---------|---------|
| 方法長度在限制內 | ≤ 40 行（Action ≤ 20 行） |
| 依賴透過 primary constructor 注入 | 無 `new Service()` 在類別內部 |
| 無投機性抽象 | 只有當前需求用到的程式碼 |
| 依賴方向正確 | Web → Service → Data → Library |
| Entity 不外洩 | Controller/View 只看到 ViewModel |
| Service 方法有日誌 | 入口 LogInformation + 失敗 LogWarning |
| 日誌無 PII | 不記錄 Email、密碼、姓名 |
| .ConfigureAwait(false) | Data / Service 層的 await |
