---
name: csharp-system-development
description: C# coding standards, best practices, and system development patterns for .NET applications, high-performance services, and enterprise systems.
---

# C# System Development Standards

System-level coding standards and patterns for enterprise C# and .NET development.

## Code Quality Principles

### 1. Readability First
- Code is read more than written
- Clear variable and method names
- Self-documenting code preferred over comments
- Consistent formatting (use .editorconfig)

### 2. KISS (Keep It Simple, Stupid)
- Simplest solution that works
- Avoid over-engineering
- Profile before optimizing
- Clear code > clever code

### 3. DRY (Don't Repeat Yourself)
- Extract common logic into methods
- Create reusable components
- Share utilities across modules
- Avoid copy-paste programming

### 4. Dispose Pattern (IDisposable)
- Implement IDisposable for unmanaged resources
- Use 'using' statement for automatic disposal
- Ensure cleanup in all code paths
- Follow dispose pattern correctly

## C# Standards

### Naming Conventions

```csharp
// ✅ GOOD: PascalCase for public members
public class MarketProcessor 
{
    public string MarketSearchQuery { get; set; } = "election";
    public bool IsUserAuthenticated { get; set; } = true;
    public long TotalRevenue { get; set; } = 1000;
    
    public void ProcessMarketData() { }
}

// ✅ GOOD: camelCase for private fields with underscore
public class UserService 
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger<UserService> _logger;
    
    public UserService(IUserRepository userRepository, ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _logger = logger;
    }
}

// ❌ BAD: Unclear or inconsistent names
public class processor
{
    public string q;
    public bool flag;
    public int x;
}
```

### Method Naming

```csharp
// ✅ GOOD: PascalCase verb-noun pattern
public async Task<MarketData> FetchMarketDataAsync(string marketId)
{
    // Implementation
}

public double CalculateSimilarity(List<double> a, List<double> b)
{
    // Implementation
}

public bool IsValidEmail(string email)
{
    // Implementation
}

// ❌ BAD: Unclear or noun-only
public async Task<object> Market(string id) { }
public double Similarity(object a, object b) { }
public bool Email(string e) { }
```

### Resource Management with IDisposable

```csharp
// ✅ GOOD: IDisposable pattern implementation
public class FileProcessor : IDisposable
{
    private FileStream? _fileStream;
    private bool _disposed = false;
    
    public FileProcessor(string path)
    {
        _fileStream = File.OpenRead(path);
    }
    
    public void Process()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(FileProcessor));
            
        // Process file
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Dispose managed resources
                _fileStream?.Dispose();
            }
            
            // Free unmanaged resources
            _disposed = true;
        }
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}

// ✅ GOOD: Using statement for automatic disposal
using (var processor = new FileProcessor("data.txt"))
{
    processor.Process();
} // Automatically disposed

// ✅ GOOD: Using declaration (C# 8.0+)
using var processor = new FileProcessor("data.txt");
processor.Process();
// Disposed at end of scope
```

### Error Handling

```csharp
// ✅ GOOD: Comprehensive exception handling
public async Task<Data> FetchDataAsync(string url)
{
    try
    {
        using var client = new HttpClient();
        var response = await client.GetAsync(url);
        
        if (!response.IsSuccessStatusCode)
        {
            throw new HttpRequestException(
                $"HTTP {(int)response.StatusCode}: {response.ReasonPhrase}");
        }
        
        return await response.Content.ReadFromJsonAsync<Data>()
            ?? throw new InvalidDataException("Response body is null");
    }
    catch (HttpRequestException ex)
    {
        _logger.LogError(ex, "Failed to fetch data from {Url}", url);
        throw;
    }
    catch (JsonException ex)
    {
        _logger.LogError(ex, "Failed to deserialize response");
        throw new InvalidDataException("Invalid JSON response", ex);
    }
}

// ✅ GOOD: Custom exceptions
public class MarketNotFoundException : Exception
{
    public string MarketId { get; }
    
    public MarketNotFoundException(string marketId)
        : base($"Market with ID {marketId} not found")
    {
        MarketId = marketId;
    }
    
    public MarketNotFoundException(string marketId, Exception inner)
        : base($"Market with ID {marketId} not found", inner)
    {
        MarketId = marketId;
    }
}
```

