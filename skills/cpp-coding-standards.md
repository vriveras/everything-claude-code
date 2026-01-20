---
name: cpp-coding-standards
description: Modern C++ coding standards, best practices, naming conventions, and design patterns for C++11/14/17/20.
---

# C++ Coding Standards & Best Practices

Modern C++ coding standards emphasizing safety, performance, and maintainability.

## Code Quality Principles

### 1. Modern C++ First
- Use C++11/14/17/20 features
- Prefer standard library over custom implementations
- RAII for all resource management
- Smart pointers over raw pointers

### 2. Zero-Cost Abstractions
- Templates for compile-time polymorphism
- constexpr for compile-time computation
- Inline functions for performance
- Avoid virtual functions unless needed

### 3. Type Safety
- Strong typing over primitive types
- enum class over plain enum
- Explicit constructors
- const correctness everywhere

### 4. Memory Safety
- No manual new/delete in application code
- RAII wrappers for all resources
- Move semantics for efficiency
- Smart pointers for ownership

## Naming Conventions

### Variables and Functions

```cpp
// ✅ GOOD: Clear, descriptive names
int connectionTimeout = 30;
std::string userName = "john_doe";
bool isAuthenticated = false;

void processRequest(const Request& request);
size_t calculateChecksum(const uint8_t* data, size_t length);
bool validateInput(const std::string& input);

// ❌ BAD: Unclear abbreviations
int ct = 30;
std::string un = "john_doe";
bool auth = false;
```

### Classes and Structs

```cpp
// ✅ GOOD: PascalCase for types
class TcpConnection {
    // Implementation
};

struct HttpRequest {
    std::string method;
    std::string path;
    std::map<std::string, std::string> headers;
};

// ❌ BAD: Inconsistent naming
class tcp_connection { };
struct http_request { };
```

### Constants and Enums

```cpp
// ✅ GOOD: Named constants
constexpr int kMaxConnections = 100;
constexpr double kPi = 3.14159265359;
constexpr std::string_view kDefaultEncoding = "UTF-8";  // C++17+
// For pre-C++17: const char* kDefaultEncoding = "UTF-8";

// ✅ GOOD: enum class for type safety
enum class Status {
    Idle,
    Running,
    Paused,
    Stopped
};

// ❌ BAD: Plain enum (pollutes namespace)
enum Status {
    STATUS_IDLE,
    STATUS_RUNNING
};
```

### Member Variables

```cpp
// ✅ GOOD: Prefix or suffix to distinguish members
class Connection {
private:
    int socket_;           // Trailing underscore
    std::string address_;
    bool isConnected_;

public:
    int getSocket() const { return socket_; }
};

// Alternative style
class Connection {
private:
    int m_socket;          // m_ prefix
    std::string m_address;
};
```

## RAII Patterns

### Resource Management

```cpp
// ✅ GOOD: RAII wrapper for file
class File {
private:
    FILE* file_;

public:
    explicit File(const char* filename, const char* mode)
        : file_(fopen(filename, mode)) {
        if (!file_) {
            throw std::runtime_error("Failed to open file");
        }
    }

    ~File() {
        if (file_) {
            fclose(file_);
        }
    }

    // Delete copy, allow move
    File(const File&) = delete;
    File& operator=(const File&) = delete;

    File(File&& other) noexcept : file_(other.file_) {
        other.file_ = nullptr;
    }

    FILE* get() const { return file_; }
};

// Usage
void processFile(const char* filename) {
    File f(filename, "r");
    // File automatically closed
}
```

### Lock Guards

```cpp
// ✅ GOOD: RAII for mutex locking
class Counter {
private:
    mutable std::mutex mutex_;
    int value_ = 0;

public:
    void increment() {
        std::lock_guard<std::mutex> lock(mutex_);
        ++value_;
    }

    int getValue() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return value_;
    }
};

// ❌ BAD: Manual lock/unlock
void increment() {
    mutex_.lock();
    ++value_;
    mutex_.unlock();  // Easy to forget!
}
```

## Smart Pointers

### Unique Ownership

```cpp
// ✅ GOOD: unique_ptr for exclusive ownership
std::unique_ptr<Connection> createConnection(const std::string& host) {
    return std::make_unique<Connection>(host);
}

class Server {
private:
    std::vector<std::unique_ptr<Connection>> connections_;

public:
    void addConnection(std::unique_ptr<Connection> conn) {
        connections_.push_back(std::move(conn));
    }
};

// ❌ BAD: Raw pointer
Connection* createConnection(const std::string& host) {
    return new Connection(host);  // Who owns this?
}
```

### Shared Ownership

