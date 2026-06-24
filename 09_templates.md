# Templates

## Overview

Templates are the foundation of generic programming in C++. They allow you to write code once and use it with any type that meets the requirements. C++20 [concepts](07_modern_features.md) and [SFINAE](#sfinae-substitution-failure-is-not-an-error) provide two eras of constraint syntax; deep compile-time computation continues in [Metaprogramming](11_metaprogramming.md).

```
┌──────────────────────────────────────────────────────────┐
│                    TEMPLATE SYSTEM                       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  FUNCTION          CLASS            VARIABLE             │
│  TEMPLATES         TEMPLATES        TEMPLATES (C++14)    │
│  ─────────         ─────────        ─────────            │
│  • Generic funcs   • Generic types  • Generic constants  │
│  • Overloading     • Inheritance    • Compile-time       │
│  • Deduction       • Specialization │                    │
│                                                          │
│  ─────────────────────────────────────────────────────   │
│  VARIADIC          CONCEPTS         CONSTRAINTS          │
│  TEMPLATES         (C++20)          (C++20)              │
│  ─────────         ────────         ───────────          │
│  • Variable args   • Type reqs      • Template reqs      │
│  • Pack expansion  • Clearer errors │                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Function Templates

### Basic Function Template
```cpp
#include <iostream>

// Template declaration
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    // Explicit instantiation
    int i = max<int>(10, 20);           // T = int
    
    // Type deduction (compiler infers T)
    int j = max(10, 20);                // T = int (deduced)
    double d = max(3.14, 2.71);         // T = double (deduced)
    
    // std::string works too (has operator>)
    std::string s = max(std::string("hello"), std::string("world"));
    
    return 0;
}
```

### Visual: Template Instantiation
```
Template Definition:
┌────────────────────────────────┐
│ template<typename T>           │
│ T max(T a, T b) {              │
│     return (a > b) ? a : b;    │
│ }                              │
└────────────────────────────────┘
        │
        │ Compiler instantiates for each type used
        ├──────────────┬──────────────┬──────────────┐
        ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ int max(    │ │ double max( │ │ string max( │ │ char max(   │
│   int,int)  │ │   double,   │ │   string,   │ │   char,char)│
│             │ │   double)   │ │   string)   │ │             │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

Each instantiation is a separate function!
```

### Multiple Template Parameters
```cpp
// C++11: trailing return type with decltype is needed because the return
// type depends on the parameters.
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}

// C++14 lets you drop the trailing return type (return-type deduction).
// This is the SAME signature as the version above, so the two cannot
// coexist in one program — pick one. Shown here only as the C++14 form:
//
//   template<typename T, typename U>
//   auto add(T a, U b) { return a + b; }

int main() {
    auto result1 = add(5, 3.14);        // int + double -> double
    auto result2 = add(1.5, 2);         // double + int -> double
    
    return 0;
}
```

### Template Specialization (Function Templates)

Function templates support **full** specialization only — there is no partial specialization for function templates. To customize behavior for a family of types (e.g. all pointers), use **overloading** instead.

```cpp
#include <cstring>
#include <iostream>
#include <type_traits>

// Primary template
template<typename T>
bool equal(T a, T b) {
    return a == b;
}

// Full specialization for const char*
template<>
bool equal<const char*>(const char* a, const char* b) {
    return std::strcmp(a, b) == 0;
}

// Overload (preferred over specialization when partial-like behavior is needed)
template<typename T>
bool equal(T* a, T* b) {
    return a == b;  // pointer identity
}

int main() {
    bool b1 = equal(5, 5);                // Primary template
    bool b2 = equal("hello", "hello");    // Full specialization
    int x = 1, y = 1;
    bool b3 = equal(&x, &y);              // Overload for pointers
    return 0;
}
```

💡 **Hunch**: Class templates allow both full and partial specialization (see below). When you reach for `template<> void foo<int*>(...)`, consider a separate overload or a constrained template with [concepts](#concepts-c20) instead.

### Overloading vs Specialization
```cpp
// Overload for different parameter types
template<typename T>
void process(T value) {
    std::cout << "Generic: " << value << '\n';
}

void process(int value) {
    std::cout << "Int: " << value << '\n';
}

// Specialization (less preferred, use overloading)
template<typename T>
void handle(T value) {
    std::cout << "Generic\n";
}

template<>
void handle<int>(int value) {
    std::cout << "Specialized for int\n";
}
```

---

## Class Templates

### Basic Class Template
```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elements;
    
