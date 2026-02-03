
# GitHub Copilot Unit Test Guidelines  
**For .NET 8 Encompass BFF – xUnit, NSubstitute, FluentAssertions**

---
description: "This file provides guidelines for writing effective, maintainable tests for the Encompass BFF using xUnit and related tools"
applyTo: "tests/**/*.cs"
---

## Role Definition

- .NET Developer working on Encompass BFF (Backend For Frontend)
- Test Engineer for lending platform integration services  
- Quality Assurance Specialist for multi-service orchestration  

---

## General

**Description:**  
Unit tests must be reliable, maintainable, and provide meaningful, high-quality coverage of business logic for the Encompass BFF (.NET 8) service that orchestrates interactions between lending platform services.  
Use **xUnit** framework with **NSubstitute** for mocking/faking and **FluentAssertions** for assertions.  
Tests must work for BFF controllers, service orchestration handlers, and external service integrations.

**Requirements:**
- Use xUnit as the test framework (project standard)
- Use NSubstitute for mocking all dependencies (including Refit clients, repositories, handlers)
- Use FluentAssertions for all assertions
- Tests must be:
  - Fast (milliseconds)
  - Isolated (no infrastructure dependencies – use mocks/fakes only)
  - Repeatable and deterministic (results always the same)
  - Self-checking (assertions must fail on any unexpected result)
  - Maintainable, timely, and readable
- Maximize code coverage and meaningfulness

---

## Test Class Structure

- Use `ITestOutputHelper` for logging in xUnit tests
- Prefer `[Theory]` with `[InlineData]` or `[MemberData]` for parameterized tests; otherwise, use `[Fact]`
- Use helper methods or builders for object creation (no setup duplication)
- Shared state only via fixtures or constructor injection; avoid global/static data
- Mock keyed services using NSubstitute patterns for services registered with `AddKeyedScoped`

---

## Test Method Structure & Best Practices

**Test Naming:**
- Method names:  
  ```
  [MethodUnderTest]_[Scenario]_[ExpectedOutcome]
  ```
  **Examples:**
  - `CalculateTax_NegativeAmount_ThrowsArgumentException`
  - `GetUserById_UserExists_ReturnsUser`

**Arrange-Act-Assert (AAA):**
- All tests use AAA pattern:
  1. **Arrange:** Setup test data, mocks, SUT
  2. **Act:** Perform action (call method)
  3. **Assert:** Use FluentAssertions for assertions

**Test Isolation:**
- No shared/global state
- Use NSubstitute for all external dependencies (repos, services, loggers, etc.)
- Never touch real DB/files/network – use fakes/mocks only

**Clarity and Focus:**
- Each test = one logical scenario (one "Act" per test)
- No loops, branches, or logic in test code
- Avoid magic strings/numbers – use named constants or builders

**Async Support:**
- Test all async code with `async Task` test methods (never `.Result` or `.Wait()`)
- Use `await Assert.ThrowsAsync<T>(...)` for async exception assertions

**Arrange:**
- Use helper/factory methods for object creation (avoid [SetUp] unless required)

**Act:**
- Only one main action per test

**Assert:**
- Use **FluentAssertions** for all assertions (e.g. `result.Should().Be(expected)`)
- For NSubstitute, verify interactions (e.g. `mock.Received().SomeMethod()`)

**Static References:**
- If SUT uses static methods/properties (e.g., `DateTime.Now`), refactor to use interface seam (`IDateTimeProvider`) and substitute in test

**No Private Method Testing:**
- Only test public/internal methods (never test private methods directly)

**Meaningful Assertions:**
- Cover all code branches and edge cases
- For collections: `result.Should().BeEmpty()`, `result.Should().Contain(item)`, etc.
- For exceptions: `Assert.Throws<T>` or `await Assert.ThrowsAsync<T>`

**Web API & Minimal API Tests:**
- Test controller/minimal API handlers by injecting fakes/mocks
- Assert status codes and returned values (e.g. `result.Should().BeOfType<OkObjectResult>()`)
- Test minimal API handlers as plain functions (not HTTP calls)

---

## Encompass BFF Specific Patterns

