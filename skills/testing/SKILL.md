---
name: testing
description: '強制執行 TDD 紀律與 AI 護欄。在寫測試、實作功能或修 bug 時自動載入。確保測試在實作前寫、先失敗再通過（Shadow Run）、且 AI 永遠不刪除或弱化測試。'
user-invocable: false
---

# 測試紀律

## 適用時機

- 實作新功能
- 修復 Bug
- 改變現有行為
- 寫或修改測試

跳過：純重構且已有完整測試覆蓋、純文件變更。

---

## 整合測試優先

先測試元件協作，再測試單獨元件。

- 先寫整合測試 — 驗證各部分正確協作
- 只在整合測試無法經濟地涵蓋的邊界案例加 unit test
- 沒有整合路徑的程式碼是**死程式碼** — 刪除或連接

### 三層架構的測試策略

| 層級 | 測試類型 | 範例 |
|------|---------|------|
| Controller | 整合測試（WebApplicationFactory） | HTTP 請求 → Service → Repository → DB |
| Service | Unit Test + 整合測試 | Mock Repository 測邏輯 / 真實 DB 測端到端 |
| Repository | 整合測試（TestContainers） | 真實資料庫操作 |

---

## TDD 循環

三個步驟按順序。不跳過、不合併。

**Red** — 在任何實作前寫失敗測試。
- 確認：「測試失敗，因為 [具體原因]。」
- 不要寫任何 production code。
- **Shadow Run：** 對現有 codebase 執行 `dotnet test`。測試**必須**失敗。如果直接通過，測試無效 — 重寫。
- 明確陳述：「Shadow Run: 測試以 [錯誤] 失敗，因為 [方法/功能] 尚未實作。」

**Green** — 寫最少的程式碼讓測試通過。
- 確認：「所有測試通過。」
- 忍住不要在此步驟清理。

**Refactor** — 在不改變行為的情況下改善程式碼。
- 確認：「測試仍然通過。」

---

## AI 護欄

違反任何紅色規則 = **立即停止** — 回退並調查。

| 信號 | 等級 | 動作 |
|------|:----:|------|
| AI 刪除失敗測試 | 紅 | 立即停止。測試必須通過，不是消失。 |
| AI 標記測試為 `[Skip]` / `[Fact(Skip=...)]` | 紅 | 立即停止。等同刪除。 |
| AI 弱化斷言（例如 `Should().NotBeNull()` 取代具體值檢查） | 紅 | 立即停止。斷言強度不得降低。 |
| AI 在實作後才寫測試 | 黃 | 測試可能確認錯誤行為。需要審查。 |
| CI 通過但新行為無測試 | 黃 | 缺少測試 = Red 步驟未完成。 |

**根本規則：** 測試約束 AI 行為。通過但測試更少的套件，比失敗但測試更多的套件更糟。

---

## 測試命名慣例

```
[方法名]_[情境]_[預期結果]
```

範例：
```csharp
[Fact]
public async Task CreateOrder_ValidRequest_ReturnsOrderDto()

[Fact]
public async Task CreateOrder_EmptyItems_ReturnsValidationError()

[Fact]
public async Task CreateOrder_InsufficientStock_ReturnsStockError()
```

---

## 測試結構（AAA）

```csharp
[Fact]
public async Task CreateOrder_ValidRequest_ReturnsOrderDto()
{
    // Arrange — 準備測試資料和依賴
    var request = new CreateOrderRequest(
        CustomerId: Guid.NewGuid(),
        Items: [new OrderItemRequest(ProductId: _productId, Quantity: 2)]);

    // Act — 執行被測方法
    var result = await _orderService.CreateOrderAsync(request);

    // Assert — 驗證結果
    result.IsSuccess.Should().BeTrue();
    result.Value.Items.Should().HaveCount(1);
    result.Value.Total.Should().Be(200m);
}
```

---

## Claude 常見錯誤

**1. 預設 Mock 一切**
Claude 傾向 mock 每個依賴、只寫隔離 unit test。這會漏掉真實整合 bug。至少一個測試要讓兩個以上元件真正協作。

**2. 測試鏡像實作**
Claude 讀你的 if-else 然後寫完全對應的 expect()。這些測試只驗證「程式碼做了程式碼做的事」。從需求/spec 角度寫測試。

**3. 跳過 Red 步驟**
Claude 常把測試和實作一起寫。測試從未在失敗狀態執行過，你無法確定它測了正確的東西。

---

## 完成檢查表

| 檢查項目 | 通過條件 |
|---------|---------|
| TDD 順序遵守 | 失敗測試在 production code 之前 |
| Shadow Run 確認 | 新測試對現有 codebase 確實失敗 |
| 無刪除/跳過測試 | 測試數量未減少；無新 Skip |
| 斷言強度維持 | 無斷言被弱化 |
| 至少一個協作測試 | Service 層測試有元件互動 |
| 非鏡像測試 | 測試源自 spec/需求，非實作 |
