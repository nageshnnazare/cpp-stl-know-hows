# Advanced C++ Features

## Overview

Modern C++ introduces powerful features for resource management, performance optimization, and safer code. This chapter covers move semantics, [smart pointers](https://en.cppreference.com/w/cpp/memory), RAII, perfect forwarding, and value categories — foundations also used in [lambdas](10_lambdas.md) and [metaprogramming](11_metaprogramming.md).

```
┌────────────────────────────────────────────────────────────┐
│              ADVANCED C++ FEATURES                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  RESOURCE MGMT    │  PERFORMANCE      │  TYPE SAFETY       │
│  ──────────────   │  ────────────     │  ──────────        │
│  • RAII           │  • Move semantics │  • Strong types    │
│  • Smart pointers │  • RVO/NRVO       │  • std::variant    │
│  • Destructors    │  • Perfect fwd    │  • std::optional   │
│                   │  • Forwarding refs│                    │
│                                                            │
│  MODERN IDIOMS    │  CONCURRENCY      │  ATTRIBUTES        │
│  ──────────────   │  ────────────     │  ──────────        │
│  • Rule of 5/0    │  • std::thread    │  • [[nodiscard]]   │
│  • Copy elision   │  • std::atomic    │  • [[likely]]      │
│  • Value semantics│  • Mutexes        │  • [[maybe_unused]]│
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## RAII (Resource Acquisition Is Initialization)

### The RAII Principle

```
┌────────────────────────────────────────────────────────┐
│                  RAII Pattern                          │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. Acquire resource in constructor                    │
│  2. Release resource in destructor                     │
│  3. Resource lifetime = Object lifetime                │
│                                                        │
│  Benefits:                                             │
│  • Exception-safe                                      │
│  • No resource leaks                                   │
│  • Automatic cleanup                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Basic RAII Example
```cpp
#include <fstream>

// BAD: Manual resource management
void process_file_bad() {
    FILE* file = fopen("data.txt", "r");
    if (!file) return;
    
    // ... process file ...
    // If exception thrown here, file is leaked!
    
    fclose(file);  // Easy to forget!
}

// GOOD: RAII with std::fstream
void process_file_good() {
    std::ifstream file("data.txt");
    if (!file) return;
    
    // ... process file ...
    // File automatically closed when leaving scope
    // Even if exception is thrown!
}  // Destructor closes file
```

### Custom RAII Class
```cpp
class FileHandle {
    FILE* file_;
    
public:
    explicit FileHandle(const char* filename)
        : file_(fopen(filename, "r")) {
        if (!file_) {
            throw std::runtime_error("Failed to open file");
        }
    }
    
    ~FileHandle() {
        if (file_) {
            fclose(file_);
        }
    }
    
    // Delete copy (file handle shouldn't be copied)
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    
    // Allow move
    FileHandle(FileHandle&& other) noexcept
        : file_(other.file_) {
        other.file_ = nullptr;
    }
    
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (file_) fclose(file_);
            file_ = other.file_;
            other.file_ = nullptr;
        }
        return *this;
    }
    
    FILE* get() const { return file_; }
};
```

### RAII for Locks

See [Multithreading](14_multithreading.md) for full coverage. `std::lock_guard` and `std::unique_lock` are RAII wrappers around mutexes.

---

## Move Semantics

### Value Categories (C++11/17)

C++ classifies every expression into one of three **value categories**:

```
┌────────────────────────────────────────────────────────┐
│                Value Categories                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│                    expression                          │
│                        │                               │
│          ┌─────────────┴─────────────┐                 │
│          ▼                           ▼                 │
│      glvalue                      rvalue               │
│          │                           │                 │
│     ┌────┴────┐               ┌──────┴──────┐          │
│     ▼         ▼               ▼             ▼          │
│  lvalue   xvalue          prvalue                      │
│                                                        │
│  lvalue:  has identity, not expiring (named variable)  │
│  xvalue:  has identity, expiring (result of std::move) │
│  prvalue: no identity, expiring (literal, temporary)   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

- **Lvalue** — has a name and address; `int&`, `T&`
- **Xvalue** (expiring value) — can be moved from; `std::move(x)` produces an xvalue
- **Prvalue** (pure rvalue) — temporaries, literals; `42`, `std::string("hi")`

`std::move` does **not** move anything — it is only an `static_cast` to an rvalue reference, enabling overload resolution to pick move constructors/assignment. The actual transfer happens in the move operation.

⚠️ **Gotchas**
- Moved-from objects remain **valid but unspecified** — you may assign to them or destroy them, but don't assume their value.
- Never `std::move` a `return` of a local variable — it can **block NRVO** and pessimizes codegen. Just `return local;`.
- Move constructors should be marked `noexcept` when possible — `std::vector` uses copy instead of move if the move operation may throw.

### Lvalues vs Rvalues (Classic View)
```
┌────────────────────────────────────────────────────────┐
│              Lvalues vs Rvalues                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Lvalue: Has a name, persists beyond expression        │
│    int x = 5;     // x is lvalue                       │
│    x = 10;        // OK: can assign to lvalue          │
│                                                        │
│  Rvalue: Temporary, doesn't persist                    │
│    int y = 5 + 3; // 5+3 is prvalue                    │
│    5 + 3 = 10;    // ERROR: can't assign to rvalue     │
│                                                        │
│  References:                                           │
│    int& lref = x;      // Lvalue reference             │
│    int&& rref = 5 + 3; // Rvalue reference (C++11)     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Move Constructor and Move Assignment
```cpp
#include <utility>
#include <cstring>

class String {
    char* data_;
    size_t size_;
    
public:
    // Constructor
    String(const char* str) {
        size_ = std::strlen(str);
        data_ = new char[size_ + 1];
        std::strcpy(data_, str);
    }
    
    // Destructor
    ~String() {
        delete[] data_;
    }
    
    // Copy constructor (deep copy)
    String(const String& other)
        : size_(other.size_) {
        data_ = new char[size_ + 1];
        std::strcpy(data_, other.data_);
    }
    
    // Copy assignment
    String& operator=(const String& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new char[size_ + 1];
            std::strcpy(data_, other.data_);
        }
        return *this;
    }
    
    // Move constructor (C++11)
    String(String&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // Move assignment (C++11)
    String& operator=(String&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
    
    const char* c_str() const { return data_; }
};
```

### Visual: Move vs Copy
```
Copy Constructor:
Source:  [ptr] → [H][e][l][l][o]
                    ↓   (copy)
Dest:    [ptr] → [H][e][l][l][o]  (new allocation)

Move Constructor:
Source:  [ptr] → [H][e][l][l][o]
           │ \
           │  \__ (steal pointer)
           ↓     \
Dest:    [ptr] → [H][e][l][l][o]  (no new allocation!)
Source:  [nullptr]  (source is empty but valid)

Cost: Copy = O(n), Move = O(1)
```

### std::move
```cpp
#include <iostream>
#include <utility>
#include <vector>
#include <string>

int main() {
    std::string s1 = "Hello";

    // std::move is a cast to rvalue reference — enables move overload
    std::string s2 = std::move(s1);
    // s1 is now valid-but-unspecified; s2 owns the data

    std::cout << "s1: " << s1 << '\n';  // May be empty
    std::cout << "s2: " << s2 << '\n';  // "Hello"

    std::vector<std::string> vec;
    vec.push_back(std::move(s2));  // Explicit move into container

    return 0;
}
```

### When Move Happens Automatically
```cpp
#include <vector>
#include <string>

std::string make_string() {
    std::string temp = "Hello";
    return temp;  // NRVO or implicit move — never std::move here!
}

int main() {
    std::string s = make_string();  // Constructed in s (RVO) or moved

    std::vector<std::string> vec;
    vec.push_back(std::string("World"));  // Temporary bound to rvalue ref, then moved

    return 0;
}
```

---

## Smart Pointers

### std::unique_ptr

```
┌────────────────────────────────────────────────────────┐
│                  std::unique_ptr                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  • Exclusive ownership                                 │
│  • Cannot be copied, only moved                        │
│  • Zero overhead                                       │
│  • Automatic cleanup                                   │
│                                                        │
│  Use when: Single owner needed                         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

```cpp
#include <iostream>
#include <memory>

class Widget {
public:
    Widget() { std::cout << "Widget created\n"; }
    ~Widget() { std::cout << "Widget destroyed\n"; }
    void use() { std::cout << "Using widget\n"; }
};

int main() {
    // Prefer make_unique / make_shared — never raw new/delete for ownership
    auto ptr1 = std::make_unique<Widget>();
    auto ptr2 = std::make_unique<Widget>();

    ptr2->use();

    std::unique_ptr<Widget> ptr3 = std::move(ptr2);
    // ptr2 is now nullptr

    std::shared_ptr<Widget> shared = std::make_shared<Widget>();
    std::cout << "Ref count: " << shared.use_count() << '\n';  // 1

    return 0;
}  // All widgets automatically destroyed
```

### std::shared_ptr

```
┌────────────────────────────────────────────────────────┐
│                  std::shared_ptr                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  • Shared ownership via reference counting             │
│  • Can be copied                                       │
│  • Small overhead (ref count)                          │
│  • Thread-safe ref counting (atomic)                   │
│  • NOT thread-safe for the pointee itself              │
│                                                        │
│  Use when: Multiple owners needed                      │
│  Prefer: std::make_shared (single allocation)          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

⚠️ **Gotchas**: Concurrent access to the **managed object** still requires your own synchronization. Only the reference count is atomic. Avoid `shared_ptr(new T)` — use `std::make_shared<T>(...)` for exception safety and fewer allocations.

```cpp
#include <iostream>
#include <memory>

int main() {
    std::shared_ptr<Widget> ptr1 = std::make_shared<Widget>();  // Preferred
    std::shared_ptr<Widget> ptr2 = ptr1;  // Ref count = 2

    std::cout << "Ref count: " << ptr1.use_count() << '\n';  // 2

    ptr2.reset();
    std::cout << "Ref count: " << ptr1.use_count() << '\n';  // 1

    return 0;
}
```

### Visual: shared_ptr Reference Counting
```
Creation:
ptr1 → [Widget][RefCount:1]

After ptr2 = ptr1:
ptr1 ─┐
      ├→ [Widget][RefCount:2]
ptr2 ─┘

After ptr3 = ptr1:
ptr1 ─┐
ptr2 ─┤→ [Widget][RefCount:3]
ptr3 ─┘

After ptr2.reset():
ptr1 ─┐
      ├→ [Widget][RefCount:2]
ptr3 ─┘

After ptr1 and ptr3 destroyed:
[Widget destroyed]
```

### std::weak_ptr

```
┌────────────────────────────────────────────────────────┐
│                  std::weak_ptr                         │
├────────────────────────────────────────────────────────┤
│                                                        │
│  • Non-owning observer                                 │
│  • Doesn't affect ref count                            │
│  • Can check if object still exists                    │
│                                                        │
│  Use when: Need to observe without owning              │
│  Solves: Circular reference problem                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

```cpp
#include <memory>

int main() {
    std::weak_ptr<Widget> weak;
    
    {
        std::shared_ptr<Widget> shared = std::make_shared<Widget>();
        weak = shared;  // Weak pointer observes
        
        std::cout << "Ref count: " << shared.use_count() << '\n';  // 1
        
        // Check if still alive and use
        if (auto locked = weak.lock()) {  // Returns shared_ptr
            locked->use();
            std::cout << "Object exists\n";
        }
    }  // shared destroyed
    
    // Object is gone
    if (auto locked = weak.lock()) {
        // Won't execute
    } else {
        std::cout << "Object destroyed\n";
    }
    
    return 0;
}
```

### Circular Reference Problem
```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // BAD: Circular reference!
    ~Node() { std::cout << "Node destroyed\n"; }
};