public:
    void push(const T& elem) {
        elements.push_back(elem);
    }
    
    void pop() {
        if (elements.empty()) {
            throw std::out_of_range("Stack empty");
        }
        elements.pop_back();
    }
    
    T& top() {
        if (elements.empty()) {
            throw std::out_of_range("Stack empty");
        }
        return elements.back();
    }
    
    bool empty() const {
        return elements.empty();
    }
};

int main() {
    Stack<int> int_stack;
    int_stack.push(42);
    int_stack.push(100);
    
    Stack<std::string> string_stack;
    string_stack.push("Hello");
    string_stack.push("World");
    
    return 0;
}
```

### Multiple Template Parameters
```cpp
template<typename Key, typename Value>
class KeyValuePair {
private:
    Key key_;
    Value value_;
    
public:
    KeyValuePair(const Key& k, const Value& v) 
        : key_(k), value_(v) {}
    
    const Key& key() const { return key_; }
    const Value& value() const { return value_; }
    
    void set_value(const Value& v) { value_ = v; }
};

int main() {
    KeyValuePair<std::string, int> age("Alice", 25);
    KeyValuePair<int, double> ratio(42, 3.14);
    
    return 0;
}
```

### Default Template Arguments
```cpp
template<typename T, typename Container = std::vector<T>>
class Stack {
private:
    Container elements;
    
public:
    void push(const T& elem) {
        elements.push_back(elem);
    }
    // ... other methods
};

int main() {
    Stack<int> s1;                      // Uses std::vector<int>
    Stack<int, std::deque<int>> s2;     // Uses std::deque<int>
    
    return 0;
}
```

### Non-Type Template Parameters
```cpp
template<typename T, size_t N>
class Array {
private:
    T data_[N];
    
public:
    size_t size() const { return N; }
    
    T& operator[](size_t i) { return data_[i]; }
    const T& operator[](size_t i) const { return data_[i]; }
    
    T* begin() { return data_; }
    T* end() { return data_ + N; }
};

int main() {
    Array<int, 5> arr1;     // Size 5
    Array<int, 10> arr2;    // Size 10
    // arr1 and arr2 are DIFFERENT types!
    
    for (size_t i = 0; i < arr1.size(); ++i) {
        arr1[i] = i;
    }
    
    return 0;
}
```

### Class Template Specialization

Class templates support **full** specialization (concrete type list) and **partial** specialization (pattern over template parameters, e.g. `T*`).
```cpp
// Primary template
template<typename T>
class Storage {
private:
    T data_;
public:
    Storage(T data) : data_(data) {}
    T get() const { return data_; }
};

// Full specialization for bool
template<>
class Storage<bool> {
private:
    unsigned char data_;  // Use char instead of bool
public:
    Storage(bool data) : data_(data ? 1 : 0) {}
    bool get() const { return data_ != 0; }
};

// Partial specialization for pointers
template<typename T>
class Storage<T*> {
private:
    T* data_;
public:
    Storage(T* data) : data_(data) {}
    T* get() const { return data_; }
    T& operator*() { return *data_; }
};
```

### Visual: Template Specialization Hierarchy
```
                Primary Template
              template<typename T>
              class Storage<T>
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
  Storage<int>  Storage<bool>  Storage<int*>
  (primary)     (specialized)  (partial specialization)
```

---

## Template Argument Deduction

### C++17: Class Template Argument Deduction (CTAD)
```cpp
template<typename T>
class Container {
private:
    T value_;
public:
    Container(T v) : value_(v) {}
};

int main() {
    // Before C++17: Must specify type
    Container<int> c1(42);
    
    // C++17: Type deduced from constructor argument
    Container c2(42);           // Container<int>
    Container c3(3.14);         // Container<double>
    Container c4("hello");      // Container<const char*>
    
    // With std containers
    std::vector v{1, 2, 3};     // std::vector<int>
    std::pair p{1, 3.14};       // std::pair<int, double>
    
    return 0;
}
```

### Deduction Guides (C++17)
```cpp
template<typename T>
class Container {
    T value_;
public:
    Container(T v) : value_(v) {}
};

// Custom deduction guide
template<typename T>
Container(T) -> Container<T>;

// For arrays
template<typename T, size_t N>
Container(T(&)[N]) -> Container<std::vector<T>>;

int main() {
    Container c1(42);           // Container<int>
    
    int arr[] = {1, 2, 3};
    Container c2(arr);          // Container<std::vector<int>>
    
    return 0;
}
```

---

## Variadic Templates (C++11)

### Basic Variadic Template
```cpp
#include <iostream>

