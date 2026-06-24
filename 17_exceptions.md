# Exception Handling and Error Management

## Overview

Exception handling is C++'s primary mechanism for dealing with runtime errors. This chapter covers exception mechanics, safety guarantees, modern error handling with `std::error_code`, and `std::expected` (C++23). RAII — covered in depth in [Advanced Features](12_advanced_features.md) — is the foundation of exception-safe resource management.

```
┌────────────────────────────────────────────────────────┐
│           EXCEPTION HANDLING ECOSYSTEM                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│  EXCEPTIONS      │  ERROR CODES     │  MODERN          │
│  ──────────      │  ───────────     │  ──────          │
│  • try/catch     │  • error_code    │  • expected      │
│  • throw         │  • error_category│  • noexcept      │
│  • Standard exc  │  • system_error  │  • Contracts     │
│  • Custom exc    │                  │                  │
│                                                        │
│  SAFETY GUARANTEES                                     │
│  ─────────────────                                     │
│  • No-throw (noexcept)                                 │
│  • Strong (commit-or-rollback)                         │
│  • Basic (no leaks, valid state)                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Exception Basics

### try-catch-throw
```cpp
#include <iostream>
#include <stdexcept>

double divide(double a, double b) {
    if (b == 0.0) {
        throw std::invalid_argument("Division by zero");
    }
    return a / b;
}

