---
name: csharp-system-patterns
description: C# system programming patterns, async/await, memory management, networking, and Windows system integration best practices.
---

# C# System Development Patterns

System programming patterns and best practices for C# applications with focus on performance and system integration.

## Memory Management Patterns

### IDisposable Pattern

```csharp
// ✅ GOOD: Proper IDisposable implementation
public class FileProcessor : IDisposable
{
    private FileStream _fileStream;
    private bool _disposed = false;

    public FileProcessor(string path)
    {
        _fileStream = new FileStream(path, FileMode.Open);
    }

    public void Process()
    {
        // Processing logic
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

            // Free unmanaged resources if any
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    ~FileProcessor()
    {
        Dispose(false);
    }
}

// Usage with using statement
using (var processor = new FileProcessor("data.txt"))
{
    processor.Process();
} // Automatically disposed
```

### Modern Using Declaration (C# 8.0+)

```csharp
// ✅ GOOD: Simplified using declaration
public void ProcessFile(string path)
{
    using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    
    string content = reader.ReadToEnd();
    // Process content
    
} // Automatically disposed at end of scope
```

### Memory<T> and Span<T> for Zero-Copy Operations

```csharp
// ✅ GOOD: Using Span<T> for efficient memory operations
public static int ParseInt32(ReadOnlySpan<char> text)
{
    int result = 0;
    foreach (char c in text)
    {
        if (c < '0' || c > '9')
            throw new FormatException();
        result = result * 10 + (c - '0');
    }
    return result;
}

// Usage - no allocation
string numberStr = "12345";
int value = ParseInt32(numberStr.AsSpan());

// ✅ GOOD: Slicing without allocation
ReadOnlySpan<byte> data = GetData();
ReadOnlySpan<byte> header = data.Slice(0, 16);
ReadOnlySpan<byte> body = data.Slice(16);
```

### ArrayPool for Reducing Allocations

```csharp
using System.Buffers;

// ✅ GOOD: Rent and return buffers
public byte[] ProcessData(int size)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(size);
    try
    {
        // Use buffer
        FillBuffer(buffer, size);
        return buffer.Take(size).ToArray();
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

## Async/Await Patterns

### Async All the Way

```csharp
// ✅ GOOD: Async throughout the call stack
public async Task<string> FetchDataAsync(string url)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(url);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
}

public async Task ProcessDataAsync()
{
    string data = await FetchDataAsync("https://api.example.com/data");
    await SaveToFileAsync(data);
}

// ❌ BAD: Blocking on async code
public string FetchData(string url)
{
    return FetchDataAsync(url).Result; // Can cause deadlocks!
}
```

### ConfigureAwait for Library Code

```csharp
// ✅ GOOD: Use ConfigureAwait(false) in library code
public async Task<Data> GetDataAsync()
{
    var response = await _httpClient
        .GetAsync(_endpoint)
        .ConfigureAwait(false);
    
    var content = await response.Content
        .ReadAsStringAsync()
        .ConfigureAwait(false);
    
    return ParseData(content);
}
```

### Task.WhenAll for Parallel Operations

```csharp
// ✅ GOOD: Execute multiple async operations in parallel
public async Task<(User user, Order[] orders, Product[] products)> GetUserDashboardAsync(int userId)
{
    var userTask = GetUserAsync(userId);
    var ordersTask = GetUserOrdersAsync(userId);
    var productsTask = GetRecommendedProductsAsync(userId);

    await Task.WhenAll(userTask, ordersTask, productsTask);

    return (
        await userTask,
        await ordersTask,
        await productsTask
    );
}

// ❌ BAD: Sequential execution
public async Task<(User, Order[], Product[])> GetUserDashboardAsync(int userId)
{
    var user = await GetUserAsync(userId);
    var orders = await GetUserOrdersAsync(userId);
    var products = await GetRecommendedProductsAsync(userId);
    return (user, orders, products);
}
```

### Cancellation Tokens

```csharp
// ✅ GOOD: Support cancellation
public async Task<string> DownloadFileAsync(string url, CancellationToken cancellationToken)
{
    using var client = new HttpClient();
    using var response = await client.GetAsync(url, cancellationToken);
    
    response.EnsureSuccessStatusCode();
    
    return await response.Content.ReadAsStringAsync();
}