### Async/Await Best Practices

```csharp
// ✅ GOOD: Parallel execution when possible
var tasks = new[]
{
    FetchUsersAsync(),
    FetchMarketsAsync(),
    FetchStatsAsync()
};

var results = await Task.WhenAll(tasks);
var users = results[0];
var markets = results[1];
var stats = results[2];

// ✅ GOOD: ConfigureAwait(false) in libraries
public async Task<Data> GetDataAsync()
{
    var response = await _httpClient.GetAsync(url).ConfigureAwait(false);
    return await response.Content.ReadFromJsonAsync<Data>().ConfigureAwait(false);
}

// ❌ BAD: Sequential when unnecessary
var users = await FetchUsersAsync();
var markets = await FetchMarketsAsync();
var stats = await FetchStatsAsync();

// ❌ BAD: Async void (except event handlers)
public async void ProcessData() // BAD - exceptions can't be caught
{
    await FetchDataAsync();
}

// ✅ GOOD: Async Task
public async Task ProcessDataAsync()
{
    await FetchDataAsync();
}
```

### Nullable Reference Types (C# 8.0+)

```csharp
#nullable enable

// ✅ GOOD: Explicit nullability
public class Market
{
    public string Id { get; set; } = string.Empty;
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; } // Nullable
    public DateTime CreatedAt { get; set; }
}

public Market? FindMarket(string id)
{
    return _markets.FirstOrDefault(m => m.Id == id);
}

// ✅ GOOD: Null checking with pattern matching
if (FindMarket(id) is { } market)
{
    Console.WriteLine(market.Name);
}

// ✅ GOOD: Null-coalescing operators
var name = market?.Name ?? "Unknown";
var description = market?.Description ?? throw new ArgumentException("Market required");
```

### Modern C# Features

```csharp
// ✅ GOOD: Pattern matching
public decimal CalculateDiscount(Customer customer) => customer switch
{
    { IsPremium: true, YearsActive: > 5 } => 0.20m,
    { IsPremium: true } => 0.15m,
    { YearsActive: > 3 } => 0.10m,
    _ => 0.05m
};

// ✅ GOOD: Records for immutable data
public record Market(string Id, string Name, MarketStatus Status, DateTime CreatedAt);

// ✅ GOOD: Init-only properties
public class MarketDto
{
    public string Id { get; init; } = string.Empty;
    public string Name { get; init; } = string.Empty;
}

// ✅ GOOD: Target-typed new (C# 9.0+)
List<Market> markets = new();
Dictionary<string, User> users = new();

// ✅ GOOD: LINQ for data manipulation
var activeMarkets = markets
    .Where(m => m.Status == MarketStatus.Active)
    .OrderByDescending(m => m.Volume)
    .Take(10)
    .ToList();
```

## System Development Patterns

### Dependency Injection

```csharp
// ✅ GOOD: Constructor injection with interfaces
public class MarketService
{
    private readonly IMarketRepository _repository;
    private readonly ILogger<MarketService> _logger;
    private readonly IMemoryCache _cache;
    
    public MarketService(
        IMarketRepository repository,
        ILogger<MarketService> logger,
        IMemoryCache cache)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _cache = cache ?? throw new ArgumentNullException(nameof(cache));
    }
    
    public async Task<Market?> GetMarketAsync(string id)
    {
        var cacheKey = $"market:{id}";
        
        if (_cache.TryGetValue<Market>(cacheKey, out var market))
        {
            return market;
        }
        
        market = await _repository.GetByIdAsync(id);
        
        if (market != null)
        {
            _cache.Set(cacheKey, market, TimeSpan.FromMinutes(5));
        }
        
        return market;
    }
}

// ✅ GOOD: Service registration
services.AddScoped<IMarketRepository, MarketRepository>();
services.AddScoped<IMarketService, MarketService>();
services.AddMemoryCache();
services.AddLogging();
```

### Repository Pattern

