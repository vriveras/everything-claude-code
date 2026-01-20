---
name: cpp-system-patterns
description: C++ system architecture patterns, network programming, database integration, and high-performance backend patterns for systems programming.
---

# C++ System Development Patterns

Advanced system architecture patterns and best practices for scalable C++ applications.

## Network Programming Patterns

### TCP Server with Epoll

```cpp
class TcpServer {
private:
    int server_fd_;
    int epoll_fd_;
    std::unordered_map<int, std::unique_ptr<Connection>> connections_;
    std::atomic<bool> running_{false};
    
public:
    explicit TcpServer(uint16_t port) {
        // Create socket
        server_fd_ = socket(AF_INET, SOCK_STREAM, 0);
        if (server_fd_ < 0) {
            throw std::runtime_error("Failed to create socket");
        }
        
        // Set socket options
        int opt = 1;
        setsockopt(server_fd_, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
        
        // Bind
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_addr.s_addr = INADDR_ANY;
        addr.sin_port = htons(port);
        
        if (bind(server_fd_, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
            close(server_fd_);
            throw std::runtime_error("Failed to bind socket");
        }
        
        // Listen
        if (listen(server_fd_, SOMAXCONN) < 0) {
            close(server_fd_);
            throw std::runtime_error("Failed to listen on socket");
        }
        
        // Create epoll
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ < 0) {
            close(server_fd_);
            throw std::runtime_error("Failed to create epoll");
        }
        
        // Add server socket to epoll
        epoll_event ev{};
        ev.events = EPOLLIN;
        ev.data.fd = server_fd_;
        epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, server_fd_, &ev);
    }
    
    ~TcpServer() {
        stop();
        close(epoll_fd_);
        close(server_fd_);
    }
    
    void run() {
        running_ = true;
        std::vector<epoll_event> events(100);
        
        while (running_) {
            int n = epoll_wait(epoll_fd_, events.data(), events.size(), 1000);
            
            for (int i = 0; i < n; ++i) {
                if (events[i].data.fd == server_fd_) {
                    accept_connection();
                } else {
                    handle_client(events[i].data.fd);
                }
            }
        }
    }
    
    void stop() { running_ = false; }
    
private:
    void accept_connection() {
        sockaddr_in client_addr{};
        socklen_t addr_len = sizeof(client_addr);
        
        int client_fd = accept(server_fd_, (struct sockaddr*)&client_addr, &addr_len);
        if (client_fd < 0) return;
        
        // Set non-blocking
        int flags = fcntl(client_fd, F_GETFL, 0);
        fcntl(client_fd, F_SETFL, flags | O_NONBLOCK);
        
        // Add to epoll
        epoll_event ev{};
        ev.events = EPOLLIN | EPOLLET;
        ev.data.fd = client_fd;
        epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, client_fd, &ev);
        
        connections_[client_fd] = std::make_unique<Connection>(client_fd);
    }
    
    void handle_client(int fd) {
        auto it = connections_.find(fd);
        if (it == connections_.end()) return;
        
        auto& conn = it->second;
        
        if (!conn->read_data()) {
            // Connection closed or error
            epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
            connections_.erase(it);
            return;
        }
        
        // Process request and send response
        auto response = process_request(conn->get_data());
        conn->write_data(response);
    }
    
    std::vector<uint8_t> process_request(const std::vector<uint8_t>& request) {
        // Process request and generate response
        return {};
    }
};
```

### HTTP Client with Connection Pooling

