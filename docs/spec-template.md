# [{Scope}] {Feature 名稱}

> **狀態**：🔲 待實作 / 🚧 實作中 / ✅ 完成 / ❌ 封鎖中
> **建立時間**：YYYY-MM-DD
> **完成時間**：－

## 功能描述
<!-- 一句話說明這個功能做什麼 -->

## 使用者故事
身為 [角色]，我希望 [做什麼]，以便 [目的]

## 驗收條件
- [ ] ...
- [ ] ...
- [ ] ...

## Entity 設計

<!-- 列出新增或修改的 Entity 欄位，Claude 才不用猜 -->

```
Entity: Wishlist
├── Id              int         PK
├── UserId          string      FK → AppUser（必填）
├── TripId          int         FK → Trip（必填）
├── CreatedAt       DateTime    UTC
├── IsDeleted       bool        軟刪除
└── DeletedAt       DateTime?   軟刪除時間

Fluent API:
├── HasQueryFilter(w => !w.IsDeleted)
├── HasIndex(w => new { w.UserId, w.TripId }).IsUnique()  ← 同一使用者不能重複收藏
└── HasOne(w => w.Trip).WithMany().OnDelete(Restrict)
```

## 影響範圍

| 層 | 異動項目 | 新增/修改 |
|----|----------|----------|
| Data | Wishlist Entity + Configuration | 新增 |
| Data | Migration: AddWishlistTable | 新增 |
| Data | IWishlistRepository + WishlistRepository | 新增 |
| Data | IUnitOfWork + UnitOfWork（加入 WishlistRepository） | 修改 |
| Service | WishlistCardViewModel | 新增 |
| Service | MapsterConfig（Wishlist → WishlistCardViewModel） | 修改 |
| Service | IWishlistService + WishlistService | 新增 |
| Service | DI: AddApplicationServices() | 修改 |
| Web | WishlistController（Index, Toggle） | 新增 |
| Web | Views/Wishlist/Index.cshtml | 新增 |
| Web | _TripCard.cshtml（加愛心按鈕） | 修改 |

## API / 路由

| Method | 路由 | 說明 | 授權 |
|--------|------|------|------|
| GET | /Wishlist | 我的收藏列表 | [Authorize] |
| POST | /Wishlist/Toggle | 收藏/取消收藏 | [Authorize] |

## 邊界案例

<!-- 列出 Claude 容易忽略的例外情況 -->

- 未登入使用者點擊收藏 → 導向登入頁
- 同一行程重複收藏 → Toggle 取消（不是新增第二筆）
- 行程被軟刪除後 → 收藏列表不顯示（Global Query Filter 自動處理）
- 大量收藏分頁 → 每頁 12 筆

## 日誌要求

| 操作 | 等級 | 訊息範例 |
|------|------|---------|
| 新增收藏 | Info | 使用者收藏行程：{UserId}, {TripId} |
| 取消收藏 | Info | 使用者取消收藏：{UserId}, {TripId} |
| 行程不存在 | Warning | 收藏失敗，行程不存在：{TripId} |

## 測試計畫

### 單元測試
- [ ] Toggle_新增收藏_成功
- [ ] Toggle_取消收藏_成功
- [ ] Toggle_行程不存在_回傳失敗

### HTTP 驗證
- [ ] GET /Wishlist → 200（登入後）
- [ ] GET /Wishlist → 302 → /Account/Login（未登入）
- [ ] POST /Wishlist/Toggle → 200（登入後）

## 實際產出
<!-- 實作完成後填寫 -->
- Commit：
- Migration：
- 備註：
