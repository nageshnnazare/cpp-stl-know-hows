# C++ Best Practices and Idioms

## Overview

This chapter consolidates best practices, common idioms, and guidelines for writing modern, efficient, and maintainable C++. It builds on [Advanced Features](12_advanced_features.md) (RAII, move semantics) and [Modern Features](07_modern_features.md) (`auto`, ranges).

```
┌──────────────────────────────────────────────────────────┐
│                  BEST PRACTICES AREAS                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  CODE QUALITY      │  PERFORMANCE     │  SAFETY          │
│  ─────────────     │  ────────────    │  ──────          │
│  • Readability     │  • Optimization  │  • Type safety   │
│  • Maintainability │  • Profiling     │  • Bounds checks │
│  • Documentation   │  • Benchmarking  │  • Exception safe│
│                                                          │
│  MODERN C++        │  DESIGN          │  TESTING         │
│  ──────────        │  ──────          │  ───────         │
│  • auto, range-for │  • SOLID         │  • Unit tests    │
│  • Smart pointers  │  • Patterns      │  • TDD           │
│  • Algorithms      │  • Modularity    │  • Coverage      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Core Guidelines Highlights

### Use RAII for Resource Management

See [Advanced Features](12_advanced_features.md) for the full RAII treatment. Summary: acquire in constructor, release in destructor.

### Prefer `auto` for Type Deduction
```cpp
// BAD: Redundant type information
std::vector<int>::iterator it = vec.begin();
std::map<std::string, std::vector<int>>::const_iterator mit = map.cbegin();

// GOOD: Let compiler deduce types
auto it = vec.begin();
auto mit = map.cbegin();

// EXCEPTION: When explicit type aids readability
int count = calculate();  // Clear that it's an int
auto value = calculate(); // What type is this?
```

### Use Range-based for Loops
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// BAD: Manual iteration
for (size_t i = 0; i < vec.size(); ++i) {
    process(vec[i]);
}

// GOOD: Range-based for
for (int value : vec) {
    process(value);
}

// BEST: const ref to avoid copies
for (const auto& value : vec) {
    process(value);
}
```

### Prefer `emplace` over `insert`/`push_back`
```cpp
std::vector<std::string> vec;

// BAD: Creates temporary then copies/moves
vec.push_back(std::string("Hello"));

// GOOD: Constructs in-place
vec.emplace_back("Hello");

// For complex types:
struct Point { int x, y; Point(int x, int y) : x(x), y(y) {} };
std::vector<Point> points;

points.push_back(Point(1, 2));  // BAD: Creates temporary
points.emplace_back(1, 2);      // GOOD: Constructs in-place
```

---

## Performance Best Practices

### Pass by const Reference
```cpp
// BAD: Copies for large objects
void process(std::string s, std::vector<int> v) {
    // ...
}

// GOOD: Pass by const reference
void process(const std::string& s, const std::vector<int>& v) {
    // ...
}

// For small types, pass by value is OK
void process(int x, double y) {  // OK: cheap to copy
    // ...
}
```

### Return by Value (Enable RVO)
```cpp
// BAD: Output parameter
void make_vector(std::vector<int>& out) {
    out = {1, 2, 3, 4, 5};
}

// GOOD: Return by value (RVO)
std::vector<int> make_vector() {
    return {1, 2, 3, 4, 5};  // No copy with RVO
}
```

### Reserve Capacity for Containers
```cpp
std::vector<int> vec;

// BAD: Multiple reallocations
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // May reallocate many times
}

// GOOD: Reserve first
std::vector<int> vec;
vec.reserve(10000);
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // No reallocations
}
```

### Use Appropriate Containers

See [Sequence Containers](01_sequence_containers.md), [Associative Containers](02_associative_containers.md), [Unordered Containers](03_unordered_containers.md), and [Container Adaptors](04_container_adaptors.md) for detailed trade-offs.

### Prefer Algorithms over Loops

See [Algorithms](06_algorithms.md) and [Iterators](05_iterators.md). With C++20, prefer [ranges](07_modern_features.md) when they simplify the code.

