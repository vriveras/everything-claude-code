---
name: csharp-system-patterns
description: C# system architecture patterns, API design, database integration, message queues, and enterprise backend patterns for .NET applications.
---

# C# System Development Patterns

Advanced system architecture patterns and best practices for scalable .NET applications.

## API Design Patterns

### RESTful API with ASP.NET Core

```csharp
// ✅ GOOD: Clean API structure with proper error handling
[ApiController]
[Route("api/[controller]")]
public class MarketsController : ControllerBase
{
    private readonly IMarketService _marketService;
    private readonly ILogger<MarketsController> _logger;
    
    public MarketsController(
        IMarketService marketService,
        ILogger<MarketsController> logger)
    {
        _marketService = marketService;
        _logger = logger;
    }
    
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<MarketDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<MarketDto>>> GetMarkets(
        [FromQuery] MarketQueryParameters parameters,
        CancellationToken cancellationToken)
    {
        try
        {
            var result = await _marketService.GetPagedAsync(parameters, cancellationToken);
            return Ok(result);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to fetch markets");
            return StatusCode(500, new ErrorResponse("Failed to fetch markets"));
        }
    }
    
    [HttpPost]
    [ProducesResponseType(typeof(MarketDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<MarketDto>> CreateMarket(
        [FromBody] CreateMarketRequest request,
        CancellationToken cancellationToken)
    {
        var market = await _marketService.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetMarket), new { id = market.Id }, market);
    }
}

// Query parameters with pagination
public class MarketQueryParameters
{
    [Range(1, int.MaxValue)]
    public int Page { get; set; } = 1;
    
    [Range(1, 100)]
    public int PageSize { get; set; } = 20;
    
    public string? Status { get; set; }
    public string? SortBy { get; set; }
    public string? SortOrder { get; set; } = "desc";
}

// Paged result wrapper
public record PagedResult<T>(
    IEnumerable<T> Items,
    int Page,
    int PageSize,
    int TotalCount,
    int TotalPages);
```

### API Versioning

```csharp
// ✅ GOOD: API versioning with ASP.NET Core
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version"));
});

// Version 1
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class MarketsV1Controller : ControllerBase
{
    [HttpGet]
    public async Task<IEnumerable<MarketDtoV1>> GetMarkets()
    {
        // V1 implementation
        return Array.Empty<MarketDtoV1>();
    }
}

// Version 2 with breaking changes
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class MarketsV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<PagedResult<MarketDtoV2>> GetMarkets(
        [FromQuery] MarketQueryParameters parameters)
    {
        // V2 implementation with pagination
        return new PagedResult<MarketDtoV2>(
            Array.Empty<MarketDtoV2>(), 1, 20, 0, 0);
    }
}
```

## Repository Pattern

### Generic Repository with Specifications

