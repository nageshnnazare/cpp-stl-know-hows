# C++ STL Quick Reference Guide

A condensed cheat sheet for daily C++ development. Bookmark this page!

---

## Container Selection Guide

### Decision Tree

```
┌───────────────────────────────────────────────────────────────────────────────┐
│            WHICH CONTAINER SHOULD I USE?                                      │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Need random access (by index)?                                               │
│  ├─ YES ─> Fixed size?                                                        │
│  │         ├─ YES ─> std::array                                               │
│  │         └─ NO ──> Fast insert/delete at ends?                              │
│  │                   ├─ Back only ───> std::vector                            │
│  │                   └─ Both ends ───> std::deque                             │
│  │                                                                            │
│  └─ NO ──> Need sorted order?                                                 │
│            ├─ YES ─> Unique keys?                                             │
│            │         ├─ YES ─> Key only?                                      │
│            │         │         ├─ YES ─> std::set                             │
│            │         │         └─ NO ──> std::map                             │
│            │         └─ NO ──> Key only?                                      │
│            │                   ├─ YES ─> std::multiset                        │
│            │                   └─ NO ──> std::multimap                        │
│            │                                                                  │
│            └─ NO ──> Need fast lookup?                                        │
│                      ├─ YES ─> Unique keys?                                   │
│                      │         ├─ YES ─> Key only?                            │
│                      │         │         ├─ YES ─> std::unordered_set         │
│                      │         │         └─ NO ──> std::unordered_map         │
│                      │         └─ NO ──> Key only?                            │
│                      │                   ├─ YES ─> std::unordered_multiset    │
│                      │                   └─ NO ──> std::unordered_multimap    │
│                      │                                                        │
│                      └─ NO ──> Frequent insert/delete middle?                 │
│                                ├─ YES ─> Forward only?                        │
│                                │         ├─ YES ─> std::forward_list          │
│                                │         └─ NO ──> std::list                  │
│                                └─ NO ──> std::vector                          │
│                                                                               │
│  Special cases:                                                               │
│  - LIFO (stack) ────────────────────> std::stack                              │
│  - FIFO (queue) ────────────────────> std::queue                              │
│  - Priority queue ──────────────────> std::priority_queue                     │
│  - Bits/flags ──────────────────────> std::bitset                             │
│  - String ──────────────────────────> std::string                             │
│  - Optional value ──────────────────> std::optional                           │
│  - One of several types ────────────> std::variant                            │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Time Complexity Quick Reference

```
┌────────────────────────────────────────────────────────────────────┐
│                   OPERATION COMPLEXITY TABLE                       │
├────────────────────────────────────────────────────────────────────┤
│ Container       │ Access │ Insert │ Delete  │ Search │ Memory      │
├─────────────────┼────────┼────────┼─────────┼────────┼─────────────┤
│ vector          │ O(1)   │ O(n)   │ O(n)    │ O(n)   │ Lowest      │
│                 │        │ O(1)*  │ O(1)*   │        │             │
│ deque           │ O(1)   │ O(n)   │ O(n)    │ O(n)   │ Medium      │
│                 │        │ O(1)*  │ O(1)*   │        │             │
│ list            │ O(n)   │ O(1)** │ O(1)**  │ O(n)   │ High        │
│ forward_list    │ O(n)   │ O(1)** │ O(1)**  │ O(n)   │ Lower       │
│ array           │ O(1)   │ N/A    │ N/A     │ O(n)   │ Lowest      │
├─────────────────┼────────┼────────┼─────────┼────────┼─────────────┤
│ set/map         │O(log n)│O(log n)│ O(log n)│O(log n)│ Medium      │
│ multiset/map    │O(log n)│O(log n)│ O(log n)│O(log n)│ Medium      │
├─────────────────┼────────┼────────┼─────────┼────────┼─────────────┤
│ unordered_set   │ N/A    │ O(1)***│ O(1)*** │ O(1)***│ High        │
│ unordered_map   │ O(1)   │ O(1)***│ O(1)*** │ O(1)***│ High        │
├─────────────────┼────────┼────────┼─────────┼────────┼─────────────┤
│ stack           │ O(1)†  │ O(1)   │ O(1)    │ N/A    │ Low         │
│ queue           │ O(1)†  │ O(1)   │ O(1)    │ N/A    │ Low         │
│ priority_queue  │ O(1)†  │O(log n)│O(log n) │ N/A    │ Low         │
└────────────────────────────────────────────────────────────────────┘

* at back/front
** if you have iterator to position
*** average case, O(n) worst case
† top/front only
```

---

## Most Common Algorithms - Quick Syntax

### Sorting & Searching

```cpp
// Sort
std::sort(v.begin(), v.end());                    // Ascending
std::sort(v.begin(), v.end(), std::greater<>());  // Descending
std::sort(v.begin(), v.end(), [](auto& a, auto& b) { return a.x < b.x; });

// Partial sort (top N)
std::partial_sort(v.begin(), v.begin() + 10, v.end());

// Find
auto it = std::find(v.begin(), v.end(), value);
auto it = std::find_if(v.begin(), v.end(), [](auto& x) { return x > 10; });