// BAD: Memory leak!
void create_cycle() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;
    node2->prev = node1;  // Circular reference
    // Neither node destroyed! Ref counts never reach 0
}

// GOOD: Break cycle with weak_ptr
struct NodeGood {
    std::shared_ptr<NodeGood> next;
    std::weak_ptr<NodeGood> prev;  // GOOD: No ownership
    ~NodeGood() { std::cout << "Node destroyed\n"; }
};
```

---

## Perfect Forwarding

### Forwarding References

`T&&` in a **deduced** context (template parameter or `auto&&`) is a forwarding (universal) reference — it binds to both lvalues and rvalues. Use `std::forward<T>` (not `std::move`) to preserve the original value category when passing along.

```cpp
#include <utility>

void process(int& x);
void process(int&& x);

template<typename T>
void wrapper(T&& arg) {  // forwarding reference when T is deduced
    process(std::forward<T>(arg));
}

int main() {
    int x = 42;
    wrapper(x);               // T = int&,  forwards as lvalue
    wrapper(42);              // T = int,   forwards as rvalue
    wrapper(std::move(x));    // T = int,   forwards as rvalue

    return 0;
}
```

### Reference Collapsing Rules
```
┌────────────────────────────────────────────────────────┐
│            Reference Collapsing Rules                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  T&  &  → T&      (lvalue ref)                         │
│  T&  && → T&      (lvalue ref)                         │
│  T&& &  → T&      (lvalue ref)                         │
│  T&& && → T&&     (rvalue ref)                         │
│                                                        │
│  Result: Only T&& && produces T&&                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Perfect Forwarding Example
```cpp
#include <iostream>
#include <memory>
#include <utility>

// Use the standard factory — shown here to illustrate forwarding mechanics
template<typename T, typename... Args>
std::unique_ptr<T> make(Args&&... args) {
    return std::make_unique<T>(std::forward<Args>(args)...);
}

struct Point {
    int x, y;
    Point(int x, int y) : x(x), y(y) {}
    Point(const Point&) { std::cout << "Copy\n"; }
    Point(Point&&) noexcept { std::cout << "Move\n"; }
};

int main() {
    auto p1 = make<Point>(10, 20);

    Point temp(1, 2);
    auto p2 = make<Point>(temp);                    // Forwards lvalue → copy
    auto p3 = make<Point>(std::move(temp));          // Forwards xvalue → move

    return 0;
}
```