```csharp
// ✅ GOOD: Generic repository interface
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<IEnumerable<T>> FindAsync(
        ISpecification<T> specification,
        CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
}

// Specification pattern for queries
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>>? OrderBy { get; }
    Expression<Func<T, object>>? OrderByDescending { get; }
    int Take { get; }
    int Skip { get; }
    bool IsPagingEnabled { get; }
}

// Base specification implementation
public abstract class BaseSpecification<T> : ISpecification<T>
{
    protected BaseSpecification(Expression<Func<T, bool>>? criteria)
    {
        Criteria = criteria;
    }
    
    public Expression<Func<T, bool>>? Criteria { get; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; private set; }
    public Expression<Func<T, object>>? OrderByDescending { get; private set; }
    public int Take { get; private set; }
    public int Skip { get; private set; }
    public bool IsPagingEnabled { get; private set; }
    
    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }
    
    protected void ApplyPaging(int skip, int take)
    {
        Skip = skip;
        Take = take;
        IsPagingEnabled = true;
    }
    
    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }
    
    protected void ApplyOrderByDescending(Expression<Func<T, object>> orderByDescExpression)
    {
        OrderByDescending = orderByDescExpression;
    }
}

// Concrete specification
public class ActiveMarketsSpecification : BaseSpecification<Market>
{
    public ActiveMarketsSpecification(int page, int pageSize)
        : base(m => m.Status == MarketStatus.Active)
    {
        AddInclude(m => m.Creator);
        ApplyOrderByDescending(m => m.Volume);
        ApplyPaging((page - 1) * pageSize, pageSize);
    }
}

// Repository implementation with EF Core
public class EfRepository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    
    public EfRepository(DbContext context)
    {
        _context = context;
    }
    
    public async Task<IEnumerable<T>> FindAsync(
        ISpecification<T> spec,
        CancellationToken cancellationToken = default)
    {
        var query = ApplySpecification(spec);
        return await query.ToListAsync(cancellationToken);
    }
    
    private IQueryable<T> ApplySpecification(ISpecification<T> spec)
    {
        var query = _context.Set<T>().AsQueryable();
        
        if (spec.Criteria != null)
        {
            query = query.Where(spec.Criteria);
        }
        
        query = spec.Includes.Aggregate(query,
            (current, include) => current.Include(include));
        
        if (spec.OrderBy != null)
        {
            query = query.OrderBy(spec.OrderBy);
        }
        else if (spec.OrderByDescending != null)
        {
            query = query.OrderByDescending(spec.OrderByDescending);
        }
        
        if (spec.IsPagingEnabled)
        {
            query = query.Skip(spec.Skip).Take(spec.Take);
        }
        
        return query;
    }
}
```

## Unit of Work Pattern

```csharp
// ✅ GOOD: Unit of Work for transaction management
public interface IUnitOfWork : IDisposable
{
    IMarketRepository Markets { get; }
    IUserRepository Users { get; }
    IPositionRepository Positions { get; }
    
    Task<int> CompleteAsync(CancellationToken cancellationToken = default);
    Task BeginTransactionAsync(CancellationToken cancellationToken = default);
    Task CommitTransactionAsync(CancellationToken cancellationToken = default);
    Task RollbackTransactionAsync(CancellationToken cancellationToken = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _transaction;
    
    private IMarketRepository? _markets;
    private IUserRepository? _users;
    private IPositionRepository? _positions;
    
    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public IMarketRepository Markets => 
        _markets ??= new MarketRepository(_context);
    
    public IUserRepository Users => 
        _users ??= new UserRepository(_context);
    
    public IPositionRepository Positions => 
        _positions ??= new PositionRepository(_context);
    
    public async Task<int> CompleteAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }
    
    public async Task BeginTransactionAsync(CancellationToken cancellationToken = default)
    {
        _transaction = await _context.Database.BeginTransactionAsync(cancellationToken);
    }
    
    public async Task CommitTransactionAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.SaveChangesAsync(cancellationToken);
            await _transaction?.CommitAsync(cancellationToken)!;
        }
        catch
        {
            await RollbackTransactionAsync(cancellationToken);
            throw;
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }
    
    public async Task RollbackTransactionAsync(CancellationToken cancellationToken = default)
    {
        await _transaction?.RollbackAsync(cancellationToken)!;
        _transaction?.Dispose();
        _transaction = null;
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}

// Usage
public class MarketService
{
    private readonly IUnitOfWork _unitOfWork;
    
    public async Task CreateMarketWithPositionAsync(
        CreateMarketRequest marketRequest,
        CreatePositionRequest positionRequest)
    {
        await _unitOfWork.BeginTransactionAsync();
        
        try
        {
            var market = await _unitOfWork.Markets.AddAsync(
                new Market { Name = marketRequest.Name });
            
            await _unitOfWork.CompleteAsync();
            
            var position = await _unitOfWork.Positions.AddAsync(
                new Position { MarketId = market.Id });
            
            await _unitOfWork.CompleteAsync();
            await _unitOfWork.CommitTransactionAsync();
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}
```