// Binary search (requires sorted)
bool found = std::binary_search(v.begin(), v.end(), value);
auto it = std::lower_bound(v.begin(), v.end(), value);  // First >= value
auto it = std::upper_bound(v.begin(), v.end(), value);  // First > value

// Min/Max
auto min_it = std::min_element(v.begin(), v.end());
auto max_it = std::max_element(v.begin(), v.end());
auto [min_it, max_it] = std::minmax_element(v.begin(), v.end());
```

### Modifying Sequences

```cpp
// Copy
std::copy(src.begin(), src.end(), dest.begin());
std::copy_if(src.begin(), src.end(), dest.begin(), predicate);

// Transform
std::transform(v.begin(), v.end(), v.begin(), [](auto x) { return x * 2; });

// Remove (erase-remove idiom)
v.erase(std::remove(v.begin(), v.end(), value), v.end());
v.erase(std::remove_if(v.begin(), v.end(), predicate), v.end());

// Unique (remove consecutive duplicates - requires sorted)
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());

// Reverse
std::reverse(v.begin(), v.end());

// Rotate
std::rotate(v.begin(), v.begin() + 3, v.end());  // Shift left by 3

// Fill
std::fill(v.begin(), v.end(), value);
std::fill_n(v.begin(), 10, value);

// Generate
std::generate(v.begin(), v.end(), []() { return rand(); });
```

### Numeric Operations

```cpp
// Sum
int sum = std::accumulate(v.begin(), v.end(), 0);
int sum = std::reduce(v.begin(), v.end(), 0);  // C++17, parallel

// Product
int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<>());

// Inner product
int dot = std::inner_product(v1.begin(), v1.end(), v2.begin(), 0);

// Partial sum
std::partial_sum(v.begin(), v.end(), result.begin());

// Adjacent difference
std::adjacent_difference(v.begin(), v.end(), result.begin());
```

### Counting & Testing

```cpp
// Count
int n = std::count(v.begin(), v.end(), value);
int n = std::count_if(v.begin(), v.end(), predicate);

// All, any, none
bool all = std::all_of(v.begin(), v.end(), predicate);
bool any = std::any_of(v.begin(), v.end(), predicate);
bool none = std::none_of(v.begin(), v.end(), predicate);

// Equal
bool equal = std::equal(v1.begin(), v1.end(), v2.begin());

// Mismatch
auto [it1, it2] = std::mismatch(v1.begin(), v1.end(), v2.begin());
```

---

## C++20 Ranges - Quick Syntax

```cpp
#include <ranges>
namespace rng = std::ranges;
namespace vws = std::views;

// Filter
auto evens = v | vws::filter([](int x) { return x % 2 == 0; });

// Transform
auto doubled = v | vws::transform([](int x) { return x * 2; });

// Take/Drop
auto first10 = v | vws::take(10);
auto skip10 = v | vws::drop(10);

// Chain operations
auto result = v 
    | vws::filter([](int x) { return x > 0; })
    | vws::transform([](int x) { return x * x; })
    | vws::take(5);

// Ranges algorithms (no .begin()/.end() needed!)
rng::sort(v);
rng::find(v, value);
rng::copy(v, std::back_inserter(dest));

// With projections
rng::sort(people, {}, &Person::age);  // Sort by age
```

---

## Modern C++ Features Quick Reference

### C++11

```cpp
// Auto type deduction
auto x = 42;
auto ptr = std::make_unique<int>(42);

// Range-based for loop
for (const auto& item : container) { }

// Lambda expressions
auto lambda = [](int x) { return x * 2; };
auto capture = [&, x](int y) { return x + y; };

// Move semantics
std::vector<int> v2 = std::move(v1);

// Smart pointers
std::unique_ptr<T> ptr = std::make_unique<T>(args);
std::shared_ptr<T> ptr = std::make_shared<T>(args);

// nullptr
T* ptr = nullptr;

// Uniform initialization
std::vector<int> v{1, 2, 3, 4, 5};

// Deleted/defaulted functions
ClassName(const ClassName&) = delete;
ClassName() = default;

// override and final
void func() override;
void func() final;
```

### C++14

```cpp
// Generic lambdas
auto lambda = [](auto x) { return x * 2; };

// Lambda capture with initializers
auto lambda = [x = getValue()](){ return x; };

// make_unique
auto ptr = std::make_unique<T>(args);

// Return type deduction
auto func() { return 42; }
```

### C++17

```cpp
// Structured bindings
auto [x, y] = std::pair{1, 2};
auto [key, value] = *map.find("key");

// if with initializer
if (auto it = map.find(key); it != map.end()) { }

// if constexpr
if constexpr (condition) { } else { }

// std::optional
std::optional<int> opt = getValue();
if (opt) { use(*opt); }

// std::variant
std::variant<int, double, std::string> var = 42;
std::visit([](auto& x) { /* ... */ }, var);

// std::filesystem
std::filesystem::path p = "/path/to/file";
if (std::filesystem::exists(p)) { }

// Parallel algorithms
std::sort(std::execution::par, v.begin(), v.end());
```

### C++20

```cpp
// Concepts
template<std::integral T>
void func(T value) { }