```cpp
// ✅ GOOD: shared_ptr for shared ownership
class Cache {
private:
    std::map<std::string, std::shared_ptr<Resource>> resources_;

public:
    std::shared_ptr<Resource> getResource(const std::string& key) {
        auto it = resources_.find(key);
        if (it != resources_.end()) {
            return it->second;  // Shared ownership
        }
        return nullptr;
    }

    void addResource(const std::string& key, std::shared_ptr<Resource> resource) {
        resources_[key] = resource;
    }
};
```

### Weak References

```cpp
// ✅ GOOD: weak_ptr to break circular references
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // Prevents circular reference

    void setPrev(std::shared_ptr<Node> node) {
        prev = node;
    }

    std::shared_ptr<Node> getPrev() const {
        return prev.lock();  // Returns nullptr if expired
    }
};
```

## Move Semantics

### Implementing Move Operations

```cpp
// ✅ GOOD: Support move for efficiency
class Buffer {
private:
    uint8_t* data_;
    size_t size_;

public:
    // Constructor
    explicit Buffer(size_t size)
        : data_(new uint8_t[size]), size_(size) { }

    // Destructor
    ~Buffer() {
        delete[] data_;
    }

    // Delete copy operations
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    // Move constructor
    Buffer(Buffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

    size_t size() const { return size_; }
};
```

### Using std::move

```cpp
// ✅ GOOD: Move expensive objects
std::vector<std::string> createLargeVector() {
    std::vector<std::string> result;
    // Fill with data
    return result;  // Implicit move
}

void processData() {
    auto data = createLargeVector();  // Move, no copy
    
    Buffer buffer(1024);
    storeBuffer(std::move(buffer));  // Explicit move
    // buffer is now in moved-from state
}
```

## Const Correctness

### Const Member Functions

```cpp
// ✅ GOOD: Mark non-modifying functions const
class Point {
private:
    double x_, y_;

public:
    Point(double x, double y) : x_(x), y_(y) { }

    // Const member functions
    double getX() const { return x_; }
    double getY() const { return y_; }
    double distance() const {
        return std::sqrt(x_ * x_ + y_ * y_);
    }

    // Non-const member functions
    void setX(double x) { x_ = x; }
    void setY(double y) { y_ = y; }
};

// Benefits const correctness
void printPoint(const Point& p) {
    std::cout << p.getX() << ", " << p.getY() << std::endl;
}
```

### Const Pointers and References

```cpp
// ✅ GOOD: Use const for parameters that shouldn't change
void processData(const std::vector<int>& data) {
    // data cannot be modified
}

// Pointer to const data
void readBuffer(const uint8_t* buffer, size_t size) {
    // Cannot modify buffer contents
}

// Const pointer to data
void initBuffer(uint8_t* const buffer, size_t size) {
    // Cannot change buffer pointer, but can modify contents
}

// Const pointer to const data
void inspectBuffer(const uint8_t* const buffer, size_t size) {
    // Cannot modify buffer or pointer
}
```

## Template Best Practices

### Function Templates

```cpp
// ✅ GOOD: Generic algorithms with templates
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// ✅ GOOD: Constrain templates with concepts (C++20)
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<Numeric T>
T add(T a, T b) {
    return a + b;
}
```

### Class Templates

```cpp
// ✅ GOOD: Type-safe generic containers
template<typename T>
class Stack {
private:
    std::vector<T> elements_;

public:
    void push(const T& element) {
        elements_.push_back(element);
    }

    void push(T&& element) {
        elements_.push_back(std::move(element));
    }

    T pop() {
        if (elements_.empty()) {
            throw std::runtime_error("Stack is empty");
        }
        T element = std::move(elements_.back());
        elements_.pop_back();
        return element;
    }

    bool empty() const {
        return elements_.empty();
    }
};
```

### Template Specialization

```cpp
// ✅ GOOD: Specialize for specific types
template<typename T>
struct Hasher {
    size_t operator()(const T& value) const {
        return std::hash<T>{}(value);
    }
};

// Specialization for custom type
template<>
struct Hasher<Point> {
    size_t operator()(const Point& p) const {
        size_t h1 = std::hash<double>{}(p.getX());
        size_t h2 = std::hash<double>{}(p.getY());
        return h1 ^ (h2 << 1);
    }
};
```

## Error Handling

### Exceptions vs Error Codes