## Caching Strategies

### Distributed Caching with Redis

```csharp
// ✅ GOOD: Cache-aside pattern with IDistributedCache
public class CachedMarketService : IMarketService
{
    private readonly IMarketService _innerService;
    private readonly IDistributedCache _cache;
    private readonly ILogger<CachedMarketService> _logger;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);
    
    public CachedMarketService(
        IMarketService innerService,
        IDistributedCache cache,
        ILogger<CachedMarketService> logger)
    {
        _innerService = innerService;
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<Market?> GetByIdAsync(
        string id,
        CancellationToken cancellationToken = default)
    {
        var cacheKey = $"market:{id}";
        
        // Try cache first
        var cachedData = await _cache.GetStringAsync(cacheKey, cancellationToken);
        if (cachedData != null)
        {
            _logger.LogDebug("Cache hit for market {MarketId}", id);
            return JsonSerializer.Deserialize<Market>(cachedData);
        }
        
        // Cache miss - fetch from service
        _logger.LogDebug("Cache miss for market {MarketId}", id);
        var market = await _innerService.GetByIdAsync(id, cancellationToken);
        
        if (market != null)
        {
            // Store in cache
            var serialized = JsonSerializer.Serialize(market);
            await _cache.SetStringAsync(
                cacheKey,
                serialized,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = CacheDuration
                },
                cancellationToken);
        }
        
        return market;
    }
    
    public async Task InvalidateCacheAsync(string id)
    {
        await _cache.RemoveAsync($"market:{id}");
    }
}

// Configuration
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration["Redis:ConnectionString"];
    options.InstanceName = "Markets:";
});

services.AddScoped<IMarketService, MarketService>();
services.Decorate<IMarketService, CachedMarketService>();
```

### In-Memory Caching with IMemoryCache

```csharp
// ✅ GOOD: Memory cache with eviction policies
public class MemoryCachedMarketService : IMarketService
{
    private readonly IMarketService _innerService;
    private readonly IMemoryCache _cache;
    private readonly ILogger<MemoryCachedMarketService> _logger;
    
    public async Task<Market?> GetByIdAsync(string id, CancellationToken cancellationToken = default)
    {
        return await _cache.GetOrCreateAsync(
            $"market:{id}",
            async entry =>
            {
                entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(5));
                entry.SetSlidingExpiration(TimeSpan.FromMinutes(2));
                entry.SetSize(1);
                entry.RegisterPostEvictionCallback(OnEviction);
                
                _logger.LogDebug("Fetching market {MarketId} from service", id);
                return await _innerService.GetByIdAsync(id, cancellationToken);
            });
    }
    
    private void OnEviction(object key, object? value, EvictionReason reason, object? state)
    {
        _logger.LogDebug("Cache entry {Key} evicted: {Reason}", key, reason);
    }
}

// Configure memory cache with size limit
services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // Max 1024 entries
});
```

## Message Queue Patterns

### Background Processing with Channels

```csharp
// ✅ GOOD: Channel-based message queue
public interface IMessageQueue<T>
{
    ValueTask EnqueueAsync(T message, CancellationToken cancellationToken = default);
}

public class ChannelMessageQueue<T> : IMessageQueue<T>
{
    private readonly Channel<T> _channel;
    
    public ChannelMessageQueue(int capacity = 1000)
    {
        var options = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };
        _channel = Channel.CreateBounded<T>(options);
    }
    
    public async ValueTask EnqueueAsync(T message, CancellationToken cancellationToken = default)
    {
        await _channel.Writer.WriteAsync(message, cancellationToken);
    }
    
    public ChannelReader<T> Reader => _channel.Reader;
}

// Background service to process messages
public class MarketIndexingBackgroundService : BackgroundService
{
    private readonly ChannelMessageQueue<string> _queue;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<MarketIndexingBackgroundService> _logger;
    
    public MarketIndexingBackgroundService(
        ChannelMessageQueue<string> queue,
        IServiceProvider serviceProvider,
        ILogger<MarketIndexingBackgroundService> logger)
    {
        _queue = queue;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var marketId in _queue.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await ProcessMarketAsync(marketId, stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process market {MarketId}", marketId);
            }
        }
    }
    
    private async Task ProcessMarketAsync(string marketId, CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var indexer = scope.ServiceProvider.GetRequiredService<IMarketIndexer>();
        
        await indexer.IndexAsync(marketId, cancellationToken);
    }
}

// Usage
await _messageQueue.EnqueueAsync(marketId);
```