// Base case: no arguments
void print() {
    std::cout << '\n';
}

// Recursive case: at least one argument
template<typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << ' ';
    print(rest...);  // Recursive call with remaining args
}

int main() {
    print(1, 2, 3, 4, 5);
    print("Hello", 42, 3.14, "World");
    
    return 0;
}
```

### Visual: Variadic Template Recursion
```
print(1, 2, 3)
│
├─ first = 1, rest = {2, 3}
│  Output: "1 "
│  └─ print(2, 3)
│     │
│     ├─ first = 2, rest = {3}
│     │  Output: "2 "
│     │  └─ print(3)
│     │     │
│     │     ├─ first = 3, rest = {}
│     │     │  Output: "3 "
│     │     │  └─ print()  // Base case
│     │     │     Output: "\n"
│
Final output: "1 2 3 \n"
```

### Parameter Pack Expansion
```cpp
#include <iostream>

// sizeof... operator: number of arguments
template<typename... Args>
size_t count(Args... args) {
    return sizeof...(Args);  // or sizeof...(args)
}

// Fold expressions (C++17)
template<typename... Args>
auto sum(Args... args) {
    return (... + args);  // Left fold: ((a + b) + c) + d
}

template<typename... Args>
auto sum_right(Args... args) {
    return (args + ...);  // Right fold: a + (b + (c + d))
}

// Print all with fold (C++17)
template<typename... Args>
void print_fold(Args... args) {
    ((std::cout << args << ' '), ...);
    std::cout << '\n';
}

int main() {
    std::cout << count(1, 2, 3, 4) << '\n';  // 4
    std::cout << sum(1, 2, 3, 4, 5) << '\n'; // 15
    
    print_fold(1, 2, 3, "hello", 3.14);
    
    return 0;
}
```

### Fold Expressions (C++17)
```
┌────────────────────────────────────────────────────────┐
│                  FOLD EXPRESSIONS                      │
├────────────────────────────────────────────────────────┤
│ Type         │ Expression      │ Expansion            │
├──────────────┼─────────────────┼──────────────────────┤
│ Unary right  │ (args op ...)   │ a op (b op (c op d)) │
│ Unary left   │ (... op args)   │ ((a op b) op c) op d │
│ Binary right │ (args op ... op init) │ a op (b op (c op (d op init))) │
│ Binary left  │ (init op ... op args) │ ((((init op a) op b) op c) op d) │
└────────────────────────────────────────────────────────┘

Supported operators:
+ - * / % ^ & | << >> += -= *= /= %= ^= &= |= <<= >>=
== != < > <= >= && || , .* ->*
```

### Variadic Class Template
```cpp
template<typename... Types>
class Tuple;

// Base case: empty tuple
template<>
class Tuple<> {};

// Recursive case
template<typename Head, typename... Tail>
class Tuple<Head, Tail...> : private Tuple<Tail...> {
    Head head_;
public:
    Tuple() = default;
    Tuple(Head h, Tail... t) 
        : Tuple<Tail...>(t...), head_(h) {}
    
    Head& head() { return head_; }
    Tuple<Tail...>& tail() { return *this; }
};

int main() {
    Tuple<int, double, std::string> t(42, 3.14, "hello");
    
    return 0;
}
```

### Perfect Forwarding with Variadic Templates

Preserve value category when forwarding arguments into constructors — covered in depth in [Advanced Features](12_advanced_features.md) (forwarding references, reference collapsing). The variadic pattern is the mechanism `std::make_unique`, `std::emplace`, and `std::vector::emplace_back` use internally.

```cpp
#include <memory>
#include <utility>

template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

struct Point {
    int x, y;
    Point(int x, int y) : x(x), y(y) {}
};

int main() {
    auto p = make_unique<Point>(10, 20);
    // Forwards arguments to Point constructor
    
    return 0;
}
```

---

## SFINAE (Substitution Failure Is Not An Error)

SFINAE is the pre-C++20 technique for constraining templates: if substituting template arguments makes an expression ill-formed, that overload is **removed** from the overload set rather than causing a hard error. [Concepts](#concepts-c20) (C++20) express the same intent with clearer syntax and diagnostics — prefer concepts in new code; SFINAE remains common in legacy libraries and `std::enable_if` traits.

### Basic SFINAE
```cpp
#include <type_traits>

