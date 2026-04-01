---
name: write
description: '以 TDD 方式實作功能。當使用者要求「寫一個功能」、「實作」、「新增功能」、「建立」或描述新功能時使用此 skill。'
argument-hint: [功能描述]
disable-model-invocation: true
allowed-tools: Read, Edit, Write, Grep, Glob,
  Bash(git log:*), Bash(git diff:*), Bash(git show:*),
  Bash(dotnet test:*), Bash(dotnet build:*), Bash(dotnet run:*),
  Skill
---

# /write — 功能實作工作流程

## Skills 載入規則

| Skill | 載入時機 |
|-------|---------|
| **spec** | 當 Spec Gate 回答 NO 時（見步驟 2） |
| **principles** | 總是 — 每個功能都涉及設計決策 |
| **testing** | 總是 — 新行為必須有測試 |
| **done** | 總是 — 完成前執行驗證 |
| **three-tier-architecture** | 總是 — 確保新程式碼放在正確的層 |

## 步驟 1：理解上下文

功能需求：$ARGUMENTS

在寫任何程式碼前，先閱讀相關檔案。

- 確認哪些檔案 / 模組會受影響
- 檢查相鄰程式碼的現有模式（`git log --oneline -10` 了解最近變更）
- 確認此功能屬於三層架構的哪一層或哪幾層
- **不要閱讀框架原始碼** — 只看專案程式碼

## 步驟 2：Spec Gate — 先 TDD 還是先定義介面？

在載入 skill 或撰寫計畫前，回答以下三個問題：

| 問題 | 如果 YES |
|------|---------|
| 這是 bug fix 或不影響公開介面的內部修改？ | 跳過 spec → 步驟 3 |
| 此功能的 C# interface / method signature 已經存在？ | 跳過 spec → 步驟 3 |
| 你現在可以立刻說出 3 個以上的邊界案例？ | 跳過 spec → 步驟 3 |

**如果任一問題答 NO → 現在載入 `spec` skill。完成介面契約定義後再繼續。**

## 步驟 3：載入 Skills

套用上方的載入規則。`principles`、`testing`、`three-tier-architecture` 總是載入。

## 步驟 4：計畫模式 — TDD 任務拆解

進入計畫模式。將功能拆成最小有意義的任務。

- 最小變更原則：優先擴展而非修改現有程式碼
- 每個任務應可在一個 Red/Green/Refactor 循環內完成
- 明確標示：
  - 哪些在 **Data Access Layer**（Entity、Repository、UnitOfWork）
  - 哪些在 **Business Logic Layer**（Service、DTO、Validator）
  - 哪些在 **Presentation Layer**（Controller、ViewModel、View）
- 列出任何需要使用者確認的問題或假設

**等待使用者確認後才開始寫程式碼。**

## 步驟 5：實作（每個任務走 Red → Green → Refactor）

每個任務：

**Red** — 寫一個描述預期行為的失敗測試。
- 確認：「測試失敗，因為 [原因]。」
- 使用 `dotnet test` 執行

**Green** — 寫最少的程式碼讓測試通過。
- 確認：「所有測試通過。」

**Refactor** — 在不改變行為的情況下清理程式碼。
- 確認：「測試仍然通過。」

## 步驟 6：完成檢查

驗證所有 skill 的規則：

- **principles 規則** — 命名、型別、方法長度、SOLID
- **testing 規則** — AAA 結構、單一斷言、確定性、整合測試路徑
- **three-tier-architecture 規則** — 依賴方向正確、無跨層違規

---

下一步：執行 `/review` 確認風格一致性和測試品質。
