---
name: cpp-system-development
description: C++ coding standards, best practices, and system development patterns for high-performance systems, operating systems, embedded systems, and low-level programming.
---

# C++ System Development Standards

System-level coding standards and patterns for high-performance C++ development.

## Code Quality Principles

### 1. Readability First
- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting (clang-format recommended)

### 2. KISS (Keep It Simple, Stupid)
- Simplest solution that works
- Avoid over-engineering
- Profile before optimizing
- Clear code > clever code

### 3. DRY (Don't Repeat Yourself)
- Extract common logic into functions
- Use templates for generic algorithms
- Create reusable components
- Avoid copy-paste programming

### 4. RAII (Resource Acquisition Is Initialization)
- Resources tied to object lifetime
- Automatic cleanup via destructors
- No manual resource management
- Exception-safe resource handling

## C++ Standards

### Variable Naming

```cpp
// ✅ GOOD: Descriptive names
const std::string market_search_query = "election";
bool is_user_authenticated = true;
int64_t total_revenue = 1000;

// CamelCase for classes/types
class MarketProcessor { };
struct UserData { };

// snake_case for variables and functions
int calculate_similarity(const std::vector<double>& a);

// ❌ BAD: Unclear names
const std::string q = "election";
bool flag = true;
int x = 1000;
```

### Function Naming

```cpp
// ✅ GOOD: Verb-noun pattern with snake_case
std::future<MarketData> fetch_market_data(const std::string& market_id);
double calculate_similarity(const std::vector<double>& a, const std::vector<double>& b);
bool is_valid_email(const std::string& email);

// ❌ BAD: Unclear or noun-only
auto market(std::string id);
double similarity(auto a, auto b);
bool email(std::string e);
```

### RAII and Resource Management

```cpp
// ✅ GOOD: RAII for automatic resource management
class FileHandle {
private:
    int fd_;
public:
    explicit FileHandle(const std::string& path) 
        : fd_(open(path.c_str(), O_RDONLY)) {
        if (fd_ < 0) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~FileHandle() {
        if (fd_ >= 0) {
            close(fd_);
        }
    }
    
    // Delete copy, allow move
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }
    
    int get() const { return fd_; }
};

// ✅ GOOD: Smart pointers for memory management
auto market = std::make_unique<Market>();
auto shared_cache = std::make_shared<Cache>();

// ❌ BAD: Manual resource management
int fd = open("file.txt", O_RDONLY);
// ... forgetting to close(fd)

Market* market = new Market();
// ... forgetting to delete market
```

### Error Handling

```cpp
// ✅ GOOD: Use exceptions for error conditions
class NetworkError : public std::runtime_error {
public:
    explicit NetworkError(const std::string& msg) 
        : std::runtime_error(msg) {}
};

std::vector<uint8_t> fetch_data(const std::string& url) {
    try {
        auto response = http_client.get(url);
        
        if (!response.ok()) {
            throw NetworkError("HTTP " + std::to_string(response.status_code()));
        }
        
        return response.body();
    } catch (const std::exception& e) {
        std::cerr << "Fetch failed: " << e.what() << std::endl;
        throw;
    }
}

// ✅ GOOD: Use std::optional for nullable returns
std::optional<User> find_user_by_id(int64_t id) {
    auto it = users.find(id);
    if (it != users.end()) {
        return it->second;
    }
    return std::nullopt;
}

// ✅ GOOD: Use std::expected (C++23) or Result type for error handling
std::expected<Data, Error> load_data(const std::string& path);
```

### Const Correctness

```cpp
// ✅ GOOD: Const correctness
class Market {
private:
    std::string name_;
    double volume_;
    
public:
    // Const member function - doesn't modify object
    const std::string& get_name() const { return name_; }
    double get_volume() const { return volume_; }
    
    // Non-const member function
    void set_volume(double volume) { volume_ = volume; }
    
    // Const parameters
    void process(const std::vector<int>& data) const;
};

// ✅ GOOD: Const references to avoid copies
void process_markets(const std::vector<Market>& markets);

// ❌ BAD: Missing const
class Market {
    std::string get_name() { return name_; }  // Should be const
    void process(std::vector<int>& data);     // Unclear if modifies
};
```

### Modern C++ Best Practices

```cpp
// ✅ GOOD: Use auto for type deduction
auto markets = fetch_markets();
auto it = std::find_if(markets.begin(), markets.end(), 
                       [](const auto& m) { return m.is_active(); });

// ✅ GOOD: Range-based for loops
for (const auto& market : markets) {
    process(market);
}

// ✅ GOOD: Structured bindings (C++17)
for (const auto& [key, value] : market_map) {
    std::cout << key << ": " << value << std::endl;
}

// ✅ GOOD: Lambda expressions
auto is_active = [](const Market& m) { return m.status() == Status::Active; };
std::erase_if(markets, std::not_fn(is_active));

// ✅ GOOD: Use STL algorithms
std::vector<int> values = {1, 2, 3, 4, 5};
auto sum = std::accumulate(values.begin(), values.end(), 0);
std::transform(values.begin(), values.end(), values.begin(), 
               [](int x) { return x * 2; });
```