---

## Modern C++ Idioms

### Almost Always Auto (AAA)
```cpp
// Use auto to avoid repetition and errors
auto value = calculate_something();
auto it = container.find(key);
auto result = expensive_function();

// Especially with lambdas
auto lambda = [](int x) { return x * 2; };

// For iterators
for (auto it = vec.begin(); it != vec.end(); ++it) {
    // ...
}
```

### Immediately Invoked Lambda Expression (IILE)
```cpp
// Initialize const variables with complex logic
const auto value = [&]() {
    if (condition1) return 42;
    if (condition2) return 100;
    return 0;
}();

// One-time initialization
const auto config = []() {
    Config c;
    c.load("config.txt");
    c.validate();
    return c;
}();
```

### Copy-and-Swap Idiom
```cpp
class Resource {
    int* data_;
    
public:
    // Copy constructor
    Resource(const Resource& other) 
        : data_(new int(*other.data_)) {}
    
    // Copy assignment using copy-and-swap
    Resource& operator=(Resource other) {  // Pass by value
        swap(other);  // Swap with temporary
        return *this;
    }  // Temporary destroyed, cleaning up old data
    
    void swap(Resource& other) noexcept {
        std::swap(data_, other.data_);
    }
};
```

### RAII Wrapper Pattern

Same pattern as [RAII in Advanced Features](12_advanced_features.md) — wrap C APIs (`FILE*`, sockets, handles) in a move-only class with a custom deleter or `unique_ptr` + function pointer deleter.

### Pimpl (Pointer to Implementation)
```cpp
// Header: widget.h
class Widget {
public:
    Widget();
    ~Widget();
    void do_something();
    
private:
    class Impl;  // Forward declaration
    std::unique_ptr<Impl> pimpl_;
};

// Implementation: widget.cpp
class Widget::Impl {
public:
    void do_something() { /* implementation */ }
private:
    // Private members hidden from header
    int data_;
    std::vector<int> more_data_;
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // Must be in cpp (Impl is complete here)
void Widget::do_something() { pimpl_->do_something(); }
```

---

## Safety Best Practices

### Avoid Raw Pointers for Ownership

Use `std::unique_ptr` / `std::shared_ptr` with `make_unique` / `make_shared`. Raw pointers are fine as **non-owning** observers when lifetime is guaranteed. See [Advanced Features](12_advanced_features.md).

### Use `const` Liberally
```cpp
// Mark what doesn't change
void process(const std::string& input) {  // Won't modify input
    // ...
}

// Const member functions
class Calculator {
    int value_;
public:
    int get_value() const { return value_; }  // Won't modify object
    void set_value(int v) { value_ = v; }     // Can modify
};

// Const correctness enables optimization
```

### Initialize Variables
```cpp
// BAD: Uninitialized
int x;
std::string s;  // OK: default constructed
int* ptr;       // BAD: dangling

// GOOD: Initialize
int x = 0;
int y{42};  // Uniform initialization
std::string s = "Hello";
int* ptr = nullptr;

// C++17: Structured bindings
if (auto [iter, inserted] = map.insert({key, value}); inserted) {
    // Use iter
}
```

### Check Return Values
```cpp
// BAD: Ignore errors
file.open("data.txt");
file.write(data);

// GOOD: Check for errors (std::fstream::open returns void — test the stream)
file.open("data.txt");
if (!file) {
    std::cerr << "Failed to open file\n";
    return;
}

// BETTER: Use exceptions for errors
try {
    File file("data.txt");
    file.write(data);
} catch (const std::exception& e) {
    std::cerr << "Error: " << e.what() << '\n';
}
```

---

## Code Organization

### Header-Only Libraries
```cpp
// my_utility.hpp
#ifndef MY_UTILITY_HPP
#define MY_UTILITY_HPP

namespace my_utility {

// Inline for header-only
inline int add(int a, int b) {
    return a + b;
}

// Template (naturally header-only)
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// C++17: Inline variables
inline constexpr int version = 1;

}  // namespace my_utility

#endif
```

