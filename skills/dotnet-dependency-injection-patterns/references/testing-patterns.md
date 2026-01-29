# Testing Patterns

The main advantage of extension methods: **reuse production configuration in tests**.

## WebApplicationFactory

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public ApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Production services already registered via Add* methods
                // Only override what's different for testing

                // Replace email sender with test double
                services.RemoveAll<IEmailSender>();
                services.AddSingleton<IEmailSender, TestEmailSender>();

                // Replace external payment processor
                services.RemoveAll<IPaymentProcessor>();
                services.AddSingleton<IPaymentProcessor, FakePaymentProcessor>();
            });
        });
    }

    [Fact]
    public async Task CreateOrder_SendsConfirmationEmail()
    {
        var client = _factory.CreateClient();
        var emailSender = _factory.Services.GetRequiredService<IEmailSender>() as TestEmailSender;

        await client.PostAsJsonAsync("/api/orders", new CreateOrderRequest(...));

        Assert.Single(emailSender!.SentEmails);
    }
}
```

## Standalone Unit Tests

```csharp
public class UserServiceTests
{
    private readonly ServiceProvider _provider;

    public UserServiceTests()
    {
        var services = new ServiceCollection();

        // Reuse production registrations
        services.AddUserServices();

        // Add test infrastructure
        services.AddSingleton<IUserRepository, InMemoryUserRepository>();

        _provider = services.BuildServiceProvider();
    }

    [Fact]
    public async Task CreateUser_ValidData_Succeeds()
    {
        var service = _provider.GetRequiredService<IUserService>();
        var result = await service.CreateUserAsync(new CreateUserRequest(...));

        Assert.True(result.IsSuccess);
    }
}
```

## Testing with TestContainers

```csharp
public class IntegrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();
    private ServiceProvider _provider;

    public async Task InitializeAsync()
    {
        await _sqlContainer.StartAsync();

        var services = new ServiceCollection();

        // Reuse production service registrations
        services.AddOrderServices();
        services.AddEmailServices();

        // Add real database
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(_sqlContainer.GetConnectionString()));

        // Override external dependencies
        services.AddSingleton<IPaymentProcessor, FakePaymentProcessor>();

        _provider = services.BuildServiceProvider();

        // Run migrations
        using var scope = _provider.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _provider.DisposeAsync();
        await _sqlContainer.DisposeAsync();
    }

    [Fact]
    public async Task CreateOrder_PersistsToDatabase()
    {
        using var scope = _provider.CreateScope();
        var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();

        var result = await orderService.CreateOrderAsync(new CreateOrderRequest(...));

        result.IsSuccess.Should().BeTrue();
    }
}
```

## Benefits

- **Confidence**: Tests use production service registrations
- **Less Duplication**: Don't repeat service setup in every test
- **Refactoring Safe**: Changes to dependencies automatically flow to tests
- **Fast**: Only override what's different for testing