### Testing Keyed Services
When testing services that use keyed dependency injection:
```csharp
public class ServiceProviderControllerTests
{
    private readonly IServiceProviderService _mockService = Substitute.For<IServiceProviderService>();
    private readonly ServiceProvidersController _controller;

    public ServiceProviderControllerTests()
    {
        // Test the controller with mocked keyed service
        _controller = new ServiceProvidersController(_mockService);
    }

    [Fact]
    public async Task GetProviders_ValidRequest_ReturnsProviders()
    {
        // Arrange
        var expectedProviders = new List<Provider> { new Provider { Id = 1, Name = "Test" } };
        _mockService.GetProvidersAsync(Arg.Any<CancellationToken>())
                   .Returns(expectedProviders);

        // Act
        var result = await _controller.GetProviders();

        // Assert
        var okResult = result.Should().BeOfType<OkObjectResult>();
        okResult.Subject.Value.Should().Be(expectedProviders);
    }
}
```

### Testing Chain of Responsibility Handlers
For testing ordered handler chains (like `IProcessOrderHandler`):
```csharp
public class OrderProcessorTests
{
    private readonly List<IProcessOrderHandler> _handlers;
    private readonly OrderProcessor _processor;
    private readonly TestLogger<OrderProcessor> _logger;

    public OrderProcessorTests()
    {
        _logger = new TestLogger<OrderProcessor>();
        _handlers = new List<IProcessOrderHandler>
        {
            Substitute.For<IProcessOrderHandler>(), // Handler 1
            Substitute.For<IProcessOrderHandler>(), // Handler 2
            Substitute.For<IProcessOrderHandler>()  // Handler 3
        };
        _processor = new OrderProcessor(_handlers, _logger);
    }

    [Fact]
    public async Task ProcessAsync_AllHandlersSucceed_ProcessesAllHandlers()
    {
        // Arrange
        var context = new OrderProcessContext();
        foreach (var handler in _handlers)
        {
            handler.HandleAsync(context, Arg.Any<CancellationToken>()).Returns(true);
        }

        // Act
        await _processor.ProcessAsync(context, CancellationToken.None);

        // Assert
        foreach (var handler in _handlers)
        {
            await handler.Received(1).HandleAsync(context, Arg.Any<CancellationToken>());
        }
    }

    [Fact]
    public async Task ProcessAsync_SecondHandlerFails_StopsProcessing()
    {
        // Arrange
        var context = new OrderProcessContext();
        _handlers[0].HandleAsync(context, Arg.Any<CancellationToken>()).Returns(true);
        _handlers[1].HandleAsync(context, Arg.Any<CancellationToken>()).Returns(false);
        _handlers[2].HandleAsync(context, Arg.Any<CancellationToken>()).Returns(true);

        // Act
        await _processor.ProcessAsync(context, CancellationToken.None);

        // Assert
        await _handlers[0].Received(1).HandleAsync(context, Arg.Any<CancellationToken>());
        await _handlers[1].Received(1).HandleAsync(context, Arg.Any<CancellationToken>());
        await _handlers[2].DidNotReceive().HandleAsync(Arg.Any<OrderProcessContext>(), Arg.Any<CancellationToken>());
    }
}
```

### Testing Logging with Custom TestLogger
**Problem**: NSubstitute mocks can cause `RedundantArgumentMatcherException` with structured logging.
**Solution**: Use a custom `TestLogger<T>` implementation for real logging verification:

```csharp
public class TestLogger<T> : ILogger<T>
{
    public List<LogEntry> LogEntries { get; } = new();

    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => null;
    public bool IsEnabled(LogLevel logLevel) => true;

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, 
        Exception? exception, Func<TState, Exception?, string> formatter)
    {
        var message = formatter(state, exception);
        LogEntries.Add(new LogEntry
        {
            LogLevel = logLevel,
            EventId = eventId,
            Message = message,
            Exception = exception,
            State = state
        });
    }
}

public class LogEntry
{
    public LogLevel LogLevel { get; set; }
    public EventId EventId { get; set; }
    public string Message { get; set; } = string.Empty;
    public Exception? Exception { get; set; }
    public object? State { get; set; }
}

// Usage in tests:
[Fact]
public async Task ProcessAsync_HandlerFails_LogsError()
{
    // Arrange
    var context = CreateTestContext();
    context.ErrorMessage = "Handler failed";
    _handlers[0].HandleAsync(context, cancellationToken).Returns(false);

    // Act
    await _sut.ProcessAsync(context, cancellationToken);

    // Assert
    var errorLogs = _logger.LogEntries.Where(log => log.LogLevel == LogLevel.Error).ToList();
    errorLogs.Should().HaveCount(1);
    errorLogs.First().Message.Should().Contain("Handler failed");
}
```

