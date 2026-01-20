---
name: csharp-coding-standards
description: C# coding standards and best practices following .NET conventions, modern C# features, and SOLID principles.
---

# C# Coding Standards & Best Practices

Modern C# coding standards following .NET conventions and best practices.

## Code Quality Principles

### 1. Follow .NET Conventions
- PascalCase for public members
- Use framework design guidelines
- Async all the way
- Leverage language features

### 2. SOLID Principles
- Single Responsibility
- Open/Closed Principle
- Liskov Substitution
- Interface Segregation
- Dependency Inversion

### 3. Immutability When Possible
- Record types for immutable data
- Init-only properties
- Readonly fields
- Avoid mutable static state

### 4. Null Safety
- Nullable reference types (C# 8.0+)
- Null-conditional operators
- Null-coalescing operators
- Pattern matching for null checks

## Naming Conventions

### Classes, Methods, and Properties

```csharp
// ✅ GOOD: PascalCase for public members
public class UserService
{
    public string UserName { get; set; }
    public int UserId { get; private set; }

    public async Task<User> GetUserAsync(int id)
    {
        // Implementation
    }

    public bool IsUserActive()
    {
        return true;
    }
}

// ❌ BAD: Incorrect casing
public class userService
{
    public string user_name { get; set; }
    public int userid { get; private set; }
}
```

### Private Fields and Local Variables

```csharp
// ✅ GOOD: camelCase for private/local, underscore prefix for fields
public class Connection
{
    private readonly string _connectionString;
    private bool _isConnected;

    public void Connect()
    {
        string hostName = "localhost";
        int portNumber = 5432;
        // Implementation
    }
}

// Alternative style (no underscore)
public class Connection
{
    private readonly string connectionString;
    private bool isConnected;
}
```

### Interfaces and Constants

```csharp
// ✅ GOOD: 'I' prefix for interfaces
public interface IRepository<T>
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
}

// ✅ GOOD: PascalCase for constants
public class Constants
{
    public const int MaxRetryCount = 3;
    public const string DefaultEncoding = "UTF-8";
}

// ❌ BAD: ALL_CAPS for constants (C/C++ style)
public const int MAX_RETRY_COUNT = 3;
```

### Method Naming

```csharp
// ✅ GOOD: Verb-noun pattern, Async suffix
public async Task<Order> CreateOrderAsync(OrderRequest request)
{
    // Implementation
}

public bool ValidateInput(string input)
{
    // Implementation
}

public void ProcessPayment(Payment payment)
{
    // Implementation
}

// ❌ BAD: Unclear or inconsistent names
public async Task<Order> CreateOrder()  // Missing Async suffix
{
    // Implementation
}
```

## Modern C# Features

### Records for Immutable Data

```csharp
// ✅ GOOD: Record for data transfer objects
public record UserDto(int Id, string Name, string Email);

// With validation using init
public record CreateOrderRequest
{
    public int UserId { get; init; }
    public List<OrderItem> Items { get; init; } = new();
    public string ShippingAddress { get; init; } = string.Empty;

    public CreateOrderRequest()
    {
        // Default constructor
    }

    // Validation can be done in a separate method
    public void Validate()
    {
        if (Items.Count == 0)
            throw new ArgumentException("Items cannot be empty", nameof(Items));
    }
};

// Record with mutable properties (use sparingly)
public record UserProfile
{
    public int Id { get; init; }
    public string Name { get; init; }
    public string Email { get; set; }  // Mutable
}
```

### Nullable Reference Types

```csharp
#nullable enable

// ✅ GOOD: Explicit nullability
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public User? FindUser(int id)
    {
        // May return null
        return _repository.Find(id);
    }

    public User GetUser(int id)
    {
        // Never returns null
        return _repository.Find(id) 
            ?? throw new NotFoundException($"User {id} not found");
    }

    public void ProcessUser(User? user)
    {
        if (user is null)
            return;

        // user is not null here
        Console.WriteLine(user.Name);
    }
}
```

### Pattern Matching

```csharp
// ✅ GOOD: Modern pattern matching
public string GetDescription(object value) => value switch
{
    null => "null value",
    int i when i > 0 => $"Positive number: {i}",
    int i when i < 0 => $"Negative number: {i}",
    int => "Zero",
    string s when s.Length > 10 => "Long string",
    string s => $"String: {s}",
    IEnumerable<int> list => $"List with {list.Count()} items",
    _ => "Unknown type"
};

// ✅ GOOD: Type pattern with is
if (response is SuccessResponse success)
{
    Console.WriteLine($"Success: {success.Data}");
}
else if (response is ErrorResponse error)
{
    Console.WriteLine($"Error: {error.Message}");
}
```

### Init-only Properties

```csharp
// ✅ GOOD: Init-only properties for immutability
public class Configuration
{
    public string ApiKey { get; init; }
    public string BaseUrl { get; init; }
    public int Timeout { get; init; }

    // Can only be set during object initialization
}

var config = new Configuration
{
    ApiKey = "key123",
    BaseUrl = "https://api.example.com",
    Timeout = 30
};

// config.ApiKey = "new_key";  // Compile error!
```

### Target-typed New Expressions

```csharp
// ✅ GOOD: Simplified object creation
List<string> names = new();
Dictionary<int, User> users = new();

User user = new() { Id = 1, Name = "John" };

// In method calls
ProcessData(new() { Id = 1, Value = "test" });
```

## Async/Await Patterns

### Async Method Naming

```csharp
// ✅ GOOD: Async suffix for async methods
public async Task<User> GetUserAsync(int id)
{
    return await _repository.GetByIdAsync(id);
}

public async Task SaveChangesAsync()
{
    await _context.SaveChangesAsync();
}

// ❌ BAD: No async suffix
public async Task<User> GetUser(int id)
{
    return await _repository.GetByIdAsync(id);
}
```

### Async Best Practices

```csharp
// ✅ GOOD: ConfigureAwait in library code
public async Task<string> FetchDataAsync()
{
    var response = await _httpClient
        .GetAsync(_endpoint)
        .ConfigureAwait(false);

    return await response.Content
        .ReadAsStringAsync()
        .ConfigureAwait(false);
}

// ✅ GOOD: ValueTask for frequently synchronous results
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
    {
        return new ValueTask<int>(value);  // Synchronous path
    }

    return new ValueTask<int>(FetchFromDatabaseAsync(key));
}

// ❌ BAD: Blocking on async
public string FetchData()
{
    return FetchDataAsync().Result;  // Can deadlock!
}
```

### Cancellation Support

```csharp
// ✅ GOOD: Always support cancellation
public async Task<Data> ProcessDataAsync(
    string input,
    CancellationToken cancellationToken = default)
{
    cancellationToken.ThrowIfCancellationRequested();

    var data = await FetchDataAsync(input, cancellationToken);
    
    cancellationToken.ThrowIfCancellationRequested();

    await ProcessAsync(data, cancellationToken);
    
    return data;
}
```

## LINQ Best Practices

### Query Syntax vs Method Syntax

```csharp
// ✅ GOOD: Method syntax (preferred for simple queries)
var activeUsers = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new UserDto(u.Id, u.Name, u.Email));

// ✅ GOOD: Query syntax (for complex queries with multiple from)
var orders = from order in _context.Orders
             where order.Status == OrderStatus.Pending
             join customer in _context.Customers on order.CustomerId equals customer.Id
             select new { order.Id, customer.Name, order.Total };

// ❌ BAD: Mixed syntax without clear benefit
var result = (from user in users
              where user.IsActive
              select user)
             .OrderBy(u => u.Name);
```

### Deferred Execution Awareness

```csharp
// ✅ GOOD: Understand when query executes
var query = users.Where(u => u.IsActive);  // Not executed yet

var count = query.Count();     // Executes here
var list = query.ToList();     // Executes again

// ✅ GOOD: Execute once and store
var activeUsers = users
    .Where(u => u.IsActive)
    .ToList();  // Execute once

var count = activeUsers.Count;  // No query
```

### Avoid Multiple Enumerations

```csharp
// ❌ BAD: Multiple enumerations
public void ProcessUsers(IEnumerable<User> users)
{
    if (users.Any())  // Enumeration 1
    {
        var count = users.Count();  // Enumeration 2
        var first = users.First();  // Enumeration 3
    }
}

// ✅ GOOD: Single enumeration
public void ProcessUsers(IEnumerable<User> users)
{
    var userList = users.ToList();  // Single enumeration
    
    if (userList.Count > 0)
    {
        var first = userList[0];
    }
}
```

## Dependency Injection

### Constructor Injection

```csharp
// ✅ GOOD: Constructor injection (preferred)
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;
    private readonly IEmailService _emailService;

    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger,
        IEmailService emailService)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _emailService = emailService ?? throw new ArgumentNullException(nameof(emailService));
    }

    public async Task<Order> CreateOrderAsync(OrderRequest request)
    {
        _logger.LogInformation("Creating order for user {UserId}", request.UserId);
        
        var order = await _repository.CreateAsync(request);
        await _emailService.SendOrderConfirmationAsync(order);
        
        return order;
    }
}

// Registration
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderService, OrderService>();
```

### Service Lifetimes

```csharp
// ✅ GOOD: Appropriate lifetimes
public void ConfigureServices(IServiceCollection services)
{
    // Transient: Created each time
    services.AddTransient<IEmailService, EmailService>();

    // Scoped: One per request
    services.AddScoped<IOrderService, OrderService>();
    services.AddScoped<DbContext>();

    // Singleton: One for application lifetime
    services.AddSingleton<IConfiguration>(Configuration);
    services.AddSingleton<IMemoryCache, MemoryCache>();
}
```

## Error Handling

### Exception Guidelines

```csharp
// ✅ GOOD: Specific exception types
public class OrderService
{
    public async Task<Order> GetOrderAsync(int id)
    {
        var order = await _repository.GetByIdAsync(id);
        
        if (order == null)
        {
            throw new NotFoundException($"Order {id} not found");
        }

        return order;
    }

    public void ValidateOrder(Order order)
    {
        if (order.Items.Count == 0)
        {
            throw new ValidationException("Order must have at least one item");
        }

        if (order.Total <= 0)
        {
            throw new ValidationException("Order total must be positive");
        }
    }
}

// Custom exceptions
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class ValidationException : Exception
{
    public ValidationException(string message) : base(message) { }
}
```

### Try-Catch Best Practices

```csharp
// ✅ GOOD: Catch specific exceptions
public async Task<Result> ProcessPaymentAsync(Payment payment)
{
    try
    {
        return await _paymentGateway.ProcessAsync(payment);
    }
    catch (PaymentDeclinedException ex)
    {
        _logger.LogWarning(ex, "Payment declined for order {OrderId}", payment.OrderId);
        return Result.Failure("Payment was declined");
    }
    catch (NetworkException ex)
    {
        _logger.LogError(ex, "Network error processing payment");
        return Result.Failure("Payment service unavailable");
    }
}

// ❌ BAD: Catch all exceptions
public async Task ProcessPaymentAsync(Payment payment)
{
    try
    {
        await _paymentGateway.ProcessAsync(payment);
    }
    catch (Exception ex)
    {
        // Too broad!
    }
}
```

## Resource Management

### Using Statements

```csharp
// ✅ GOOD: Using statement for IDisposable
public async Task<string> ReadFileAsync(string path)
{
    using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    return await reader.ReadToEndAsync();
}

// ✅ GOOD: Using declaration (C# 8.0+)
public void ProcessFile(string path)
{
    using var file = File.OpenRead(path);
    using var reader = new BinaryReader(file);
    
    // Process file
    
} // Automatically disposed
```

### Implementing IDisposable

```csharp
// ✅ GOOD: Complete IDisposable pattern
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;
    private bool _disposed;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _connection?.Dispose();
            }

            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

## Performance Best Practices

### String Concatenation

```csharp
// ✅ GOOD: StringBuilder for multiple concatenations
public string BuildReport(IEnumerable<string> lines)
{
    var builder = new StringBuilder();
    
    foreach (var line in lines)
    {
        builder.AppendLine(line);
    }
    
    return builder.ToString();
}

// ✅ GOOD: String interpolation for few operations
var message = $"User {userId} performed {action} at {timestamp}";

// ❌ BAD: String concatenation in loop
string result = "";
foreach (var line in lines)
{
    result += line + "\n";  // Creates new string each time
}
```

### Collection Initialization

```csharp
// ✅ GOOD: Collection expressions (C# 12)
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob", "Charlie"];

// ✅ GOOD: Collection initializer
var users = new List<User>
{
    new User { Id = 1, Name = "Alice" },
    new User { Id = 2, Name = "Bob" }
};

// ✅ GOOD: Capacity hint
var largeList = new List<int>(capacity: 1000);
```

### Avoid Boxing

```csharp
// ✅ GOOD: Use generic collections
var numbers = new List<int> { 1, 2, 3 };  // No boxing

// ❌ BAD: Non-generic collections cause boxing
var numbers = new ArrayList { 1, 2, 3 };  // Boxing occurs
```

## Code Organization

### File Structure

```csharp
// ✅ GOOD: One public type per file
// File: OrderService.cs
namespace MyApp.Services;

public class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<Order> CreateOrderAsync(OrderRequest request)
    {
        // Implementation
    }
}

