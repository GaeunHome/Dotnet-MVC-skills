# dotnet-mvc-skills

一個 [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins)，針對 C# .NET 10 MVC 開發設計 — 包含四層架構（Web / Service / Data / Library）、UnitOfWork + IDbContextFactory、TDD 工作流程、SOLID 原則與結構化 Code Review。

## 安裝

```bash
git clone https://github.com/GaeunHome/Dotnet-MVC-skills.git
claude --plugin-dir ./Dotnet-MVC-skills
```

可用指令：`/write`、`/fix`、`/review`、`/refactor`、`/debug`、`/spec`、`/decision`

## Skills 總覽

### Workflow Skills（手動呼叫）

| Skill | 用途 |
|-------|------|
| `/write <功能>` | 以 TDD 方式實作功能 |
| `/fix <bug>` | 修復錯誤（診斷 → Red → Green → Refactor） |
| `/review [--staged \| path]` | 審查本地變更（風格、測試、架構） |
| `/refactor [path \| module]` | 安全重構，含 Code Smell 分析與 TDD 驗證 |
| `/debug <錯誤>` | 系統性根因調查，修復前先找出問題 |
| `/spec <功能>` | 定義介面契約（Given/When/Then + C# interface + invariants） |
| `/decision <A vs B>` | 技術決策框架（四維度評分 + Pre-Mortem + 退場方案） |

### Methodology Skills（自動載入，不會出現在 `/` 選單）

| Skill | 載入時機 |
|-------|---------|
| `principles` | 設計功能、架構決策、SOLID 違反時 |
| `testing` | 實作功能、修 bug、改變行為時 |
| `done` | 任何產出程式碼變更的工作流程結尾 |

### Domain Skills（依情境自動載入）

| Skill | 涵蓋範圍 |
|-------|----------|
| `three-tier-architecture` | 四層架構各層責任、依賴方向、檔案結構 |
| `unitofwork-dbcontextfactory` | UnitOfWork、IDbContextFactory、平行查詢、背景服務 |
| `efcore-patterns` | Entity Configuration、Migration、Query 最佳化、Global Query Filter |
| `database-performance` | 讀寫分離、N+1 防範、AsNoTracking、Row Limit |
| `csharp-coding-standards` | Record、Pattern Matching、Nullable Types、async/await |
| `concurrency` | async/await、Channel、Parallel.ForEachAsync |
| `di-patterns` | IServiceCollection 擴展方法、Lifetime 管理、Keyed Services |
| `configuration` | IOptions 模式、啟動時驗證、強型別設定 |

### 各指令載入的 Skills

| 指令 | `spec` | `principles` | `testing` | `debug` | `done` | `decision` |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| `/write` | Spec Gate¹ 觸發時 | 總是 | 總是 | — | 總是 | — |
| `/fix` | — | 設計問題時 | 總是 | 總是 | 總是 | — |
| `/review` | — | 總是 | 總是 | — | — | — |
| `/refactor` | — | SOLID 違反時 | 總是 | — | 總是 | — |
| `/decision` | — | — | — | — | — | 獨立使用 |

> ¹ **Spec Gate** — 寫程式碼前的三個問題：(1) 這是 bug fix 或介面不變的修改？(2) C# interface 已存在？(3) 能立刻列出 3 個以上 boundary cases？全 YES → 跳過 spec 直接 TDD，任一 NO → 先載入 `spec`。

## 運作原理

Workflow Skills 提供步驟化流程，Methodology 和 Domain Skills 提供知識。

- **Workflow Skills** — 有編號步驟和確認關卡，Claude 在你核准計畫前不會寫程式碼
- **Methodology Skills** — 由 Workflow 根據情境自動載入（例如 `/fix` 總是載入 `testing`，根因涉及設計問題時加載 `principles`）
- **Domain Skills** — .NET MVC 專屬知識（架構、EF Core、UnitOfWork、DI），任務涉及時自動載入
- Workflow Skills 設定 `disable-model-invocation: true`，不會意外自動觸發

## 架構

強制執行四層架構：

```
{Project}.Web/          → Controllers, Views, Middleware
{Project}.Service/      → 商業邏輯, ViewModels, Mapper
{Project}.Data/         → Entities, Repositories, UnitOfWork, EF Core
{Project}.Library/      → 共用工具, Enums, Exceptions（無框架相依）
```

依賴方向：`Web → Service → Data → Library`

關鍵模式：UnitOfWork、IDbContextFactory、ServiceResult\<T\>、軟刪除（Soft Delete）、IAuthService

## 工作流程範例

```
/write "新增使用者驗證"
  → Spec Gate（介面已定義？邊界案例清楚？）
      全 YES → 計畫 → 確認 → TDD 循環 → /review
      任一 NO → /spec（介面 + 不變量）→ TDD 循環 → /review

/fix "登入頁面當機"      →  診斷  →  確認  →  Red/Green/Refactor  →  /review
/refactor src/auth/      →  Code Smell 分析  →  確認  →  逐步重構  →  /review
/decision "Dapper vs EF Core"  →  假設審查  →  四維度評分  →  Pre-Mortem  →  建議
```

## Agents

Agent 是擁有深度領域專業的 AI 人格，Claude 偵測到相關任務時自動啟用，不需手動呼叫。

| Agent | 專業領域 |
|-------|---------|
| **efcore-migration-specialist** | Migration 衝突、Schema 演進、Fluent API 除錯、Global Query Filter、Seed Data |
| **mvc-runtime-troubleshooter** | DI Lifetime 問題（Scoped-in-Singleton、ObjectDisposedException）、async/await Deadlock、DbContext 執行緒安全、CancellationToken 傳遞、IDbContextFactory 平行查詢、BackgroundService Scope 管理 |

## 建議的 CLAUDE.md 設定

在你的專案根目錄 `CLAUDE.md` 加入以下片段，讓 Claude 自動路由到對應 Skill：

```markdown
# Agent Guidance: dotnet-mvc-skills

Skill 路由
- C# 品質：csharp-coding-standards, concurrency
- 架構：three-tier-architecture
- 資料存取：efcore-patterns, database-performance, unitofwork-dbcontextfactory
- DI / 設定：di-patterns, configuration
- 品質關卡：每次完成功能後執行 /review

Agents（自動啟用）
- efcore-migration-specialist, mvc-runtime-troubleshooter
```

## 致謝

本 plugin 的構想與模式參考自以下專案：

- **[bouob/coding-skills](https://github.com/bouob/coding-skills)** — TDD 工作流程架構、Spec-Driven Design、雙層 Skill 架構（Workflow + Methodology）
- **[Aaronontheweb/dotnet-skills](https://github.com/Aaronontheweb/dotnet-skills)** — .NET 生態系 Skill 模式，包含 EF Core、C# Coding Standards、Database Performance、Concurrency、DI/Configuration 等最佳實踐

## 授權

MIT
