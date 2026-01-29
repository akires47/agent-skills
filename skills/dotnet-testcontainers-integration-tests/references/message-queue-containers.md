# Message Queue Containers - RabbitMQ

## Basic RabbitMQ Integration Test

```csharp
using Testcontainers.RabbitMq;

public class RabbitMqTests : IAsyncLifetime
{
    private readonly RabbitMqContainer _rabbitMqContainer = new RabbitMqBuilder()
        .WithImage("rabbitmq:3-management-alpine")
        .Build();

    private IConnection _connection;
    private IModel _channel;

    public async Task InitializeAsync()
    {
        await _rabbitMqContainer.StartAsync();

        var factory = new ConnectionFactory
        {
            Uri = new Uri(_rabbitMqContainer.GetConnectionString())
        };

        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public async Task DisposeAsync()
    {
        _channel?.Close();
        _connection?.Close();
        await _rabbitMqContainer.DisposeAsync();
    }

    [Fact]
    public void CanPublishAndConsumeMessage()
    {
        // Arrange
        const string queueName = "test-queue";
        const string message = "Hello TestContainers!";

        _channel.QueueDeclare(
            queue: queueName,
            durable: false,
            exclusive: false,
            autoDelete: false);

        // Act - Publish
        var body = Encoding.UTF8.GetBytes(message);
        _channel.BasicPublish(
            exchange: "",
            routingKey: queueName,
            basicProperties: null,
            body: body);

        // Act - Consume
        var result = _channel.BasicGet(queueName, autoAck: true);

        // Assert
        result.Should().NotBeNull();
        var receivedMessage = Encoding.UTF8.GetString(result.Body.ToArray());
        receivedMessage.Should().Be(message);
    }
}
```

## Testing Message Handler

```csharp
public class OrderMessageHandlerTests : IAsyncLifetime
{
    private readonly RabbitMqContainer _rabbitMqContainer = new RabbitMqBuilder().Build();
    private IConnection _connection;
    private OrderMessageHandler _handler;

    public async Task InitializeAsync()
    {
        await _rabbitMqContainer.StartAsync();

        var factory = new ConnectionFactory
        {
            Uri = new Uri(_rabbitMqContainer.GetConnectionString())
        };

        _connection = factory.CreateConnection();
        _handler = new OrderMessageHandler(_connection);
    }

    public async Task DisposeAsync()
    {
        _connection?.Close();
        await _rabbitMqContainer.DisposeAsync();
    }

    [Fact]
    public async Task ProcessOrderMessage_UpdatesDatabase()
    {
        // Arrange
        var orderCreatedEvent = new OrderCreatedEvent
        {
            OrderId = "ORD-123",
            CustomerId = "CUST-456",
            Total = 99.99m
        };

        // Act
        await _handler.PublishAsync(orderCreatedEvent);
        await Task.Delay(500); // Allow processing time

        // Assert
        // Verify the message was processed
        var processedOrders = await _handler.GetProcessedOrdersAsync();
        processedOrders.Should().Contain(o => o.OrderId == "ORD-123");
    }
}
```

## Testing Publisher-Subscriber Pattern

```csharp
public class PubSubTests : IAsyncLifetime
{
    private readonly RabbitMqContainer _rabbitMqContainer = new RabbitMqBuilder().Build();
    private IConnection _connection;
    private IModel _channel;

    public async Task InitializeAsync()
    {
        await _rabbitMqContainer.StartAsync();

        var factory = new ConnectionFactory
        {
            Uri = new Uri(_rabbitMqContainer.GetConnectionString())
        };

        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public async Task DisposeAsync()
    {
        _channel?.Close();
        _connection?.Close();
        await _rabbitMqContainer.DisposeAsync();
    }

    [Fact]
    public void FanoutExchange_DeliveresToAllQueues()
    {
        // Arrange
        const string exchangeName = "test-fanout";
        const string queue1 = "subscriber1";
        const string queue2 = "subscriber2";

        _channel.ExchangeDeclare(exchangeName, ExchangeType.Fanout);
        _channel.QueueDeclare(queue1, false, false, false);
        _channel.QueueDeclare(queue2, false, false, false);
        _channel.QueueBind(queue1, exchangeName, "");
        _channel.QueueBind(queue2, exchangeName, "");

        // Act
        var message = "Broadcast message";
        var body = Encoding.UTF8.GetBytes(message);
        _channel.BasicPublish(exchangeName, "", null, body);

        // Assert
        var result1 = _channel.BasicGet(queue1, true);
        var result2 = _channel.BasicGet(queue2, true);

        Encoding.UTF8.GetString(result1.Body.ToArray()).Should().Be(message);
        Encoding.UTF8.GetString(result2.Body.ToArray()).Should().Be(message);
    }
}
```