**Benefits:**
- Tests actual formatted log messages (what users see)
- Eliminates NSubstitute argument matcher conflicts
- Provides better debugging information
- Works with structured logging arrays like `[handler.GetType().Name, context.ErrorMessage]`

### Testing Notification Handler Factory
For testing factory patterns that resolve handlers by key:
```csharp
public class NotificationHandlerFactoryTests
{
    private readonly IKeyedServiceProvider _mockKeyedServiceProvider;
    private readonly NotificationHandlerFactory _sut;

    public NotificationHandlerFactoryTests()
    {
        _mockKeyedServiceProvider = Substitute.For<IKeyedServiceProvider, IServiceProvider>();
        _sut = new NotificationHandlerFactory((IServiceProvider)_mockKeyedServiceProvider);
    }

    [Theory]
    [InlineData("created", "transaction", nameof(TransactionCreatedHandler))]
    [InlineData("updated", "transaction", nameof(TransactionUpdatedHandler))]
    public void CreateHandler_ValidEventAndResource_ReturnsCorrectHandler(
        string eventType, string resourceType, string expectedKey)
    {
        // Arrange
        var expectedHandler = Substitute.For<INotificationHandler>();
        _mockKeyedServiceProvider
            .GetRequiredKeyedService(typeof(INotificationHandler), expectedKey)
            .Returns(expectedHandler);

        // Act
        var result = _sut.CreateHandler(eventType, resourceType);

        // Assert
        result.Should().Be(expectedHandler);
        _mockKeyedServiceProvider
            .Received(1)
            .GetRequiredKeyedService(typeof(INotificationHandler), expectedKey);
    }
}
```

### Testing External Service Integrations
For testing Refit clients and external service calls:
```csharp
public class EncompassPartnerServiceTests
{
    private readonly IEncompassPatnerClient _mockClient = Substitute.For<IEncompassPatnerClient>();
    private readonly IWebhookRepository _mockRepository = Substitute.For<IWebhookRepository>();
    private readonly HttpClient _httpClient = new HttpClient();
    private readonly ILogger<EncompassPartnerService> _logger = Substitute.For<ILogger<EncompassPartnerService>>();
    private readonly EncompassPartnerService _service;

    public EncompassPartnerServiceTests()
    {
        _service = new EncompassPartnerService(_mockClient, _mockRepository, _httpClient, _logger);
    }

    [Fact]
    public async Task GetTransactionAsync_ValidId_ReturnsTransaction()
    {
        // Arrange
        var transactionId = "trans-123";
        var expectedTransaction = new Transaction { Id = transactionId };
        _mockClient.GetTransactionAsync(transactionId).Returns(expectedTransaction);

        // Act
        var result = await _service.GetTransactionAsync(transactionId);

        // Assert
        result.Should().Be(expectedTransaction);
        await _mockClient.Received(1).GetTransactionAsync(transactionId);
    }
}
```

### Testing Entity Framework Repositories
For testing repository patterns with Entity Framework:
```csharp
public class BrandRepositoryTests : IDisposable
{
    private readonly EncompassDbContext _context;
    private readonly BrandRepository _repository;

    public BrandRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<EncompassDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        _context = new EncompassDbContext(options);
        _repository = new BrandRepository(_context);
    }

    [Fact]
    public async Task GetByIdAsync_ExistingBrand_ReturnsBrand()
    {
        // Arrange
        var brand = new Brand { Id = 1, Name = "Test Brand" };
        _context.Brands.Add(brand);
        await _context.SaveChangesAsync();

        // Act
        var result = await _repository.GetByIdAsync(1);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be("Test Brand");
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### Working with C# Records in Tests

**Creating Record Instances:**
When creating record instances in tests, use the correct parameter names from the record definition:

```csharp
// ❌ AVOID: Guessing parameter names
var notification = new OrderRequestNotification(
    id: "test-id",  // Wrong parameter name
    details: orderDetails  // Wrong parameter name
);