---

## Rule of Five / Rule of Zero

### Rule of Five
```
If you define any of these, you should probably define all:
1. Destructor
2. Copy constructor
3. Copy assignment operator
4. Move constructor
5. Move assignment operator
```

```cpp
class Resource {
    int* data_;
    
public:
    // 1. Destructor
    ~Resource() {
        delete data_;
    }
    
    // 2. Copy constructor
    Resource(const Resource& other)
        : data_(new int(*other.data_)) {}
    
    // 3. Copy assignment
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete data_;
            data_ = new int(*other.data_);
        }
        return *this;
    }
    
    // 4. Move constructor
    Resource(Resource&& other) noexcept
        : data_(other.data_) {
        other.data_ = nullptr;
    }
    
    // 5. Move assignment
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) {
            delete data_;
            data_ = other.data_;
            other.data_ = nullptr;
        }
        return *this;
    }
};
```

### Rule of Zero

Prefer the **Rule of Zero**: if all members manage their own resources (e.g. `std::unique_ptr`, `std::vector`, `std::string`), the compiler-generated special members are correct. See [Best Practices](13_best_practices.md).

```cpp
class BetterResource {
    std::unique_ptr<int> data_;  // Manages itself!
    
public:
    // No need to define any special member functions!
    // Compiler-generated versions work perfectly:
    // - Destructor: unique_ptr cleans up
    // - Move: unique_ptr moves
    // - Copy: deleted automatically (unique_ptr is not copyable)
};
```