```csharp
// ✅ GOOD: Generic repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(string id, CancellationToken cancellationToken = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(string id, CancellationToken cancellationToken = default);
}

// ✅ GOOD: Specific repository with query methods
public interface IMarketRepository : IRepository<Market>
{
    Task<IEnumerable<Market>> GetActiveMarketsAsync(CancellationToken cancellationToken = default);
    Task<IEnumerable<Market>> SearchAsync(string query, int limit, CancellationToken cancellationToken = default);
}

// ✅ GOOD: Implementation with Entity Framework Core
public class MarketRepository : IMarketRepository
{
    private readonly ApplicationDbContext _context;
    
    public MarketRepository(ApplicationDbContext context)
    {
        _context = context ?? throw new ArgumentNullException(nameof(context));
    }
    
    public async Task<Market?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
    {
        return await _context.Markets
            .AsNoTracking()
            .FirstOrDefaultAsync(m => m.Id == id, cancellationToken);
    }
    
    public async Task<IEnumerable<Market>> GetActiveMarketsAsync(CancellationToken cancellationToken = default)
    {
        return await _context.Markets
            .AsNoTracking()
            .Where(m => m.Status == MarketStatus.Active)
            .OrderByDescending(m => m.Volume)
            .ToListAsync(cancellationToken);
    }
}
```

### Concurrency Patterns

```csharp
// ✅ GOOD: Thread-safe singleton with Lazy<T>
public sealed class ConfigurationManager
{
    private static readonly Lazy<ConfigurationManager> _instance = 
        new(() => new ConfigurationManager());
    
    public static ConfigurationManager Instance => _instance.Value;
    
    private ConfigurationManager()
    {
        // Load configuration
    }
}

// ✅ GOOD: Thread-safe collection operations
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, TValue> _cache = new();
    
    public bool TryAdd(TKey key, TValue value)
    {
        return _cache.TryAdd(key, value);
    }
    
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory)
    {
        return _cache.GetOrAdd(key, valueFactory);
    }
    
    public bool TryRemove(TKey key, out TValue? value)
    {
        return _cache.TryRemove(key, out value);
    }
}

// ✅ GOOD: SemaphoreSlim for async synchronization
public class AsyncRateLimiter
{
    private readonly SemaphoreSlim _semaphore;
    private readonly TimeSpan _timeUnit;
    
    public AsyncRateLimiter(int maxConcurrent, TimeSpan timeUnit)
    {
        _semaphore = new SemaphoreSlim(maxConcurrent, maxConcurrent);
        _timeUnit = timeUnit;
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        await _semaphore.WaitAsync();
        
        try
        {
            return await action();
        }
        finally
        {
            _ = Task.Delay(_timeUnit).ContinueWith(_ => _semaphore.Release());
        }
    }
}
```

### Task Parallel Library (TPL)

```csharp
// ✅ GOOD: Parallel processing with TPL
public async Task ProcessMarketsAsync(List<Market> markets)
{
    await Parallel.ForEachAsync(markets, 
        new ParallelOptions { MaxDegreeOfParallelism = 4 },
        async (market, ct) =>
        {
            await ProcessMarketAsync(market, ct);
        });
}

// ✅ GOOD: Dataflow blocks for pipeline processing
public class DataPipeline
{
    private readonly TransformBlock<string, Market> _fetchBlock;
    private readonly TransformBlock<Market, MarketData> _processBlock;
    private readonly ActionBlock<MarketData> _saveBlock;
    
    public DataPipeline()
    {
        _fetchBlock = new TransformBlock<string, Market>(
            async id => await FetchMarketAsync(id),
            new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4 });
            
        _processBlock = new TransformBlock<Market, MarketData>(
            async market => await ProcessMarketAsync(market),
            new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4 });
            
        _saveBlock = new ActionBlock<MarketData>(
            async data => await SaveMarketDataAsync(data),
            new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 2 });
        
        _fetchBlock.LinkTo(_processBlock, new DataflowLinkOptions { PropagateCompletion = true });
        _processBlock.LinkTo(_saveBlock, new DataflowLinkOptions { PropagateCompletion = true });
    }
    
    public async Task ProcessAsync(IEnumerable<string> marketIds)
    {
        foreach (var id in marketIds)
        {
            await _fetchBlock.SendAsync(id);
        }
        
        _fetchBlock.Complete();
        await _saveBlock.Completion;
    }
}
```

