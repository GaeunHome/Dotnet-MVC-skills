# dotnet-mvc-skills

[English](./README.md)

一個 [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins)，提供具有主觀設定的 C# .NET 10 MVC 開發流程 — 四層架構（Web / Service / Data / Library）、UnitOfWork + IDbContextFactory、TDD 工作流程、SOLID 原則與結構化程式碼審查。

## 安裝

```bash
# 從 marketplace 新增並安裝（推薦）
/plugin marketplace add {your-github-username}/dotnet-mvc-skills
/plugin install dotnet-mvc-skills

# 或在開發時直接載入
claude --plugin-dir ./dotnet-mvc-skills
```

Skills：`/write`、`/fix`、`/review`、`/refactor`、`/debug`、`/spec`、`/decision`

## Skills

### 工作流程 Skills（手動呼叫）

| Skill | 用途 |
|-------|------|
| `/write <功能描述>` | 以 TDD 方式實作功能 |
| `/fix <錯誤描述>` | 修復錯誤（診斷 → Red → Green → Refactor） |
| `/review [--staged \| path]` | 審查本地變更（風格、測試、架構） |
| `/refactor [path \| module]` | 安全重構，含壞味道分析與 TDD 驗證 |
| `/debug <錯誤描述>` | 系統性根因調查，在修復前先找出問題 |
| `/spec <功能描述>` | 定義介面契約（Given/When/Then + C# interface + 不變量） |
| `/decision <A vs B>` | AI 時代技術決策框架（四維度評分 + 預想失敗分析 + 退場方案） |

### 方法論 Skills（由工作流程自動載入）

不會出現在 `/` 選單，Claude 在需要時自動載入。

| Skill | 自動載入時機 |
|-------|------------|
| `principles` | 設計新功能、架構決策、SOLID 違反 |
| `testing` | 實作功能、修復錯誤、改變行為 |
| `done` | 任何產出程式碼變更的工作流程結尾 |

### 領域 Skills（依情境自動載入）

專業 .NET MVC 知識，在相關任務時自動載入。

| Skill | 涵蓋範圍 |
|-------|----------|
| `three-tier-architecture` | 四層架構的各層責任、依賴方向、檔案結構 |
| `unitofwork-dbcontextfactory` | UnitOfWork 模式、IDbContextFactory、平行查詢、背景服務 |
| `efcore-patterns` | Entity Configuration、Migration、查詢最佳化、Global Query Filter |
| `database-performance` | 讀寫分離、避免 N+1、AsNoTracking、Row Limit |
| `csharp-coding-standards` | Record、Pattern Matching、Nullable Types、async/await 最佳實踐 |
| `concurrency` | async/await、Channel、Parallel.ForEachAsync、選擇正確的並行抽象 |
| `di-patterns` | IServiceCollection 擴展方法、Lifetime 管理、Keyed Services |
| `configuration` | IOptions 模式、啟動時驗證、強型別設定 |

### 各指令載入的 Skills

| 指令 | `spec` | `principles` | `testing` | `debug` | `done` | `decision` |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| `/write` | Spec Gate 觸發時¹ | 總是 | 總是 | — | 總是 | — |
| `/fix` | — | 設計問題時 | 總是 | 總是 | 總是 | — |
| `/review` | — | 總是 | 總是 | — | — | — |
| `/refactor` | — | SOLID 違反時 | 總是 | — | 總是 | — |
| `/decision` | — | — | — | — | — | 獨立使用 |

> ¹ **Spec Gate** — 寫程式碼前的三個問題：(1) 這是 bug fix 或介面不變的修改嗎？(2) C# interface 已存在嗎？(3) 現在就能列出 3 個以上的邊界案例嗎？三問全 YES → 跳過 spec 直接 TDD。任一 NO → 先載入 `spec`。

## 運作原理

**兩層架構** — 工作流程 Skills 提供步驟化流程，方法論與領域 Skills 提供知識。

- **工作流程 Skills** 使用編號步驟加上明確確認關卡 — Claude 在你核准計畫前不會撰寫任何程式碼
- **方法論 Skills** 由工作流程根據情境自動載入（例如 `/fix` 總是載入 `testing`，若根因涉及設計問題則額外載入 `principles`）
- **領域 Skills** 提供 .NET MVC 專屬模式（架構、EF Core、UnitOfWork、DI），在任務涉及這些領域時自動載入
- 工作流程 Skills 皆設定 `disable-model-invocation: true` — 不會意外自動觸發

## 架構

此 plugin 強制執行四層架構：

```
{Project}.Web/          → Controllers, Views, Middleware
{Project}.Service/      → 商業邏輯, ViewModels, Mapper
{Project}.Data/         → Entities, Repositories, UnitOfWork, EF Core
{Project}.Library/      → 共用工具, 列舉, 例外（無框架相依）
```

**依賴方向：** `Web → Service → Data → Library`

關鍵模式：UnitOfWork、IDbContextFactory、ServiceResult\<T\>、軟刪除、IAuthService。

## 工作流程

```
/write "新增使用者驗證"
  → Spec Gate（介面已定義？邊界案例清楚？）
      全 YES → 計畫 → 確認 → TDD 循環 → /review
      任一 NO → /spec（介面 + 不變量）→ TDD 循環 → /review

/fix "登入頁面當機"      →  診斷  →  確認  →  Red/Green/Refactor  →  /review
/refactor src/auth/      →  壞味道分析  →  確認  →  逐步重構  →  /review
/decision "Dapper vs EF Core"  →  假設審查  →  四維度評分  →  預想失敗分析  →  建議
```

## 專業 Agents

Agent 是擁有深度領域專業的 AI 人格。當 Claude 偵測到相關任務時會自動啟用。

| Agent | 專業 |
|-------|------|
| **efcore-migration-specialist** | Migration 衝突、Schema 演進、Fluent API 除錯、Global Query Filter 問題、種子資料 |
| **mvc-runtime-troubleshooter** | DI 生命週期錯誤（Scoped-in-Singleton、ObjectDisposedException）、async/await 死結、DbContext 執行緒安全、CancellationToken 傳遞、IDbContextFactory 平行查詢、BackgroundService scope 管理 |

## 建議的 CLAUDE.md 片段

將以下內容加入專案的 `CLAUDE.md`，確保 Skill 路由一致：

```markdown
# Agent Guidance: dotnet-mvc-skills

路由（依名稱呼叫）
- C# 品質：csharp-coding-standards, concurrency
- 架構：three-tier-architecture
- 資料存取：efcore-patterns, database-performance, unitofwork-dbcontextfactory
- DI / 設定：di-patterns, configuration
- 品質關卡：每次完成功能後執行 /review

專業 agents（自動啟用）
- efcore-migration-specialist, mvc-runtime-troubleshooter
```

## 致謝

本 plugin 的構想與模式來自以下專案：

- **[bouob/coding-skills](https://github.com/bouob/coding-skills)** — TDD 工作流程架構、規格驅動設計、以及雙層 Skill 架構（工作流程 + 方法論）
- **[Aaronontheweb/dotnet-skills](https://github.com/Aaronontheweb/dotnet-skills)** — .NET 生態系 Skill 模式，包含 EF Core、C# 編碼標準、資料庫效能、並行處理、DI/設定等最佳實踐

## 授權

MIT