---

## Copy Elision and RVO

### Return Value Optimization (RVO)
```cpp
std::string make_string() {
    std::string result = "Hello";
    // ... modify result ...
    return result;  // NRVO: compilers elide the copy/move here, but unlike
                    // returning a prvalue this elision is NOT guaranteed by
                    // the standard (it is a permitted optimization).
}

int main() {
    std::string s = make_string();  // Constructed directly in s
    // No temporary created, no move/copy
    
    return 0;
}
```

### Named Return Value Optimization (NRVO)
```cpp
// NRVO may be applied (compiler-dependent)
std::vector<int> make_vector() {
    std::vector<int> vec;
    vec.push_back(1);
    vec.push_back(2);
    return vec;  // May be constructed directly at call site
}
```

### Guaranteed Copy Elision (C++17)
```cpp
// C++17: Guaranteed elision for temporaries
std::string s = std::string("Hello");  // No move, direct construction

Widget get_widget() {
    return Widget();  // Guaranteed copy elision (C++17): initializes the
                      // caller's object directly — no copy/move is performed.
}

// Note: a prvalue still *denotes* an object; under C++17 "guaranteed copy
// elision" it materializes a temporary only when actually needed (e.g. when
// bound to a reference), and here it initializes the result object directly.
```

---

## Attributes (C++11+)