### Namespaces
```cpp
// Avoid `using namespace std` in headers
// BAD (in header):
using namespace std;

// GOOD (in cpp):
namespace {  // Anonymous namespace for file-local
    constexpr int BUFFER_SIZE = 1024;
    void helper_function() { /* ... */ }
}

namespace mylib {
    void public_function() { /* ... */ }
}

// Avoid deeply nested namespaces
namespace mycompany::myproject::mymodule {  // C++17
    // ...
}
```

### Forward Declarations
```cpp
// header.hpp
class BigClass;  // Forward declare

class MyClass {
    std::unique_ptr<BigClass> member_;  // OK: pointer to incomplete type
public:
    void process(const BigClass& obj);  // OK: reference
};

// cpp file includes full definition
#include "big_class.hpp"
```

---

## Error Handling

### Prefer Exceptions for Errors

See [Exceptions](17_exceptions.md) for guarantees, `noexcept`, and when error codes are appropriate.

### Exception Safety Guarantees
```
┌────────────────────────────────────────────────────────┐
│            Exception Safety Levels                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│ 1. No-throw (noexcept)                                 │
│    - Never throws exceptions                           │
│    - Destructors, move operations, swap                │
│                                                        │
│ 2. Strong guarantee                                    │
│    - Either succeeds or leaves unchanged               │
│    - Commit-or-rollback semantics                      │
│                                                        │
│ 3. Basic guarantee                                     │
│    - No resources leaked                               │
│    - Object remains in valid state                     │
│                                                        │
│ 4. No guarantee                                        │
│    - May leak resources or leave inconsistent          │
│    - Avoid this!                                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

```cpp
// Strong exception guarantee using copy-and-swap
void Widget::set_data(const Data& new_data) {
    Data temp = new_data;  // May throw
    data_.swap(temp);      // noexcept
    // If exception thrown, Widget unchanged
}

// Mark noexcept when appropriate
void important_operation() noexcept {
    // Must not throw
}
```

---

## Testing Best Practices

### Unit Testing Example
```cpp
// Using a testing framework (e.g., Catch2, Google Test)
#include <catch2/catch.hpp>

TEST_CASE("Vector operations", "[vector]") {
    std::vector<int> vec;
    
    SECTION("push_back adds elements") {
        vec.push_back(1);
        vec.push_back(2);
        
        REQUIRE(vec.size() == 2);
        REQUIRE(vec[0] == 1);
        REQUIRE(vec[1] == 2);
    }
    
    SECTION("clear empties vector") {
        vec.push_back(1);
        vec.clear();
        
        REQUIRE(vec.empty());
    }
}
```

### Assertions and Contracts
```cpp
#include <cassert>

void process(int* data, size_t size) {
    assert(data != nullptr);  // Precondition
    assert(size > 0);         // Precondition
    
    // ... processing ...
    
    assert(is_valid_result());  // Postcondition
}

// C++20: Contracts (planned, not yet standardized)
// void process(int* data, size_t size)
//     [[expects: data != nullptr]]
//     [[expects: size > 0]]
//     [[ensures: result >= 0]]
// { ... }
```

---

## Documentation

### Comments Best Practices
```cpp
// BAD: States the obvious
int count = 0;  // Set count to 0

// GOOD: Explains why
int count = 0;  // Track number of retries for exponential backoff

// BAD: Outdated comment
int limit = 1000;  // Maximum of 100 connections

// GOOD: Self-documenting code
constexpr int MAX_CONNECTIONS = 1000;

// Document non-obvious behavior
/// Returns the index of the element, or -1 if not found.
/// Time complexity: O(log n) for sorted containers.
/// Thread-safe: No
int find_index(const std::vector<int>& vec, int target);
```

### Doxygen-style Documentation
```cpp
/**
 * @brief Calculates the distance between two points.
 * 
 * @param p1 The first point
 * @param p2 The second point
 * @return The Euclidean distance between p1 and p2
 * 
 * @note This function uses floating-point arithmetic.
 * @warning Large coordinate values may cause overflow.
 */
