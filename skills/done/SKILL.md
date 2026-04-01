---
name: done
description: '完成檢查。在任何產出程式碼變更的工作流程結尾自動載入。驗證建置、架構、日誌、程式碼品質。'
user-invocable: false
---

# 完成檢查

## 適用時機

任何產出程式碼變更的工作流程結尾：`/write`、`/fix`、`/refactor`。

---

## 檢查清單

### 1. 建置與測試

```bash
dotnet build
dotnet test  # 如有測試專案
```

- [ ] 建置無錯誤、無新 warning
- [ ] 所有測試通過（如有測試專案）

### 2. 四層架構合規

- [ ] Controller 只呼叫 Service interface
- [ ] Service 透過 IUnitOfWork 存取 Repository
- [ ] Entity 沒有洩漏到 View（透過 ViewModel + Mapper）
- [ ] Web 層沒有直接用 UserManager/SignInManager
- [ ] 依賴方向：Web → Service → Data → Library
- [ ] Repository 沒有呼叫 SaveChanges（由 UnitOfWork 管理）

### 3. 日誌合規

- [ ] 每個 Service 方法有入口日誌（LogInformation）
- [ ] 失敗路徑有 Warning 日誌
- [ ] 使用 `{PropertyName}` 佔位符（非字串插值）
- [ ] 無 PII 洩漏（Email、密碼、姓名）
- [ ] catch 區塊用 LogError 並附帶 Exception 物件

### 4. 程式碼品質

- [ ] 方法長度合理（Service ≤ 40 行，Action ≤ 20 行）
- [ ] 命名遵循慣例（PascalCase / _camelCase / Async 後綴）
- [ ] 無 magic number（使用 DisplayConstants 等）
- [ ] async 方法接受 CancellationToken
- [ ] Data / Service 層有 .ConfigureAwait(false)
- [ ] POST 有 [ValidateAntiForgeryToken]
- [ ] 後台有 [Area("Admin")] + [Authorize(Roles = "Admin")]

### 5. 資料存取

- [ ] 讀取查詢使用 AsNoTracking（或全域預設 NoTracking）
- [ ] 無 N+1 查詢
- [ ] 查詢有 row limit（分頁或 Take）
- [ ] ISoftDeletable Entity 只做軟刪除

### 6. DI 註冊

- [ ] 新 Service 已在 AddApplicationServices() 註冊
- [ ] 新 Repository 已在 IUnitOfWork + UnitOfWork 加入

### 7. 同步更新

- [ ] Entity 變更有對應 Migration
- [ ] 新 Repository 同步更新 IUnitOfWork + UnitOfWork
- [ ] Mapper 映射已更新

---

## 輸出格式

```
✅ 完成檢查通過
- 建置：成功
- 架構：四層邊界正確
- 日誌：所有 Service 方法有日誌
- 品質：無違規

或

⚠️ 完成檢查有問題
- [問題 1]
- [問題 2]
建議：[修復建議]
```
