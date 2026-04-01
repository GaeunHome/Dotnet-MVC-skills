# CLAUDE.md — C# .NET 10 MVC 通用專案指引

> 此檔案為**通用範本**。複製到新專案根目錄後，補上「專案專屬」的區段（Entity 清單、路由表、Area 定義、任務清單）。

## 技術棧

- C# / .NET 10 / ASP.NET Core MVC
- Entity Framework Core + SQL Server
- UnitOfWork + IDbContextFactory
- Serilog 結構化日誌
- Mapster 或 AutoMapper（依專案決定）
- ASP.NET Core Identity

## 架構原則

### 四層架構（嚴格遵守）

```
{ProjectName}.Web/              # 展示層
    ├── Controllers/            #   前台 Controller
    ├── Areas/Admin/Controllers/#   後台 Controller
    ├── Views/                  #   Razor Views + Partials
    ├── Infrastructure/         #   Middleware, Extensions, ViewComponent, BackgroundService
    └── wwwroot/                #   CSS, JS, images

{ProjectName}.Service/          # 商業邏輯層
    ├── Services/               #   Interfaces/ + Implementation/（含 AuthService）
    ├── ViewModels/             #   按功能分目錄
    ├── Mapper/                 #   AutoMapper Profile 或 Mapster IRegister
    └── Constants/              #   CacheKeys, DisplayConstants

{ProjectName}.Data/             # 資料存取層
    ├── Entities/               #   Entity + Fluent API Configuration（同檔）
    ├── Repositories/           #   Interfaces/ + Implementation/
    ├── UnitOfWork/             #   IUnitOfWork + UnitOfWork
    ├── ApplicationDbContext.cs #   軟刪除攔截 + Global Query Filter
    ├── DbInitializer.cs        #   種子資料
    └── Migrations/

{ProjectName}.Library/          # 共用工具層（無框架相依）
    ├── Helpers/                #   ServiceResult<T>, PagedResult<T>
    ├── Enums/                  #   OrderStatus 等共用列舉
    └── Exceptions/             #   NotFoundException, BusinessException
```

**依賴方向：**
```
Web → Service → Data → Library（所有層都可引用 Library，Library 不引用其他層）
```

### 關鍵模式

- **UnitOfWork** — `IUnitOfWork` 暴露所有 Repository 屬性 + `SaveChangesAsync`
- **IDbContextFactory** — 平行查詢、背景服務、多工場景用 `IDbContextFactory<ApplicationDbContext>`
- **ServiceResult<T>** — Service 統一回傳 `ServiceResult<T>`（Success / Failure），不對預期錯誤 throw Exception
- **軟刪除** — `ISoftDeletable` 介面 + DbContext 自動攔截 + Global Query Filter
- **IAuthService** — 封裝所有 Identity 操作，Web 層不直接碰 UserManager/SignInManager

## Skills 路由

- **C# 品質：** csharp-coding-standards, concurrency
- **架構：** three-tier-architecture
- **資料存取：** efcore-patterns, database-performance, unitofwork-dbcontextfactory
- **DI / 設定：** di-patterns, configuration
- **品質關卡：** 每次完成功能後執行 `/review`

## 日誌規範（Serilog — 每個操作必須有日誌）

### 規則

- Service 注入 `ILogger<T>`，使用 `{PropertyName}` 結構化佔位符
- **每個 Service 方法的入口和出口都要有日誌**
- 日誌中**禁止記錄 PII**（Email、密碼、完整姓名）

### 日誌等級

| 等級 | 用途 | 範例 |
|------|------|------|
| `Debug` | 開發除錯 | 查詢參數、中間計算值 |
| `Information` | 業務事件 | 「訂單已建立：{OrderId}」、「使用者登入：{UserId}」 |
| `Warning` | 可恢復異常 | 「商品庫存不足：{ProductId}，剩餘 {Stock}」 |
| `Error` | 系統錯誤 | catch 區塊，附帶 Exception 物件 |

### 範例

```csharp
public async Task<ServiceResult<OrderDto>> CreateOrderAsync(
    CreateOrderViewModel model, CancellationToken ct = default)
{
    _logger.LogInformation("開始建立訂單，客戶：{CustomerId}，品項數：{ItemCount}",
        model.CustomerId, model.Items.Count);

    // ... 商業邏輯 ...

    if (product.Stock < item.Quantity)
    {
        _logger.LogWarning("庫存不足，商品：{ProductId}，需求：{Requested}，剩餘：{Available}",
            item.ProductId, item.Quantity, product.Stock);
        return ServiceResult<OrderDto>.Failure("庫存不足");
    }

    await _uow.SaveChangesAsync(ct).ConfigureAwait(false);

    _logger.LogInformation("訂單建立成功：{OrderId}，總金額：{Total}",
        order.Id, order.Total);

    return ServiceResult<OrderDto>.Success(mapper.Map<OrderDto>(order));
}
```