// Usage with timeout
using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30)))
{
    try
    {
        var result = await DownloadFileAsync(url, cts.Token);
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("Download timed out");
    }
}
```

### AsyncLocal for Async Context

```csharp
// ✅ GOOD: Store context that flows with async operations
public class RequestContext
{
    private static readonly AsyncLocal<string> _requestId = new();

    public static string RequestId
    {
        get => _requestId.Value ?? string.Empty;
        set => _requestId.Value = value;
    }
}

public async Task ProcessRequestAsync()
{
    RequestContext.RequestId = Guid.NewGuid().ToString();
    
    await Task.Delay(100);
    
    // RequestId is preserved across await boundaries
    Console.WriteLine($"Processing: {RequestContext.RequestId}");
}
```

## Concurrency Patterns

### Thread-Safe Collections

```csharp
using System.Collections.Concurrent;

// ✅ GOOD: Use concurrent collections
public class JobQueue
{
    private readonly ConcurrentQueue<Job> _queue = new();
    private readonly SemaphoreSlim _signal = new(0);

    public void Enqueue(Job job)
    {
        _queue.Enqueue(job);
        _signal.Release();
    }

    public async Task<Job> DequeueAsync(CancellationToken cancellationToken)
    {
        await _signal.WaitAsync(cancellationToken);
        
        if (_queue.TryDequeue(out var job))
        {
            return job;
        }
        
        throw new InvalidOperationException("Queue is empty");
    }
}
```

### Channel for Producer-Consumer Pattern

```csharp
using System.Threading.Channels;

// ✅ GOOD: Modern producer-consumer with Channels
public class DataPipeline
{
    private readonly Channel<string> _channel;

    public DataPipeline(int capacity = 100)
    {
        _channel = Channel.CreateBounded<string>(capacity);
    }

    public async Task ProducerAsync(CancellationToken cancellationToken)
    {
        await foreach (var item in GenerateDataAsync(cancellationToken))
        {
            await _channel.Writer.WriteAsync(item, cancellationToken);
        }
        
        _channel.Writer.Complete();
    }

    public async Task ConsumerAsync(CancellationToken cancellationToken)
    {
        await foreach (var item in _channel.Reader.ReadAllAsync(cancellationToken))
        {
            await ProcessItemAsync(item);
        }
    }

    private async IAsyncEnumerable<string> GenerateDataAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            yield return await FetchDataAsync();
        }
    }
}
```

### ReaderWriterLockSlim for Shared Resources

```csharp
// ✅ GOOD: Optimize for many readers, few writers
public class CachedDataStore
{
    private readonly Dictionary<string, string> _cache = new();
    private readonly ReaderWriterLockSlim _lock = new();

    public string Read(string key)
    {
        _lock.EnterReadLock();
        try
        {
            return _cache.TryGetValue(key, out var value) ? value : null;
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }

    public void Write(string key, string value)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }

    public void Dispose()
    {
        _lock?.Dispose();
    }
}
```

## File I/O Patterns

### Async File Operations

```csharp
// ✅ GOOD: Async file I/O
public async Task<string> ReadFileAsync(string path)
{
    using var stream = new FileStream(
        path,
        FileMode.Open,
        FileAccess.Read,
        FileShare.Read,
        bufferSize: 4096,
        useAsync: true);
    
    using var reader = new StreamReader(stream);
    return await reader.ReadToEndAsync();
}

public async Task WriteFileAsync(string path, string content)
{
    using var stream = new FileStream(
        path,
        FileMode.Create,
        FileAccess.Write,
        FileShare.None,
        bufferSize: 4096,
        useAsync: true);
    
    using var writer = new StreamWriter(stream);
    await writer.WriteAsync(content);
}
```

### Memory-Mapped Files

```csharp
using System.IO.MemoryMappedFiles;

// ✅ GOOD: Memory-mapped file for large files
public class MemoryMappedFileReader : IDisposable
{
    private readonly MemoryMappedFile _mmf;
    private readonly MemoryMappedViewAccessor _accessor;