// Ranges
auto result = v | std::views::filter(pred) | std::views::transform(fn);

// Coroutines (C++20 adds the language keywords co_await/co_yield/co_return).
// std::generator itself is C++23 (<generator>); in C++20 you hand-roll a type.
std::generator<int> range(int start, int end) {   // <generator>, C++23
    for (int i = start; i < end; ++i) {
        co_yield i;
    }
}

// Three-way comparison (spaceship)
auto operator<=>(const T&) const = default;

// Designated initializers
Person p{.name = "John", .age = 30};

// consteval (immediate functions)
consteval int square(int x) { return x * x; }

// std::span
void func(std::span<int> data) { }

// std::format
std::string s = std::format("Hello, {}!", name);
```

### C++23

```cpp
// std::expected
std::expected<int, Error> result = compute();
if (result) { use(*result); }

// std::mdspan (multidimensional span)
std::mdspan<int, std::dextents<int, 2>> matrix(data, 3, 4);

// ranges::to
auto vec = range | std::ranges::to<std::vector>();

// std::print
std::print("Hello, {}!\n", name);

// Deducing this
struct S {
    template<typename Self>
    auto&& get(this Self&& self) { return std::forward<Self>(self).value; }
};
```

---

## Common Patterns Quick Reference

### RAII Pattern

```cpp
class Resource {
public:
    Resource() { /* acquire */ }
    ~Resource() { /* release */ }
    
    // Rule of Five
    Resource(const Resource&) = delete;
    Resource& operator=(const Resource&) = delete;
    Resource(Resource&&) noexcept = default;
    Resource& operator=(Resource&&) noexcept = default;
};

// Usage
{
    Resource r;  // Automatically cleaned up
}  // Destructor called here
```

### Factory Pattern

```cpp
class Product {
public:
    virtual ~Product() = default;
    virtual void use() = 0;
};

class Factory {
public:
    static std::unique_ptr<Product> create(Type type) {
        switch (type) {
            case Type::A: return std::make_unique<ProductA>();
            case Type::B: return std::make_unique<ProductB>();
        }
    }
};

// Usage
auto product = Factory::create(Type::A);
product->use();
```

### Observer Pattern (Modern)

```cpp
class Subject {
    std::vector<std::function<void(int)>> observers_;
    
public:
    void attach(std::function<void(int)> observer) {
        observers_.push_back(std::move(observer));
    }
    
    void notify(int value) {
        for (auto& observer : observers_) {
            observer(value);
        }
    }
};

// Usage
Subject subject;
subject.attach([](int x) { std::cout << "Observer 1: " << x << "\n"; });
subject.attach([](int x) { std::cout << "Observer 2: " << x * 2 << "\n"; });
subject.notify(42);
```

### Strategy Pattern (Modern)

```cpp
class Context {
    std::function<int(int, int)> strategy_;
    
public:
    void set_strategy(std::function<int(int, int)> strategy) {
        strategy_ = std::move(strategy);
    }
    
    int execute(int a, int b) {
        return strategy_(a, b);
    }
};

// Usage
Context ctx;
ctx.set_strategy([](int a, int b) { return a + b; });
std::cout << ctx.execute(5, 3) << "\n";  // 8

ctx.set_strategy([](int a, int b) { return a * b; });
std::cout << ctx.execute(5, 3) << "\n";  // 15
```

### Singleton Pattern (Thread-Safe)

```cpp
class Singleton {
public:
    static Singleton& instance() {
        static Singleton instance;  // C++11 guarantees thread safety
        return instance;
    }
    
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    
private:
    Singleton() = default;
};

// Usage
auto& s = Singleton::instance();
```

### Pimpl Idiom

```cpp
// Header
class Widget {
public:
    Widget();
    ~Widget();
    void operation();
    
private:
    class Impl;
    std::unique_ptr<Impl> pimpl_;
};