// ✅ PREFER: Use actual record parameter names from definition
var notification = new OrderRequestNotification(
    Id: "test-id",  // Matches actual record definition
    OrderTypeDetails: orderDetails,  // Matches actual record definition
    ServiceProvider: serviceProvider,
    Comment: "test comment",
    Attachments: Array.Empty<Attachment>(),
    User: user,
    CreatedDate: DateTime.UtcNow,
    CreatedBy: "TestUser"
);
```

**Handling Namespace Conflicts:**
When multiple classes have the same name across different namespaces, use fully qualified names:

```csharp
// ❌ AVOID: Ambiguous type references
var transaction = new TransactionResponse(); // Could be multiple types

// ✅ PREFER: Explicit namespace qualification
var transaction = new Encompass.BFF.Application.Models.Transaction.TransactionResponse();

// Or use alias at top of file
using AppTransactionResponse = Encompass.BFF.Application.Models.Transaction.TransactionResponse;
// Then use: var transaction = new AppTransactionResponse();
```

**Record Modification in Tests:**
For immutable records, create helper methods instead of using `with` expressions in tests:

```csharp
// ❌ AVOID: Complex with expressions in tests
var modifiedProvider = serviceProvider with 
{ 
    Profile = serviceProvider.Profile with { RoutingNumber = "new-routing" }
};

// ✅ PREFER: Helper methods for variations
private static ServiceProvider CreateTestServiceProviderWithRoutingNumber(string routingNumber)
{
    return new ServiceProvider(
        BrandCode: "TEST",
        Profile: new Profile(
            ProfileId: "test-profile-123",
            OfficeName: "Test Office", 
            RoutingNumber: routingNumber,
            NotificationEmailAddress: "test@example.com",
            IsFavorite: false
        ),
        Organization: new Organization("0", "Test Organization"),
        Products: Array.Empty<Product>(),
        Agent: new Agent("Test", "Agent", "agent-id", "agent@example.com")
    );
}
```

**Testing Record Properties:**
```csharp
[Fact]
public void CreateServiceProvider_WithValidData_SetsAllProperties()
{
    // Arrange
    var profile = new Profile("profile-123", "Test Office", "routing-456", "test@example.com", true);
    
    // Act
    var serviceProvider = new ServiceProvider("BRAND", profile, organization, products, agent);

    // Assert
    serviceProvider.BrandCode.Should().Be("BRAND");
    serviceProvider.Profile.Should().Be(profile);
    serviceProvider.Profile.ProfileId.Should().Be("profile-123");
    serviceProvider.Profile.OfficeName.Should().Be("Test Office");
}
```

---

## Common Testing Anti-Patterns & Solutions

### ❌ NSubstitute Argument Matcher Conflicts
```csharp
// AVOID: Complex argument matching with logging
_mockLogger.Received(1).LogError("message", 
    Arg.Is<object[]>(args => args[0].ToString().Contains("Handler")));