// Private helper classes in same file
internal class OrderValidator
{
    // Helper implementation
}
```

### Namespace Organization

```csharp
// ✅ GOOD: File-scoped namespace (C# 10+)
namespace MyApp.Services;

public class UserService
{
    // Implementation
}

// ✅ GOOD: Using statements
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Extensions.Logging;
using MyApp.Models;
```

## Comments and Documentation

### XML Documentation

```csharp
/// <summary>
/// Retrieves a user by their unique identifier.
/// </summary>
/// <param name="id">The unique identifier of the user.</param>
/// <param name="cancellationToken">Cancellation token.</param>
/// <returns>The user if found; otherwise, null.</returns>
/// <exception cref="ArgumentException">Thrown when id is less than 1.</exception>
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken cancellationToken = default)
{
    if (id < 1)
        throw new ArgumentException("Id must be positive", nameof(id));

    return await _repository.GetByIdAsync(id, cancellationToken);
}
```

### When to Comment

```csharp
// ✅ GOOD: Explain WHY, not WHAT
// Using exponential backoff to avoid overwhelming the API during outages
var delay = TimeSpan.FromSeconds(Math.Pow(2, retryCount));

// Complex regex requires explanation
// Match email: local-part@domain.tld
var emailRegex = @"^[^@\s]+@[^@\s]+\.[^@\s]+$";

// ❌ BAD: Stating the obvious
// Increment counter
counter++;

// Create new user
var user = new User();
```

**Remember**: C# emphasizes clarity, async programming, LINQ expressiveness, and leveraging the rich .NET ecosystem. Write idiomatic C# that follows framework guidelines and modern language features.