// Implementation
class Widget::Impl {
    // Private implementation details
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // Must be in .cpp for unique_ptr<Impl>
void Widget::operation() { pimpl_->do_something(); }
```

---

## Performance Tips Quick Reference

### Container Performance

```cpp
// ✓ GOOD: Reserve capacity for vector
std::vector<int> v;
v.reserve(1000);  // Avoid reallocations
for (int i = 0; i < 1000; ++i) {
    v.push_back(i);
}

// ✗ BAD: Random access in list
std::list<int> lst;
auto it = lst.begin();
std::advance(it, 500);  // O(n) - slow!

// ✓ GOOD: Use vector for random access
std::vector<int> vec;
int x = vec[500];  // O(1) - fast!

// ✓ GOOD: emplace instead of push
v.emplace_back(arg1, arg2);  // Construct in-place
// ✗ BAD:
v.push_back(Object(arg1, arg2));  // Construct then move

// ✓ GOOD: Use unordered containers for large datasets
std::unordered_map<std::string, int> fast_lookup;  // O(1)
// ✗ BAD:
std::map<std::string, int> slow_lookup;  // O(log n)
```

### Algorithm Performance

```cpp
// ✓ GOOD: Use algorithm instead of hand-written loop
auto it = std::find(v.begin(), v.end(), value);

// ✗ BAD: Hand-written loop
for (size_t i = 0; i < v.size(); ++i) {
    if (v[i] == value) { /* ... */ }
}

// ✓ GOOD: Parallel algorithms for large data
std::sort(std::execution::par, v.begin(), v.end());

// ✓ GOOD: Use ranges for composability without temporaries
auto result = v 
    | std::views::filter(pred)
    | std::views::transform(fn);
// No intermediate containers created!
```

### Move Semantics

```cpp
// ✓ GOOD: Move instead of copy
std::vector<int> v2 = std::move(v1);
func(std::move(obj));

// ✓ GOOD: Return by value (NRVO/RVO)
std::vector<int> func() {
    std::vector<int> result;
    // ...
    return result;  // No std::move needed!
}

// ✗ BAD: Unnecessary copy
void func(std::string s) {  // Copy!
    // ...
}

// ✓ GOOD: Pass by const reference
void func(const std::string& s) {  // No copy
    // ...
}

// ✓ BEST: Use string_view for read-only strings
void func(std::string_view s) {  // No copy, more flexible
    // ...
}
```

### Cache-Friendly Code

```cpp
// ✓ GOOD: Contiguous memory (cache-friendly)
std::vector<int> v;  // Elements stored contiguously

// ✗ BAD: Non-contiguous memory (cache-unfriendly)
std::list<int> lst;  // Elements scattered in memory

// ✓ GOOD: Struct of Arrays (SoA) for large datasets
struct Players {
    std::vector<int> health;
    std::vector<int> mana;
    std::vector<Position> positions;
};

// ✗ BAD: Array of Structs (AoS) when processing one field
struct Player {
    int health;
    int mana;
    Position pos;
};
std::vector<Player> players;
```

---

## Common Gotchas Checklist

### Container Gotchas

```cpp
// ✗ Iterator invalidation after modification
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);  // it is now INVALID!

// ✗ Modifying container while iterating
for (auto it = v.begin(); it != v.end(); ++it) {
    v.erase(it);  // it is now INVALID!
}
// ✓ CORRECT:
for (auto it = v.begin(); it != v.end(); ) {
    it = v.erase(it);  // Get new valid iterator
}

// ✗ Using [] on map creates entry
std::map<int, int> m;
int x = m[5];  // Creates entry with value 0!
// ✓ CORRECT:
auto it = m.find(5);
if (it != m.end()) {
    int x = it->second;
}

// ✗ Assuming vector bool is like vector<T>
std::vector<bool> vb;
bool& ref = vb[0];  // ERROR! vector<bool> is specialized
// ✓ CORRECT: Use std::vector<char> or std::deque<bool>
```

### OOP Gotchas

```cpp
// ✗ Non-virtual destructor in base class
class Base {
    ~Base() {}  // NOT virtual!
};
Base* ptr = new Derived();
delete ptr;  // Derived destructor NOT called - memory leak!

// ✓ CORRECT:
class Base {
    virtual ~Base() {}
};

// ✗ Object slicing
Derived d;
Base b = d;  // Sliced! Only Base part copied
// ✓ CORRECT: Use pointers/references
Base& b = d;

// ✗ Calling virtual functions in constructor
class Base {
    Base() { init(); }  // Calls Base::init, not Derived::init!
    virtual void init() {}
};

// ✗ Forgetting override keyword
class Derived : public Base {
    void func() {}  // Might not override Base::func()
};
// ✓ CORRECT:
void func() override {}  // Compiler error if not overriding
```

### Template Gotchas

```cpp
// ✗ Most vexing parse
Widget w();  // Function declaration, not object!
// ✓ CORRECT:
Widget w;      // Object
Widget w{};    // Also correct (uniform initialization)

// ✗ Missing typename in dependent types
template<typename T>
void func() {
    T::value_type x;  // ERROR if value_type is a type!
}
// ✓ CORRECT:
typename T::value_type x;

// ✗ Forgetting template keyword for dependent member templates
obj.template member<T>();

// ✗ Integer division in templates
template<int N>
struct X {
    static constexpr int half = N / 2;  // Integer division!
};
```

### Move Semantics Gotchas

```cpp
// ✗ std::move doesn't move!
std::vector<int> v1 = {1, 2, 3};
std::move(v1);  // Does NOTHING! Just casts to rvalue
// ✓ CORRECT:
std::vector<int> v2 = std::move(v1);  // Actually moves

// ✗ Using moved-from object
std::string s1 = "hello";
std::string s2 = std::move(s1);
std::cout << s1;  // UB or unspecified behavior!

// ✗ Moving in return statement prevents RVO
std::vector<int> func() {
    std::vector<int> v;
    return std::move(v);  // Prevents RVO! DON'T DO THIS!
}
// ✓ CORRECT:
return v;  // Let compiler optimize

// ✗ Const objects can't be moved
const std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1);  // Copies, doesn't move!
```

### Concurrency Gotchas

```cpp
// ✗ Data race
int counter = 0;
std::thread t1([&]() { ++counter; });  // Race condition!
std::thread t2([&]() { ++counter; });

// ✓ CORRECT: Use mutex or atomic
std::atomic<int> counter{0};
std::thread t1([&]() { ++counter; });
std::thread t2([&]() { ++counter; });