```cpp
class HttpClient {
private:
    struct Connection {
        int socket_fd;
        std::chrono::steady_clock::time_point last_used;
        std::string host;
        uint16_t port;
    };
    
    std::mutex mutex_;
    std::vector<Connection> pool_;
    static constexpr size_t MAX_POOL_SIZE = 10;
    static constexpr auto CONNECTION_TIMEOUT = std::chrono::seconds(30);
    
public:
    struct Response {
        int status_code;
        std::unordered_map<std::string, std::string> headers;
        std::vector<uint8_t> body;
    };
    
    Response get(const std::string& url) {
        auto [host, port, path] = parse_url(url);
        
        int socket_fd = acquire_connection(host, port);
        
        // Build HTTP request
        std::ostringstream request;
        request << "GET " << path << " HTTP/1.1\r\n"
                << "Host: " << host << "\r\n"
                << "Connection: keep-alive\r\n"
                << "\r\n";
        
        std::string request_str = request.str();
        send(socket_fd, request_str.data(), request_str.size(), 0);
        
        // Read response
        Response response = read_response(socket_fd);
        
        // Return connection to pool
        release_connection(socket_fd, host, port);
        
        return response;
    }
    
private:
    int acquire_connection(const std::string& host, uint16_t port) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        // Remove stale connections
        auto now = std::chrono::steady_clock::now();
        pool_.erase(
            std::remove_if(pool_.begin(), pool_.end(),
                [now](const Connection& conn) {
                    return now - conn.last_used > CONNECTION_TIMEOUT;
                }),
            pool_.end());
        
        // Find existing connection
        auto it = std::find_if(pool_.begin(), pool_.end(),
            [&host, port](const Connection& conn) {
                return conn.host == host && conn.port == port;
            });
        
        if (it != pool_.end()) {
            int fd = it->socket_fd;
            pool_.erase(it);
            return fd;
        }
        
        // Create new connection
        return create_connection(host, port);
    }
    
    void release_connection(int socket_fd, const std::string& host, uint16_t port) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        if (pool_.size() < MAX_POOL_SIZE) {
            pool_.push_back({
                socket_fd,
                std::chrono::steady_clock::now(),
                host,
                port
            });
        } else {
            close(socket_fd);
        }
    }
    
    int create_connection(const std::string& host, uint16_t port) {
        // DNS resolution and socket creation
        // Implementation details...
        return 0;
    }
    
    Response read_response(int socket_fd) {
        // Read and parse HTTP response
        // Implementation details...
        return {};
    }
    
    std::tuple<std::string, uint16_t, std::string> parse_url(const std::string& url) {
        // Parse URL into host, port, path
        return {"", 0, ""};
    }
};
```

## Database Integration Patterns

### Connection Pool Pattern

```cpp
template<typename ConnectionType>
class ConnectionPool {
private:
    struct PooledConnection {
        std::unique_ptr<ConnectionType> conn;
        std::chrono::steady_clock::time_point last_used;
        bool in_use = false;
    };
    
    std::vector<PooledConnection> pool_;
    std::mutex mutex_;
    std::condition_variable cv_;
    size_t max_size_;
    std::function<std::unique_ptr<ConnectionType>()> factory_;
    
public:
    ConnectionPool(size_t max_size, 
                   std::function<std::unique_ptr<ConnectionType>()> factory)
        : max_size_(max_size), factory_(std::move(factory)) {
        
        // Pre-allocate connections
        for (size_t i = 0; i < max_size / 2; ++i) {
            pool_.push_back({factory_(), std::chrono::steady_clock::now(), false});
        }
    }
    
    class Handle {
    private:
        ConnectionPool* pool_;
        ConnectionType* conn_;
        
    public:
        Handle(ConnectionPool* pool, ConnectionType* conn) 
            : pool_(pool), conn_(conn) {}
        
        ~Handle() {
            if (pool_ && conn_) {
                pool_->release(conn_);
            }
        }
        
        Handle(const Handle&) = delete;
        Handle& operator=(const Handle&) = delete;
        
        Handle(Handle&& other) noexcept 
            : pool_(other.pool_), conn_(other.conn_) {
            other.pool_ = nullptr;
            other.conn_ = nullptr;
        }
        
        ConnectionType* operator->() { return conn_; }
        ConnectionType& operator*() { return *conn_; }
    };
    
    Handle acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        
        // Wait for available connection
        cv_.wait(lock, [this] {
            return std::any_of(pool_.begin(), pool_.end(),
                [](const PooledConnection& pc) { return !pc.in_use; });
        });
        
        // Find available connection
        auto it = std::find_if(pool_.begin(), pool_.end(),
            [](const PooledConnection& pc) { return !pc.in_use; });
        
        if (it == pool_.end() && pool_.size() < max_size_) {
            // Create new connection
            pool_.push_back({factory_(), std::chrono::steady_clock::now(), true});
            return Handle(this, pool_.back().conn.get());
        }
        
        it->in_use = true;
        it->last_used = std::chrono::steady_clock::now();
        return Handle(this, it->conn.get());
    }
    
private:
    void release(ConnectionType* conn) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto it = std::find_if(pool_.begin(), pool_.end(),
            [conn](const PooledConnection& pc) {
                return pc.conn.get() == conn;
            });
        
        if (it != pool_.end()) {
            it->in_use = false;
            it->last_used = std::chrono::steady_clock::now();
            cv_.notify_one();
        }
    }
};

// Usage example
class PostgresConnection {
public:
    void execute(const std::string& query) {
        // Execute query
    }
};

auto pool = ConnectionPool<PostgresConnection>(
    10,
    []() { return std::make_unique<PostgresConnection>(); }
);

// Acquire and use connection
{
    auto conn = pool.acquire();
    conn->execute("SELECT * FROM markets");
} // Connection automatically released
```

