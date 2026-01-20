---
name: cpp-system-patterns
description: C++ system programming patterns, memory management, concurrency, networking, and low-level system design best practices.
---

# C++ System Development Patterns

System programming patterns and best practices for high-performance C++ applications.

## Memory Management Patterns

### RAII (Resource Acquisition Is Initialization)

```cpp
// ✅ GOOD: RAII pattern for automatic resource management
class FileHandle {
private:
    int fd;
    
public:
    explicit FileHandle(const char* filename, int flags) {
        fd = open(filename, flags);
        if (fd == -1) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~FileHandle() {
        if (fd != -1) {
            close(fd);
        }
    }
    
    // Delete copy, allow move
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : fd(other.fd) {
        other.fd = -1;
    }
    
    int get() const { return fd; }
};

// Usage - automatic cleanup
void processFile(const char* filename) {
    FileHandle file(filename, O_RDONLY);
    // Use file.get()
} // File automatically closed here
```

### Smart Pointers

```cpp
#include <memory>

// ✅ GOOD: Use smart pointers for automatic memory management
class Connection {
public:
    void send(const std::string& data);
    void receive(std::string& buffer);
};

// Unique ownership
std::unique_ptr<Connection> createConnection(const std::string& host) {
    return std::make_unique<Connection>(host);
}

// Shared ownership
class ConnectionPool {
private:
    std::vector<std::shared_ptr<Connection>> connections;
    
public:
    std::shared_ptr<Connection> acquire() {
        // Return shared pointer to connection
        return connections.empty() ? nullptr : connections.back();
    }
};

// ❌ BAD: Manual memory management
Connection* conn = new Connection();
// ... easy to forget delete
delete conn;
```

### Custom Allocators

```cpp
// ✅ Pool allocator for frequent allocations
template<typename T, size_t PoolSize = 1024>
class PoolAllocator {
private:
    std::array<T, PoolSize> pool;
    std::vector<size_t> freeList;
    
public:
    PoolAllocator() {
        freeList.reserve(PoolSize);
        for (size_t i = 0; i < PoolSize; ++i) {
            freeList.push_back(i);
        }
    }
    
    T* allocate() {
        if (freeList.empty()) {
            throw std::bad_alloc();
        }
        size_t index = freeList.back();
        freeList.pop_back();
        return &pool[index];
    }
    
    void deallocate(T* ptr) {
        size_t index = ptr - pool.data();
        freeList.push_back(index);
    }
};
```

## Concurrency Patterns

### Thread-Safe Queue

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>

template<typename T>
class ThreadSafeQueue {
private:
    std::queue<T> queue;
    mutable std::mutex mutex;
    std::condition_variable condVar;
    
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mutex);
        queue.push(std::move(value));
        condVar.notify_one();
    }
    
    bool tryPop(T& value) {
        std::lock_guard<std::mutex> lock(mutex);
        if (queue.empty()) {
            return false;
        }
        value = std::move(queue.front());
        queue.pop();
        return true;
    }
    
    void waitAndPop(T& value) {
        std::unique_lock<std::mutex> lock(mutex);
        condVar.wait(lock, [this] { return !queue.empty(); });
        value = std::move(queue.front());
        queue.pop();
    }
    
    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex);
        return queue.empty();
    }
};
```

### Thread Pool

```cpp
#include <thread>
#include <functional>
#include <future>

class ThreadPool {
private:
    std::vector<std::thread> workers;
    ThreadSafeQueue<std::function<void()>> tasks;
    std::atomic<bool> stop;
    
public:
    explicit ThreadPool(size_t numThreads) : stop(false) {
        for (size_t i = 0; i < numThreads; ++i) {
            workers.emplace_back([this] {
                while (!stop) {
                    std::function<void()> task;
                    if (tasks.tryPop(task)) {
                        task();
                    } else {
                        std::this_thread::yield();
                    }
                }
            });
        }
    }
    
    ~ThreadPool() {
        stop = true;
        for (auto& worker : workers) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::result_of<F(Args...)>::type> {
        using ReturnType = typename std::result_of<F(Args...)>::type;
        
        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<ReturnType> result = task->get_future();
        tasks.push([task]() { (*task)(); });
        
        return result;
    }
};
```

### Reader-Writer Lock

```cpp
#include <shared_mutex>

class DataStore {
private:
    std::map<std::string, std::string> data;
    mutable std::shared_mutex mutex;
    
public:
    // Multiple readers can access simultaneously
    std::string read(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex);
        auto it = data.find(key);
        return it != data.end() ? it->second : "";
    }
    