    public MemoryMappedFileReader(string path)
    {
        var fileInfo = new FileInfo(path);
        _mmf = MemoryMappedFile.CreateFromFile(
            path,
            FileMode.Open,
            null,
            fileInfo.Length);
        
        _accessor = _mmf.CreateViewAccessor(
            0,
            fileInfo.Length,
            MemoryMappedFileAccess.Read);
    }

    public T Read<T>(long position) where T : struct
    {
        _accessor.Read(position, out T value);
        return value;
    }

    public void Dispose()
    {
        _accessor?.Dispose();
        _mmf?.Dispose();
    }
}
```

### Pipelines for Stream Processing

```csharp
using System.IO.Pipelines;

// ✅ GOOD: Use Pipelines for efficient stream processing
public async Task ProcessStreamAsync(Stream stream)
{
    var reader = PipeReader.Create(stream);

    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        while (TryReadLine(ref buffer, out ReadOnlySequence<byte> line))
        {
            ProcessLine(line);
        }

        reader.AdvanceTo(buffer.Start, buffer.End);

        if (result.IsCompleted)
        {
            break;
        }
    }

    await reader.CompleteAsync();
}

private bool TryReadLine(
    ref ReadOnlySequence<byte> buffer,
    out ReadOnlySequence<byte> line)
{
    var position = buffer.PositionOf((byte)'\n');
    
    if (position == null)
    {
        line = default;
        return false;
    }

    line = buffer.Slice(0, position.Value);
    buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
    return true;
}
```

## Network Programming Patterns

### HttpClient Best Practices

```csharp
// ✅ GOOD: Reuse HttpClient with IHttpClientFactory
public class ApiClient
{
    private readonly HttpClient _httpClient;

    public ApiClient(IHttpClientFactory httpClientFactory)
    {
        _httpClient = httpClientFactory.CreateClient("api");
    }

    public async Task<T> GetAsync<T>(string endpoint)
    {
        var response = await _httpClient.GetAsync(endpoint);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<T>();
    }
}