// ✗ Deadlock
std::mutex m1, m2;
// Thread 1:
m1.lock(); m2.lock();
// Thread 2:
m2.lock(); m1.lock();  // DEADLOCK!

// ✓ CORRECT: Lock in same order or use scoped_lock
std::scoped_lock lock(m1, m2);  // Deadlock-free
```

---

## Quick Debugging Checklist

```
Before Code Review:
☐ Compiles with -Wall -Wextra -Werror
☐ No memory leaks (valgrind)
☐ All tests pass
☐ const-correctness verified
☐ Move semantics used where appropriate
☐ Smart pointers used (no raw new/delete)
☐ RAII for all resource management
☐ Virtual destructor in base classes
☐ override keyword used
☐ No iterator invalidation issues
☐ Thread-safe (if concurrent)
☐ Exception-safe (RAII)
☐ No unnecessary copies
☐ Proper use of auto
☐ Range-based for where possible
☐ Algorithms instead of raw loops
```

---

## Container Operation Cheat Sheet

```cpp
// Vector
v.push_back(x);          // Add to end - O(1) amortized
v.pop_back();            // Remove from end - O(1)
v.insert(it, x);         // Insert at position - O(n)
v.erase(it);             // Erase at position - O(n)
v.reserve(n);            // Reserve capacity - avoid reallocations
v.shrink_to_fit();       // Reduce capacity to size
v[i]                     // Access by index - O(1)

// Deque
d.push_back(x);          // Add to end - O(1)
d.push_front(x);         // Add to front - O(1)
d.pop_back();            // Remove from end - O(1)
d.pop_front();           // Remove from front - O(1)

// List
l.push_back(x);          // Add to end - O(1)
l.push_front(x);         // Add to front - O(1)
l.insert(it, x);         // Insert at position - O(1) if you have iterator
l.erase(it);             // Erase at position - O(1)
l.remove(x);             // Remove all x - O(n)
l.sort();                // Sort list - O(n log n)
l.unique();              // Remove consecutive duplicates - O(n)
l.splice(it, other);     // Move elements from other - O(1) or O(n)

// Set/Map
s.insert(x);             // Insert - O(log n)
s.erase(x);              // Erase - O(log n)
s.find(x);               // Find - O(log n)
s.count(x);              // Count - O(log n)
s.lower_bound(x);        // First >= x - O(log n)
s.upper_bound(x);        // First > x - O(log n)

// Unordered Set/Map
us.insert(x);            // Insert - O(1) average
us.erase(x);             // Erase - O(1) average
us.find(x);              // Find - O(1) average
us.count(x);             // Count - O(1) average
us.bucket_count();       // Number of buckets
us.load_factor();        // Elements / buckets
us.rehash(n);            // Set bucket count to >= n

// Stack
stk.push(x);             // Push - O(1)
stk.pop();               // Pop - O(1)
stk.top();               // Top element - O(1)

// Queue
q.push(x);               // Push to back - O(1)
q.pop();                 // Pop from front - O(1)
q.front();               // Front element - O(1)
q.back();                // Back element - O(1)

// Priority Queue
pq.push(x);              // Push - O(log n)
pq.pop();                // Pop - O(log n)
pq.top();                // Top element - O(1)

// String
s.append(str);           // Append - O(n)
s.substr(pos, len);      // Substring - O(n)
s.find(str);             // Find - O(n*m)
s.replace(pos, len, str);// Replace - O(n)
s += str;                // Concatenate - O(n)
```

---

## Common Type Traits

```cpp
// Type checks
std::is_integral_v<T>
std::is_floating_point_v<T>
std::is_arithmetic_v<T>
std::is_pointer_v<T>
std::is_reference_v<T>
std::is_const_v<T>
std::is_class_v<T>
std::is_enum_v<T>
std::is_function_v<T>
std::is_array_v<T>

// Type properties
std::is_trivial_v<T>
std::is_standard_layout_v<T>
std::is_pod_v<T>
std::is_empty_v<T>
std::is_polymorphic_v<T>
std::is_abstract_v<T>
std::is_final_v<T>

// Type relationships
std::is_same_v<T, U>
std::is_base_of_v<Base, Derived>
std::is_convertible_v<From, To>

// Type transformations
std::remove_const_t<T>
std::remove_reference_t<T>
std::remove_cv_t<T>
std::add_const_t<T>
std::add_pointer_t<T>
std::decay_t<T>
std::underlying_type_t<Enum>

// Conditional
std::conditional_t<condition, T, U>
std::enable_if_t<condition, T>
```

---

## Lambda Syntax Quick Reference

```cpp
// Basic lambda
auto lambda = []() { return 42; };

// With parameters
auto add = [](int a, int b) { return a + b; };

// Capture by value
int x = 10;
auto f1 = [x]() { return x; };

// Capture by reference
auto f2 = [&x]() { x++; };

// Capture all by value
auto f3 = [=]() { return x + y; };

// Capture all by reference
auto f4 = [&]() { x++; y++; };

// Mixed capture
auto f5 = [&, x]() { y++; return x; };  // x by value, rest by ref
auto f6 = [=, &x]() { x++; return y; }; // x by ref, rest by value