    // Exclusive write access
    void write(const std::string& key, const std::string& value) {
        std::unique_lock<std::shared_mutex> lock(mutex);
        data[key] = value;
    }
};
```

## File I/O Patterns

### Memory-Mapped Files

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

class MemoryMappedFile {
private:
    void* addr;
    size_t length;
    int fd;
    
public:
    MemoryMappedFile(const char* filename) {
        fd = open(filename, O_RDONLY);
        if (fd == -1) {
            throw std::runtime_error("Failed to open file");
        }
        
        struct stat sb;
        if (fstat(fd, &sb) == -1) {
            close(fd);
            throw std::runtime_error("Failed to get file size");
        }
        
        length = sb.st_size;
        addr = mmap(nullptr, length, PROT_READ, MAP_PRIVATE, fd, 0);
        
        if (addr == MAP_FAILED) {
            close(fd);
            throw std::runtime_error("Failed to map file");
        }
    }
    
    ~MemoryMappedFile() {
        if (addr != MAP_FAILED) {
            munmap(addr, length);
        }
        if (fd != -1) {
            close(fd);
        }
    }
    
    const void* data() const { return addr; }
    size_t size() const { return length; }
};
```

### Buffered I/O

```cpp
class BufferedFileWriter {
private:
    static constexpr size_t BUFFER_SIZE = 8192;
    std::vector<char> buffer;
    size_t bufferPos;
    int fd;
    
public:
    explicit BufferedFileWriter(const char* filename) 
        : buffer(BUFFER_SIZE), bufferPos(0) {
        fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd == -1) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~BufferedFileWriter() {
        try {
            flush();
        } catch (...) {
            // Don't throw from destructor
        }
        if (fd != -1) {
            close(fd);
        }
    }
    
    void write(const void* data, size_t size) {
        const char* ptr = static_cast<const char*>(data);
        
        while (size > 0) {
            size_t available = BUFFER_SIZE - bufferPos;
            size_t toWrite = std::min(size, available);
            
            std::memcpy(buffer.data() + bufferPos, ptr, toWrite);
            bufferPos += toWrite;
            ptr += toWrite;
            size -= toWrite;
            
            if (bufferPos == BUFFER_SIZE) {
                flush();
            }
        }
    }
    
    void flush() {
        if (bufferPos > 0) {
            ssize_t written = ::write(fd, buffer.data(), bufferPos);
            if (written != static_cast<ssize_t>(bufferPos)) {
                throw std::runtime_error("Write failed");
            }
            bufferPos = 0;
        }
    }
};
```

## Network Programming Patterns

### TCP Server

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

class TcpServer {
private:
    int serverSocket;
    
public:
    TcpServer(uint16_t port) {
        serverSocket = socket(AF_INET, SOCK_STREAM, 0);
        if (serverSocket == -1) {
            throw std::runtime_error("Failed to create socket");
        }
        
        // Allow address reuse
        int opt = 1;
        setsockopt(serverSocket, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
        
        sockaddr_in address{};
        address.sin_family = AF_INET;
        address.sin_addr.s_addr = INADDR_ANY;
        address.sin_port = htons(port);
        
        if (bind(serverSocket, (sockaddr*)&address, sizeof(address)) < 0) {
            close(serverSocket);
            throw std::runtime_error("Bind failed");
        }
        
        if (listen(serverSocket, 10) < 0) {
            close(serverSocket);
            throw std::runtime_error("Listen failed");
        }
    }
    
    ~TcpServer() {
        if (serverSocket != -1) {
            close(serverSocket);
        }
    }
    
    int acceptConnection() {
        sockaddr_in clientAddress{};
        socklen_t clientLen = sizeof(clientAddress);
        
        int clientSocket = accept(serverSocket, 
                                  (sockaddr*)&clientAddress, 
                                  &clientLen);
        if (clientSocket < 0) {
            throw std::runtime_error("Accept failed");
        }
        
        return clientSocket;
    }
};
```

### Async I/O with epoll

```cpp
#include <sys/epoll.h>

class EpollEventLoop {
private:
    int epollFd;
    static constexpr int MAX_EVENTS = 64;
    
public:
    EpollEventLoop() {
        epollFd = epoll_create1(0);
        if (epollFd == -1) {
            throw std::runtime_error("Failed to create epoll");
        }
    }
    
    ~EpollEventLoop() {
        if (epollFd != -1) {
            close(epollFd);
        }
    }
    
