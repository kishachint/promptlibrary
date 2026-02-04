---
agent: 'agent'
tools: ['search', 'changes']
model: 'gpt-4o-mini'
name: 'dotnetcore-unit-testing'
description: 'Guidelines for writing effective, maintainable unit tests using xUnit, NSubstitute, and FluentAssertions on .NET 8.'
applyTo: "tests/**/*.cs"
---

# GitHub Copilot Unit Test Guidelines
**For .NET 8 — xUnit, NSubstitute, FluentAssertions**

This document describes the standardized testing conventions, patterns, and examples used across the {{PROJECT_NAME}} repository to ensure tests are reliable, fast, and maintainable.

## Audience / Roles
- .NET developers working on {{PROJECT_NAME}}
- Test engineers and QA specialists responsible for unit and small-scope integration tests
- Contributors who add or modify service orchestration and integration logic

## High-level Requirements
- Test framework: xUnit
- Mocking: NSubstitute (all external dependencies, Refit clients, repositories, keyed services)
- Assertions: FluentAssertions
- Tests must be:
  - Fast (execute in milliseconds)
  - Isolated (no DB/files/network; use mocks/fakes)
  - Deterministic and repeatable
  - Self-checking (clear assertions)
  - Maintainable and readable

## Test Class & Method Conventions
- Test filenames and classes should mirror the class under test (e.g., OrderServiceTests for OrderService).
- Use `ITestOutputHelper` for per-test logging when needed.
- Prefer `[Theory]` with `[InlineData]` / `[MemberData]` for parameterized cases; otherwise use `[Fact]`.
- Test method name pattern:
  MethodUnderTest_Scenario_ExpectedOutcome
  e.g. `CalculateTax_NegativeAmount_ThrowsArgumentException`
- Use Arrange–Act–Assert (AAA) in every test:
  1. Arrange — setup data, SUT, and mocks
  2. Act — perform single action
  3. Assert — assert expected outcome(s)
- Avoid shared mutable static/global state. Use fixtures or constructor injection for shared setup.
- Each test should exercise one logical scenario and one Act.

## Async & Exception Testing
- Use `async Task` for async tests. Never block on tasks (no `.Result` / `.Wait()`).
- Use `await Assert.ThrowsAsync<T>(...)` for async exceptions.
- Prefer `await` for all async interactions in tests.

## Mocking & Isolation
- Mock all external dependencies using NSubstitute, including:
  - Refit clients
  - Repositories
  - HttpClient interactions (where applicable)
  - Keyed services registered with AddKeyedScoped
- Use real implementations only when necessary and lightweight (e.g., value objects).
- Replace static/time references with an interface (e.g., `IDateTimeProvider`) to allow deterministic tests.

## Logging in Tests
- Avoid complex NSubstitute verification of structured logging (can cause matcher issues).
- Prefer a small, test-focused logger implementation (TestLogger<T>) to capture formatted messages for assertions.
- TestLogger benefits:
  - Validates the final formatted message
  - Avoids NSubstitute matcher conflicts
  - Works reliably with structured logging

Example TestLogger:
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
```

## Patterns & Examples

### Controller with Keyed Service
- Inject the keyed service wrapper/interface in the controller constructor and substitute it in tests.
```csharp
public class ServiceProviderControllerTests
{
    private readonly IServiceProviderService _mockService = Substitute.For<IServiceProviderService>();
    private readonly ServiceProvidersController _controller;

    public ServiceProviderControllerTests()
    {
        _controller = new ServiceProvidersController(_mockService);
    }

    [Fact]
    public async Task GetProviders_ValidRequest_ReturnsProviders()
    {
        // Arrange
        var expected = new List<Provider> { new Provider { Id = 1, Name = "Test" } };
        _mockService.GetProvidersAsync(Arg.Any<CancellationToken>()).Returns(expected);

        // Act
        var result = await _controller.GetProviders();

        // Assert
        var ok = result.Should().BeOfType<OkObjectResult>();
        ok.Subject.Value.Should().Be(expected);
    }
}
```

### Chain-of-Responsibility Handler Chains
- Mock individual handlers, verify invocation/order, and handle early termination cases.
```csharp
// See guidelines above for verifying Received()/DidNotReceive() patterns on handlers
```

### External Service Integrations (Refit / HTTP)
- Substitute Refit clients and verify interactions.
- Avoid calling real endpoints; if HTTP behavior must be validated, use HttpMessageHandler stubs or test-specific HttpClient instances.

### Entity Framework Repositories
- For repository behavior tests prefer the EF InMemory provider with a unique database name per test.
- Dispose DbContext between tests to avoid cross-test leakage.

### Records & Immutable Types
- Use actual record parameter names when constructing instances.
- Use fully-qualified names or using aliases when namespace collisions occur.
- Prefer small helper factory methods for creating variations of records rather than complex `with` expressions in tests.

## Anti-patterns & Solutions
- Don't verify private state or internal flags—test observable behavior.
- Avoid heavy setup duplication — extract factory/helper methods or fixtures.
- Avoid complex logic in tests (no loops/branches).
- Avoid NSubstitute structured logging matchers—use TestLogger.

## Decision Matrix (summary)
- External HTTP APIs: Mock (NSubstitute)
- EF Repositories: InMemory provider (real)
- Loggers: Custom TestLogger (real)
- Handler chains: Mock handlers and verify ordering
- Value objects: Real instances
- Time/date dependencies: Mock via IDateTimeProvider

## Quick-check Checklist for PR reviewers
- Tests use xUnit + NSubstitute + FluentAssertions
- Names follow MethodUnderTest_Scenario_ExpectedOutcome
- Tests are fast and isolated
- No real network or DB calls
- Async tests use async Task and await
- Logging assertions use TestLogger where formatting matters
- Helpers/factories are used to avoid setup duplication

## Examples Index (short)
- Controller / Keyed service: ServiceProviderControllerTests
- Handler chain verification: OrderProcessorTests
- TestLogger usage: ProcessAsync_HandlerFails_LogsError
- EF repository: BrandRepositoryTests (InMemory provider)
- External client: EncompassPartnerServiceTests (Refit client substitute)