### Common Attributes
```cpp
// [[nodiscard]] (C++17): Warn if return value ignored
[[nodiscard]] int calculate() {
    return 42;
}

void test() {
    calculate();  // Warning: return value ignored
    int x = calculate();  // OK
}

// [[maybe_unused]]: Suppress unused variable warning
void func([[maybe_unused]] int debug_param) {
    // debug_param used only in debug builds
    #ifdef DEBUG
        std::cout << debug_param << '\n';
    #endif
}

// [[likely]], [[unlikely]] (C++20): Branch prediction hints
int process(int x) {
    if (x > 0) [[likely]] {
        return x * 2;
    } else [[unlikely]] {
        return 0;
    }
}

// [[fallthrough]] (C++17): Intentional switch fallthrough
switch (value) {
    case 1:
        do_something();
        [[fallthrough]];
    case 2:
        do_something_else();
        break;
}

// [[deprecated]] (C++14): Mark as deprecated
[[deprecated("Use new_function() instead")]]
void old_function() {
    // ...
}

// [[noreturn]] (C++11): Function never returns
[[noreturn]] void fatal_error() {
    throw std::runtime_error("Fatal");
}
```

---

## Inline Variables (C++17)

```cpp
// Header file
// Before C++17: Problem with multiple definitions
extern const int value;  // Declaration
// cpp file needed: const int value = 42;

// C++17: inline variables
inline const int value = 42;  // Definition in header, OK!

// Especially useful for static members
class Config {
public:
    inline static int max_connections = 100;  // C++17
    inline static const std::string name = "MyApp";
};
```

---

## Structured Bindings (C++17)

```cpp
#include <map>
#include <tuple>

int main() {
    // Decompose pair
    std::pair<int, std::string> p(42, "hello");
    auto [num, str] = p;  // num = 42, str = "hello"
    
    // Decompose tuple
    std::tuple<int, double, std::string> t(1, 3.14, "world");
    auto [i, d, s] = t;
    
    // Iterate map
    std::map<std::string, int> ages = { {"Alice", 25}, {"Bob", 30} };
    for (const auto& [name, age] : ages) {
        std::cout << name << ": " << age << '\n';
    }
    
    // Decompose struct
    struct Point { int x, y; };
    Point point{10, 20};
    auto [x, y] = point;
    
    return 0;
}
```

---

## std::launder (C++17)

### Strict Aliasing and Object Lifetime
```cpp
#include <new>

// Placement new with different type
struct A { int x; };
struct B { int y; };

alignas(B) char buffer[sizeof(B)];

A* a = new (buffer) A{42};
// Object lifetime of A begins

a->~A();  // Destroy A
// Object lifetime of A ends

B* b = new (buffer) B{100};
// Object lifetime of B begins — the storage now holds a B, NOT an A.

// Accessing the storage as an A is UB now (there is no A object there):
// int val = a->x;            // UB! 'a' points to an object whose life ended
// A* bad = std::launder(reinterpret_cast<A*>(buffer));  // also UB: no A here

// Just use the pointer the placement-new expression returned:
b->y = 200;
// std::launder is only needed to "refresh" a pointer to the SAME type that
// now lives in reused storage (e.g. after re-placement-newing a B), not to
// reinterpret a B as an A.
B* valid_b = std::launder(reinterpret_cast<B*>(buffer));
```

---

## Best Practices Summary

### 1. Resource Management
```cpp
// ALWAYS use RAII
// - Smart pointers for dynamic allocation
// - Lock guards for mutexes
// - unique_ptr for ownership
// - shared_ptr for shared ownership
```

### 2. Move Semantics
```cpp
// Enable move for custom types
// Mark move operations noexcept
// Use std::move explicitly when needed
// Don't std::move on return (RVO)
```

### 3. Rule of Five/Zero
```cpp
// Prefer Rule of Zero
// If managing resources, follow Rule of Five
```

### 4. Smart Pointers
```cpp
// Prefer unique_ptr over shared_ptr
// Use make_unique/make_shared
// Use weak_ptr to break cycles
```

---

## Common Pitfalls

### 1. Using Moved-From Objects
```cpp
std::string s1 = "Hello";
std::string s2 = std::move(s1);
std::cout << s1;  // UB? No, but value is unspecified
s1 = "New value";  // OK: assign new value
```

### 2. Returning by const value
```cpp
// BAD: Prevents move
const std::string bad_function() {
    return std::string("Hello");  // Can't move!
}

// GOOD: Allow move
std::string good_function() {
    return std::string("Hello");  // Can move
}
```