double distance(const Point& p1, const Point& p2);
```

---

## Compilation and Build

### Compiler Warnings
```bash
# Enable all warnings and treat as errors
g++ -Wall -Wextra -Werror -pedantic -std=c++20 main.cpp

# Additional useful warnings
g++ -Wshadow -Wconversion -Wsign-conversion -Wnon-virtual-dtor
```

### Optimization Flags
```bash
# Debug build
g++ -g -O0 -DDEBUG main.cpp

# Release build
g++ -O3 -DNDEBUG -march=native main.cpp

# Profiling build
g++ -pg -O2 main.cpp
```

---

## Common Anti-Patterns to Avoid

### 1. God Class
```cpp
// BAD: Class does everything
class System {
    void process_input();
    void render_graphics();
    void handle_network();
    void manage_audio();
    void save_file();
    // ... 100 more methods
};

// GOOD: Separate responsibilities
class InputHandler { /*...*/ };
class Renderer { /*...*/ };
class NetworkManager { /*...*/ };
```

### 2. Premature Optimization
```cpp
// BAD: Optimizing before profiling
void process(const std::vector<int>& data) {
    // Complex bit-twiddling optimization
    // that's hard to read and maintain
}

// GOOD: Write clear code first
void process(const std::vector<int>& data) {
    // Clear, simple implementation
    // Optimize later if profiling shows bottleneck
}
```

### 3. Naked `new` and `delete`

Never use raw `new`/`delete` for ownership — use smart pointers or containers. Mark move operations `noexcept` so containers like `std::vector` use move during reallocation instead of copy.

### 4. Ignoring const-correctness
```cpp
// BAD: Non-const getters
class Point {
    int x_, y_;
public:
    int x() { return x_; }  // Can't be called on const Point
    int y() { return y_; }
};

