# Options Lifetime

Understanding the three options interfaces:

| Interface | Lifetime | Reloads on Change | Use Case |
|-----------|----------|-------------------|----------|
| `IOptions<T>` | Singleton | No | Static config, read once |
| `IOptionsSnapshot<T>` | Scoped | Yes (per request) | Web apps needing fresh config |
| `IOptionsMonitor<T>` | Singleton | Yes (with callback) | Background services, real-time updates |

## IOptions<T> - Singleton

Read configuration once at startup:

```csharp
public class EmailService
{
    private readonly SmtpSettings _settings;

    public EmailService(IOptions<SmtpSettings> options)
    {
        _settings = options.Value; // Read once
    }
}
```

**Use when:** Configuration doesn't change at runtime (most cases).

## IOptionsSnapshot<T> - Scoped

Read fresh configuration per request:

```csharp
public class DynamicService
{
    private readonly FeatureFlags _flags;

    public DynamicService(IOptionsSnapshot<FeatureFlags> options)
    {
        _flags = options.Value; // Reloaded per request
    }
}
```

**Use when:** Configuration can change and you need per-request updates (web apps).

## IOptionsMonitor<T> - Singleton with Change Notifications

Subscribe to configuration changes in real-time:

```csharp
public class BackgroundWorker : BackgroundService
{
    private readonly IOptionsMonitor<WorkerSettings> _optionsMonitor;
    private WorkerSettings _currentSettings;

    public BackgroundWorker(IOptionsMonitor<WorkerSettings> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
        _currentSettings = optionsMonitor.CurrentValue;

        // Subscribe to configuration changes
        _optionsMonitor.OnChange(settings =>
        {
            _currentSettings = settings;
            _logger.LogInformation("Worker settings updated: Interval={Interval}",
                settings.PollingInterval);
        });
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync();
            await Task.Delay(_currentSettings.PollingInterval, stoppingToken);
        }
    }
}
```

**Use when:** Background services that need to react to configuration changes without restart.

## Comparison Example

```csharp
// Singleton - same value for entire app lifetime
public class SingletonService(IOptions<Settings> options)
{
    private readonly Settings _settings = options.Value;
    // _settings never changes
}

// Scoped - fresh value per HTTP request
public class ScopedService(IOptionsSnapshot<Settings> options)
{
    private readonly Settings _settings = options.Value;
    // _settings refreshed per request
}

// Monitor - reactive to changes
public class MonitoredService
{
    private Settings _settings;

    public MonitoredService(IOptionsMonitor<Settings> monitor)
    {
        _settings = monitor.CurrentValue;
        monitor.OnChange(settings => _settings = settings);
        // _settings updates when config file changes
    }
}
```

## When to Use Which

```csharp
// IOptions<T> - 95% of cases
services.AddSingleton<EmailService>();  // Uses IOptions<SmtpSettings>

// IOptionsSnapshot<T> - Web apps with dynamic config
services.AddScoped<FeatureService>();   // Uses IOptionsSnapshot<FeatureFlags>

// IOptionsMonitor<T> - Background services needing updates
services.AddHostedService<WorkerService>();  // Uses IOptionsMonitor<WorkerSettings>
```