// Capture with initialization (C++14)
auto f7 = [x = getValue()]() { return x; };
auto f8 = [ptr = std::make_unique<int>(42)]() { return *ptr; };

// Mutable lambda
auto f9 = [x]() mutable { return ++x; };

// Generic lambda (C++14)
auto f10 = [](auto x, auto y) { return x + y; };

// Template lambda (C++20)
auto f11 = []<typename T>(T x, T y) { return x + y; };

// Return type specification
auto f12 = [](int x) -> double { return x * 1.5; };

// Trailing return type with decltype
auto f13 = [](auto x, auto y) -> decltype(x + y) { return x + y; };

// Immediate invocation
int result = []() { return 42; }();

// In algorithms
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
std::transform(v.begin(), v.end(), v.begin(), [](int x) { return x * 2; });
```

---

## Smart Pointer Quick Reference

```cpp
// unique_ptr - exclusive ownership
std::unique_ptr<T> ptr = std::make_unique<T>(args);
std::unique_ptr<T> ptr(new T(args));  // Less preferred
ptr.reset(new T(args));               // Reset with new object
ptr.reset();                          // Delete and set to nullptr
T* raw = ptr.get();                   // Get raw pointer (doesn't transfer ownership)
T* raw = ptr.release();               // Release ownership (must delete manually)
auto ptr2 = std::move(ptr);           // Transfer ownership

// shared_ptr - shared ownership
std::shared_ptr<T> ptr = std::make_shared<T>(args);
std::shared_ptr<T> ptr(new T(args));  // Less preferred (2 allocations)
std::shared_ptr<T> ptr2 = ptr;        // Both own the object
long count = ptr.use_count();         // Reference count
ptr.reset();                          // Decrement refcount

// weak_ptr - non-owning reference
std::weak_ptr<T> weak = shared;
if (auto ptr = weak.lock()) {         // Try to get shared_ptr
    // Use ptr - object still exists
}
bool expired = weak.expired();        // Check if object deleted

// Custom deleter
auto deleter = [](T* ptr) { 
    custom_cleanup(ptr);
    delete ptr;
};
std::unique_ptr<T, decltype(deleter)> ptr(new T, deleter);
std::shared_ptr<T> ptr2(new T, deleter);

// Arrays
std::unique_ptr<T[]> arr = std::make_unique<T[]>(size);
arr[0] = value;  // Operator[] available

std::shared_ptr<T[]> arr2(new T[size]);  // C++17
// Or better: use std::vector
```

---

## Commonly Used Utilities

```cpp
// std::optional
std::optional<int> opt = getValue();
if (opt.has_value()) {
    int x = *opt;
    int y = opt.value();  // Throws if empty
}
int z = opt.value_or(42);  // Default if empty

// std::variant
std::variant<int, double, std::string> var = 42;
std::holds_alternative<int>(var);  // true
int x = std::get<int>(var);        // Get by type
int y = std::get<0>(var);          // Get by index
std::visit([](auto&& arg) {        // Visit with lambda
    std::cout << arg;
}, var);

// std::any
std::any a = 42;
a = std::string("hello");
if (a.has_value()) {
    std::string s = std::any_cast<std::string>(a);
}

// std::tuple
std::tuple<int, double, std::string> t{1, 3.14, "hello"};
int x = std::get<0>(t);
auto [a, b, c] = t;  // Structured binding

// std::pair
std::pair<int, std::string> p{42, "hello"};
int x = p.first;
std::string s = p.second;
auto [key, value] = p;  // Structured binding

// std::tie (for unpacking)
int a; double b;
std::tie(a, b) = std::tuple{1, 3.14};
std::tie(a, std::ignore, b) = std::tuple{1, 2, 3};  // Ignore middle

// std::exchange
int x = 5;
int old = std::exchange(x, 10);  // x = 10, old = 5

// std::swap
std::swap(a, b);
```

---

## String Operations Quick Reference

```cpp
// Construction
std::string s1 = "hello";
std::string s2(10, 'x');           // "xxxxxxxxxx"
std::string s3(s1, 1, 3);          // "ell" (from pos 1, length 3)

// Concatenation
s = s1 + s2;
s = s1 + " world";
s += "!";
s.append("text");

// Substring
std::string sub = s.substr(0, 5);  // From pos 0, length 5
std::string sub = s.substr(5);     // From pos 5 to end

// Find
size_t pos = s.find("world");      // Returns pos or npos
size_t pos = s.find('o');
size_t pos = s.find("world", 5);   // Start from pos 5
size_t pos = s.rfind("o");         // Reverse find

// Replace
s.replace(0, 5, "hi");             // Replace first 5 chars
s.replace(pos, len, "new");

// Insert/Erase
s.insert(5, "text");
s.erase(5, 4);                     // Erase 4 chars from pos 5

// Comparison
if (s1 == s2) {}
if (s1 < s2) {}
int cmp = s1.compare(s2);          // -1, 0, or 1

// Convert
int i = std::stoi(s);
double d = std::stod(s);
long l = std::stol(s);
std::string s = std::to_string(42);