// GOOD: Const-correct getters
class Point {
    int x_, y_;
public:
    int x() const { return x_; }
    int y() const { return y_; }
};
```

---

## Performance Measurement

### Basic Benchmarking

See [Time & Chrono](18_time_chrono.md) for precise timing utilities.

---

## Quick Reference Checklist

### Before Committing Code
```
✓ Code compiles with -Wall -Wextra -Werror
✓ No memory leaks (run valgrind)
✓ All tests pass
✓ Code is formatted consistently
✓ Complex logic is commented
✓ Public API is documented
✓ No compiler warnings
✓ Performance is acceptable (if relevant)
✓ Exception safety considered
✓ const-correctness verified
```

### Code Review Checklist
```
✓ Correct algorithm/data structure choice
✓ Resource management (RAII)
✓ Error handling appropriate
✓ Thread safety (if concurrent)
✓ No unnecessary copies
✓ Const-correctness
✓ Clear variable/function names
✓ Appropriate use of modern C++ features
✓ Tests cover edge cases
✓ Documentation clear and accurate
```

---

## Recommended Tools

### Static Analysis
- **Clang-Tidy**: Comprehensive linter
- **Cppcheck**: Static analysis
- **Clang Static Analyzer**: Deep analysis

### Dynamic Analysis
- **Valgrind**: Memory leak detection
- **AddressSanitizer**: Memory errors
- **ThreadSanitizer**: Data races
- **UndefinedBehaviorSanitizer**: UB detection

### Profiling
- **gprof**: Call graph profiler
- **Perf**: Linux performance analyzer
- **Valgrind/Callgrind**: Call profiling

### Build Systems
- **CMake**: Cross-platform build generator
- **Meson**: Fast build system
- **Bazel**: Google's build system

---

## Further Learning

### Essential Reading
- C++ Core Guidelines (Bjarne Stroustrup, Herb Sutter)
- Effective Modern C++ (Scott Meyers)
- C++ Concurrency in Action (Anthony Williams)
- The C++ Programming Language (Bjarne Stroustrup)

### Online Resources
- cppreference.com (comprehensive reference)
- CppCon talks (YouTube)
- C++ Weekly (Jason Turner)
- Compiler Explorer (godbolt.org)

---

## Conclusion

Modern C++ is a powerful language that rewards careful design and attention to best practices. Key takeaways:

1. **Use RAII** for all resource management
2. **Prefer value semantics** and smart pointers
3. **Embrace modern C++** features (auto, range-for, lambdas)
4. **Write clear code first**, optimize later
5. **Use STL algorithms** instead of raw loops
6. **Test thoroughly** and use static analysis
7. **Document your code** for maintainability
8. **Follow the Core Guidelines**

Remember: Correct, clear, and maintainable code is more valuable than clever code!

---

## Complete Practical Example: Modern C++ Application

Here's a comprehensive example applying all best practices - a complete, production-quality application:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>
#include <optional>
#include <variant>
#include <memory>
#include <algorithm>
#include <ranges>
#include <chrono>
#include <stdexcept>
#include <cassert>

// 1. Strong types for type safety
class UserId {
    int value_;
public:
    explicit UserId(int value) : value_(value) {
        if (value < 0) {
            throw std::invalid_argument("UserId must be non-negative");
        }
    }
    int value() const noexcept { return value_; }
    auto operator<=>(const UserId&) const = default;
};

class Email {
    std::string value_;
public:
    explicit Email(std::string email) : value_(std::move(email)) {
        if (!is_valid(value_)) {
            throw std::invalid_argument("Invalid email format");
        }
    }
    const std::string& value() const noexcept { return value_; }
    
private:
    static bool is_valid(std::string_view email) {
        return email.find('@') != std::string_view::npos;
    }
};

// 2. Well-designed entity class (Rule of Zero)
class User {
private:
    UserId id_;
    std::string name_;
    Email email_;
    std::chrono::system_clock::time_point created_at_;
    
public:
    User(UserId id, std::string name, Email email)
        : id_(id)
        , name_(std::move(name))
        , email_(std::move(email))
        , created_at_(std::chrono::system_clock::now()) {
        
        if (name_.empty()) {
            throw std::invalid_argument("Name cannot be empty");
        }
    }
    
    // Getters (const-correct, noexcept where possible)
    UserId id() const noexcept { return id_; }
    const std::string& name() const noexcept { return name_; }
    const Email& email() const noexcept { return email_; }
    auto created_at() const noexcept { return created_at_; }
    
    // Business logic
    bool is_recently_created(std::chrono::hours threshold = std::chrono::hours{24}) const {
        auto now = std::chrono::system_clock::now();
        return now - created_at_ < threshold;
    }
};

// 3. Error handling with optional and variant
using DatabaseError = std::string;
template<typename T>
using Result = std::variant<T, DatabaseError>;

// 4. Repository pattern with clear ownership
class UserRepository {
private:
    std::vector<std::unique_ptr<User>> users_;
    
public:
    // Add user (transfer ownership)
    UserId add_user(std::unique_ptr<User> user) {
        if (!user) {
            throw std::invalid_argument("Cannot add null user");
        }
        
        auto id = user->id();
        users_.push_back(std::move(user));
        return id;
    }
    
    // Find user (return non-owning pointer)
    const User* find_by_id(UserId id) const noexcept {
        auto it = std::ranges::find_if(users_, [id](const auto& user) {
            return user->id() == id;
        });
        
        return it != users_.end() ? it->get() : nullptr;
    }
    
    // Find users by predicate (return view, not copies)
    auto find_if(auto predicate) const {
        return users_ 
            | std::views::transform([](const auto& ptr) -> const User& { 
                return *ptr; 
              })
            | std::views::filter(predicate);
    }
    
    // Get all recently created users
    auto get_recent_users() const {
        return find_if([](const User& user) {
            return user.is_recently_created();
        });
    }
    
    size_t size() const noexcept { return users_.size(); }
    
    // Clear users
    void clear() noexcept {
        users_.clear();
    }
};

// 5. Service layer with dependency injection
class UserService {
private:
    UserRepository& repository_;
    
public:
    explicit UserService(UserRepository& repository) 
        : repository_(repository) {}
    
    // Create user with validation
    Result<UserId> create_user(int id, std::string name, std::string email) noexcept {
        try {
            auto user = std::make_unique<User>(
                UserId(id),
                std::move(name),
                Email(std::move(email))
            );
            
            auto user_id = repository_.add_user(std::move(user));
            return user_id;
            
        } catch (const std::exception& e) {
            return DatabaseError{e.what()};
        }
    }
    
    // Get user details
    std::optional<std::string> get_user_info(UserId id) const noexcept {
        if (const User* user = repository_.find_by_id(id)) {
            return std::string("User: ") + user->name() + 
                   " <" + user->email().value() + ">";
        }
        return std::nullopt;
    }
    
    // Get recent user count
    size_t get_recent_user_count() const noexcept {
        return std::ranges::distance(repository_.get_recent_users());
    }
    
    // List all users (string_view parameter, no copying)
    void list_users(std::string_view prefix = "") const {
        // The index isn't needed here, so iterate the elements directly.
        // (std::views::enumerate yields (index, element) pairs and is C++23,
        // so a binding would have to be `[index, user]`, not `[user]`.)
        for (const auto& user :
             repository_.find_if([](const User&) { return true; })) {
            std::cout << prefix << "User " << user.name()
                      << " (" << user.email().value() << ")\n";
        }
    }
};

// 6. RAII timer for performance measurement
class ScopedTimer {
private:
    std::string_view name_;
    std::chrono::high_resolution_clock::time_point start_;
    
public:
    explicit ScopedTimer(std::string_view name)
        : name_(name)
        , start_(std::chrono::high_resolution_clock::now()) {
        std::cout << "[" << name_ << "] Started\n";
    }
    
    ~ScopedTimer() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
            end - start_
        );
        std::cout << "[" << name_ << "] Completed in " 
                  << duration.count() << " μs\n";
    }
    
    // Non-copyable, non-movable
    ScopedTimer(const ScopedTimer&) = delete;
    ScopedTimer& operator=(const ScopedTimer&) = delete;
};

// 7. Configuration with modern C++ features
struct AppConfig {
    static constexpr int DEFAULT_MAX_USERS = 1000;
    static constexpr std::string_view APP_NAME = "UserManager";
    
    int max_users = DEFAULT_MAX_USERS;
    bool debug_mode = false;
    
    // Validation
    [[nodiscard]] bool is_valid() const noexcept {
        return max_users > 0 && max_users <= 100000;
    }
};

// 8. Application class using all best practices
class Application {
private:
    AppConfig config_;
    UserRepository repository_;
    UserService service_;
    
public:
    explicit Application(AppConfig config = {})
        : config_(std::move(config))
        , repository_()
        , service_(repository_) {
        
        if (!config_.is_valid()) {
            throw std::invalid_argument("Invalid configuration");
        }
        
        std::cout << "=== " << config_.APP_NAME << " Started ===\n";
        if (config_.debug_mode) {
            std::cout << "DEBUG MODE ENABLED\n";
        }
    }
    
    void run() {
        ScopedTimer timer("Application Run");
        
        // Create sample users
        create_sample_users();
        
        // Demonstrate queries
        demonstrate_queries();
        
        // Show statistics
        show_statistics();
    }
    
private:
    void create_sample_users() {
        ScopedTimer timer("Create Users");
        
        struct UserData { int id; const char* name; const char* email; };
        constexpr UserData users[] = {
            {1, "Alice Johnson", "alice@example.com"},
            {2, "Bob Smith", "bob@example.com"},
            {3, "Charlie Brown", "charlie@example.com"},
            {4, "Diana Prince", "diana@example.com"},
            {5, "Eve Wilson", "eve@example.com"}
        };
        
        for (const auto& [id, name, email] : users) {
            auto result = service_.create_user(id, name, email);
            
            std::visit([&](auto&& value) {
                using T = std::decay_t<decltype(value)>;
                if constexpr (std::is_same_v<T, UserId>) {
                    if (config_.debug_mode) {
                        std::cout << "  Created user: " << name << "\n";
                    }
                } else {
                    std::cerr << "  Failed: " << value << "\n";
                }
            }, result);
        }
        
        std::cout << "Created " << repository_.size() << " users\n";
    }
    
    void demonstrate_queries() {
        std::cout << "\n--- User Queries ---\n";
        
        // Find specific user
        if (auto info = service_.get_user_info(UserId{2})) {
            std::cout << *info << "\n";
        }
        
        // Find all users (using ranges)
        std::cout << "\nAll users:\n";
        service_.list_users("  ");
    }
    
    void show_statistics() {
        std::cout << "\n--- Statistics ---\n";
        std::cout << "Total users: " << repository_.size() << "\n";
        std::cout << "Recent users: " << service_.get_recent_user_count() << "\n";
        std::cout << "Max users: " << config_.max_users << "\n";
    }
};

// 9. Main with proper error handling
int main() {
    try {
        AppConfig config{
            .max_users = 100,
            .debug_mode = true
        };
        
        Application app(config);
        app.run();
        
        std::cout << "\n=== Application Completed Successfully ===\n";
        return 0;
        
    } catch (const std::exception& e) {
        std::cerr << "FATAL ERROR: " << e.what() << "\n";
        return 1;
    } catch (...) {
        std::cerr << "FATAL ERROR: Unknown exception\n";
        return 2;
    }
}
```