```cpp
// ✅ GOOD: Use exceptions for exceptional cases
class FileReader {
public:
    std::string readFile(const std::string& path) {
        std::ifstream file(path);
        if (!file) {
            throw std::runtime_error("Failed to open file: " + path);
        }

        std::stringstream buffer;
        buffer << file.rdbuf();
        return buffer.str();
    }
};

// ✅ GOOD: Use error codes for expected failures
enum class ErrorCode {
    Success,
    NotFound,
    PermissionDenied,
    InvalidInput
};

class Cache {
public:
    ErrorCode get(const std::string& key, std::string& value) {
        auto it = data_.find(key);
        if (it == data_.end()) {
            return ErrorCode::NotFound;
        }
        value = it->second;
        return ErrorCode::Success;
    }

private:
    std::map<std::string, std::string> data_;
};
```

### Exception Safety

```cpp
// ✅ GOOD: Strong exception guarantee
class Transaction {
private:
    Database& db_;
    bool committed_ = false;

public:
    explicit Transaction(Database& db) : db_(db) {
        db_.beginTransaction();
    }

    ~Transaction() {
        if (!committed_) {
            try {
                db_.rollback();
            } catch (...) {
                // Log but don't throw from destructor
            }
        }
    }

    void commit() {
        db_.commit();
        committed_ = true;
    }

    // No-throw swap for exception-safe assignment
    void swap(Transaction& other) noexcept {
        std::swap(db_, other.db_);
        std::swap(committed_, other.committed_);
    }
};
```

## Modern C++ Features

### Range-Based For Loops

```cpp
// ✅ GOOD: Use range-based for
std::vector<int> numbers = {1, 2, 3, 4, 5};

for (const auto& num : numbers) {
    std::cout << num << std::endl;
}

// Modify elements
for (auto& num : numbers) {
    num *= 2;
}
```

### Auto Type Deduction

```cpp
// ✅ GOOD: Use auto for obvious types
auto connection = createConnection("localhost");
auto result = calculateComplexValue();

// ❌ BAD: Don't use auto when type is important
auto x = 5;  // Is this int, long, or size_t?
int x = 5;   // Explicit is better
```

### Lambda Expressions

```cpp
// ✅ GOOD: Use lambdas for callbacks and algorithms
std::vector<int> numbers = {1, 2, 3, 4, 5};

// Simple lambda
auto print = [](int n) { std::cout << n << std::endl; };
std::for_each(numbers.begin(), numbers.end(), print);

// Lambda with capture
int threshold = 3;
auto filtered = std::count_if(numbers.begin(), numbers.end(),
    [threshold](int n) { return n > threshold; });

// Mutable lambda
int sum = 0;
std::for_each(numbers.begin(), numbers.end(),
    [&sum](int n) mutable { sum += n; });
```

### Structured Bindings (C++17)

```cpp
// ✅ GOOD: Unpack tuples and structs
std::map<std::string, int> scores = {{"Alice", 95}, {"Bob", 87}};

for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << std::endl;
}

// Unpack function return
auto [success, value] = parseInteger("123");
```

## Code Organization

### Header Files

```cpp
// ✅ GOOD: Header file structure
#pragma once  // or include guards

#include <string>
#include <vector>

// Forward declarations
class Connection;

class Server {
public:
    explicit Server(int port);
    ~Server();

    void start();
    void stop();

private:
    class Impl;  // Forward declare implementation
    std::unique_ptr<Impl> impl_;
};
```

### Source Files

```cpp
// ✅ GOOD: Implementation file
#include "server.h"

#include <iostream>
#include <thread>

// Implementation details in anonymous namespace
namespace {
    constexpr int kDefaultTimeout = 30;

    void helperFunction() {
        // Internal helper
    }
}

// Public interface implementation
Server::Server(int port) : impl_(std::make_unique<Impl>(port)) {
}

Server::~Server() = default;
```

## Performance Best Practices

### Avoid Unnecessary Copies

```cpp
// ✅ GOOD: Pass by const reference
void processData(const std::vector<int>& data) {
    // No copy
}

// ✅ GOOD: Return by value (RVO/move)
std::vector<int> createData() {
    std::vector<int> result;
    // Fill data
    return result;  // No copy due to RVO
}

// ❌ BAD: Pass by value unnecessarily
void processData(std::vector<int> data) {
    // Unnecessary copy
}
```

### Reserve Container Capacity

```cpp
// ✅ GOOD: Reserve capacity when size is known
std::vector<int> numbers;
numbers.reserve(1000);  // Avoid reallocations

for (int i = 0; i < 1000; ++i) {
    numbers.push_back(i);
}
```

### Use emplace Instead of push

```cpp
// ✅ GOOD: Construct in place
std::vector<std::pair<int, std::string>> items;
items.emplace_back(1, "first");   // Construct directly

// ❌ BAD: Construct then move
items.push_back(std::make_pair(1, "first"));
```

**Remember**: Modern C++ emphasizes safety through RAII, type safety, const correctness, and leveraging the standard library. Write clear, safe code that the compiler can optimize.
