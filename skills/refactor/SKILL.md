---
name: refactor
description: '安全重構現有程式碼。識別壞味道、安全變換、行為不可改變。當使用者要求「重構」、「清理程式碼」、「簡化」、「提取方法」、「降低複雜度」時使用。'
argument-hint: [path | module]
disable-model-invocation: true
allowed-tools: Read, Edit, Write, Grep, Glob,
  Bash(git log:*), Bash(git diff:*), Bash(git show:*),
  Bash(dotnet test:*), Bash(dotnet build:*),
  WebSearch, Skill
---

# /refactor — 安全重構工作流程

## Skills 載入規則

| Skill | 載入時機 |
|-------|---------|
| **testing** | 總是 — 重構前必須有測試覆蓋 |
| **principles** | 重構涉及 SOLID 違反、命名或依賴方向 |
| **three-tier-architecture** | 重構涉及跨層移動程式碼 |
| **done** | 總是 — 確認測試仍然通過且無回歸 |

## 步驟 1：分析壞味道

重構目標：$ARGUMENTS

閱讀目標路徑，掃描問題。

- 使用 `git log -- <path>` 查看近期變更歷史（排除最近已重構的區域）
- 按嚴重度分類：

| 壞味道 | 範例 | 嚴重度 |
|--------|------|--------|
| 過長方法 | 方法超過 40 行 | 高 |
| 過大類別 | 類別有太多責任 | 高 |
| 重複程式碼 | 同樣邏輯複製貼上 ≥ 3 處 | 中 |
| 死程式碼 | public 但從未被使用 | 中 |
| Magic Number | `if (status == 3)` 沒有命名常數 | 中 |
| 錯誤的層 | 商業邏輯在 Controller / Repository 中 | 高 |
| 循環依賴 | A 引用 B，B 引用 A | 高 |
| Entity 洩漏 | Entity 直接傳到 Controller/View | 高 |
| 缺少 UnitOfWork | Service 直接呼叫 SaveChanges 多次 | 中 |

## 步驟 2：載入 Skills

套用載入規則。**如果測試不存在就不要開始重構** — 先用 `/write` 補測試，再回來。

## 步驟 3：計畫模式 — 重構計畫

進入計畫模式。優先順序：高影響 + 低回歸風險。

每個壞味道：
- 確認最小變換（Extract Method、Rename、Move、Inline 等）
- 確認變換保持行為不變（無邏輯變更）
- 確認依賴方向在 Move 後仍然正確

**等待使用者確認後才開始寫程式碼。**

## 步驟 4：重構（一次一個壞味道）

每個項目：

1. 確認測試在變更前通過：「變更前測試全綠。」
2. 套用單一變換 — 不做其他事
3. 確認測試在變更後通過：「變更後測試全綠。」
4. 如果測試失敗，回退變換並調查

不要在一個步驟中合併多個變換。只做小的、可驗證的增量。

## 步驟 5：完成檢查

- **principles 規則** — 命名、方法長度、無 magic number
- **testing 規則** — 測試仍然通過、無測試壞味道、覆蓋率未降低
- **three-tier-architecture 規則** — 依賴方向正確

---

下一步：執行 `/review` 審查重構的 diff。