// Format (C++20)
std::string s = std::format("Hello, {}!", name);
std::string s = std::format("{0} {1} {0}", "hello", "world");

// string_view (non-owning)
std::string_view sv = s;
sv.remove_prefix(2);               // Skip first 2 chars
sv.remove_suffix(2);               // Skip last 2 chars
```

---

## Iterator Utilities

```cpp
// Advance iterator
std::advance(it, n);               // Move iterator by n positions

// Distance between iterators
auto dist = std::distance(first, last);

// Next/Prev (doesn't modify original)
auto next_it = std::next(it, 3);   // 3 positions ahead
auto prev_it = std::prev(it, 2);   // 2 positions back

// Iterator categories
std::input_iterator_tag
std::output_iterator_tag
std::forward_iterator_tag
std::bidirectional_iterator_tag
std::random_access_iterator_tag
std::contiguous_iterator_tag       // C++20

// Iterator adaptors
std::back_inserter(container);     // Insert at back
std::front_inserter(container);    // Insert at front
std::inserter(container, it);      // Insert at position
std::reverse_iterator(it);         // Reverse iteration
std::move_iterator(it);            // Move elements

// Using iterator adaptors
std::copy(v1.begin(), v1.end(), std::back_inserter(v2));
```

---

## Memory Management Quick Tips

```cpp
// Alignment
alignas(16) int x;                 // Align to 16 bytes
size_t align = alignof(T);         // Get alignment

// Placement new
void* mem = std::malloc(sizeof(T));
T* ptr = new (mem) T(args);        // Construct at specific address
ptr->~T();                         // Must manually destroy
std::free(mem);

// Allocation/Deallocation
T* ptr = new T;                    // ✗ Prefer smart pointers
T* arr = new T[n];
delete ptr;
delete[] arr;

// Memory operations
std::memcpy(dest, src, n);         // Copy memory
std::memmove(dest, src, n);        // Copy (handles overlap)
std::memset(ptr, value, n);        // Set memory
std::memcmp(p1, p2, n);            // Compare memory

// std::byte (C++17)
std::byte b{42};
b <<= 2;
int x = std::to_integer<int>(b);
```

---

## Concurrency Quick Reference

```cpp
// Thread creation
std::thread t(func, arg1, arg2);
std::thread t([](){ /* lambda */ });
t.join();                          // Wait for completion
t.detach();                        // Detach from thread

// jthread (C++20) - auto-joins
std::jthread t(func);

// Mutex
std::mutex mtx;
mtx.lock();
// critical section
mtx.unlock();

// Lock guards (RAII)
std::lock_guard<std::mutex> lock(mtx);
std::unique_lock<std::mutex> lock(mtx);  // Can unlock before scope end
lock.unlock();

// Multiple locks
std::scoped_lock lock(mtx1, mtx2);  // Deadlock-free

// Atomic
std::atomic<int> counter{0};
counter++;                         // Atomic increment
int val = counter.load();
counter.store(42);
int old = counter.exchange(42);
bool success = counter.compare_exchange_strong(expected, desired);

// Condition variable
std::condition_variable cv;
std::mutex mtx;

// Wait
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [&]{ return ready; });

// Notify
cv.notify_one();
cv.notify_all();

// Future/Promise
std::promise<int> prom;
std::future<int> fut = prom.get_future();
prom.set_value(42);
int result = fut.get();

// Async
auto fut = std::async(std::launch::async, func, args);
int result = fut.get();
```

---

## Exception Handling

```cpp
// Try-catch
try {
    // code that might throw
} catch (const SpecificException& e) {
    // handle specific
} catch (const std::exception& e) {
    // handle standard
} catch (...) {
    // handle any
}

// Throw
throw std::runtime_error("Error message");
throw std::invalid_argument("Bad argument");
throw std::out_of_range("Out of range");

// Custom exception
class MyException : public std::exception {
    const char* what() const noexcept override {
        return "My error";
    }
};

// noexcept
void func() noexcept;              // Guarantees no throw
void func() noexcept(expr);        // Conditionally noexcept

// Exception safety guarantees
// 1. No-throw (noexcept)
// 2. Strong (commit or rollback)
// 3. Basic (no leaks, valid state)
// 4. No guarantee

// RAII ensures exception safety
{
    std::lock_guard lock(mtx);     // Released on exception
    Resource r;                    // Cleaned up on exception
    // ...
}  // All cleaned up even if exception thrown
```

---

## File I/O Quick Reference

```cpp
// Reading file
std::ifstream file("path.txt");
if (file) {
    std::string line;
    while (std::getline(file, line)) {
        // process line
    }
}

// Writing file
std::ofstream file("path.txt");
file << "Hello, World!\n";

// Reading/Writing
std::fstream file("path.txt", std::ios::in | std::ios::out);

// Binary I/O
std::ofstream file("data.bin", std::ios::binary);
file.write(reinterpret_cast<const char*>(&data), sizeof(data));

std::ifstream file("data.bin", std::ios::binary);
file.read(reinterpret_cast<char*>(&data), sizeof(data));

// Filesystem (C++17)
namespace fs = std::filesystem;

// Check existence
if (fs::exists(path)) {}