### Repository Pattern with Prepared Statements

```cpp
class MarketRepository {
private:
    ConnectionPool<PostgresConnection>& pool_;
    
public:
    explicit MarketRepository(ConnectionPool<PostgresConnection>& pool)
        : pool_(pool) {}
    
    std::optional<Market> find_by_id(const std::string& id) {
        auto conn = pool_.acquire();
        
        // Use prepared statement
        auto stmt = conn->prepare("SELECT * FROM markets WHERE id = $1");
        stmt.bind(0, id);
        
        auto result = stmt.execute();
        
        if (result.empty()) {
            return std::nullopt;
        }
        
        return Market{
            result[0]["id"].as_string(),
            result[0]["name"].as_string(),
            result[0]["volume"].as_double()
        };
    }
    
    std::vector<Market> find_all(const MarketFilters& filters) {
        auto conn = pool_.acquire();
        
        std::ostringstream query;
        query << "SELECT * FROM markets WHERE 1=1";
        
        std::vector<std::string> params;
        
        if (filters.status) {
            query << " AND status = $" << (params.size() + 1);
            params.push_back(*filters.status);
        }
        
        if (filters.limit) {
            query << " LIMIT $" << (params.size() + 1);
            params.push_back(std::to_string(*filters.limit));
        }
        
        auto stmt = conn->prepare(query.str());
        for (size_t i = 0; i < params.size(); ++i) {
            stmt.bind(i, params[i]);
        }
        
        auto result = stmt.execute();
        
        std::vector<Market> markets;
        for (const auto& row : result) {
            markets.push_back({
                row["id"].as_string(),
                row["name"].as_string(),
                row["volume"].as_double()
            });
        }
        
        return markets;
    }
    
    Market create(const CreateMarketDto& dto) {
        auto conn = pool_.acquire();
        
        auto stmt = conn->prepare(
            "INSERT INTO markets (id, name, description, created_at) "
            "VALUES ($1, $2, $3, NOW()) RETURNING *");
        
        stmt.bind(0, dto.id);
        stmt.bind(1, dto.name);
        stmt.bind(2, dto.description);
        
        auto result = stmt.execute();
        
        return Market{
            result[0]["id"].as_string(),
            result[0]["name"].as_string(),
            result[0]["volume"].as_double()
        };
    }
};
```

## Caching Patterns

### LRU Cache Implementation

```cpp
template<typename Key, typename Value>
class LruCache {
private:
    struct Node {
        Key key;
        Value value;
        std::shared_ptr<Node> prev;
        std::shared_ptr<Node> next;
    };
    
    size_t capacity_;
    std::unordered_map<Key, std::shared_ptr<Node>> map_;
    std::shared_ptr<Node> head_;
    std::shared_ptr<Node> tail_;
    mutable std::shared_mutex mutex_;
    
public:
    explicit LruCache(size_t capacity) : capacity_(capacity) {
        head_ = std::make_shared<Node>();
        tail_ = std::make_shared<Node>();
        head_->next = tail_;
        tail_->prev = head_;
    }
    
    std::optional<Value> get(const Key& key) {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        
        auto it = map_.find(key);
        if (it == map_.end()) {
            return std::nullopt;
        }
        
        lock.unlock();
        std::unique_lock<std::shared_mutex> write_lock(mutex_);
        
        // Move to front
        move_to_front(it->second);
        return it->second->value;
    }
    
    void put(const Key& key, Value value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        
        auto it = map_.find(key);
        if (it != map_.end()) {
            // Update existing
            it->second->value = std::move(value);
            move_to_front(it->second);
            return;
        }
        
        // Add new node
        auto node = std::make_shared<Node>();
        node->key = key;
        node->value = std::move(value);
        
        map_[key] = node;
        add_to_front(node);
        
        // Evict if necessary
        if (map_.size() > capacity_) {
            auto removed = remove_tail();
            map_.erase(removed->key);
        }
    }
    
private:
    void move_to_front(std::shared_ptr<Node> node) {
        remove_node(node);
        add_to_front(node);
    }
    
    void add_to_front(std::shared_ptr<Node> node) {
        node->next = head_->next;
        node->prev = head_;
        head_->next->prev = node;
        head_->next = node;
    }
    
    void remove_node(std::shared_ptr<Node> node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }
    
    std::shared_ptr<Node> remove_tail() {
        auto node = tail_->prev;
        remove_node(node);
        return node;
    }
};

// Usage
LruCache<std::string, Market> market_cache(1000);

// Get with fallback
auto get_market_with_cache(const std::string& id, MarketRepository& repo) {
    if (auto cached = market_cache.get(id)) {
        return *cached;
    }
    
    auto market = repo.find_by_id(id);
    if (market) {
        market_cache.put(id, *market);
    }
    
    return market;
}
```