## System Development Patterns

### Concurrency Patterns

```cpp
// ✅ GOOD: Thread-safe singleton with std::call_once
class Logger {
private:
    Logger() = default;
    static std::once_flag init_flag_;
    static std::unique_ptr<Logger> instance_;
    
public:
    static Logger& instance() {
        std::call_once(init_flag_, []() {
            instance_ = std::unique_ptr<Logger>(new Logger());
        });
        return *instance_;
    }
    
    // Delete copy and move
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
};

// ✅ GOOD: Mutex for thread safety
class ThreadSafeQueue {
private:
    std::queue<int> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    
public:
    void push(int value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(value);
        cv_.notify_one();
    }
    
    std::optional<int> pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        
        if (queue_.empty()) return std::nullopt;
        
        int value = queue_.front();
        queue_.pop();
        return value;
    }
};

// ✅ GOOD: Async operations with std::future
std::future<Data> fetch_async(const std::string& url) {
    return std::async(std::launch::async, [url]() {
        return fetch_data(url);
    });
}

// Usage
auto future1 = fetch_async("url1");
auto future2 = fetch_async("url2");
auto data1 = future1.get();
auto data2 = future2.get();
```

### Memory Management Patterns

```cpp
// ✅ GOOD: Custom allocator for performance-critical code
template<typename T>
class PoolAllocator {
private:
    std::vector<T> pool_;
    std::vector<size_t> free_slots_;
    
public:
    explicit PoolAllocator(size_t size) : pool_(size) {
        free_slots_.reserve(size);
        for (size_t i = 0; i < size; ++i) {
            free_slots_.push_back(i);
        }
    }
    
    T* allocate() {
        if (free_slots_.empty()) return nullptr;
        
        size_t slot = free_slots_.back();
        free_slots_.pop_back();
        return &pool_[slot];
    }
    
    void deallocate(T* ptr) {
        size_t slot = ptr - pool_.data();
        free_slots_.push_back(slot);
    }
};

// ✅ GOOD: Arena allocator for batch allocations
class Arena {
private:
    std::vector<uint8_t> buffer_;
    size_t offset_ = 0;
    
public:
    explicit Arena(size_t size) : buffer_(size) {}
    
    void* allocate(size_t size, size_t alignment = alignof(std::max_align_t)) {
        size_t aligned_offset = (offset_ + alignment - 1) & ~(alignment - 1);
        
        if (aligned_offset + size > buffer_.size()) {
            throw std::bad_alloc();
        }
        
        void* ptr = buffer_.data() + aligned_offset;
        offset_ = aligned_offset + size;
        return ptr;
    }
    
    void reset() { offset_ = 0; }
};
```

### Lock-Free Programming

```cpp
// ✅ GOOD: Atomic operations for lock-free data structures
class LockFreeStack {
private:
    struct Node {
        int value;
        Node* next;
    };
    
    std::atomic<Node*> head_{nullptr};
    
public:
    void push(int value) {
        Node* new_node = new Node{value, head_.load()};
        
        while (!head_.compare_exchange_weak(new_node->next, new_node)) {
            // Retry on failure
        }
    }
    
    std::optional<int> pop() {
        Node* old_head = head_.load();
        
        while (old_head && 
               !head_.compare_exchange_weak(old_head, old_head->next)) {
            // Retry on failure
        }
        
        if (!old_head) return std::nullopt;
        
        int value = old_head->value;
        delete old_head;
        return value;
    }
};
```

### System I/O Patterns

```cpp
// ✅ GOOD: Buffered I/O for performance
class BufferedReader {
private:
    int fd_;
    std::vector<uint8_t> buffer_;
    size_t buffer_pos_ = 0;
    size_t buffer_size_ = 0;
    static constexpr size_t BUFFER_CAPACITY = 8192;
    
public:
    explicit BufferedReader(int fd) 
        : fd_(fd), buffer_(BUFFER_CAPACITY) {}
    
    std::optional<std::string> read_line() {
        std::string line;
        
        while (true) {
            if (buffer_pos_ >= buffer_size_) {
                ssize_t n = read(fd_, buffer_.data(), BUFFER_CAPACITY);
                if (n <= 0) {
                    return line.empty() ? std::nullopt : std::make_optional(line);
                }
                buffer_size_ = n;
                buffer_pos_ = 0;
            }
            
            char c = buffer_[buffer_pos_++];
            if (c == '\n') break;
            line += c;
        }
        
        return line;
    }
};

// ✅ GOOD: Memory-mapped file I/O for large files
class MappedFile {
private:
    int fd_;
    void* addr_;
    size_t size_;
    
public:
    explicit MappedFile(const std::string& path) {
        fd_ = open(path.c_str(), O_RDONLY);
        if (fd_ < 0) throw std::runtime_error("Failed to open file");
        
        struct stat sb;
        if (fstat(fd_, &sb) < 0) {
            close(fd_);
            throw std::runtime_error("Failed to stat file");
        }
        
        size_ = sb.st_size;
        addr_ = mmap(nullptr, size_, PROT_READ, MAP_PRIVATE, fd_, 0);
        
        if (addr_ == MAP_FAILED) {
            close(fd_);
            throw std::runtime_error("Failed to mmap file");
        }
    }
    
    ~MappedFile() {
        munmap(addr_, size_);
        close(fd_);
    }
    
    const uint8_t* data() const { 
        return static_cast<const uint8_t*>(addr_); 
    }
    
    size_t size() const { return size_; }
};
```