// Enabled only if T is integral
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
increment(T value) {
    return value + 1;
}

// Enabled only if T is floating point
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
increment(T value) {
    return value + 0.5;
}

int main() {
    int i = increment(5);       // Calls integral version
    double d = increment(3.14); // Calls floating point version
    
    return 0;
}
```

### C++14: std::enable_if_t
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
increment(T value) {
    return value + 1;
}
```

### Visual: SFINAE Process
```
Template Instantiation Attempt:
┌────────────────────────────────────┐
│ increment<int>(5)                  │
└────────────────────────────────────┘
        │
        ├─► Try Template 1 (integral)
        │   std::is_integral<int>::value = true
        │   ✓ SUCCESS → Use this template
        │
        └─► Try Template 2 (floating point)
            std::is_floating_point<int>::value = false
            ✗ SUBSTITUTION FAILURE
            (Not an error! Just skip this template)

Result: Template 1 is selected
```

### Tag Dispatching (Alternative to SFINAE)
```cpp
// Tags for dispatch
struct integral_tag {};
struct floating_point_tag {};

// Implementation functions
template<typename T>
T increment_impl(T value, integral_tag) {
    return value + 1;
}

template<typename T>
T increment_impl(T value, floating_point_tag) {
    return value + 0.5;
}

// Dispatcher
template<typename T>
T increment(T value) {
    using tag = std::conditional_t<
        std::is_integral_v<T>,
        integral_tag,
        floating_point_tag
    >;
    return increment_impl(value, tag{});
}
```

---

## Concepts (C++20)

### What are Concepts?

Concepts provide a way to specify constraints on template parameters, giving clearer errors and better documentation. They largely replace SFINAE for new code. See also [Modern C++20/23 Features](07_modern_features.md) for concepts used with ranges.

```cpp
#include <concepts>
#include <iostream>

// Define a concept with 'concept' keyword
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

// Abbreviated template parameter: template<Numeric T> ≡ template<typename T> requires Numeric<T>
template<Numeric T>
T add(T a, T b) {
    return a + b;
}

// requires clause before function body
template<typename T>
requires Numeric<T>
T multiply(T a, T b) {
    return a * b;
}

// Trailing requires on return type
template<typename T>
T divide(T a, T b) requires Numeric<T> {
    return a / b;
}

// Standard library constraint — no custom concept needed
template<std::integral T>
T double_int(T x) { return x * 2; }

// Constrained abbreviated function template parameter
void print_integral(std::integral auto value) {
    std::cout << value << '\n';
}

int main() {
    auto sum = add(5, 3);            // OK
    auto prod = multiply(3.14, 2.0); // OK
    print_integral(42);
    // auto bad = add("hello", "world");  // ERROR: does not satisfy Numeric
    return 0;
}
```

### requires Expressions

A `requires` expression checks that types support specific operations — the building block inside `concept` definitions:

```cpp
#include <concepts>
#include <iterator>

template<typename T>
concept HasSize = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;  // valid expression + return-type constraint
};

template<typename T>
concept InputIter = requires(T it) {
    typename std::iterator_traits<T>::value_type;
    { *it } -> std::convertible_to<typename std::iterator_traits<T>::value_type>;
    ++it;
};
```

### Standard Concepts
```cpp
#include <concepts>
#include <string>
#include <vector>

// Library concepts from <concepts> and related headers:
template<typename T>
void demo() {
    static_assert(std::integral<int>);
    static_assert(std::floating_point<double>);
    static_assert(std::signed_integral<int>);
    static_assert(std::unsigned_integral<unsigned>);
    
    static_assert(std::same_as<int, int>);
    static_assert(std::convertible_to<int, double>);
    static_assert(std::derived_from<std::string, std::basic_string<char>>);
    
    static_assert(std::default_initializable<int>);
    static_assert(std::move_constructible<std::string>);
    static_assert(std::copy_constructible<std::vector<int>>);
}
```

### Custom Concepts
```cpp
#include <concepts>
#include <vector>

// Concept: Type must support addition
template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

// Concept: Type must be a container
template<typename T>
concept Container = requires(T t) {
    typename T::value_type;
    typename T::iterator;
    { t.begin() } -> std::same_as<typename T::iterator>;
    { t.end() } -> std::same_as<typename T::iterator>;
    { t.size() } -> std::convertible_to<std::size_t>;
};

// Using custom concept
template<Container C>
void process(const C& container) {
    for (const auto& elem : container) {
        // Process element
    }
}

int main() {
    std::vector<int> vec = {1, 2, 3};
    process(vec);  // OK: vector satisfies Container
    
    return 0;
}
```