### Redis-style Cache Integration

```cpp
class RedisCache {
private:
    std::unique_ptr<redisContext> context_;
    std::mutex mutex_;
    
public:
    explicit RedisCache(const std::string& host, int port) {
        context_.reset(redisConnect(host.c_str(), port));
        if (!context_ || context_->err) {
            throw std::runtime_error("Failed to connect to Redis");
        }
    }
    
    std::optional<std::string> get(const std::string& key) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto reply = static_cast<redisReply*>(
            redisCommand(context_.get(), "GET %s", key.c_str()));
        
        if (!reply) {
            throw std::runtime_error("Redis command failed");
        }
        
        std::unique_ptr<redisReply, decltype(&freeReplyObject)> reply_ptr(
            reply, freeReplyObject);
        
        if (reply->type == REDIS_REPLY_STRING) {
            return std::string(reply->str, reply->len);
        }
        
        return std::nullopt;
    }
    
    void set(const std::string& key, const std::string& value, 
             std::optional<int> ttl_seconds = std::nullopt) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        redisReply* reply;
        if (ttl_seconds) {
            reply = static_cast<redisReply*>(
                redisCommand(context_.get(), "SETEX %s %d %s",
                           key.c_str(), *ttl_seconds, value.c_str()));
        } else {
            reply = static_cast<redisReply*>(
                redisCommand(context_.get(), "SET %s %s",
                           key.c_str(), value.c_str()));
        }
        
        if (!reply) {
            throw std::runtime_error("Redis command failed");
        }
        
        freeReplyObject(reply);
    }
    
    void del(const std::string& key) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        auto reply = static_cast<redisReply*>(
            redisCommand(context_.get(), "DEL %s", key.c_str()));
        
        if (reply) {
            freeReplyObject(reply);
        }
    }
};
```

## Logging and Monitoring

### Structured Logger

```cpp
enum class LogLevel {
    Debug,
    Info,
    Warning,
    Error,
    Fatal
};

class Logger {
private:
    LogLevel min_level_ = LogLevel::Info;
    std::mutex mutex_;
    std::ofstream file_;
    
public:
    explicit Logger(const std::string& log_file)
        : file_(log_file, std::ios::app) {}
    
    template<typename... Args>
    void log(LogLevel level, const std::string& message, Args&&... args) {
        if (level < min_level_) return;
        
        std::lock_guard<std::mutex> lock(mutex_);
        
        // Build JSON log entry
        nlohmann::json entry;
        entry["timestamp"] = get_timestamp();
        entry["level"] = level_to_string(level);
        entry["message"] = message;
        
        add_context(entry, std::forward<Args>(args)...);
        
        std::cout << entry.dump() << std::endl;
        file_ << entry.dump() << std::endl;
    }
    
    template<typename... Args>
    void info(const std::string& message, Args&&... args) {
        log(LogLevel::Info, message, std::forward<Args>(args)...);
    }
    
    template<typename... Args>
    void error(const std::string& message, Args&&... args) {
        log(LogLevel::Error, message, std::forward<Args>(args)...);
    }
    
private:
    std::string get_timestamp() {
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);
        std::stringstream ss;
        ss << std::put_time(std::localtime(&time), "%Y-%m-%d %H:%M:%S");
        return ss.str();
    }
    
    std::string level_to_string(LogLevel level) {
        switch (level) {
            case LogLevel::Debug: return "DEBUG";
            case LogLevel::Info: return "INFO";
            case LogLevel::Warning: return "WARNING";
            case LogLevel::Error: return "ERROR";
            case LogLevel::Fatal: return "FATAL";
        }
        return "UNKNOWN";
    }
    
    void add_context(nlohmann::json& entry) {}
    
    template<typename T, typename... Rest>
    void add_context(nlohmann::json& entry, const std::string& key, T&& value, Rest&&... rest) {
        entry[key] = std::forward<T>(value);
        add_context(entry, std::forward<Rest>(rest)...);
    }
};

// Usage
Logger logger("app.log");
logger.info("Processing market", "market_id", "123", "user_id", "456");
logger.error("Database connection failed", "error", "timeout");
```