## 開發工作流程

### 新增功能標準流程

```
━━━ Spec ━━━━━━━━━━━━━━━━━━━━━━━━━
Step 0  docs/specs/{scope}-{feature}.md（狀態 🔲）
        git commit: docs({scope}): 新增 {feature} spec

━━━ 實作 ━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1  閱讀相關現有程式碼
Step 2  Entity + Fluent API Configuration（Data/Entities/）
Step 3  EF Migration
Step 4  Repository Interface + Implementation
Step 5  更新 IUnitOfWork + UnitOfWork
Step 6  ViewModel（Service/ViewModels/）
Step 7  Mapper 映射
Step 8  Service Interface + Implementation（含日誌）
Step 9  DI 註冊（Infrastructure/Extensions/）
Step 10 Controller + Views + CSS

━━━ 驗證 ━━━━━━━━━━━━━━━━━━━━━━━━━
Step 11 dotnet build（0 error 0 warning）
Step 12 dotnet test（如有測試專案）
Step 13 啟動 server → curl 驗證端點 → kill

━━━ 完成 ━━━━━━━━━━━━━━━━━━━━━━━━━
Step 14 更新 spec 狀態為 ✅
Step 15 git commit（type(scope): 繁中動詞開頭）
```

### 每次動作前自我檢查

- 每建立一個新檔案後，立即 `dotnet build` 確認可編譯
- 新增 Migration 前，先確認 Entity Configuration 無語法錯誤
- 後台管理放 `Areas/Admin`，前台功能放 `Controllers/`

## 禁止事項

| 禁止 | 原因 |
|------|------|
| Controller 直接用 DbContext / Repository | 必須透過 Service 層 |
| Entity 直接傳入 View | 必須透過 ViewModel + Mapper |
| Web 層直接用 UserManager/SignInManager | 必須透過 IAuthService |
| `.Result` / `.Wait()` | deadlock |
| Service 方法無日誌 | 無法追蹤問題 |
| 硬刪除 ISoftDeletable Entity | 必須軟刪除 |
| 刪除已套用的 Migration | 破壞資料庫狀態 |
| Magic Number | 使用常數（DisplayConstants 等）|

## C# 慣例

- .NET 10 語言特性：file-scoped namespace、primary constructor、collection expressions、pattern matching
- ViewModel 用 DataAnnotations 驗證（Required, StringLength, Range, Display）
- Entity 用 Fluent API（不用 DataAnnotations），Configuration 與 Entity 同檔
- 所有 async 方法接受 `CancellationToken ct = default`
- Data / Service 層加 `.ConfigureAwait(false)`
- POST 一律 `[ValidateAntiForgeryToken]`
- 後台一律 `[Area("Admin")]` + `[Authorize(Roles = "Admin")]`
- 使用 `TempData["Success"]` / `TempData["Error"]` 傳遞單次訊息

## 命名規範

| 項目 | 規則 | 範例 |
|------|------|------|
| Service | `I{Feature}Service` → `{Feature}Service` | `IOrderService` → `OrderService` |
| Repository | `I{Entity}Repository` → `{Entity}Repository` | `IOrderRepository` → `OrderRepository` |
| ViewModel | 功能 + 後綴 | `CreateOrderViewModel`, `OrderListViewModel` |
| Entity | 單數名詞 | `Order`（不是 `Orders`） |
| 測試方法 | `方法_情境_預期結果` | `CreateAsync_WhenTitleExists_ReturnsFailure` |
| Git Commit | `type(scope): 繁中動詞開頭` | `feat(order): 新增訂單分頁功能` |

## DI 註冊結構

```csharp
// Program.cs
builder.Services
    .AddDataServices(configuration)      // DbContext, Factory, UoW, Repositories
    .AddApplicationServices();           // Mapper, Services
```

---

*以下為「專案專屬」區段 — 新專案請在此補上：*

## 專案描述
<!-- 一句話描述 -->

## Entity 清單
<!-- 列出所有 Entity -->

## 路由表
<!-- 列出所有路由 -->

## Area 定義
<!-- 列出 Area 和對應角色 -->

## 任務清單
<!-- Phase 1, 2, 3... -->