### 3. std::move in Return
```cpp
// BAD: Prevents RVO
std::string bad() {
    std::string s = "Hello";
    return std::move(s);  // Don't do this!
}

// GOOD: Allow RVO
std::string good() {
    std::string s = "Hello";
    return s;  // RVO or move automatically
}
```

---

## Complete Practical Example: Resource Manager with Advanced Features

Here's a comprehensive example integrating RAII, move semantics, smart pointers, perfect forwarding, and all advanced C++ features:

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>
#include <utility>
#include <type_traits>
#include <optional>

// 1. RAII wrapper for file handle
class FileHandle {
private:
    FILE* file_;
    std::string filename_;
    
public:
    // Constructor acquires resource
    explicit FileHandle(const std::string& filename, const char* mode = "r")
        : file_(fopen(filename.c_str(), mode)), filename_(filename) {
        if (!file_) {
            throw std::runtime_error("Failed to open file: " + filename);
        }
        std::cout << "Opened file: " << filename_ << "\n";
    }
    
    // Destructor releases resource
    ~FileHandle() {
        if (file_) {
            fclose(file_);
            std::cout << "Closed file: " << filename_ << "\n";
        }
    }
    
    // Delete copy operations
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    
    // Move constructor
    FileHandle(FileHandle&& other) noexcept
        : file_(other.file_), filename_(std::move(other.filename_)) {
        other.file_ = nullptr;
        std::cout << "Moved file handle: " << filename_ << "\n";
    }
    
    // Move assignment
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            // Release current resource
            if (file_) {
                fclose(file_);
            }
            
            // Transfer ownership
            file_ = other.file_;
            filename_ = std::move(other.filename_);
            other.file_ = nullptr;
        }
        return *this;
    }
    
    FILE* get() const { return file_; }
    bool is_open() const { return file_ != nullptr; }
};

// 2. Resource with Rule of Five
class Buffer {
private:
    char* data_;
    size_t size_;
    
public:
    // Constructor
    explicit Buffer(size_t size) : data_(new char[size]), size_(size) {
        std::cout << "Buffer allocated: " << size_ << " bytes\n";
    }
    
    // Destructor
    ~Buffer() {
        delete[] data_;
        std::cout << "Buffer deallocated\n";
    }
    
    // Copy constructor
    Buffer(const Buffer& other) : data_(new char[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
        std::cout << "Buffer copied: " << size_ << " bytes\n";
    }
    
    // Copy assignment
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            // Allocate new
            char* new_data = new char[other.size_];
            std::copy(other.data_, other.data_ + other.size_, new_data);
            
            // Release old
            delete[] data_;
            
            // Transfer
            data_ = new_data;
            size_ = other.size_;
            
            std::cout << "Buffer copy-assigned: " << size_ << " bytes\n";
        }
        return *this;
    }
    
    // Move constructor
    Buffer(Buffer&& other) noexcept 
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
        std::cout << "Buffer moved: " << size_ << " bytes\n";
    }
    
    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            // Release old
            delete[] data_;
            
            // Transfer
            data_ = other.data_;
            size_ = other.size_;
            
            // Reset other
            other.data_ = nullptr;
            other.size_ = 0;
            
            std::cout << "Buffer move-assigned: " << size_ << " bytes\n";
        }
        return *this;
    }
    
    size_t size() const { return size_; }
    char* data() { return data_; }
    const char* data() const { return data_; }
};

// 3. Perfect forwarding factory
template<typename T, typename... Args>
std::unique_ptr<T> make_resource(Args&&... args) {
    std::cout << "Creating resource with " << sizeof...(args) << " arguments\n";
    return std::make_unique<T>(std::forward<Args>(args)...);
}

// 4. Smart pointer hierarchy
class Resource {
protected:
    std::string name_;
    
public:
    explicit Resource(std::string name) : name_(std::move(name)) {
        std::cout << "Resource created: " << name_ << "\n";
    }
    
    virtual ~Resource() {
        std::cout << "Resource destroyed: " << name_ << "\n";
    }
    
    virtual void use() {
        std::cout << "Using resource: " << name_ << "\n";
    }
    
    const std::string& name() const { return name_; }
};

class DatabaseConnection : public Resource {
private:
    bool connected_;
    
public:
    explicit DatabaseConnection(std::string name) 
        : Resource(std::move(name)), connected_(true) {
        std::cout << "  Database connected\n";
    }
    