// This can cause RedundantArgumentMatcherException
```

### ✅ Use Custom TestLogger Instead
```csharp
// PREFER: Real logger implementation for verification
var errorLogs = _logger.LogEntries.Where(log => log.LogLevel == LogLevel.Error).ToList();
errorLogs.Should().HaveCount(1);
errorLogs.First().Message.Should().Contain("Expected error content");
```

### ❌ Testing Implementation Details
```csharp
// AVOID: Testing private methods or internal state
Assert.True(processor._internalFlag); // Don't test private fields
```

### ✅ Test Public Behavior
```csharp
// PREFER: Test observable outcomes
context.ErrorMessage.Should().Be("Expected error");
await handler.Received(1).HandleAsync(context, cancellationToken);
```

### ❌ Complex Test Setup
```csharp
// AVOID: Repeated setup code in each test
public async Task Test1() 
{
    var handler1 = Substitute.For<IHandler>();
    var handler2 = Substitute.For<IHandler>();
    // ... repeated in every test
}
```

### ✅ Use Helper Methods
```csharp
// PREFER: Extract common setup to helper methods
private static OrderProcessContext CreateTestContext() => new()
{
    Notification = new Notification 
    { 
        EventId = "test-event-123",
        EventTime = DateTimeOffset.UtcNow,
        EventType = EventType.Created,
        Meta = new Metadata
        {
            ResourceId = "test-resource-id",
            ResourceType = ResourceType.Transaction,
            InstanceId = "test-instance-id",
            ResourceRef = "test-resource-reference"
        }
    }
};
```

### ❌ Record Construction Parameter Errors
```csharp
// AVOID: Incorrect parameter names or order
var record = new MyRecord(
    name: "test",      // Wrong parameter name
    id: 123,          // Wrong parameter name
    status: "active"   // Wrong parameter name
);
```

### ✅ Verify Record Parameter Names
```csharp
// PREFER: Check actual record definition and use correct names
// For record MyRecord(string Name, int Id, string Status)
var record = new MyRecord(
    Name: "test",      // Correct parameter name (matches record definition)
    Id: 123,          // Correct parameter name  
    Status: "active"   // Correct parameter name
);
```

---

## Example Patterns

### Service Method Test (xUnit, NSubstitute, FluentAssertions)
```csharp
public class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _service;

    public OrderServiceTests()
    {
        _service = new OrderService(_repo);
    }

    [Fact]
    public async Task PlaceOrder_ValidOrder_ReturnsOrderId()
    {
        // Arrange
        var order = new Order("SKU1", 3);
        _repo.SaveAsync(order).Returns(Task.CompletedTask);

        // Act
        var result = await _service.PlaceOrderAsync(order);

        // Assert
        result.Should().BeOfType<Guid>();
        await _repo.Received(1).SaveAsync(order);
    }
}
```

### Async Exception Test
```csharp
[Fact]
public async Task PlaceOrder_InvalidOrder_ThrowsOrderException()
{
    // Arrange
    var order = new Order("", 0);

    // Act & Assert
    await Assert.ThrowsAsync<OrderException>(async () => await _service.PlaceOrderAsync(order));
}
```

### Parameterized Theory with MemberData
```csharp
public static IEnumerable<object[]> DiscountData =>
    new List<object[]>
    {
        new object[] { 100m, 1, 0m },
        new object[] { 100m, 5, 5m },
    };