int main() {
    try {
        double result = divide(10.0, 0.0);
        std::cout << "Result: " << result << std::endl;
    }
    catch (const std::invalid_argument& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    catch (const std::exception& e) {
        std::cerr << "Unknown error: " << e.what() << std::endl;
    }
    catch (...) {
        std::cerr << "Unknown exception" << std::endl;
    }
    
    return 0;
}
```

### Exception Flow
```
Normal execution:
main() → function_a() → function_b() → return → return → return

Exception thrown:
main() → function_a() → function_b() → throw
   ↑                                      │
   └──────────── unwinds stack ───────────┘
                 (catch or terminate)

Stack unwinding:
┌─────────────────────────────────────────────┐
│ function_c() throws exception               │
│ function_b() stack unwound, destructors run │
│ function_a() stack unwound, destructors run │
│ main() catches exception                    │
└─────────────────────────────────────────────┘
```

---

## Standard Exceptions

### Exception Hierarchy
```cpp
#include <exception>
#include <stdexcept>

/*
Exception Hierarchy:

std::exception
├── std::bad_alloc
├── std::bad_cast
├── std::bad_typeid
├── std::bad_exception
├── std::bad_function_call (C++11)
├── std::bad_weak_ptr (C++11)
├── std::bad_optional_access (C++17)
├── std::bad_variant_access (C++17)
├── std::bad_any_cast (C++17)
└── std::logic_error
    ├── std::invalid_argument
    ├── std::domain_error
    ├── std::length_error
    ├── std::out_of_range
    └── std::future_error (C++11)
└── std::runtime_error
    ├── std::range_error
    ├── std::overflow_error
    ├── std::underflow_error
    └── std::system_error (C++11)
        ├── std::ios_base::failure
        └── std::filesystem::filesystem_error (C++17)
*/

void example_exceptions() {
    try {
        // Logic errors (programming bugs)
        throw std::invalid_argument("Invalid argument");
        throw std::out_of_range("Index out of range");
        throw std::length_error("Length exceeded");
        throw std::domain_error("Domain error");
        
        // Runtime errors (external conditions)
        throw std::runtime_error("Runtime error");
        throw std::overflow_error("Overflow");
        throw std::underflow_error("Underflow");
        
        // System errors
        throw std::system_error(errno, std::system_category());
        
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
    }
}
```

### Using Standard Exceptions
```cpp
#include <stdexcept>
#include <vector>

class BankAccount {
    double balance_;
    
public:
    BankAccount(double initial) : balance_(initial) {
        if (initial < 0) {
            throw std::invalid_argument("Initial balance cannot be negative");
        }
    }
    
    void withdraw(double amount) {
        if (amount < 0) {
            throw std::invalid_argument("Amount cannot be negative");
        }
        if (amount > balance_) {
            throw std::runtime_error("Insufficient funds");
        }
        balance_ -= amount;
    }
    
    double get_balance() const { return balance_; }
};

void process_accounts(const std::vector<BankAccount>& accounts, size_t index) {
    if (index >= accounts.size()) {
        throw std::out_of_range("Account index out of range");
    }
    // Process account...
}
```

---

## Custom Exceptions

### Creating Custom Exception Classes
```cpp
#include <exception>
#include <string>

// Simple custom exception
class MyException : public std::exception {
    std::string message_;
    
public:
    explicit MyException(const std::string& msg) : message_(msg) {}
    
    const char* what() const noexcept override {
        return message_.c_str();
    }
};

// Exception with error code
class DatabaseError : public std::runtime_error {
    int error_code_;
    
public:
    DatabaseError(const std::string& msg, int code)
        : std::runtime_error(msg), error_code_(code) {}
    
    int error_code() const { return error_code_; }
};

// Exception hierarchy for domain
class NetworkException : public std::runtime_error {
    using std::runtime_error::runtime_error;
};

class ConnectionError : public NetworkException {
    using NetworkException::NetworkException;
};

class TimeoutError : public NetworkException {
    using NetworkException::NetworkException;
};

// Usage
void connect_to_server() {
    throw ConnectionError("Failed to connect to server");
}

int main() {
    try {
        connect_to_server();
    }
    catch (const ConnectionError& e) {
        std::cerr << "Connection error: " << e.what() << std::endl;
    }
    catch (const NetworkException& e) {
        std::cerr << "Network error: " << e.what() << std::endl;
    }
    
    return 0;
}
```

---

## Exception Safety Guarantees

### Three Levels of Safety
```
┌────────────────────────────────────────────────────────┐
│          Exception Safety Guarantees                   │
├────────────────────────────────────────────────────────┤
│                                                        │
│ 1. NO-THROW GUARANTEE (noexcept)                       │
│    • Never throws exceptions                           │
│    • Required for: destructors, move operations, swap  │
│    • Throwing from a noexcept function → std::terminate│
│                                                        │
│ 2. STRONG GUARANTEE (commit-or-rollback)               │
│    • Either completes successfully OR                  │
│    • Leaves state unchanged (as if not called)         │
│    • Example: vector::push_back (if realloc fails)     │
│                                                        │
│ 3. BASIC GUARANTEE                                     │
│    • No resource leaks                                 │
│    • Object remains in valid state                     │
│    • State may be changed but is consistent            │
│    • Example: most STL operations                      │
│                                                        │
│ 4. NO GUARANTEE (avoid!)                               │
│    • May leak resources                                │
│    • May leave object in invalid state                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

💡 **Hunch**: Containers use `noexcept` move operations to choose move-over-copy during reallocation (strong guarantee). Mark moves `noexcept` when possible — see [Advanced Features](12_advanced_features.md).

### No-Throw Guarantee (noexcept)
```cpp
#include <utility>

class Resource {
    int* data_;
    
public:
    // Destructor must be noexcept
    ~Resource() noexcept {
        delete data_;
    }
    
    // Move operations should be noexcept
    Resource(Resource&& other) noexcept
        : data_(other.data_) {
        other.data_ = nullptr;
    }
    
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) {
            delete data_;
            data_ = other.data_;
            other.data_ = nullptr;
        }
        return *this;
    }
    
    // Swap should be noexcept
    void swap(Resource& other) noexcept {
        std::swap(data_, other.data_);
    }
};

// Conditional noexcept
template<typename T>
class Container {
public:
    void push_back(T&& value) noexcept(noexcept(T(std::move(value)))) {
        // noexcept if T's move constructor is noexcept
    }
};
```

### Strong Guarantee
```cpp
#include <memory>
#include <utility>

class Widget {
    int value_;
    std::unique_ptr<int[]> data_;
    
public:
    // Strong guarantee using copy-and-swap
    void set_data(int value, size_t size) {
        // Create new data
        auto new_data = std::make_unique<int[]>(size);
        // May throw, but Widget unchanged
        
        for (size_t i = 0; i < size; ++i) {
            new_data[i] = value;
        }
        
        // No throw from here
        value_ = value;
        data_ = std::move(new_data);
        // Commit: success, or above threw and Widget unchanged
    }
    
    // Strong guarantee for assignment
    Widget& operator=(const Widget& other) {
        Widget temp(other);  // May throw, but *this unchanged
        swap(temp);          // No throw
        return *this;
    }
    
    void swap(Widget& other) noexcept {
        std::swap(value_, other.value_);
        std::swap(data_, other.data_);
    }
};
```

### Basic Guarantee
```cpp
class Logger {
    std::vector<std::string> logs_;
    
public:
    void add_log(const std::string& msg) {
        // Basic guarantee: if push_back throws,
        // logs_ is in valid state (unchanged)
        logs_.push_back(msg);
        
        // But state is changed (msg added)
        // No leak, object valid, but state modified
    }
};
```

---

## noexcept Specification

### Using noexcept
```cpp
#include <type_traits>

// Always noexcept
void guaranteed_no_throw() noexcept {
    // Must not throw
}

// Conditional noexcept
template<typename T>
void conditional_no_throw(T&& value) 
    noexcept(std::is_nothrow_move_constructible_v<T>) {
    // noexcept depends on T
}

// Check if noexcept
void check_noexcept() {
    static_assert(noexcept(guaranteed_no_throw()));
    
    constexpr bool is_noexcept = noexcept(std::declval<int>() + 1);
}

// Destructors are implicitly noexcept — throwing calls std::terminate
class MyClass {
public:
    ~MyClass() {  // implicitly noexcept
        // Don't throw in destructor!
    }
};

// Override only if you truly must (almost never)
class DangerousClass {
public:
    ~DangerousClass() noexcept(false) {
        // If this throws during stack unwinding → std::terminate
    }
};
```

### noexcept and Performance
```cpp
#include <vector>

class MoveOnly {
public:
    MoveOnly() = default;
    MoveOnly(MoveOnly&&) noexcept = default;  // Important!
    MoveOnly& operator=(MoveOnly&&) noexcept = default;
    
    // If move is noexcept, vector can use move during reallocation
    // Otherwise, vector must copy (for strong guarantee)
};

void performance_example() {
    std::vector<MoveOnly> vec;
    
    // If MoveOnly's move is noexcept:
    //   - vector uses move during reallocation → fast
    // If move might throw:
    //   - vector uses copy during reallocation → slow
    
    vec.push_back(MoveOnly{});
}
```

---

## Error Codes (Alternative to Exceptions)

`std::error_code` is a lightweight, non-throwing error carrier. Ideal for performance-critical paths, C interop, and filesystem/I/O APIs that offer `error_code` overloads — see [I/O and Filesystem](16_io_filesystem.md).

### std::error_code
```cpp
#include <system_error>
#include <iostream>

// Define custom error enum
enum class MyError {
    Success = 0,
    NotFound,
    PermissionDenied,
    InvalidInput
};

// Create error category
class MyErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override {
        return "MyError";
    }
    
    std::string message(int ev) const override {
        switch (static_cast<MyError>(ev)) {
            case MyError::Success: return "Success";
            case MyError::NotFound: return "Not found";
            case MyError::PermissionDenied: return "Permission denied";
            case MyError::InvalidInput: return "Invalid input";
            default: return "Unknown error";
        }
    }
};

const MyErrorCategory& my_error_category() {
    static MyErrorCategory instance;
    return instance;
}

std::error_code make_error_code(MyError e) {
    return {static_cast<int>(e), my_error_category()};
}

// Must specialize is_error_code_enum
namespace std {
    template<>
    struct is_error_code_enum<MyError> : true_type {};
}

// Usage
std::error_code do_something() {
    // ...
    if (/* error condition */) {
        return MyError::NotFound;
    }
    return MyError::Success;
}

int main() {
    auto ec = do_something();
    
    if (ec) {  // Check if error
        std::cerr << "Error: " << ec.message() << std::endl;
    }
    
    if (ec == MyError::NotFound) {
        // Handle specific error
    }
    
    return 0;
}
```

### std::system_error
```cpp
#include <system_error>
#include <fstream>

void read_file(const char* filename) {
    std::ifstream file(filename);
    
    if (!file) {
        throw std::system_error(
            errno,
            std::system_category(),
            "Failed to open file"
        );
    }
    
    // Read file...
}

int main() {
    try {
        read_file("nonexistent.txt");
    }
    catch (const std::system_error& e) {
        std::cerr << "System error: " << e.what() << std::endl;
        std::cerr << "Error code: " << e.code() << std::endl;
        std::cerr << "Error value: " << e.code().value() << std::endl;
    }
    
    return 0;
}
```

---

## std::expected (C++23)

A value-or-error type for regular, expected failures — composable like `std::optional` but with a typed error channel. Check `__cpp_lib_expected` before use; Apple Clang/libc++ added it in recent Xcode releases.

### Result Type Pattern
```cpp
#include <expected>  // C++23
#include <string>

// Return value or error
std::expected<int, std::string> divide(int a, int b) {
    if (b == 0) {
        return std::unexpected("Division by zero");
    }
    return a / b;
}

int main() {
    auto result = divide(10, 2);
    
    if (result) {  // Has value
        std::cout << "Result: " << *result << std::endl;
        // Or: result.value()
    } else {  // Has error
        std::cerr << "Error: " << result.error() << std::endl;
    }
    
    // With default value
    int value = result.value_or(0);
    
    // Transform operations
    auto result2 = divide(10, 2)
        .transform([](int x) { return x * 2; })
        .or_else([](auto&& error) {
            std::cerr << error << std::endl;
            return std::expected<int, std::string>(0);
        });
    
    return 0;
}
```

### Comparison: Exceptions vs Error Codes vs Expected
```
┌────────────────────────────────────────────────────────┐
│     Error Handling Comparison                          │
├────────────────────────────────────────────────────────┤
│              │ Exceptions │ Error Codes │ expected     │
├──────────────┼────────────┼─────────────┼──────────────┤
│ Ergonomics   │ Good       │ Poor        │ Good         │
│ Performance  │ Slow path  │ Fast        │ Fast         │
│ Composability│ Good       │ Poor        │ Excellent    │ 
│ Type safety  │ Partial    │ Weak        │ Strong       │
│ Zero overhead│ No         │ Yes         │ Almost       │
│                                                        │
│ Use exceptions for: Rare, exceptional conditions       │
│ Use error codes for: C interop, performance critical   │
│ Use expected for: Regular errors, functional style     │
└────────────────────────────────────────────────────────┘
```

---

## RAII and Exception Safety

### Resource Management
```cpp
#include <memory>
#include <fstream>
#include <mutex>

// BAD: Manual resource management
void bad_example() {
    int* data = new int[100];
    std::mutex mtx;
    mtx.lock();
    
    // If exception thrown here, leak!
    process_data(data);
    
    delete[] data;
    mtx.unlock();
}

// GOOD: RAII handles cleanup
void good_example() {
    auto data = std::make_unique<int[]>(100);
    std::lock_guard<std::mutex> lock(mtx);
    
    // If exception thrown, automatic cleanup
    process_data(data.get());
    
    // No manual cleanup needed
}

// Custom RAII wrapper
class FileHandle {
    FILE* file_;
public:
    explicit FileHandle(const char* name, const char* mode)
        : file_(fopen(name, mode)) {
        if (!file_) throw std::runtime_error("Failed to open");
    }
    
    ~FileHandle() { if (file_) fclose(file_); }
    
    // Delete copy, allow move
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept : file_(other.file_) {
        other.file_ = nullptr;
    }
    
    FILE* get() const { return file_; }
};
```

---

## Best Practices

### 1. Never Throw from Destructors
```cpp
class Bad {
public:
    ~Bad() {
        throw std::runtime_error("Bad!");  // NEVER — calls std::terminate during unwinding
    }
};

class Good {
public:
    ~Good() noexcept {
        try {
            cleanup();
        } catch (...) {
            // Log error, but don't let it escape
        }
    }
};
```

### 2. Catch by const Reference
```cpp
try {
    // ...
} catch (const std::exception& e) {  // GOOD: const ref
    std::cerr << e.what();
}

// BAD: Catches by value (slicing!)
catch (std::exception e) {  // BAD
    // Derived class info lost
}

// OK for rethrow
catch (...) {
    log_error();
    throw;  // Rethrow same exception
}
```

### 3. Order Exception Handlers
```cpp
try {
    // ...
}
catch (const std::out_of_range& e) {  // Most specific first
    // ...
}
catch (const std::logic_error& e) {   // Less specific
    // ...
}
catch (const std::exception& e) {      // Most general
    // ...
}
catch (...) {                          // Catch-all last
    // ...
}
```

### 4. Document Exception Specifications
```cpp
/**
 * @brief Divides two numbers
 * @throws std::invalid_argument if divisor is zero
 * @throws std::overflow_error if result overflows
 */
double divide(double a, double b);
```

### 5. Use RAII for All Resources
```cpp
// Every resource should have an owning RAII wrapper
std::unique_ptr<T>     // for pointers — see [Advanced Features](12_advanced_features.md)
std::lock_guard        // for mutexes — see [Multithreading](14_multithreading.md)
std::fstream           // for files — see [I/O and Filesystem](16_io_filesystem.md)
std::vector            // for arrays
```

---

## Performance Considerations

### Zero-Overhead Principle
```cpp
// If no exception thrown, exception handling has zero overhead
// Exception path is slow, but normal path is optimized

void hot_path() noexcept {
    // Critical performance code
    // noexcept allows better optimization
}

void cold_path() {
    // Can throw exceptions
    // Less performance critical
}
```

### Avoid Exceptions in Performance-Critical Code
```cpp
// HOT PATH: Use error codes or expected
std::error_code process_packet(Packet& p);

// COLD PATH: Exceptions OK
void load_config(const char* filename);  // May throw
```

---

## Common Patterns

### Function Try Blocks
```cpp
class Widget {
    Resource r_;
    
public:
    Widget()
    try : r_(initialize()) {
        // Constructor body
    }
    catch (const std::exception& e) {
        // Handle initialization exception
        // Automatically rethrows after handler
    }
};
```

### Exception Translator
```cpp
void c_api_function() {
    try {
        cpp_function();
    }
    catch (const std::exception& e) {
        // Translate C++ exception to C error code
        set_last_error(translate_exception(e));
    }
    catch (...) {
        set_last_error(UNKNOWN_ERROR);
    }
}
```

---

## Complete Practical Example: Robust Database Connection Manager

Here's a comprehensive example integrating exception handling, RAII, error codes, and all safety guarantees:

```cpp
#include <iostream>
#include <exception>
#include <stdexcept>
#include <string>
#include <memory>
#include <vector>
#include <optional>
#include <system_error>
#include <chrono>

// Custom exception hierarchy
class DatabaseException : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class ConnectionException : public DatabaseException {
public:
    explicit ConnectionException(const std::string& msg) 
        : DatabaseException("Connection error: " + msg) {}
};

class QueryException : public DatabaseException {
    int error_code_;
public:
    QueryException(const std::string& msg, int code)
        : DatabaseException("Query error: " + msg)
        , error_code_(code) {}
    
    int error_code() const { return error_code_; }
};

class TransactionException : public DatabaseException {
public:
    using DatabaseException::DatabaseException;
};

// Error code enum for non-exception path
enum class DBError {
    Success = 0,
    ConnectionFailed,
    QueryFailed,
    TransactionFailed,
    Timeout,
    PermissionDenied
};

// RAII Connection wrapper (no-throw guarantee for destructor)
class DBConnection {
private:
    bool connected_ = false;
    std::string connection_string_;
    
    // Simulate connection
    void connect() {
        std::cout << "  [Connecting to: " << connection_string_ << "]\n";
        
        // Simulate connection that might fail
        if (connection_string_.find("invalid") != std::string::npos) {
            throw ConnectionException("Invalid connection string");
        }
        
        connected_ = true;
        std::cout << "  [Connected successfully]\n";
    }
    
public:
    explicit DBConnection(const std::string& conn_str)
        : connection_string_(conn_str) {
        connect();
    }
    
    // Destructor must be noexcept
    ~DBConnection() noexcept {
        try {
            if (connected_) {
                std::cout << "  [Disconnecting...]\n";
                connected_ = false;
            }
        } catch (...) {
            // Never let exceptions escape destructor
            std::cerr << "  [ERROR: Exception in destructor caught]\n";
        }
    }
    
    // Delete copy (connection is unique resource)
    DBConnection(const DBConnection&) = delete;
    DBConnection& operator=(const DBConnection&) = delete;
    
    // Move operations should be noexcept
    DBConnection(DBConnection&& other) noexcept
        : connected_(other.connected_)
        , connection_string_(std::move(other.connection_string_)) {
        other.connected_ = false;
    }
    
    DBConnection& operator=(DBConnection&& other) noexcept {
        if (this != &other) {
            connected_ = other.connected_;
            connection_string_ = std::move(other.connection_string_);
            other.connected_ = false;
        }
        return *this;
    }
    
    bool is_connected() const noexcept { return connected_; }
    
    // Query execution with strong exception guarantee
    std::vector<std::string> execute_query(const std::string& query) {
        if (!connected_) {
            throw DatabaseException("Not connected");
        }
        
        std::cout << "  [Executing: " << query << "]\n";
        
        // Simulate query that might fail
        if (query.find("error") != std::string::npos) {
            throw QueryException("Syntax error in query", 1064);
        }
        
        // Return dummy results
        return {"row1", "row2", "row3"};
    }
};

// Transaction with RAII and strong exception guarantee
class Transaction {
private:
    DBConnection& conn_;
    bool committed_ = false;
    bool rolled_back_ = false;
    
public:
    explicit Transaction(DBConnection& conn) : conn_(conn) {
        std::cout << "  [BEGIN TRANSACTION]\n";
    }
    
    ~Transaction() noexcept {
        try {
            if (!committed_ && !rolled_back_) {
                std::cout << "  [AUTO-ROLLBACK (transaction not committed)]\n";
                rollback();
            }
        } catch (...) {
            // Swallow exception in destructor
        }
    }
    
    void commit() {
        if (rolled_back_) {
            throw TransactionException("Cannot commit rolled-back transaction");
        }
        
        std::cout << "  [COMMIT]\n";
        committed_ = true;
    }
    
    void rollback() noexcept {
        std::cout << "  [ROLLBACK]\n";
        rolled_back_ = true;
    }
};

// Connection pool with exception safety
class ConnectionPool {
private:
    std::vector<std::unique_ptr<DBConnection>> connections_;
    std::string connection_string_;
    size_t max_size_;
    
public:
    ConnectionPool(std::string conn_str, size_t max_size)
        : connection_string_(std::move(conn_str))
        , max_size_(max_size) {
        
        connections_.reserve(max_size_);  // Reserve for exception safety
    }
    
    // Strong exception guarantee: either adds connection or leaves state unchanged
    void add_connection() {
        if (connections_.size() >= max_size_) {
            throw std::length_error("Pool is full");
        }
        
        // Create new connection (might throw)
        auto conn = std::make_unique<DBConnection>(connection_string_);
        
        // If we get here, connection succeeded
        // Now commit the change (no-throw)
        connections_.push_back(std::move(conn));
        
        std::cout << "  [Pool size: " << connections_.size() << "]\n";
    }
    
    size_t size() const noexcept { return connections_.size(); }
};

// Error handling without exceptions (error codes)
class ErrorCodeDB {
public:
    struct Result {
        DBError error;
        std::vector<std::string> data;
        
        explicit operator bool() const { return error == DBError::Success; }
    };
    
    static Result query_no_throw(const std::string& query) noexcept {
        Result result{DBError::Success, { } };
        
        try {
            // Simulate query
            if (query.find("error") != std::string::npos) {
                result.error = DBError::QueryFailed;
                return result;
            }
            
            result.data = {"row1", "row2"};
        } catch (...) {
            result.error = DBError::QueryFailed;
        }
        
        return result;
    }
};

// Function-try block example
class DatabaseManager {
    DBConnection conn_;
    
public:
    // Constructor with function-try block
    DatabaseManager(const std::string& conn_str)
    try : conn_(conn_str) {
        std::cout << "  [DatabaseManager initialized]\n";
    }
    catch (const ConnectionException& e) {
        std::cerr << "  [Failed to initialize: " << e.what() << "]\n";
        throw;  // Automatically rethrows
    }
};

// Demonstration functions
void demo_basic_exception_handling() {
    std::cout << "\n=== Basic Exception Handling ===\n";
    
    try {
        DBConnection conn("localhost:5432");
        auto results = conn.execute_query("SELECT * FROM users");
        
        std::cout << "  Results: " << results.size() << " rows\n";
    }
    catch (const QueryException& e) {
        std::cerr << "  Query failed: " << e.what() 
                  << " (code: " << e.error_code() << ")\n";
    }
    catch (const ConnectionException& e) {
        std::cerr << "  Connection failed: " << e.what() << "\n";
    }
    catch (const DatabaseException& e) {
        std::cerr << "  Database error: " << e.what() << "\n";
    }
    catch (const std::exception& e) {
        std::cerr << "  Unexpected error: " << e.what() << "\n";
    }
}

void demo_raii_cleanup() {
    std::cout << "\n=== RAII Automatic Cleanup ===\n";
    
    try {
        DBConnection conn("localhost:5432");
        
        // Simulate error
        throw std::runtime_error("Something went wrong");
        
        // Connection automatically closed in destructor
    }
    catch (const std::exception& e) {
        std::cerr << "  Caught: " << e.what() << "\n";
    }
    // Connection destroyed and closed here
}

void demo_transaction_safety() {
    std::cout << "\n=== Transaction Safety ===\n";
    
    try {
        DBConnection conn("localhost:5432");
        Transaction txn(conn);
        
        conn.execute_query("INSERT INTO users VALUES (1, 'Alice')");
        conn.execute_query("INSERT INTO users VALUES (2, 'Bob')");
        
        // Simulate error before commit
        throw std::runtime_error("Error during transaction");
        
        txn.commit();  // Never reached
    }
    catch (const std::exception& e) {
        std::cerr << "  Transaction failed: " << e.what() << "\n";
        std::cerr << "  Transaction will auto-rollback\n";
    }
}

void demo_strong_guarantee() {
    std::cout << "\n=== Strong Exception Guarantee ===\n";
    
    try {
        ConnectionPool pool("localhost:5432", 3);
        
        pool.add_connection();  // OK
        pool.add_connection();  // OK
        
        std::cout << "  Pool has " << pool.size() << " connections\n";
        
        // Try to add invalid connection
        ConnectionPool bad_pool("invalid_server", 3);
        bad_pool.add_connection();  // Throws
        
        // This line never reached
        std::cout << "  This won't print\n";
    }
    catch (const std::exception& e) {
        std::cerr << "  Failed to add connection: " << e.what() << "\n";
        std::cerr << "  Pool state remains valid\n";
    }
}

void demo_error_codes() {
    std::cout << "\n=== Error Codes (No Exceptions) ===\n";
    
    auto result1 = ErrorCodeDB::query_no_throw("SELECT * FROM users");
    if (result1) {
        std::cout << "  Query succeeded: " << result1.data.size() << " rows\n";
    } else {
        std::cerr << "  Query failed with code: " 
                  << static_cast<int>(result1.error) << "\n";
    }
    
    auto result2 = ErrorCodeDB::query_no_throw("SELECT error FROM users");
    if (result2) {
        std::cout << "  Query succeeded\n";
    } else {
        std::cerr << "  Query failed with code: " 
                  << static_cast<int>(result2.error) << "\n";
    }
}

void demo_exception_hierarchy() {
    std::cout << "\n=== Exception Hierarchy ===\n";
    
    auto handle_db_operation = []() {
        throw QueryException("Table not found", 1146);
    };
    
    try {
        handle_db_operation();
    }
    catch (const QueryException& e) {
        std::cerr << "  Caught QueryException: " << e.what() 
                  << " (code: " << e.error_code() << ")\n";
    }
    catch (const DatabaseException& e) {
        // Would catch if QueryException wasn't caught above
        std::cerr << "  Caught DatabaseException: " << e.what() << "\n";
    }
    catch (const std::runtime_error& e) {
        // Would catch if DatabaseException wasn't caught above
        std::cerr << "  Caught runtime_error: " << e.what() << "\n";
    }
}

void demo_nested_exceptions() {
    std::cout << "\n=== Nested Exceptions ===\n";
    
    try {
        try {
            throw std::runtime_error("Original error");
        }
        catch (...) {
            std::throw_with_nested(DatabaseException("Wrapped error"));
        }
    }
    catch (const DatabaseException& e) {
        std::cerr << "  Outer exception: " << e.what() << "\n";
        
        try {
            std::rethrow_if_nested(e);
        }
        catch (const std::exception& nested) {
            std::cerr << "  Nested exception: " << nested.what() << "\n";
        }
    }
}

int main() {
    std::cout << "=== Exception Handling Demo ===\n";
    
    // 1. Basic exception handling
    demo_basic_exception_handling();
    
    // 2. RAII automatic cleanup
    demo_raii_cleanup();
    
    // 3. Transaction safety
    demo_transaction_safety();
    
    // 4. Strong exception guarantee
    demo_strong_guarantee();
    
    // 5. Error codes (no exceptions)
    demo_error_codes();
    
    // 6. Exception hierarchy
    demo_exception_hierarchy();
    
    // 7. Nested exceptions
    demo_nested_exceptions();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Output (Sample):
```
=== Exception Handling Demo ===

=== Basic Exception Handling ===
  [Connecting to: localhost:5432]
  [Connected successfully]
  [Executing: SELECT * FROM users]
  Results: 3 rows
  [Disconnecting...]

=== RAII Automatic Cleanup ===
  [Connecting to: localhost:5432]
  [Connected successfully]
  Caught: Something went wrong
  [Disconnecting...]

=== Transaction Safety ===
  [Connecting to: localhost:5432]
  [Connected successfully]
  [BEGIN TRANSACTION]
  [Executing: INSERT INTO users VALUES (1, 'Alice')]
  [Executing: INSERT INTO users VALUES (2, 'Bob')]
  Transaction failed: Error during transaction
  Transaction will auto-rollback
  [AUTO-ROLLBACK (transaction not committed)]
  [ROLLBACK]
  [Disconnecting...]

=== Strong Exception Guarantee ===
  [Connecting to: localhost:5432]
  [Connected successfully]
  [Pool size: 1]
  [Connecting to: localhost:5432]
  [Connected successfully]
  [Pool size: 2]
  Pool has 2 connections
  [Connecting to: invalid_server]
  Failed to add connection: Connection error: Invalid connection string
  Pool state remains valid
  [Disconnecting...]
  [Disconnecting...]
```

### Concepts Demonstrated:
- **Custom exception hierarchy**: Organizing exceptions by type
- **RAII**: Automatic resource cleanup
- **noexcept destructors**: Never throw from destructors
- **Strong exception guarantee**: Commit-or-rollback semantics
- **Transaction safety**: Automatic rollback
- **Function-try blocks**: Constructor exception handling
- **Error codes**: Non-exception error handling
- **Nested exceptions**: Exception chaining
- **Exception hierarchy**: Catching by specificity
- **Move semantics**: noexcept move operations
- **std::unique_ptr**: Exception-safe ownership
- **Reserve for safety**: Preventing reallocations

This example shows robust, production-ready exception handling!

---

## Related Topics

- [Advanced Features](12_advanced_features.md) — RAII, smart pointers, move semantics, and the Rule of Five that underpin exception safety
- [Multithreading](14_multithreading.md) — `std::lock_guard` and why locks must survive stack unwinding
- [I/O and Filesystem](16_io_filesystem.md) — `std::filesystem::filesystem_error` and non-throwing `error_code` overloads
- [Async and Futures](15_async_futures.md) — exception propagation through `std::future::get()`; `std::future_error`
- [Best Practices](13_best_practices.md) — when to throw vs return errors; API design guidance
- [Modern C++20/23 Features](07_modern_features.md) — `std::expected` and other C++23 error-handling additions
- [Quick Reference](99_quick_reference.md) — exception hierarchy and safety guarantees at a glance

## Next Steps
- **Next**: [Time and Chrono →](18_time_chrono.md)
- **Previous**: [← I/O and Filesystem](16_io_filesystem.md)

---
*Chapter 17 — Exception Handling and Error Management*

