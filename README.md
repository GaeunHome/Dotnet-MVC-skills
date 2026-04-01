# dotnet-mvc-skills

針對 **C# .NET 10 MVC** 開發設計的 [Claude Code Plugin](https://docs.anthropic.com/en/docs/claude-code/plugins)。

涵蓋四層架構（Web / Service / Data / Library）、UnitOfWork + IDbContextFactory、TDD 工作流程、SOLID 原則與結構化 Code Review。

---

## Skills 總覽

本 Plugin 包含 **18 個 Skills**，分成三類：

### 1. Workflow Skills — 手動呼叫

透過 `/指令` 觸發，帶有步驟和確認關卡，Claude 在你核准計畫前不會寫程式碼。

| 指令 | 用途 |
|------|------|
| `/write <功能>` | 以 TDD 方式實作功能 |
| `/fix <bug>` | 修復錯誤（診斷 → Red → Green → Refactor） |
| `/review [--staged \| path]` | 審查本地變更（風格、測試、架構） |
| `/refactor [path \| module]` | 安全重構，含 Code Smell 分析與 TDD 驗證 |
| `/debug <錯誤>` | 系統性根因調查，修復前先找出問題 |
| `/spec <功能>` | 定義介面契約（Given/When/Then + C# Interface + Invariants） |
| `/decision <A vs B>` | 技術決策框架（四維度評分 + Pre-Mortem + 退場方案） |

### 2. Methodology Skills — 由 Workflow 自動載入

不會出現在 `/` 選單，Claude 在需要時自動載入。

| Skill | 載入時機 |
|-------|---------|
| `principles` | 設計功能、架構決策、違反 SOLID 時 |
| `testing` | 實作功能、修 Bug、改變行為時 |
| `done` | 任何產出程式碼變更的 Workflow 結尾 |

### 3. Domain Skills — 依情境自動載入

.NET MVC 專屬知識，任務涉及相關領域時自動載入。

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

### 各指令自動載入的 Skills 對照表

| 指令 | `spec` | `principles` | `testing` | `debug` | `done` | `decision` |
|------|:------:|:------------:|:---------:|:-------:|:------:|:----------:|
| `/write` | Spec Gate¹ | v | v | — | v | — |
| `/fix` | — | 設計問題時 | v | v | v | — |
| `/review` | — | v | v | — | — | — |
| `/refactor` | — | SOLID 違反時 | v | — | v | — |
| `/decision` | — | — | — | — | — | 獨立使用 |

> ¹ **Spec Gate** — 寫程式碼前先問三個問題：
> 1. 這是 Bug Fix 或介面不變的修改？
> 2. C# Interface 已存在？
> 3. 能立刻列出 3 個以上 Boundary Cases？
>
> 全部 YES → 跳過 Spec，直接進入 TDD。任一 NO → 先載入 `/spec`。

---

## 架構

強制執行四層架構，依賴方向為 `Web → Service → Data → Library`：

```
{Project}.Web/              Controllers, Views, Middleware
{Project}.Service/          商業邏輯, ViewModels, Mapper
{Project}.Data/             Entities, Repositories, UnitOfWork, EF Core
{Project}.Library/          共用工具, Enums, Exceptions（無框架相依）
```

關鍵模式：

- **UnitOfWork** — 統一管理 Repository + `SaveChangesAsync`
- **IDbContextFactory** — 平行查詢、背景服務、多工場景
- **ServiceResult\<T\>** — Service 層統一回傳，不對預期錯誤 throw Exception
- **Soft Delete** — `ISoftDeletable` 介面 + Global Query Filter
- **IAuthService** — 封裝 Identity 操作，Web 層不直接碰 UserManager

---

## 工作流程範例

```
/write "新增使用者驗證"
  → Spec Gate（介面已定義？邊界案例清楚？）
      全 YES → 計畫 → 確認 → TDD 循環 → /review
      任一 NO → /spec → TDD 循環 → /review

/fix "登入頁面當機"
  → 診斷 → 確認 → Red/Green/Refactor → /review

/refactor src/auth/
  → Code Smell 分析 → 確認 → 逐步重構 → /review

/decision "Dapper vs EF Core"
  → 假設審查 → 四維度評分 → Pre-Mortem → 建議
```

---

## Agents

Agent 是擁有深度領域專業的 AI 人格，Claude 偵測到相關問題時**自動啟用**，不需手動呼叫。

### efcore-migration-specialist

EF Core 資料庫 Schema 專家。

- Migration 衝突與 Snapshot 不同步
- Fluent API Configuration 除錯（Cascade Delete 循環、複合鍵、Value Conversion）
- Global Query Filter 陷阱（多重 Filter、與 Include 的交互）
- Seed Data 順序與 FK 衝突
- Schema 演進策略（欄位更名、型別轉換、大型資料表 Index）

### mvc-runtime-troubleshooter

MVC 執行時期問題專家，涵蓋 DI 與 Async 兩大領域。

**DI 相關：**
- `Cannot resolve scoped service from root provider`
- `ObjectDisposedException`（DbContext 在 Request 結束後被使用）
- Captive Dependency（Scoped 被 Singleton 捕獲）
- Circular Dependency

**Async 相關：**
- `.Result` / `.Wait()` 造成的 Deadlock
- DbContext 多執行緒存取（`A second operation was started on this context`）
- CancellationToken 未傳遞到 EF Core Query
- BackgroundService 的 Scope 管理
- IDbContextFactory 平行查詢模式

---

## 建議的 CLAUDE.md 設定

在專案根目錄 `CLAUDE.md` 加入以下片段，讓 Claude 自動路由到對應 Skill：

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

---

## 致謝

本 Plugin 的構想與模式參考自以下專案：

- **[bouob/coding-skills](https://github.com/bouob/coding-skills)** — TDD 工作流程架構、Spec-Driven Design、雙層 Skill 架構（Workflow + Methodology）
- **[Aaronontheweb/dotnet-skills](https://github.com/Aaronontheweb/dotnet-skills)** — .NET 生態系 Skill 模式，包含 EF Core、C# Coding Standards、Database Performance、Concurrency、DI/Configuration 等最佳實踐

## 授權

MIT