### Best Practices Demonstrated:

**Design & Architecture:**
- ✅ Strong types (UserId, Email)
- ✅ Dependency injection
- ✅ Separation of concerns
- ✅ Clear ownership semantics
- ✅ Repository pattern

**Modern C++ Features:**
- ✅ Rule of Zero
- ✅ Smart pointers (unique_ptr)
- ✅ Ranges and views
- ✅ Structured bindings
- ✅ std::variant for errors
- ✅ std::optional for optional values
- ✅ constexpr for constants
- ✅ Designated initializers
- ✅ Spaceship operator (<=>)
- ✅ [[nodiscard]] attribute

**Resource Management:**
- ✅ RAII everywhere
- ✅ No manual memory management
- ✅ Clear ownership transfer
- ✅ No raw pointers in interfaces

**Error Handling:**
- ✅ Exceptions for construction errors
- ✅ noexcept for non-throwing functions
- ✅ Result type for recoverable errors
- ✅ Optional for missing values

**Code Quality:**
- ✅ Const-correctness
- ✅ Input validation
- ✅ Clear naming
- ✅ Single responsibility
- ✅ Comprehensive error handling
- ✅ Performance measurement

**Performance:**
- ✅ Move semantics
- ✅ string_view for parameters
- ✅ Lazy evaluation with views
- ✅ No unnecessary copies
- ✅ Reserve when needed

This is a production-quality C++ application template!

---

## Related Topics

- [Advanced Features](12_advanced_features.md) — Rule of Zero, move semantics, smart pointers, noexcept moves
- [Modern Features](07_modern_features.md) — `auto`, ranges, `optional`/`variant`, designated initializers
- [Exceptions](17_exceptions.md) — exception safety guarantees, `noexcept`, error-handling strategy
- [Algorithms](06_algorithms.md) — prefer STL algorithms over hand-written loops
- [Lambdas](10_lambdas.md) — idiomatic callbacks; avoid `std::function` on hot paths
- [I/O & Filesystem](16_io_filesystem.md) — RAII file handles, path best practices
- [Multithreading](14_multithreading.md) — thread safety, lock guards, data-race avoidance

---

## Next Steps
- **Next**: [Multithreading and Concurrency →](14_multithreading.md)
- **Previous**: [← Advanced Features](12_advanced_features.md)
- **Main**: [Main README](README.md)

---
*Chapter 13 — Best Practices and Idioms*

