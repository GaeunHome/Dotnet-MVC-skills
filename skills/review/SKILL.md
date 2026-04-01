---
name: review
description: '審查本地程式碼變更的風格、測試品質和架構。當使用者要求「review code」、「檢查程式碼品質」、「審查 diff」或需要 pre-commit 審查時使用。'
argument-hint: [--staged | path | (空=最新 commit)]
disable-model-invocation: true
allowed-tools: Read, Grep, Glob,
  Bash(git diff:*), Bash(git show:*),
  Skill
---

# /review — 程式碼審查工作流程

## Skills 載入規則

| Skill | 載入時機 |
|-------|---------|
| **principles** | 總是 — 檢查命名、型別、SOLID、方法長度 |
| **testing** | 總是 — 檢查 AAA 結構、覆蓋率、測試壞味道 |
| **three-tier-architecture** | 總是 — 檢查架構邊界 |

## 步驟 1：收集變更範圍

審查目標：$ARGUMENTS

根據參數決定審查內容：

- `--staged` → `git diff --staged`
- `path` → `git diff HEAD -- {path}`
- *(空)* → `git show HEAD`

**完整閱讀變更的檔案**（不只是 diff），理解上下文。

## 步驟 2：載入 Skills

三個 skill 全部載入。

## 步驟 3：風格一致性

比較變更程式碼與周圍未變更的程式碼。

- 命名是否遵循慣例？（PascalCase for public, _camelCase for private fields）
- using 排序是否正確？（System → Microsoft → 第三方 → 專案）
- 方法長度是否在限制內？（建議 ≤ 40 行）
- 是否引入 `object` / `dynamic` 型別或 magic number？
- 是否使用 record / readonly record struct 應該用的地方卻用了 class？

標記不一致之處，不標記個人偏好。

## 步驟 4：測試品質

評估新增或修改的測試。

- 測試是否遵循 AAA 結構（Arrange / Act / Assert）？
- 每個測試是否只有一個邏輯斷言？
- 測試名稱是否可讀（`方法名_情境_預期結果`）？
- 有無測試壞味道？（mock 過多、測試鏡像實作、缺少邊界案例）
- 新行為是否有整合測試？

## 步驟 5：四層架構檢查

掃描變更程式碼中的架構違規。

- **Controller 是否直接存取 DbContext / Repository？**（應透過 Service）
- **Controller 是否直接用 UserManager/SignInManager？**（應透過 IAuthService）
- **Service 是否直接回傳 Entity？**（應轉換為 ViewModel via Mapper）
- **依賴方向是否正確？** Web → Service → Data → Library
- **商業邏輯是否洩漏到 Controller 或 Repository？**
- **UnitOfWork 是否正確管理交易？**（Repository 不呼叫 SaveChanges）
- **新增 Repository 是否同步更新 IUnitOfWork + UnitOfWork？**

## 步驟 6：日誌檢查

- **每個 Service 方法是否有入口和出口日誌？**
- **是否使用 `{PropertyName}` 佔位符？**（禁止 `$""` 字串插值）
- **是否記錄 PII？**（Email、密碼、姓名 → 改用 UserId）
- **日誌等級是否正確？** Info/Warning/Error
- **Data / Service 層是否有 `.ConfigureAwait(false)`？**

## 步驟 7：輸出 — 分類問題清單

以三個類別報告結果：

**HIGH（高）** — 正確性或安全性問題；必須在合併前修復。
- 範例：邏輯錯誤、缺少 null 防護、安全漏洞、型別錯誤、架構邊界違規、UnitOfWork 未 commit

**MEDIUM（中）** — 程式碼品質問題；應盡快修復。
- 範例：命名違規、缺少測試、方法過長、magic number、Entity 洩漏到 Presentation Layer

**LOW（低）** — 風格建議；方便時修復。
- 範例：using 順序、小幅命名改善、註解清晰度

格式：
```
[HIGH] Controllers/OrderController.cs:42 — Controller 直接注入 IOrderRepository，違反三層架構
[MED]  Services/OrderService.cs:87 — 方法超過 40 行，建議拆分
[LOW]  ViewModels/OrderViewModel.cs:12 — 屬性排序可改善
```

如果某個類別無問題，省略該區段。

---

下一步：如果 MEDIUM 問題揭示結構性問題，執行 `/refactor`。