// Startup configuration
services.AddHttpClient("api", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

### TCP Socket Server

```csharp
using System.Net;
using System.Net.Sockets;

// ✅ GOOD: Async TCP server with proper disposal
public class TcpServer : IDisposable
{
    private readonly TcpListener _listener;
    private readonly CancellationTokenSource _cts = new();

    public TcpServer(int port)
    {
        _listener = new TcpListener(IPAddress.Any, port);
    }

    public async Task StartAsync()
    {
        _listener.Start();
        
        while (!_cts.Token.IsCancellationRequested)
        {
            var client = await _listener.AcceptTcpClientAsync(_cts.Token);
            _ = HandleClientAsync(client, _cts.Token);
        }
    }

    private async Task HandleClientAsync(TcpClient client, CancellationToken cancellationToken)
    {
        using (client)
        {
            var stream = client.GetStream();
            var buffer = new byte[1024];

            while (!cancellationToken.IsCancellationRequested)
            {
                int bytesRead = await stream.ReadAsync(buffer, cancellationToken);
                
                if (bytesRead == 0) break;

                await stream.WriteAsync(buffer.AsMemory(0, bytesRead), cancellationToken);
            }
        }
    }

    public void Stop()
    {
        _cts.Cancel();
        _listener.Stop();
    }

    public void Dispose()
    {
        _cts?.Dispose();
    }
}
```

### WebSocket Server

```csharp
using System.Net.WebSockets;

// ✅ GOOD: WebSocket handler
public async Task HandleWebSocketAsync(WebSocket webSocket)
{
    var buffer = new byte[4096];

    try
    {
        while (webSocket.State == WebSocketState.Open)
        {
            var result = await webSocket.ReceiveAsync(
                new ArraySegment<byte>(buffer),
                CancellationToken.None);

            if (result.MessageType == WebSocketMessageType.Close)
            {
                await webSocket.CloseAsync(
                    WebSocketCloseStatus.NormalClosure,
                    "Closing",
                    CancellationToken.None);
            }
            else
            {
                // Echo back
                await webSocket.SendAsync(
                    new ArraySegment<byte>(buffer, 0, result.Count),
                    result.MessageType,
                    result.EndOfMessage,
                    CancellationToken.None);
            }
        }
    }
    catch (WebSocketException ex)
    {
        Console.WriteLine($"WebSocket error: {ex.Message}");
    }
}
```

## Error Handling Patterns

### Result Pattern

```csharp
// ✅ GOOD: Result type for explicit error handling
public class Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }

    private Result(bool success, T value, string error)
    {
        IsSuccess = success;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) =>
        new(true, value, null);

    public static Result<T> Failure(string error) =>
        new(false, default, error);

    public TResult Match<TResult>(
        Func<T, TResult> onSuccess,
        Func<string, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value) : onFailure(Error);
}

// Usage
public Result<int> Divide(int a, int b)
{
    if (b == 0)
        return Result<int>.Failure("Division by zero");
    
    return Result<int>.Success(a / b);
}

var result = Divide(10, 2);
result.Match(
    onSuccess: value => Console.WriteLine($"Result: {value}"),
    onFailure: error => Console.WriteLine($"Error: {error}")
);
```

### Retry Pattern with Polly

```csharp
using Polly;

// ✅ GOOD: Retry with exponential backoff
public class ResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _retryPolicy;

    public ResilientHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
        
        _retryPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .Or<HttpRequestException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => 
                    TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    Console.WriteLine($"Retry {retryCount} after {timespan.TotalSeconds}s");
                });
    }

    public async Task<HttpResponseMessage> GetAsync(string url)
    {
        return await _retryPolicy.ExecuteAsync(
            async () => await _httpClient.GetAsync(url));
    }
}
```

## Logging and Diagnostics

### Structured Logging with ILogger

```csharp
using Microsoft.Extensions.Logging;

// ✅ GOOD: Structured logging
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation(
            "Creating order for user {UserId} with {ItemCount} items",
            request.UserId,
            request.Items.Count);

        try
        {
            var order = await _repository.CreateAsync(request);
            
            _logger.LogInformation(
                "Order {OrderId} created successfully",
                order.Id);
            
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Failed to create order for user {UserId}",
                request.UserId);
            throw;
        }
    }
}
```

### Activity Source for Distributed Tracing

```csharp
using System.Diagnostics;

// ✅ GOOD: OpenTelemetry-compatible tracing
public class PaymentService
{
    private static readonly ActivitySource ActivitySource = 
        new("PaymentService", "1.0.0");

    public async Task<PaymentResult> ProcessPaymentAsync(Payment payment)
    {
        using var activity = ActivitySource.StartActivity("ProcessPayment");
        activity?.SetTag("payment.amount", payment.Amount);
        activity?.SetTag("payment.currency", payment.Currency);

        try
        {
            var result = await _gateway.ProcessAsync(payment);
            activity?.SetTag("payment.status", result.Status);
            return result;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

## Performance Patterns

### Object Pooling

```csharp
using Microsoft.Extensions.ObjectPool;

// ✅ GOOD: Reuse expensive objects
public class ConnectionPoolPolicy : IPooledObjectPolicy<DatabaseConnection>
{
    public DatabaseConnection Create()
    {
        return new DatabaseConnection();
    }

    public bool Return(DatabaseConnection obj)
    {
        return obj.IsHealthy;
    }
}

// Usage
var pool = new DefaultObjectPool<DatabaseConnection>(
    new ConnectionPoolPolicy(),
    maximumRetained: 10);

var connection = pool.Get();
try
{
    // Use connection
}
finally
{
    pool.Return(connection);
}
```

### Value Types for Performance

```csharp
// ✅ GOOD: Use structs for small, immutable data
public readonly struct Point3D
{
    public double X { get; }
    public double Y { get; }
    public double Z { get; }

    public Point3D(double x, double y, double z)
    {
        X = x;
        Y = y;
        Z = z;
    }

    public double DistanceTo(in Point3D other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        double dz = Z - other.Z;
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}
```

**Remember**: C# system programming leverages async/await, modern memory management with Span<T>, and rich framework support. Write async, use IDisposable properly, and leverage the .NET runtime's optimizations.