### Concepts vs SFINAE
```
┌────────────────────────────────────────────────────────┐
│              Concepts vs SFINAE                        │
├────────────────────────────────────────────────────────┤
│                SFINAE          │    Concepts (C++20)   │
├────────────────────────────────┼───────────────────────┤
│ Syntax          Complex        │    Clean              │
│ Error messages  Cryptic        │    Clear              │
│ Composition     Difficult      │    Easy               │
│ Readability     Poor           │    Excellent          │
│ Performance     Same           │    Same               │
└────────────────────────────────────────────────────────┘

Example error message:

SFINAE:
  error: no matching function for call to 'add(std::string, std::string)'
  note: candidate template ignored: substitution failure
  [50 lines of template instantiation backtrace]

Concepts:          error: 'add<std::string>' does not satisfy concept 'Numeric'
  note: std::string is not arithmetic
```

For heavier compile-time computation (type lists, `constexpr` algorithms on types), see [Metaprogramming](11_metaprogramming.md).

---

## Template Metaprogramming Basics

Introductory compile-time patterns; advanced TMP in [Metaprogramming](11_metaprogramming.md).

### Compile-Time Computation
```cpp
// Compile-time factorial
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

int main() {
    constexpr int result = Factorial<5>::value;  // 120, computed at compile time
    
    return 0;
}
```

### Type Traits
```cpp
#include <type_traits>

template<typename T>
void analyze_type() {
    std::cout << "Is integral: " << std::is_integral_v<T> << '\n';
    std::cout << "Is pointer: " << std::is_pointer_v<T> << '\n';
    std::cout << "Is const: " << std::is_const_v<T> << '\n';
    std::cout << "Size: " << sizeof(T) << '\n';
}

// Remove const/reference
template<typename T>
using bare_type = std::remove_cv_t<std::remove_reference_t<T>>;

int main() {
    analyze_type<int>();
    analyze_type<const int*>();
    
    using type1 = bare_type<const int&>;  // int
    
    return 0;
}
```

### Conditional Types
```cpp
#include <type_traits>

// Choose type based on condition
template<bool Condition, typename T, typename F>
using conditional_type = typename std::conditional<Condition, T, F>::type;

template<typename T>
using storage_type = conditional_type<
    sizeof(T) <= 8,
    T,              // Small: store directly
    T*              // Large: store pointer
>;

int main() {
    storage_type<int> s1;     // int (small)
    storage_type<std::array<int, 1000>> s2;  // std::array<int,1000>* (large)
    
    return 0;
}
```

---

## Template Best Practices

### 1. Use Concepts (C++20)
```cpp
// GOOD: Clear intent
template<std::integral T>
T add(T a, T b) { return a + b; }

// BAD: Unclear constraints
template<typename T>
T add(T a, T b) { return a + b; }
```

### 2. Provide Type Aliases
```cpp
template<typename T>
class Container {
public:
    using value_type = T;
    using iterator = T*;
    using const_iterator = const T*;
    using size_type = std::size_t;
};
```

### 3. Forward Declarations
```cpp
// Header: Forward declare
template<typename T>
class Container;

// Later: Define
template<typename T>
class Container {
    // Implementation
};
```

### 4. Explicit Instantiation (Control Code Bloat)
```cpp
// In .cpp file
template class Container<int>;
template class Container<double>;
// Only these types will be instantiated
```

---

## Common Patterns

### CRTP (Curiously Recurring Template Pattern)
```cpp
// Base class with interface
template<typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

// Derived class
class Derived : public Base<Derived> {
public:
    void implementation() {
        std::cout << "Derived implementation\n";
    }
};

int main() {
    Derived d;
    d.interface();  // Calls Derived::implementation via static polymorphism
    
    return 0;
}
```

### Type Erasure
```cpp
#include <memory>

class Any {
    struct Concept {
        virtual ~Concept() = default;
        virtual void print() const = 0;
    };
    
    template<typename T>
    struct Model : Concept {
        T data;
        Model(T d) : data(std::move(d)) {}
        void print() const override {
            std::cout << data << '\n';
        }
    };
    
    std::unique_ptr<Concept> impl_;
    
public:
    template<typename T>
    Any(T data) : impl_(std::make_unique<Model<T>>(std::move(data))) {}
    
    void print() const { impl_->print(); }
};

int main() {
    Any a1 = 42;
    Any a2 = std::string("Hello");
    
    a1.print();  // 42
    a2.print();  // Hello
    
    return 0;
}
```