### RabbitMQ Integration

```csharp
// ✅ GOOD: RabbitMQ publisher
public interface IMessagePublisher
{
    Task PublishAsync<T>(string exchange, string routingKey, T message);
}

public class RabbitMqPublisher : IMessagePublisher, IDisposable
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly ILogger<RabbitMqPublisher> _logger;
    
    public RabbitMqPublisher(IOptions<RabbitMqOptions> options, ILogger<RabbitMqPublisher> logger)
    {
        _logger = logger;
        
        var factory = new ConnectionFactory
        {
            HostName = options.Value.Host,
            Port = options.Value.Port,
            UserName = options.Value.Username,
            Password = options.Value.Password,
            DispatchConsumersAsync = true
        };
        
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }
    
    public Task PublishAsync<T>(string exchange, string routingKey, T message)
    {
        var json = JsonSerializer.Serialize(message);
        var body = Encoding.UTF8.GetBytes(json);
        
        var properties = _channel.CreateBasicProperties();
        properties.Persistent = true;
        properties.ContentType = "application/json";
        
        _channel.BasicPublish(
            exchange: exchange,
            routingKey: routingKey,
            basicProperties: properties,
            body: body);
        
        _logger.LogDebug("Published message to {Exchange}/{RoutingKey}", exchange, routingKey);
        
        return Task.CompletedTask;
    }
    
    public void Dispose()
    {
        _channel?.Dispose();
        _connection?.Dispose();
    }
}

// Consumer
public class MarketEventConsumer : BackgroundService
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    private readonly IServiceProvider _serviceProvider;
    
    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var consumer = new AsyncEventingBasicConsumer(_channel);
        
        consumer.Received += async (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var json = Encoding.UTF8.GetString(body);
            var marketEvent = JsonSerializer.Deserialize<MarketEvent>(json);
            
            try
            {
                await ProcessEventAsync(marketEvent);
                _channel.BasicAck(ea.DeliveryTag, false);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process event");
                _channel.BasicNack(ea.DeliveryTag, false, true);
            }
        };
        
        _channel.BasicConsume(
            queue: "market-events",
            autoAck: false,
            consumer: consumer);
        
        return Task.CompletedTask;
    }
}
```

## Resilience Patterns with Polly

```csharp
// ✅ GOOD: Retry with exponential backoff
public class ResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _retryPolicy;
    private readonly IAsyncPolicy<HttpResponseMessage> _circuitBreakerPolicy;
    
    public ResilientHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
        
        // Retry policy with exponential backoff
        _retryPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<HttpRequestException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => 
                    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    Console.WriteLine($"Retry {retryCount} after {timespan}");
                });
        
        // Circuit breaker
        _circuitBreakerPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<HttpRequestException>()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (outcome, duration) =>
                {
                    Console.WriteLine($"Circuit breaker opened for {duration}");
                },
                onReset: () =>
                {
                    Console.WriteLine("Circuit breaker reset");
                });
    }
    
    public async Task<HttpResponseMessage> GetAsync(string url)
    {
        var policy = Policy.WrapAsync(_retryPolicy, _circuitBreakerPolicy);
        return await policy.ExecuteAsync(() => _httpClient.GetAsync(url));
    }
}

// Configuration with Polly
services.AddHttpClient<IMarketApiClient, MarketApiClient>()
    .AddTransientHttpErrorPolicy(builder => 
        builder.WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))))
    .AddTransientHttpErrorPolicy(builder =>
        builder.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

## Health Checks

```csharp
// ✅ GOOD: Custom health checks
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;
    
    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database is reachable");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is unreachable", ex);
        }
    }
}