// Create directory
fs::create_directory(path);
fs::create_directories(path);  // With parents

// Remove
fs::remove(path);
fs::remove_all(path);          // Recursive

// Copy
fs::copy(src, dest);

// Iterate directory
for (auto& entry : fs::directory_iterator(path)) {
    std::cout << entry.path() << "\n";
}

// Recursive iteration
for (auto& entry : fs::recursive_directory_iterator(path)) {
    std::cout << entry.path() << "\n";
}

// File size
size_t size = fs::file_size(path);

// Path operations
fs::path p = "/path/to/file.txt";
p.filename();                  // "file.txt"
p.extension();                 // ".txt"
p.stem();                      // "file"
p.parent_path();               // "/path/to"
p.replace_extension(".md");
```

---

## Chrono Quick Reference

```cpp
// Time point
auto now = std::chrono::system_clock::now();

// Duration
std::chrono::seconds sec(60);
std::chrono::milliseconds ms(1000);
std::chrono::hours h(1);

// Duration cast
auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(sec);

// Timing
auto start = std::chrono::high_resolution_clock::now();
// ... code to measure ...
auto end = std::chrono::high_resolution_clock::now();
auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
std::cout << duration.count() << " μs\n";

// Sleep
std::this_thread::sleep_for(std::chrono::seconds(1));
std::this_thread::sleep_until(time_point);

// Calendar (C++20)
auto today = std::chrono::year_month_day{std::chrono::floor<std::chrono::days>(now)};
```

---

## Regex Quick Reference

```cpp
#include <regex>

// Match
std::regex pattern(R"(\d+)");
if (std::regex_match(str, pattern)) {}

// Search
std::smatch matches;
if (std::regex_search(str, matches, pattern)) {
    std::string match = matches[0];
}

// Replace
std::string result = std::regex_replace(str, pattern, "replacement");

// Iterator
std::sregex_iterator it(str.begin(), str.end(), pattern);
std::sregex_iterator end;
for (; it != end; ++it) {
    std::cout << it->str() << "\n";
}

// Common patterns
R"(\d+)"              // Digits
R"([a-zA-Z]+)"        // Letters
R"(\w+)"              // Word characters
R"(\s+)"              // Whitespace
R"([^abc])"           // Not a, b, or c
R"(^...)"             // Start of string
R"(...$)"             // End of string
```

---

## Quick Index

**Containers:** vector, deque, list, array, set, map, unordered_set, unordered_map
**Algorithms:** sort, find, copy, transform, accumulate, remove, unique
**Iterators:** begin, end, advance, next, distance
**Smart Pointers:** unique_ptr, shared_ptr, weak_ptr
**Utilities:** optional, variant, any, tuple, pair
**Modern:** ranges, concepts, coroutines, span, format
**Concurrency:** thread, mutex, atomic, future, async
**Patterns:** RAII, Factory, Observer, Strategy, Singleton

---

## See Also — Full Chapter Index

**Foundations & Containers**
- [00 · OOP Concepts](00_oop_concepts.md) — classes, inheritance, polymorphism
- [01 · Sequence Containers](01_sequence_containers.md) — vector, deque, list, array
- [02 · Associative Containers](02_associative_containers.md) — set, map (ordered)
- [03 · Unordered Containers](03_unordered_containers.md) — hash-based set/map
- [04 · Container Adaptors](04_container_adaptors.md) — stack, queue, priority_queue
- [05 · Iterators](05_iterators.md) — categories & invalidation
- [06 · Algorithms](06_algorithms.md) — sort, find, transform, numeric
- [07 · Modern Features](07_modern_features.md) — ranges, views, concepts, span
- [08 · Utility Containers](08_utility_containers.md) — string, optional, variant, tuple

**Generic & Advanced C++**
- [09 · Templates](09_templates.md) — generic programming
- [10 · Lambdas](10_lambdas.md) — closures & functional patterns
- [11 · Metaprogramming](11_metaprogramming.md) — constexpr, type traits
- [12 · Advanced Features](12_advanced_features.md) — move semantics, smart pointers, RAII
- [13 · Best Practices](13_best_practices.md) — idioms & anti-patterns

**Concurrency, I/O & Systems**
- [14 · Multithreading](14_multithreading.md) — threads, mutexes, atomics
- [15 · Async & Futures](15_async_futures.md) — async, promise/future, parallelism
- [16 · I/O & Filesystem](16_io_filesystem.md) — streams, filesystem, format/print
- [17 · Exceptions](17_exceptions.md) — safety guarantees, expected
- [18 · Time & Chrono](18_time_chrono.md) — durations, clocks, calendars
- [19 · Memory & Allocators](19_memory_allocators.md) — PMR, alignment, pools
- [20 · Regex](20_regex.md) — pattern matching
- [21 · Modules](21_modules.md) — C++20 modules
- [22 · Coroutines](22_coroutines.md) — co_await/co_yield/co_return

> Tip: the [Container Selection Guide](#container-selection-guide) and
> [Time Complexity table](#time-complexity-quick-reference) above are the two
> fastest ways to pick the right container.

---

*Quick Reference Guide — Bookmark this page for fast lookup during development*