### Memory-Efficient Patterns

```csharp
// ✅ GOOD: Span<T> and Memory<T> for zero-copy operations
public int CountOccurrences(ReadOnlySpan<char> text, ReadOnlySpan<char> pattern)
{
    int count = 0;
    int index = 0;
    
    while ((index = text.Slice(index).IndexOf(pattern)) >= 0)
    {
        count++;
        index += pattern.Length;
    }
    
    return count;
}

// ✅ GOOD: ArrayPool for reducing allocations
public class BufferProcessor
{
    private static readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;
    
    public void ProcessData(int size)
    {
        byte[] buffer = _pool.Rent(size);
        
        try
        {
            // Use buffer
            ProcessBuffer(buffer.AsSpan(0, size));
        }
        finally
        {
            _pool.Return(buffer);
        }
    }
}

// ✅ GOOD: ValueTask for hot path optimization
public ValueTask<int> GetCachedValueAsync(string key)
{
    if (_cache.TryGetValue(key, out int value))
    {
        return new ValueTask<int>(value); // Synchronous completion
    }
    
    return new ValueTask<int>(FetchFromDatabaseAsync(key));
}
```

### High-Performance I/O

```csharp
// ✅ GOOD: PipeReader for efficient stream processing
public async Task ProcessStreamAsync(Stream stream)
{
    var reader = PipeReader.Create(stream);
    
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        while (TryParseLine(ref buffer, out ReadOnlySequence<byte> line))
        {
            ProcessLine(line);
        }
        
        reader.AdvanceTo(buffer.Start, buffer.End);
        
        if (result.IsCompleted)
            break;
    }
    
    await reader.CompleteAsync();
}

// ✅ GOOD: Memory-mapped files for large file processing
public class MappedFileReader : IDisposable
{
    private readonly MemoryMappedFile _mappedFile;
    private readonly MemoryMappedViewAccessor _accessor;
    
    public MappedFileReader(string path)
    {
        _mappedFile = MemoryMappedFile.CreateFromFile(path, FileMode.Open);
        _accessor = _mappedFile.CreateViewAccessor(0, 0, MemoryMappedFileAccess.Read);
    }
    
    public ReadOnlySpan<byte> Read(long offset, int length)
    {
        byte[] buffer = new byte[length];
        _accessor.ReadArray(offset, buffer, 0, length);
        return buffer;
    }
    
    public void Dispose()
    {
        _accessor.Dispose();
        _mappedFile.Dispose();
    }
}
```

### Background Services

```csharp
// ✅ GOOD: Hosted service for background processing
public class MarketIndexingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<MarketIndexingService> _logger;
    
    public MarketIndexingService(
        IServiceProvider serviceProvider,
        ILogger<MarketIndexingService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Market indexing service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await IndexMarketsAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Service is stopping
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error indexing markets");
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
            }
        }
        
        _logger.LogInformation("Market indexing service stopped");
    }
    
    private async Task IndexMarketsAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var marketService = scope.ServiceProvider.GetRequiredService<IMarketService>();
        
        await marketService.IndexAllAsync(cancellationToken);
    }
}

// Register the service
services.AddHostedService<MarketIndexingService>();
```

## API Development Patterns

### Minimal APIs (ASP.NET Core)

```csharp
// ✅ GOOD: Minimal API with proper routing and validation
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IMarketService, MarketService>();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.MapGet("/api/markets", async (IMarketService marketService) =>
{
    var markets = await marketService.GetAllAsync();
    return Results.Ok(new { Success = true, Data = markets });
});

app.MapGet("/api/markets/{id}", async (string id, IMarketService marketService) =>
{
    var market = await marketService.GetByIdAsync(id);
    return market is not null
        ? Results.Ok(new { Success = true, Data = market })
        : Results.NotFound(new { Success = false, Error = "Market not found" });
});

app.MapPost("/api/markets", async (CreateMarketRequest request, IMarketService marketService) =>
{
    var market = await marketService.CreateAsync(request);
    return Results.Created($"/api/markets/{market.Id}", new { Success = true, Data = market });
})
.WithName("CreateMarket")
.WithOpenApi();

app.Run();
```