## Message Queue Pattern

```cpp
template<typename T>
class MessageQueue {
private:
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;
    
public:
    void push(T message) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(std::move(message));
        cv_.notify_one();
    }
    
    std::optional<T> pop(std::chrono::milliseconds timeout = std::chrono::milliseconds::max()) {
        std::unique_lock<std::mutex> lock(mutex_);
        
        if (!cv_.wait_for(lock, timeout, [this] { return !queue_.empty() || stopped_; })) {
            return std::nullopt;
        }
        
        if (stopped_ && queue_.empty()) {
            return std::nullopt;
        }
        
        T message = std::move(queue_.front());
        queue_.pop();
        return message;
    }
    
    void stop() {
        std::lock_guard<std::mutex> lock(mutex_);
        stopped_ = true;
        cv_.notify_all();
    }
};

// Background worker pattern
class WorkerThread {
private:
    MessageQueue<std::function<void()>>& queue_;
    std::thread thread_;
    std::atomic<bool> running_{true};
    
public:
    explicit WorkerThread(MessageQueue<std::function<void()>>& queue)
        : queue_(queue) {
        thread_ = std::thread([this] { run(); });
    }
    
    ~WorkerThread() {
        running_ = false;
        if (thread_.joinable()) {
            thread_.join();
        }
    }
    
private:
    void run() {
        while (running_) {
            auto task = queue_.pop(std::chrono::milliseconds(100));
            if (task) {
                (*task)();
            }
        }
    }
};
```

## Metrics and Monitoring

```cpp
class Metrics {
private:
    struct Counter {
        std::atomic<int64_t> value{0};
    };
    
    struct Histogram {
        std::vector<double> buckets{0.001, 0.01, 0.1, 1.0, 10.0};
        std::vector<std::atomic<int64_t>> counts;
        
        Histogram() : counts(buckets.size() + 1) {}
    };
    
    std::unordered_map<std::string, Counter> counters_;
    std::unordered_map<std::string, Histogram> histograms_;
    std::shared_mutex mutex_;
    
public:
    void increment_counter(const std::string& name, int64_t value = 1) {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        auto it = counters_.find(name);
        
        if (it == counters_.end()) {
            lock.unlock();
            std::unique_lock<std::shared_mutex> write_lock(mutex_);
            it = counters_.emplace(name, Counter{}).first;
        }
        
        it->second.value.fetch_add(value);
    }
    
    void observe(const std::string& name, double value) {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        auto it = histograms_.find(name);
        
        if (it == histograms_.end()) {
            lock.unlock();
            std::unique_lock<std::shared_mutex> write_lock(mutex_);
            it = histograms_.emplace(name, Histogram{}).first;
        }
        
        auto& hist = it->second;
        size_t bucket = 0;
        for (size_t i = 0; i < hist.buckets.size(); ++i) {
            if (value <= hist.buckets[i]) {
                bucket = i;
                break;
            }
        }
        
        hist.counts[bucket].fetch_add(1);
    }
    
    class Timer {
    private:
        Metrics& metrics_;
        std::string name_;
        std::chrono::steady_clock::time_point start_;
        
    public:
        Timer(Metrics& metrics, std::string name)
            : metrics_(metrics), name_(std::move(name)),
              start_(std::chrono::steady_clock::now()) {}
        
        ~Timer() {
            auto duration = std::chrono::steady_clock::now() - start_;
            auto seconds = std::chrono::duration<double>(duration).count();
            metrics_.observe(name_, seconds);
        }
    };
    
    Timer timer(const std::string& name) {
        return Timer(*this, name);
    }
};

// Usage
Metrics metrics;

void process_request() {
    auto timer = metrics.timer("request_duration");
    
    // Process request
    
    metrics.increment_counter("requests_processed");
}
```

**Remember**: C++ system patterns enable building high-performance, scalable backend systems. Use RAII, smart pointers, and modern concurrency primitives for robust system code.