    ~DatabaseConnection() override {
        disconnect();
    }
    
    void disconnect() {
        if (connected_) {
            std::cout << "  Database disconnected\n";
            connected_ = false;
        }
    }
    
    void use() override {
        if (connected_) {
            std::cout << "Executing query on: " << name_ << "\n";
        }
    }
};

// 5. Resource manager with smart pointers
class ResourceManager {
private:
    std::vector<std::unique_ptr<Resource>> owned_resources_;
    std::vector<std::shared_ptr<Resource>> shared_resources_;
    std::vector<std::weak_ptr<Resource>> observed_resources_;
    
public:
    // Add owned resource (unique ownership)
    void add_owned(std::unique_ptr<Resource> resource) {
        std::cout << "Adding owned resource: " << resource->name() << "\n";
        owned_resources_.push_back(std::move(resource));
    }
    
    // Add shared resource
    std::shared_ptr<Resource> add_shared(std::unique_ptr<Resource> resource) {
        auto shared = std::shared_ptr<Resource>(std::move(resource));
        std::cout << "Adding shared resource: " << shared->name() 
                  << " (refcount: " << shared.use_count() << ")\n";
        shared_resources_.push_back(shared);
        return shared;
    }
    
    // Observe resource without ownership
    void observe(std::shared_ptr<Resource> resource) {
        std::cout << "Observing resource: " << resource->name() << "\n";
        observed_resources_.push_back(std::weak_ptr<Resource>(resource));
    }
    
    // Use all resources
    void use_all() {
        std::cout << "\n--- Using Owned Resources ---\n";
        for (auto& resource : owned_resources_) {
            resource->use();
        }
        
        std::cout << "\n--- Using Shared Resources ---\n";
        for (auto& resource : shared_resources_) {
            resource->use();
            std::cout << "  Refcount: " << resource.use_count() << "\n";
        }
        
        std::cout << "\n--- Checking Observed Resources ---\n";
        for (auto& weak : observed_resources_) {
            if (auto shared = weak.lock()) {
                shared->use();
                std::cout << "  Still alive, refcount: " << shared.use_count() << "\n";
            } else {
                std::cout << "  Resource expired\n";
            }
        }
    }
    
    void clear_shared() {
        std::cout << "\nClearing shared resources...\n";
        shared_resources_.clear();
    }
};

// 6. Value categories demonstration
template<typename T>
void show_value_category(T&& value) {
    using ValueType = std::remove_reference_t<T>;
    
    std::cout << "Value category: ";
    if constexpr (std::is_lvalue_reference_v<T>) {
        std::cout << "lvalue reference\n";
    } else if constexpr (std::is_rvalue_reference_v<T>) {
        std::cout << "rvalue reference\n";
    } else {
        std::cout << "value\n";
    }
}

// 7. Custom deleters
struct CustomDeleter {
    void operator()(Resource* ptr) const {
        std::cout << "Custom deleter called\n";
        delete ptr;
    }
};

// Demonstrations
void demo_raii() {
    std::cout << "\n=== RAII Demo ===\n";
    
    try {
        // Create temporary file for demo
        {
            FILE* f = fopen("/tmp/test.txt", "w");
            if (f) {
                fprintf(f, "Test data");
                fclose(f);
            }
        }
        
        FileHandle file("/tmp/test.txt", "r");
        std::cout << "File is open: " << file.is_open() << "\n";
        
        // File automatically closed when leaving scope
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n";
    }
    
    std::cout << "After scope - file closed automatically\n";
}

void demo_rule_of_five() {
    std::cout << "\n=== Rule of Five Demo ===\n";
    
    Buffer buf1(1024);
    
    // Copy
    Buffer buf2 = buf1;
    
    // Move
    Buffer buf3 = std::move(buf1);
    
    // Copy assignment
    Buffer buf4(512);
    buf4 = buf2;
    
    // Move assignment
    Buffer buf5(256);
    buf5 = std::move(buf2);
}