public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = _redis.GetDatabase();
            await db.PingAsync();
            return HealthCheckResult.Healthy("Redis is reachable");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis is unreachable", ex);
        }
    }
}

// Configuration
services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<RedisHealthCheck>("redis")
    .AddUrlGroup(new Uri("https://api.example.com/health"), "api");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description
            })
        });
        await context.Response.WriteAsync(result);
    }
});
```

## Event Sourcing Pattern

```csharp
// ✅ GOOD: Event sourcing with domain events
public abstract record DomainEvent(Guid AggregateId, DateTime OccurredAt);

public record MarketCreatedEvent(
    Guid AggregateId,
    DateTime OccurredAt,
    string Name,
    string Description) : DomainEvent(Aggregate Id, OccurredAt);

public record MarketStatusChangedEvent(
    Guid AggregateId,
    DateTime OccurredAt,
    MarketStatus OldStatus,
    MarketStatus NewStatus) : DomainEvent(AggregateId, OccurredAt);

// Event store
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, int expectedVersion);
    Task<IEnumerable<DomainEvent>> GetEventsAsync(Guid aggregateId);
}

public class EventStore : IEventStore
{
    private readonly ApplicationDbContext _context;
    
    public async Task SaveEventsAsync(
        Guid aggregateId,
        IEnumerable<DomainEvent> events,
        int expectedVersion)
    {
        foreach (var @event in events)
        {
            var eventEntity = new EventEntity
            {
                AggregateId = aggregateId,
                EventType = @event.GetType().Name,
                Data = JsonSerializer.Serialize(@event),
                OccurredAt = @event.OccurredAt,
                Version = expectedVersion++
            };
            
            _context.Events.Add(eventEntity);
        }
        
        await _context.SaveChangesAsync();
    }
    
    public async Task<IEnumerable<DomainEvent>> GetEventsAsync(Guid aggregateId)
    {
        var events = await _context.Events
            .Where(e => e.AggregateId == aggregateId)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        return events.Select(e => 
            JsonSerializer.Deserialize<DomainEvent>(e.Data)!);
    }
}

// Aggregate root
public class Market
{
    private readonly List<DomainEvent> _uncommittedEvents = new();
    
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public MarketStatus Status { get; private set; }
    public int Version { get; private set; }
    
    public static Market Create(Guid id, string name, string description)
    {
        var market = new Market();
        var @event = new MarketCreatedEvent(id, DateTime.UtcNow, name, description);
        market.Apply(@event);
        market._uncommittedEvents.Add(@event);
        return market;
    }
    
    public void ChangeStatus(MarketStatus newStatus)
    {
        if (Status == newStatus) return;
        
        var @event = new MarketStatusChangedEvent(
            Id, DateTime.UtcNow, Status, newStatus);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }
    
    public IEnumerable<DomainEvent> GetUncommittedEvents() => _uncommittedEvents;
    
    public void MarkEventsAsCommitted() => _uncommittedEvents.Clear();
    
    public void LoadFromHistory(IEnumerable<DomainEvent> events)
    {
        foreach (var @event in events)
        {
            Apply(@event);
            Version++;
        }
    }
    
    private void Apply(DomainEvent @event)
    {
        switch (@event)
        {
            case MarketCreatedEvent e:
                Id = e.AggregateId;
                Name = e.Name;
                Status = MarketStatus.Draft;
                break;
            
            case MarketStatusChangedEvent e:
                Status = e.NewStatus;
                break;
        }
    }
}
```

**Remember**: C# and .NET provide rich patterns for building scalable, maintainable enterprise systems. Use dependency injection, async/await, and framework features for robust code.