### Event Loop Pattern

```cpp
// ✅ GOOD: Epoll-based event loop for scalable I/O
class EventLoop {
private:
    int epoll_fd_;
    bool running_ = false;
    std::unordered_map<int, std::function<void()>> handlers_;
    
public:
    EventLoop() {
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ < 0) {
            throw std::runtime_error("Failed to create epoll");
        }
    }
    
    ~EventLoop() {
        close(epoll_fd_);
    }
    
    void add_fd(int fd, std::function<void()> handler) {
        epoll_event ev{};
        ev.events = EPOLLIN | EPOLLET;
        ev.data.fd = fd;
        
        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
            throw std::runtime_error("Failed to add fd to epoll");
        }
        
        handlers_[fd] = std::move(handler);
    }
    
    void run() {
        running_ = true;
        std::vector<epoll_event> events(10);
        
        while (running_) {
            int n = epoll_wait(epoll_fd_, events.data(), events.size(), -1);
            
            for (int i = 0; i < n; ++i) {
                int fd = events[i].data.fd;
                auto it = handlers_.find(fd);
                if (it != handlers_.end()) {
                    it->second();
                }
            }
        }
    }
    
    void stop() { running_ = false; }
};
```

## Performance Optimization

### Cache-Friendly Code

```cpp
// ✅ GOOD: Struct of Arrays (SoA) for better cache locality
struct ParticlesSoA {
    std::vector<float> x;
    std::vector<float> y;
    std::vector<float> z;
    std::vector<float> mass;
    
    void update(size_t n) {
        for (size_t i = 0; i < n; ++i) {
            x[i] += 1.0f;  // Sequential memory access
            y[i] += 1.0f;
            z[i] += 1.0f;
        }
    }
};

// ❌ BAD: Array of Structs (AoS) causes cache misses
struct Particle {
    float x, y, z;
    float mass;
    char padding[48];  // Poor cache utilization
};

std::vector<Particle> particles;
```

### SIMD Optimization

```cpp
// ✅ GOOD: Vectorized operations with SIMD
#include <immintrin.h>

void add_vectors_simd(const float* a, const float* b, float* result, size_t n) {
    size_t i = 0;
    
    // Process 8 floats at a time with AVX
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vr = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(result + i, vr);
    }
    
    // Handle remaining elements
    for (; i < n; ++i) {
        result[i] = a[i] + b[i];
    }
}
```

## Testing Standards

### Unit Testing with Google Test

```cpp
#include <gtest/gtest.h>

// ✅ GOOD: Descriptive test names
TEST(MarketTest, ReturnsEmptyArrayWhenNoMarketsMatchQuery) {
    MarketSearcher searcher;
    auto results = searcher.search("nonexistent");
    EXPECT_TRUE(results.empty());
}

TEST(MarketTest, ThrowsExceptionWhenApiKeyMissing) {
    MarketClient client("");
    EXPECT_THROW(client.fetch_markets(), std::runtime_error);
}

// ✅ GOOD: Test fixtures for shared setup
class DatabaseTest : public ::testing::Test {
protected:
    void SetUp() override {
        db_ = std::make_unique<Database>(":memory:");
    }
    
    void TearDown() override {
        db_.reset();
    }
    
    std::unique_ptr<Database> db_;
};

TEST_F(DatabaseTest, InsertAndRetrieve) {
    db_->insert("key", "value");
    auto result = db_->get("key");
    ASSERT_TRUE(result.has_value());
    EXPECT_EQ(result.value(), "value");
}
```

## Code Smell Detection

### Long Functions
```cpp
// ❌ BAD: Function > 50 lines
void process_market_data() {
    // 100 lines of code
}

// ✅ GOOD: Split into smaller functions
void process_market_data() {
    auto validated = validate_data();
    auto transformed = transform_data(validated);
    save_data(transformed);
}
```

### Raw Pointers
```cpp
// ❌ BAD: Raw pointer ownership unclear
Market* create_market() {
    return new Market();  // Who owns this?
}

// ✅ GOOD: Clear ownership with smart pointers
std::unique_ptr<Market> create_market() {
    return std::make_unique<Market>();
}
```

### Magic Numbers
```cpp
// ❌ BAD: Unexplained numbers
if (retry_count > 3) { }
std::this_thread::sleep_for(std::chrono::milliseconds(500));

// ✅ GOOD: Named constants
constexpr int MAX_RETRIES = 3;
constexpr auto RETRY_DELAY_MS = std::chrono::milliseconds(500);

if (retry_count > MAX_RETRIES) { }
std::this_thread::sleep_for(RETRY_DELAY_MS);
```

**Remember**: Modern C++ enables safe, high-performance system development. Use RAII, smart pointers, and STL algorithms to write robust system code.