void demo_smart_pointers() {
    std::cout << "\n=== Smart Pointers Demo ===\n";
    
    ResourceManager manager;
    
    // Unique ownership
    auto db1 = std::make_unique<DatabaseConnection>("DB1");
    manager.add_owned(std::move(db1));
    
    // Shared ownership
    auto db2 = std::make_unique<DatabaseConnection>("DB2");
    auto shared_db2 = manager.add_shared(std::move(db2));
    
    // Additional shared reference
    auto external_ref = shared_db2;
    std::cout << "External reference created, refcount: " 
              << external_ref.use_count() << "\n";
    
    // Observe without owning
    manager.observe(shared_db2);
    
    // Use all resources
    manager.use_all();
    
    // Clear shared resources
    manager.clear_shared();
    
    // Check observed - should be expired
    manager.use_all();
    
    // External reference still valid
    std::cout << "\nExternal reference still valid:\n";
    external_ref->use();
    std::cout << "Refcount: " << external_ref.use_count() << "\n";
}

void demo_perfect_forwarding() {
    std::cout << "\n=== Perfect Forwarding Demo ===\n";
    
    // Create resources with perfect forwarding
    auto res1 = make_resource<Resource>("Resource1");
    auto res2 = make_resource<DatabaseConnection>("DB_Forwarded");
    
    res1->use();
    res2->use();
}

void demo_value_categories() {
    std::cout << "\n=== Value Categories Demo ===\n";
    
    int x = 42;
    
    show_value_category(x);              // lvalue
    show_value_category(std::move(x));   // rvalue
    show_value_category(42);             // rvalue
    show_value_category(x + 1);          // rvalue
}

void demo_custom_deleter() {
    std::cout << "\n=== Custom Deleter Demo ===\n";
    
    std::unique_ptr<Resource, CustomDeleter> res(
        new Resource("CustomDeleted")
    );
    
    res->use();
    
    std::cout << "Leaving scope, custom deleter will be called...\n";
}

void demo_move_semantics() {
    std::cout << "\n=== Move Semantics Demo ===\n";
    
    std::vector<Buffer> buffers;
    
    std::cout << "Adding buffer (move):\n";
    buffers.push_back(Buffer(128));
    
    std::cout << "\nAdding another buffer (move):\n";
    Buffer buf(256);
    buffers.push_back(std::move(buf));
    
    std::cout << "\nOriginal buffer size after move: " << buf.size() << "\n";
}

int main() {
    std::cout << "=== Advanced C++ Features Demo ===\n";
    
    // 1. RAII
    demo_raii();
    
    // 2. Rule of Five
    demo_rule_of_five();
    
    // 3. Smart pointers
    demo_smart_pointers();
    
    // 4. Perfect forwarding
    demo_perfect_forwarding();
    
    // 5. Value categories
    demo_value_categories();
    
    // 6. Custom deleters
    demo_custom_deleter();
    
    // 7. Move semantics
    demo_move_semantics();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **RAII**: Automatic resource management
- **Rule of Five**: All special member functions
- **Move semantics**: Efficient resource transfer
- **Perfect forwarding**: Preserving value categories
- **Smart pointers**: unique_ptr, shared_ptr, weak_ptr
- **Custom deleters**: Custom cleanup logic
- **Value categories**: lvalue, rvalue, xvalue
- **std::move**: Explicit move requests
- **std::forward**: Perfect forwarding
- **Reference counting**: shared_ptr mechanics
- **Weak references**: Breaking circular dependencies
- **Resource ownership**: Clear ownership semantics
- **Exception safety**: RAII guarantees cleanup
- **Copy elision**: RVO and NRVO

This example shows modern C++ resource management in action!

---

## Related Topics

- [Best Practices](13_best_practices.md) — Rule of Zero, `emplace`, noexcept move operations
- [Lambdas](10_lambdas.md) — perfect forwarding in template lambdas; init-capture with `std::move`
- [Metaprogramming](11_metaprogramming.md) — `if constexpr` with type traits and value categories
- [Exceptions](17_exceptions.md) — RAII and exception safety guarantees
- [Memory & Allocators](19_memory_allocators.md) — allocation patterns behind `make_shared` / `make_unique`
- [Multithreading](14_multithreading.md) — `shared_ptr` atomic ref count vs pointee thread safety
- [OOP Concepts](00_oop_concepts.md) — value semantics, encapsulation, destructor role in RAII

---

## Next Steps
- **Next**: [Best Practices and Idioms →](13_best_practices.md)
- **Previous**: [← Metaprogramming](11_metaprogramming.md)

---
*Chapter 12 — Advanced C++ Features*