[Theory]
[MemberData(nameof(DiscountData))]
public void CalculateDiscount_ValidInput_ReturnsExpectedDiscount(decimal price, int qty, decimal expected)
{
    // Arrange
    var calculator = new DiscountCalculator();

    // Act
    var discount = calculator.Calculate(price, qty);

    // Assert
    discount.Should().Be(expected);
}
```

### Web API Controller Example
```csharp
[Fact]
public async Task GetOrder_OrderExists_ReturnsOkWithOrder()
{
    // Arrange
    var fakeRepo = Substitute.For<IOrderRepository>();
    var expectedOrder = new Order("SKU1", 2);
    fakeRepo.GetByIdAsync(Arg.Any<Guid>()).Returns(expectedOrder);
    var controller = new OrdersController(fakeRepo);

    // Act
    var result = await controller.GetOrder(Guid.NewGuid());

    // Assert
    var okResult = result.Should().BeOfType<OkObjectResult>();
    okResult.Subject.Value.Should().Be(expectedOrder);
}
```

### Minimal API Handler Example
```csharp
[Fact]
public async Task GetTodo_TodoExists_ReturnsOk()
{
    // Arrange
    var fakeRepo = Substitute.For<ITodoRepository>();
    var todo = new Todo { Id = 1, Title = "Test" };
    fakeRepo.GetAsync(1).Returns(todo);

    // Act
    var result = await MinimalApiHandlers.GetTodo(1, fakeRepo);

    // Assert
    result.Should().BeOfType<Ok<Todo>>();
    result.Value.Should().Be(todo);
}
```

### Test Class Structure

- **Use ITestOutputHelper for logging (xUnit):**
    ```csharp
    using FluentAssertions;
    using NSubstitute;
    using Xunit;
    using Xunit.Abstractions;

    public class OrderProcessingTests
    {
        private readonly ITestOutputHelper _output;

        public OrderProcessingTests(ITestOutputHelper output)
        {
            _output = output;
        }

        [Fact]
        public async Task ProcessOrder_ValidOrder_Succeeds()
        {
            _output.WriteLine("Starting test with valid order");
            var orderRepository = Substitute.For<IOrderRepository>();
            var processor = new OrderProcessor(orderRepository);
            var order = new Order("SKU123", 5);

            orderRepository.SaveAsync(order).Returns(Task.CompletedTask);

            var result = await processor.ProcessAsync(order);

            result.IsSuccess.Should().BeTrue();
            await orderRepository.Received(1).SaveAsync(order);
        }
    }
    ```

- **Use fixtures for shared state (with NSubstitute):**
    ```csharp
    using System.Data.Common;
    using FluentAssertions;
    using NSubstitute;
    using Xunit;
    using Xunit.Abstractions;

    public class DatabaseFixture : IAsyncLifetime
    {
        public DbConnection Connection { get; private set; } = Substitute.For<DbConnection>();

        public Task InitializeAsync()
        {
            // Mocked connection setup if needed.
            return Task.CompletedTask;
        }

        public Task DisposeAsync()
        {
            Connection.Dispose();
            return Task.CompletedTask;
        }
    }

    public class OrderTests : IClassFixture<DatabaseFixture>
    {
        private readonly DatabaseFixture _fixture;
        private readonly ITestOutputHelper _output;

        public OrderTests(DatabaseFixture fixture, ITestOutputHelper output)
        {
            _fixture = fixture;
            _output = output;
        }

        [Fact]
        public void Fixture_ShouldProvideConnection()
        {
            _fixture.Connection.Should().NotBeNull();
        }
    }
    ```

---

### Test Methods

- **Prefer Theory over multiple Facts:**
    ```csharp
    using FluentAssertions;
    using NSubstitute;
    using Xunit;

    public class DiscountCalculatorTests
    {
        public static IEnumerable<object[]> DiscountTestData =>
            new List<object[]>
            {
                new object[] { 100m, 1, 0m },
                new object[] { 100m, 5, 5m },
                new object[] { 100m, 10, 10m },
            };

        [Theory]
        [MemberData(nameof(DiscountTestData))]
        public void CalculateDiscount_ReturnsCorrectAmount(decimal price, int quantity, decimal expectedDiscount)
        {
            var calculator = new DiscountCalculator();

            var discount = calculator.Calculate(price, quantity);

            discount.Should().Be(expectedDiscount);
        }
    }
    ```

- **Follow Arrange-Act-Assert pattern:**
    ```csharp
    using FluentAssertions;
    using NSubstitute;
    using Xunit;

    public class OrderProcessorTests
    {
        [Fact]
        public async Task ProcessOrder_ValidOrder_UpdatesInventory()
        {
            // Arrange
            var repository = Substitute.For<IInventoryRepository>();
            var order = new Order(OrderId.New(), new[] { new OrderLine("SKU123", 5) });
            var processor = new OrderProcessor(repository);

            repository.UpdateInventoryAsync(Arg.Any<string>(), Arg.Any<int>())
                      .Returns(Task.CompletedTask);

            // Act
            var result = await processor.ProcessAsync(order);

            // Assert
            result.IsSuccess.Should().BeTrue();
            await repository.Received(1)
                .UpdateInventoryAsync(Arg.Is<string>(sku => sku == "SKU123"), 5);
        }
    }
    ```

---

### Test Isolation

- **Use fresh data for each test:**
    ```csharp
    using FluentAssertions;
    using NSubstitute;
    using Xunit;

    public class OrderTests
    {
        private static Order CreateTestOrder() =>
            new Order(OrderId.New(), new[] { new OrderLine("SKU123", 2) });

        [Fact]
        public async Task ProcessOrder_Success()
        {
            var order = CreateTestOrder();
            var repository = Substitute.For<IOrderRepository>();
            var processor = new OrderProcessor(repository);

            repository.SaveAsync(order).Returns(Task.CompletedTask);

            var result = await processor.ProcessAsync(order);

            result.IsSuccess.Should().BeTrue();
            await repository.Received(1).SaveAsync(order);
        }
    }
    ```

- **Clean up resources (mocked test server example):**
    ```csharp
    using FluentAssertions;
    using NSubstitute;
    using Xunit;

    public class IntegrationTests : IAsyncDisposable
    {
        private readonly TestServer _server = Substitute.For<TestServer>();
        private readonly HttpClient _client = new HttpClient();

        public async ValueTask DisposeAsync()
        {
            _client.Dispose();
            await _server.DisposeAsync();
        }

        [Fact]
        public void Server_Should_NotBeNull()
        {
            _server.Should().NotBeNull();
        }
    }
    ```

---

### Best Practices

- **Name tests clearly:**
    ```csharp
    // Good: Clear test names
    [Fact]
    public async Task ProcessOrder_WhenInventoryAvailable_UpdatesStockAndReturnsSuccess()
    {
        // Test implementation using NSubstitute and FluentAssertions
    }

    // Avoid: Unclear names
    [Fact]
    public async Task TestProcessOrder()
    {
        // Avoid this naming
    }
    ```

- **Use meaningful assertions (FluentAssertions):**
    ```csharp
    // Good: Clear assertions
    actual.Should().Be(expected);
    collection.Should().Contain(expectedItem);
    Assert.Throws<OrderException>(() => processor.Process(invalidOrder));

    // Avoid: Multiple assertions without context
    result.Should().NotBeNull();
    result.Success.Should().BeTrue();
    result.Errors.Count.Should().Be(0);
    ```

- **Handle async operations properly:**
    ```csharp
    // Good: Async test method
    [Fact]
    public async Task ProcessOrder_ValidOrder_Succeeds()
    {
        var processor = new OrderProcessor();
        var result = await processor.ProcessAsync(order);
        result.IsSuccess.Should().BeTrue();
    }

    // Avoid: Sync over async
    [Fact]
    public void ProcessOrder_ValidOrder_Succeeds()
    {
        var processor = new OrderProcessor();
        var result = processor.ProcessAsync(order).Result;  // Can deadlock
        result.IsSuccess.Should().BeTrue();
    }
    ```

- **Use TestContext.Current.CancellationToken for cancellation:**
    ```csharp
    [Fact]
    public async Task ProcessOrder_CancellationRequested()
    {
        var processor = new OrderProcessor();
        var result = await processor.ProcessAsync(order, TestContext.Current.CancellationToken);
        result.IsSuccess.Should().BeTrue();
    }
    ```

---

### Assertions

- **Use FluentAssertions for assertions:**
    ```csharp
    public class OrderTests
    {
        [Fact]
        public void CalculateTotal_WithValidLines_ReturnsCorrectSum()
        {
            var order = new Order(
                OrderId.New(),
                new[]
                {
                    new OrderLine("SKU1", 2, 10.0m),
                    new OrderLine("SKU2", 1, 20.0m)
                });

            var total = order.CalculateTotal();

            total.Should().Be(40.0m);
        }

        [Fact]
        public void Order_WithInvalidLines_ThrowsException()
        {
            var invalidLines = Array.Empty<OrderLine>();

            var ex = Assert.Throws<ArgumentException>(() =>
                new Order(OrderId.New(), invalidLines));

            ex.Message.Should().Contain("Order must have at least one line");
        }

        [Fact]
        public void Order_WithValidData_HasExpectedProperties()
        {
            var id = OrderId.New();
            var lines = new[] { new OrderLine("SKU1", 1, 10.0m) };

            var order = new Order(id, lines);

            order.Should().NotBeNull();
            order.Id.Should().Be(id);
            order.Lines.Should().HaveCount(1);
            var line = order.Lines.Single();
            line.Sku.Should().Be("SKU1");
            line.Quantity.Should().Be(1);
            line.Price.Should().Be(10.0m);
        }
    }
    ```

- **Use proper assertion types (FluentAssertions):**
    ```csharp
    public class CustomerTests
    {
        [Fact]
        public void Customer_WithValidEmail_IsCreated()
        {
            customer.IsActive.Should().BeTrue();
            customer.IsDeleted.Should().BeFalse();
            customer.Email.Should().Be("john@example.com");
            customer.Id.Should().NotBe(Guid.Empty);
            customer.Orders.Should().BeEmpty();
            customer.Roles.Should().Contain("Admin");
            customer.Roles.Should().NotContain("Guest");
            customer.Orders.All(o => o.Id.Should().NotBeNull());
            customer.Should().BeOfType<PremiumCustomer>();
            customer.Should().BeAssignableTo<ICustomer>();
            customer.Reference.Should().StartWith("CUST");
            customer.Description.Should().Contain("Premium");
            customer.Reference.Should().MatchRegex("^CUST\\d{6}$");
            customer.Age.Should().BeInRange(18, 100);
            actualCustomer.Should().BeSameAs(expectedCustomer);
            actualCustomer.Should().NotBeSameAs(differentCustomer);
        }
    }
    ```

- **Use FluentAssertions for collection assertions:**
    ```csharp
    [Fact]
    public void ProcessOrder_CreatesExpectedEvents()
    {
        var processor = new OrderProcessor();
        var order = CreateTestOrder();

        var events = processor.Process(order);

        events.Should().SatisfyRespectively(
            first => first.Should().BeOfType<OrderReceivedEvent>()
                .Which.OrderId.Should().Be(order.Id),
            second => second.Should().BeOfType<InventoryReservedEvent>()
                .Which.OrderId.Should().Be(order.Id),
            third => third.Should().BeOfType<OrderConfirmedEvent>()
                .Which.OrderId.Should().Be(order.Id)
                .And.Subject.As<OrderConfirmedEvent>().IsSuccess.Should().BeTrue()
        );
    }
    ```
---

## Key Testing Decision Matrix

| Scenario | Use Mock | Use Real Implementation | Reason |
|----------|----------|------------------------|---------|
| **External HTTP APIs** | ✅ Mock with NSubstitute | ❌ | Avoid external dependencies |
| **Entity Framework Repositories** | ❌ | ✅ InMemory provider | Test real EF behavior |
| **Loggers** | ❌ | ✅ Custom TestLogger | Test actual log messages |
| **Chain of Responsibility Handlers** | ✅ Mock individual handlers | ❌ | Isolate processor logic |
| **Simple Value Objects** | ❌ | ✅ Real objects | No complex behavior to mock |
| **Time/Date Dependencies** | ✅ Mock IDateTimeProvider | ❌ | Control time in tests |
| **C# Records** | ❌ | ✅ Real record instances | Records are immutable value objects |

## Copilot Instructions Summary

**Encompass BFF Testing Standards:**
- Use xUnit framework (project standard)
- Mock dependencies with NSubstitute; assert with FluentAssertions
- Name tests: `[Method]_[Scenario]_[ExpectedOutcome]`
- Always follow Arrange-Act-Assert (AAA)
- Use async/await for async code (never block)
- Only test public APIs, not private methods
- No infrastructure dependencies in unit tests (mock all external services)
- One scenario per test (one "Act" per method)
- Prefer helper/factory methods for data, avoid setup duplication

**BFF-Specific Patterns:**
- Mock keyed services for services registered with `AddKeyedScoped`
- Test handler chains by verifying execution order and early termination
- Mock Refit clients for external service integration testing
- Use in-memory Entity Framework for repository tests
- Test notification handler factory resolution by event/resource type
- Verify webhook processing flows and transaction orchestration
- **Use custom TestLogger for real logging verification instead of mocking**

**C# Record Testing Best Practices:**
- Verify record parameter names match actual definitions before creating instances
- Use fully qualified type names when namespace conflicts exist
- Create helper methods for record variations instead of `with` expressions
- Test record properties and equality semantics appropriately

**External Dependencies to Mock:**
- VendorManagement client calls
- Rates service integration  
- Encompass Partner Connect APIs
- Exchange360 XML processing
- SQL Server via Entity Framework (use InMemory provider)

**Logging Testing Best Practice:**
- **Always use custom TestLogger<T> instead of mocking ILogger**
- Test actual formatted log messages, not logging framework internals
- Avoids NSubstitute argument matcher conflicts with structured logging
