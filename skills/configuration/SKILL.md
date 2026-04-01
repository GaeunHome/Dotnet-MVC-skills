---
name: configuration
description: 'Microsoft.Extensions.Configuration 模式。IOptions 模式、啟動時驗證、強型別設定、環境設定。'
user-invocable: false
---

# 設定管理模式

## 適用時機

- 從 appsettings.json 綁定設定到強型別類別
- 在啟動時驗證設定（fail fast）
- 設計可測試的設定類別

---

## 基本模式：強型別設定

```csharp
// 設定類別
public class SmtpSettings
{
    public const string SectionName = "Smtp";

    [Required(ErrorMessage = "SMTP Host 為必填")]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int Port { get; set; } = 587;

    public string? Username { get; set; }
    public string? Password { get; set; }
    public bool UseSsl { get; set; } = true;
}

// 註冊 + 驗證
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();  // 關鍵：啟動時驗證，而非第一次使用時

// 使用
public sealed class EmailService(IOptions<SmtpSettings> options)
{
    private readonly SmtpSettings _settings = options.Value;
}
```

---

## IOptions 生命週期

| 介面 | 生命週期 | 重新載入 | 適用場景 |
|------|---------|---------|---------|
| `IOptions<T>` | Singleton | 不會 | 靜態設定 |
| `IOptionsSnapshot<T>` | Scoped | 每個請求 | Web 應用需要新鮮設定 |
| `IOptionsMonitor<T>` | Singleton | 即時回呼 | BackgroundService |

---

## 反模式

```csharp
// ❌ 直接讀 IConfiguration — 無驗證、難測試
public class MyService(IConfiguration config)
{
    var host = config["Smtp:Host"];  // null 也不會報錯！
}

// ✅ 強型別 + 啟動驗證
public class MyService(IOptions<SmtpSettings> options)
{
    var host = options.Value.Host;  // 啟動時已驗證
}

// ❌ 忘記 ValidateOnStart
builder.Services.AddOptions<Settings>()
    .ValidateDataAnnotations();  // 只在第一次使用時驗證！

// ✅ 啟動時立即驗證
builder.Services.AddOptions<Settings>()
    .ValidateDataAnnotations()
    .ValidateOnStart();
```
