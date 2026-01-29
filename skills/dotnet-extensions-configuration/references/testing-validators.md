# Testing Configuration Validators

## Basic Validator Test

```csharp
public class SmtpSettingsValidatorTests
{
    private readonly SmtpSettingsValidator _validator = new();

    [Fact]
    public void Validate_WithValidSettings_ReturnsSuccess()
    {
        var settings = new SmtpSettings
        {
            Host = "smtp.example.com",
            Port = 587,
            Username = "user@example.com",
            Password = "secret"
        };

        var result = _validator.Validate(null, settings);

        result.Succeeded.Should().BeTrue();
    }

    [Fact]
    public void Validate_WithMissingHost_ReturnsFail()
    {
        var settings = new SmtpSettings { Host = "" };

        var result = _validator.Validate(null, settings);

        result.Succeeded.Should().BeFalse();
        result.FailureMessage.Should().Contain("Host is required");
    }

    [Fact]
    public void Validate_WithUsernameButNoPassword_ReturnsFail()
    {
        var settings = new SmtpSettings
        {
            Host = "smtp.example.com",
            Username = "user@example.com",
            Password = null  // Missing!
        };

        var result = _validator.Validate(null, settings);

        result.Succeeded.Should().BeFalse();
        result.FailureMessage.Should().Contain("Password is required");
    }
}
```

## Testing Validators with Dependencies

```csharp
public class DatabaseSettingsValidatorTests
{
    [Fact]
    public void Validate_ProductionWithLocalhost_Fails()
    {
        // Arrange
        var environment = new Mock<IHostEnvironment>();
        environment.Setup(e => e.EnvironmentName).Returns("Production");

        var validator = new DatabaseSettingsValidator(environment.Object);

        var settings = new DatabaseSettings
        {
            ConnectionString = "Server=localhost;Database=MyDb;..."
        };

        // Act
        var result = validator.Validate(null, settings);

        // Assert
        result.Succeeded.Should().BeFalse();
        result.FailureMessage.Should().Contain("Production cannot use localhost");
    }

    [Fact]
    public void Validate_DevelopmentWithLocalhost_Succeeds()
    {
        // Arrange
        var environment = new Mock<IHostEnvironment>();
        environment.Setup(e => e.EnvironmentName).Returns("Development");

        var validator = new DatabaseSettingsValidator(environment.Object);

        var settings = new DatabaseSettings
        {
            ConnectionString = "Server=localhost;Database=MyDb;..."
        };

        // Act
        var result = validator.Validate(null, settings);

        // Assert
        result.Succeeded.Should().BeTrue();
    }
}
```

## Testing Named Options Validators

```csharp
[Theory]
[InlineData("Primary", false, true)]   // Primary must be writable
[InlineData("Replica", true, true)]    // Replica should be read-only
[InlineData("Replica", false, false)]  // Replica writable fails
public void Validate_NamedOptions_ValidatesCorrectly(
    string name,
    bool isReadOnly,
    bool shouldSucceed)
{
    var settings = new DatabaseSettings
    {
        ConnectionString = "Server=sql;...",
        ReadOnly = isReadOnly
    };

    var validator = new DatabaseSettingsValidator();
    var result = validator.Validate(name, settings);

    result.Succeeded.Should().Be(shouldSucceed);
}
```

## Integration Test: Validation on Startup

```csharp
public class ConfigurationValidationTests
{
    [Fact]
    public void InvalidConfiguration_FailsOnStartup()
    {
        // Arrange
        var configuration = new ConfigurationBuilder()
            .AddInMemoryCollection(new Dictionary<string, string>
            {
                ["Smtp:Host"] = "",  // Invalid - empty host
                ["Smtp:Port"] = "587"
            })
            .Build();

        var services = new ServiceCollection();
        services.AddSingleton<IConfiguration>(configuration);
        services.AddOptions<SmtpSettings>()
            .BindConfiguration("Smtp")
            .ValidateDataAnnotations()
            .ValidateOnStart();
        services.AddSingleton<IValidateOptions<SmtpSettings>, SmtpSettingsValidator>();

        // Act & Assert
        var act = () => services.BuildServiceProvider(validateScopes: true);

        act.Should().Throw<OptionsValidationException>()
            .WithMessage("*Host is required*");
    }
}
```