### Controller-based APIs

```csharp
// ✅ GOOD: RESTful controller with proper error handling
[ApiController]
[Route("api/[controller]")]
public class MarketsController : ControllerBase
{
    private readonly IMarketService _marketService;
    private readonly ILogger<MarketsController> _logger;
    
    public MarketsController(IMarketService marketService, ILogger<MarketsController> logger)
    {
        _marketService = marketService;
        _logger = logger;
    }
    
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<IEnumerable<Market>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<IEnumerable<Market>>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        var markets = await _marketService.GetPagedAsync(page, pageSize);
        return Ok(new ApiResponse<IEnumerable<Market>>(true, markets));
    }
    
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ApiResponse<Market>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<Market>>> GetById(string id)
    {
        var market = await _marketService.GetByIdAsync(id);
        
        if (market is null)
        {
            return NotFound(new ApiResponse<object>(false, error: "Market not found"));
        }
        
        return Ok(new ApiResponse<Market>(true, market));
    }
    
    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<Market>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ApiResponse<Market>>> Create(
        [FromBody] CreateMarketRequest request)
    {
        try
        {
            var market = await _marketService.CreateAsync(request);
            return CreatedAtAction(
                nameof(GetById),
                new { id = market.Id },
                new ApiResponse<Market>(true, market));
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation failed for market creation");
            return BadRequest(new ApiResponse<object>(false, error: ex.Message));
        }
    }
}

// ✅ GOOD: Generic API response wrapper
public record ApiResponse<T>(bool Success, T? Data = default, string? Error = null);
```

## Testing Standards

### Unit Testing with xUnit

```csharp
// ✅ GOOD: Descriptive test names and AAA pattern
public class MarketServiceTests
{
    [Fact]
    public async Task GetByIdAsync_ReturnsNull_WhenMarketDoesNotExist()
    {
        // Arrange
        var mockRepo = new Mock<IMarketRepository>();
        mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<string>(), default))
            .ReturnsAsync((Market?)null);
        
        var service = new MarketService(mockRepo.Object);
        
        // Act
        var result = await service.GetByIdAsync("nonexistent");
        
        // Assert
        Assert.Null(result);
    }
    
    [Theory]
    [InlineData("")]
    [InlineData(null)]
    public async Task CreateAsync_ThrowsArgumentException_WhenNameIsNullOrEmpty(string name)
    {
        // Arrange
        var mockRepo = new Mock<IMarketRepository>();
        var service = new MarketService(mockRepo.Object);
        var request = new CreateMarketRequest { Name = name };
        
        // Act & Assert
        await Assert.ThrowsAsync<ArgumentException>(() => service.CreateAsync(request));
    }
}

// ✅ GOOD: Integration testing with WebApplicationFactory
public class MarketsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public MarketsApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task GetMarkets_ReturnsSuccessStatusCode()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/api/markets");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        Assert.Contains("\"success\":true", content);
    }
}
```

## Code Smell Detection

### Long Methods
```csharp
// ❌ BAD: Method > 50 lines
public void ProcessMarketData()
{
    // 100 lines of code
}

// ✅ GOOD: Split into smaller methods
public void ProcessMarketData()
{
    var validated = ValidateData();
    var transformed = TransformData(validated);
    SaveData(transformed);
}
```

### Improper Exception Handling
```csharp
// ❌ BAD: Swallowing exceptions
try
{
    ProcessData();
}
catch { } // BAD

// ✅ GOOD: Proper exception handling
try
{
    ProcessData();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process data");
    throw;
}
```

### Magic Strings and Numbers
```csharp
// ❌ BAD: Magic values
if (retryCount > 3) { }
await Task.Delay(500);

// ✅ GOOD: Named constants
private const int MaxRetries = 3;
private static readonly TimeSpan RetryDelay = TimeSpan.FromMilliseconds(500);

if (retryCount > MaxRetries) { }
await Task.Delay(RetryDelay);
```

**Remember**: Modern C# and .NET enable building robust, high-performance enterprise systems. Use dependency injection, async/await, and built-in patterns for maintainable code.