    void addSocket(int fd, uint32_t events) {
        epoll_event ev{};
        ev.events = events;
        ev.data.fd = fd;
        
        if (epoll_ctl(epollFd, EPOLL_CTL_ADD, fd, &ev) == -1) {
            throw std::runtime_error("Failed to add socket to epoll");
        }
    }
    
    void run(std::function<void(int, uint32_t)> handler) {
        epoll_event events[MAX_EVENTS];
        
        while (true) {
            int numEvents = epoll_wait(epollFd, events, MAX_EVENTS, -1);
            
            if (numEvents == -1) {
                if (errno == EINTR) continue;
                throw std::runtime_error("epoll_wait failed");
            }
            
            for (int i = 0; i < numEvents; ++i) {
                handler(events[i].data.fd, events[i].events);
            }
        }
    }
};
```

## Error Handling Patterns

### Result Type

```cpp
template<typename T, typename E = std::string>
class Result {
private:
    std::variant<T, E> value;
    
public:
    static Result Ok(T val) {
        Result r;
        r.value = std::move(val);
        return r;
    }
    
    static Result Err(E err) {
        Result r;
        r.value = std::move(err);
        return r;
    }
    
    bool isOk() const {
        return std::holds_alternative<T>(value);
    }
    
    bool isErr() const {
        return std::holds_alternative<E>(value);
    }
    
    T& unwrap() {
        if (isErr()) {
            throw std::runtime_error("Called unwrap on error result");
        }
        return std::get<T>(value);
    }
    
    T unwrapOr(T defaultValue) {
        return isOk() ? std::get<T>(value) : defaultValue;
    }
    
    E& error() {
        return std::get<E>(value);
    }
};

// Usage
Result<int, std::string> divide(int a, int b) {
    if (b == 0) {
        return Result<int, std::string>::Err("Division by zero");
    }
    return Result<int, std::string>::Ok(a / b);
}

auto result = divide(10, 2);
if (result.isOk()) {
    std::cout << "Result: " << result.unwrap() << std::endl;
} else {
    std::cerr << "Error: " << result.error() << std::endl;
}
```

### Exception Safety Guarantees

```cpp
class Transaction {
private:
    Database& db;
    bool committed;
    
public:
    explicit Transaction(Database& database) 
        : db(database), committed(false) {
        db.beginTransaction();
    }
    
    ~Transaction() {
        if (!committed) {
            try {
                db.rollback();
            } catch (...) {
                // Log error but don't throw from destructor
            }
        }
    }
    
    void commit() {
        db.commit();
        committed = true;
    }
    
    // Strong exception guarantee
    void execute(const std::string& query) {
        db.execute(query);  // If this throws, destructor will rollback
    }
};
```

## Performance Patterns

### Object Pooling

```cpp
template<typename T>
class ObjectPool {
private:
    std::vector<std::unique_ptr<T>> pool;
    std::vector<T*> available;
    std::mutex mutex;
    
public:
    class PooledObject {
    private:
        ObjectPool* pool;
        T* obj;
        
    public:
        PooledObject(ObjectPool* p, T* o) : pool(p), obj(o) {}
        
        ~PooledObject() {
            if (pool && obj) {
                pool->release(obj);
            }
        }
        
        T* operator->() { return obj; }
        T& operator*() { return *obj; }
    };
    
    PooledObject acquire() {
        std::lock_guard<std::mutex> lock(mutex);
        
        if (available.empty()) {
            pool.push_back(std::make_unique<T>());
            return PooledObject(this, pool.back().get());
        }
        
        T* obj = available.back();
        available.pop_back();
        return PooledObject(this, obj);
    }
    
private:
    void release(T* obj) {
        std::lock_guard<std::mutex> lock(mutex);
        available.push_back(obj);
    }
};
```

### Cache-Friendly Data Structures

```cpp
// ✅ GOOD: Structure of Arrays (cache-friendly)
struct ParticlesSoA {
    std::vector<float> x;
    std::vector<float> y;
    std::vector<float> z;
    std::vector<float> vx;
    std::vector<float> vy;
    std::vector<float> vz;
    
    void update(float dt) {
        // Process all x positions together (better cache locality)
        for (size_t i = 0; i < x.size(); ++i) {
            x[i] += vx[i] * dt;
        }
        // Then all y positions, etc.
    }
};

// ❌ BAD: Array of Structures (cache-unfriendly for this use case)
struct Particle {
    float x, y, z;
    float vx, vy, vz;
};

std::vector<Particle> particles;
```

**Remember**: C++ system programming requires careful resource management, understanding of low-level details, and attention to performance. RAII and smart pointers are your friends.