---

## Common Pitfalls

### 1. Template Bloat
```cpp
// BAD: Instantiates for every size
template<typename T, size_t N>
class Array {
    T data[N];
    // ... lots of code ...
};

// GOOD: Extract size-independent code
template<typename T>
class ArrayBase {
    // Size-independent implementation
};

template<typename T, size_t N>
class Array : public ArrayBase<T> {
    T data[N];
};
```

### 2. Dependent Names
```cpp
template<typename T>
class Container {
    typename T::value_type x;  // Need 'typename' for dependent types
    
    void foo() {
        this->template bar<int>();  // Need 'template' keyword
    }
};
```

### 3. Two-Phase Lookup
```cpp
void helper() { std::cout << "Global\n"; }

template<typename T>
void process() {
    helper();  // Which helper? Depends on T!
}

namespace N {
    struct S {};
    void helper() { std::cout << "N::helper\n"; }
}

int main() {
    process<int>();    // Calls global helper
    process<N::S>();   // Calls N::helper (ADL)
    
    return 0;
}
```

---

## Complete Practical Example: Generic Container Library

Here's a comprehensive example integrating function templates, class templates, variadic templates, SFINAE, concepts, and specialization:

```cpp
#include <iostream>
#include <vector>
#include <type_traits>
#include <concepts>
#include <memory>
#include <algorithm>
#include <numeric>

// 1. Concept definitions (C++20)
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<typename T>
concept Printable = requires(T t) {
    { std::cout << t } -> std::same_as<std::ostream&>;
};

template<typename T>
concept Container = requires(T t) {
    typename T::value_type;
    { t.begin() } -> std::same_as<typename T::iterator>;
    { t.end() } -> std::same_as<typename T::iterator>;
    { t.size() } -> std::convertible_to<std::size_t>;
};

// 2. Function template with concepts
template<Numeric T>
T add(T a, T b) {
    return a + b;
}

// 3. Function template with SFINAE (pre-C++20 style)
template<typename T>
typename std::enable_if<std::is_integral_v<T>, T>::type
multiply_by_two(T value) {
    return value * 2;
}

template<typename T>
typename std::enable_if<std::is_floating_point_v<T>, T>::type
multiply_by_two(T value) {
    std::cout << "[float version] ";
    return value * 2.0;
}

// 4. Variadic template - print all arguments
template<typename First>
void print_all(const First& first) {
    std::cout << first;
}

template<typename First, typename... Rest>
void print_all(const First& first, const Rest&... rest) {
    std::cout << first << ", ";
    print_all(rest...);  // Recursive call
}

// 5. Variadic template - sum all arguments
template<Numeric... Args>
auto sum(Args... args) {
    return (args + ...);  // Fold expression (C++17)
}

// 6. Class template - Generic container wrapper
template<typename T, typename Allocator = std::allocator<T>>
class MyContainer {
private:
    std::vector<T, Allocator> data_;
    
public:
    using value_type = T;
    using iterator = typename std::vector<T, Allocator>::iterator;
    using const_iterator = typename std::vector<T, Allocator>::const_iterator;
    
    // Constructor with initializer list
    MyContainer(std::initializer_list<T> init) : data_(init) {}
    
    // Variadic template constructor
    template<typename... Args>
    explicit MyContainer(Args&&... args) {
        (data_.push_back(std::forward<Args>(args)), ...);  // Fold expression
    }
    
    // Template member function
    template<typename Func>
    void for_each(Func func) {
        for (auto& item : data_) {
            func(item);
        }
    }
    
    // Template member function with concept
    template<Numeric U = T>
    U sum() const {
        return std::accumulate(data_.begin(), data_.end(), U{0});
    }
    
    // SFINAE member function - only for numeric types
    template<typename U = T>
    typename std::enable_if<std::is_arithmetic_v<U>, U>::type
    average() const {
        if (data_.empty()) return U{0};
        return sum<U>() / static_cast<U>(data_.size());
    }
    
    void push_back(const T& value) { data_.push_back(value); }
    void push_back(T&& value) { data_.push_back(std::move(value)); }
    
    iterator begin() { return data_.begin(); }
    iterator end() { return data_.end(); }
    const_iterator begin() const { return data_.begin(); }
    const_iterator end() const { return data_.end(); }
    
    size_t size() const { return data_.size(); }
};

// 7. Template specialization for bool
template<>
class MyContainer<bool> {
private:
    std::vector<bool> data_;
    
public:
    MyContainer(std::initializer_list<bool> init) : data_(init) {}
    
    void push_back(bool value) { data_.push_back(value); }
    
    size_t count_true() const {
        return std::count(data_.begin(), data_.end(), true);
    }
    
    size_t size() const { return data_.size(); }
    
    void print() const {
        for (bool b : data_) {
            std::cout << (b ? "true" : "false") << " ";
        }
        std::cout << "\n";
    }
};

// 8. Partial specialization for pointers
template<typename T>
class MyContainer<T*> {
private:
    std::vector<T*> data_;
    
public:
    MyContainer(std::initializer_list<T*> init) : data_(init) {}
    
    void push_back(T* ptr) { data_.push_back(ptr); }
    
    // Dereference all pointers and print
    void print_dereferenced() const {
        for (const auto* ptr : data_) {
            if (ptr) {
                std::cout << *ptr << " ";
            } else {
                std::cout << "nullptr ";
            }
        }
        std::cout << "\n";
    }
    
    size_t size() const { return data_.size(); }
};

// 9. CRTP (Curiously Recurring Template Pattern)
template<typename Derived>
class Comparable {
public:
    bool operator!=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) == other);
    }
    
    bool operator>(const Derived& other) const {
        return other < static_cast<const Derived&>(*this);
    }
    
    bool operator<=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) > other);
    }
    
    bool operator>=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other);
    }
};

struct Point : Comparable<Point> {
    int x, y;
    
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
    
    bool operator<(const Point& other) const {
        return (x < other.x) || (x == other.x && y < other.y);
    }
};

// 10. Type traits and metaprogramming
template<typename T>
struct is_container : std::false_type {};

template<typename T>
struct is_container<std::vector<T>> : std::true_type {};

template<typename T>
struct is_container<MyContainer<T>> : std::true_type {};

template<typename T>
inline constexpr bool is_container_v = is_container<T>::value;

// 11. Variadic template class
template<typename... Types>
class Tuple;

template<>
class Tuple<> {
public:
    static constexpr size_t size() { return 0; }
};

template<typename Head, typename... Tail>
class Tuple<Head, Tail...> : private Tuple<Tail...> {
private:
    Head value_;
    
public:
    Tuple(Head h, Tail... t) : Tuple<Tail...>(t...), value_(h) {}
    
    Head& head() { return value_; }
    const Head& head() const { return value_; }
    
    Tuple<Tail...>& tail() { return *this; }
    const Tuple<Tail...>& tail() const { return *this; }
    
    static constexpr size_t size() { return 1 + Tuple<Tail...>::size(); }
};

// 12. Template template parameter
template<template<typename> class Container, typename T>
void fill_container(Container<T>& container, const T& value, size_t count) {
    for (size_t i = 0; i < count; ++i) {
        container.push_back(value);
    }
}

// 13. Constexpr template function
template<typename T>
constexpr T fibonacci(T n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// 14. Generic algorithm with concepts
template<Container C, typename Func>
void transform_container(C& container, Func func) {
    std::transform(container.begin(), container.end(), 
                  container.begin(), func);
}

int main() {
    std::cout << "=== Templates Demo ===\n\n";
    
    // 1. Concepts
    std::cout << "1. Function template with concepts:\n";
    std::cout << "add(5, 3) = " << add(5, 3) << "\n";
    std::cout << "add(2.5, 1.5) = " << add(2.5, 1.5) << "\n\n";
    
    // 2. SFINAE
    std::cout << "2. SFINAE overloading:\n";
    std::cout << "multiply_by_two(10) = " << multiply_by_two(10) << "\n";
    std::cout << "multiply_by_two(3.5) = " << multiply_by_two(3.5) << "\n\n";
    
    // 3. Variadic templates
    std::cout << "3. Variadic templates:\n";
    std::cout << "print_all: ";
    print_all(1, 2.5, "hello", 'X');
    std::cout << "\nsum(1, 2, 3, 4, 5) = " << sum(1, 2, 3, 4, 5) << "\n\n";
    
    // 4. Class template
    std::cout << "4. Generic container (class template):\n";
    MyContainer<int> container{1, 2, 3, 4, 5};
    std::cout << "Sum: " << container.sum() << "\n";
    std::cout << "Average: " << container.average() << "\n";
    
    container.for_each([](int& x) { x *= 2; });
    std::cout << "After doubling: ";
    for (int x : container) std::cout << x << " ";
    std::cout << "\n\n";
    
    // 5. Template specialization for bool
    std::cout << "5. Specialization for bool:\n";
    MyContainer<bool> bools{true, false, true, true, false};
    std::cout << "Booleans: ";
    bools.print();
    std::cout << "True count: " << bools.count_true() << "\n\n";
    
    // 6. Partial specialization for pointers
    std::cout << "6. Partial specialization for pointers:\n";
    int a = 10, b = 20, c = 30;
    MyContainer<int*> ptrs{&a, &b, &c};
    std::cout << "Dereferenced: ";
    ptrs.print_dereferenced();
    std::cout << "\n";
    
    // 7. CRTP
    std::cout << "7. CRTP (Comparable):\n";
    Point p1{1, 2}, p2{3, 4};
    std::cout << "p1 < p2: " << (p1 < p2) << "\n";
    std::cout << "p1 > p2: " << (p1 > p2) << "\n";
    std::cout << "p1 <= p2: " << (p1 <= p2) << "\n\n";
    
    // 8. Type traits
    std::cout << "8. Type traits:\n";
    std::cout << "is_container<vector<int>>: " 
              << is_container_v<std::vector<int>> << "\n";
    std::cout << "is_container<MyContainer<int>>: " 
              << is_container_v<MyContainer<int>> << "\n";
    std::cout << "is_container<int>: " 
              << is_container_v<int> << "\n\n";
    
    // 9. Variadic class template
    std::cout << "9. Variadic class template (Tuple):\n";
    Tuple<int, double, std::string> tuple(42, 3.14, "Hello");
    std::cout << "Tuple size: " << tuple.size() << "\n";
    std::cout << "Head: " << tuple.head() << "\n\n";
    
    // 10. Template template parameter
    std::cout << "10. Template template parameter:\n";
    MyContainer<int> tmpl_container{};
    fill_container(tmpl_container, 7, 5);
    std::cout << "Filled container: ";
    for (int x : tmpl_container) std::cout << x << " ";
    std::cout << "\n\n";
    
    // 11. Constexpr template
    std::cout << "11. Constexpr template function:\n";
    constexpr int fib10 = fibonacci(10);  // Computed at compile time!
    std::cout << "fibonacci(10) = " << fib10 << "\n\n";
    
    // 12. Generic algorithm with concepts
    std::cout << "12. Generic algorithm with concepts:\n";
    MyContainer<int> nums{1, 2, 3, 4, 5};
    transform_container(nums, [](int x) { return x * x; });
    std::cout << "Squared: ";
    for (int x : nums) std::cout << x << " ";
    std::cout << "\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **Function templates**: Generic functions with type parameters
- **Class templates**: Generic classes
- **Concepts (C++20)**: Constraining template parameters
- **SFINAE**: Enable functions based on type traits
- **Variadic templates**: Templates with variable number of parameters
- **Fold expressions (C++17)**: Expanding parameter packs
- **Template specialization**: Custom behavior for specific types
- **Partial specialization**: Specializing for pointer types
- **CRTP**: Compile-time polymorphism
- **Type traits**: Querying type properties
- **Template template parameters**: Templates taking templates
- **Constexpr templates**: Compile-time computation
- **Perfect forwarding**: `std::forward` with templates
- **Generic algorithms**: Working with any container

This example showcases the power and flexibility of C++ templates!

---

## Related Topics

- [Modern C++20/23 Features](07_modern_features.md) — concepts in practice with ranges, views, and `std::span`.
- [Metaprogramming](11_metaprogramming.md) — advanced TMP: type lists, `constexpr` algorithms, expression templates.
- [Advanced Features](12_advanced_features.md) — perfect forwarding, `std::forward`, forwarding references, move semantics.
- [Utility Containers](08_utility_containers.md) — `optional`, `variant`, `tuple`, and `any` as class templates.
- [Lambdas](10_lambdas.md) — generic lambdas (`auto` parameters) and template-like callables without explicit `template<>`.
- [Iterators](05_iterators.md) — iterator traits and categories that concepts like `std::input_iterator` formalize.
- [Best Practices](13_best_practices.md) — managing template bloat, explicit instantiation, and API design.

---

## Next Steps
- **Next**: [Lambdas and Functional Programming →](10_lambdas.md)
- **Previous**: [← Utility Containers](08_utility_containers.md)

---
*Chapter 9 — Templates*

