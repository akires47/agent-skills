# Basic Options Binding

## Define a Settings Class

```csharp
public class SmtpSettings
{
    public const string SectionName = "Smtp";

    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string? Username { get; set; }
    public string? Password { get; set; }
    public bool UseSsl { get; set; } = true;
}
```

## Bind from Configuration

```csharp
// In Program.cs or service registration
builder.Services.AddOptions<SmtpSettings>()
    .BindConfiguration(SmtpSettings.SectionName);
```

## appsettings.json

```json
{
  "Smtp": {
    "Host": "smtp.example.com",
    "Port": 587,
    "Username": "user@example.com",
    "Password": "secret",
    "UseSsl": true
  }
}
```

## Consume in Services

```csharp
public class EmailService
{
    private readonly SmtpSettings _settings;

    // IOptions<T> - singleton, read once at startup
    public EmailService(IOptions<SmtpSettings> options)
    {
        _settings = options.Value;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        using var client = new SmtpClient(_settings.Host, _settings.Port)
        {
            EnableSsl = _settings.UseSsl
        };

        if (!string.IsNullOrEmpty(_settings.Username))
        {
            client.Credentials = new NetworkCredential(
                _settings.Username,
                _settings.Password);
        }

        await client.SendMailAsync(new MailMessage
        {
            From = new MailAddress(_settings.Username),
            To = { new MailAddress(to) },
            Subject = subject,
            Body = body
        });
    }
}
```

## Post-Configuration

Modify options after binding but before validation:

```csharp
builder.Services.AddOptions<ApiSettings>()
    .BindConfiguration("Api")
    .PostConfigure(options =>
    {
        // Ensure BaseUrl ends with /
        if (!string.IsNullOrEmpty(options.BaseUrl) && !options.BaseUrl.EndsWith('/'))
        {
            options.BaseUrl += '/';
        }

        // Set defaults
        options.Timeout ??= TimeSpan.FromSeconds(30);
    })
    .ValidateOnStart();
```

## PostConfigure with Dependencies

```csharp
builder.Services.AddOptions<ApiSettings>()
    .BindConfiguration("Api")
    .PostConfigure<IHostEnvironment>((options, env) =>
    {
        if (env.IsDevelopment())
        {
            options.Timeout = TimeSpan.FromMinutes(5); // Longer for debugging
        }
    });
```
