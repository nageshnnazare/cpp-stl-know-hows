# Template Metaprogramming

## Overview

Template metaprogramming is compile-time computation using C++ [templates](09_templates.md). Modern C++ favors `constexpr` functions and `if constexpr` over classic recursive templates; [concepts](09_templates.md) (C++20) largely replace [SFINAE](#sfinae-substitution-failure-is-not-an-error) for constraints.

```
┌──────────────────────────────────────────────────────────┐
│          TEMPLATE METAPROGRAMMING TIMELINE               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  C++98/03: Template Metaprogramming discovered           │
│            • Accidental Turing-completeness              │
│            • Complex, hard to read                       │
│                                                          │
│  C++11:    constexpr functions                           │
│            • Compile-time computation made easier        │
│            • Type traits standardized                    │
│                                                          │
│  C++14:    Relaxed constexpr                             │
│            • Loops and branches in constexpr             │
│                                                          │
│  C++17:    if constexpr                                  │
│            • Compile-time branching                      │
│            • Fold expressions                            │
│                                                          │
│  C++20:    Concepts, consteval                           │
│            • Better type constraints                     │
│            • Forced compile-time evaluation              │
│                                                          │
│  C++23:    constexpr improvements                        │
│            • More std functions as constexpr             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Classic Template Metaprogramming

### Compile-Time Factorial
```cpp
// Classic TMP style (recursive templates)
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

// Base case (specialization)
template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

int main() {
    constexpr int result = Factorial<5>::value;  // 120, computed at compile time
    
    // Can use in array size
    int array[Factorial<4>::value];  // int array[24]
    
    return 0;
}
```

### Visual: Template Recursion
```
Factorial<5>::value
    │
    └─► 5 * Factorial<4>::value
            │
            └─► 4 * Factorial<3>::value
                    │
                    └─► 3 * Factorial<2>::value
                            │
                            └─► 2 * Factorial<1>::value
                                    │
                                    └─► 1 * Factorial<0>::value
                                            │
                                            └─► 1 (base case)

Result: 5 * 4 * 3 * 2 * 1 * 1 = 120
All computed at compile time!
```

### Compile-Time Fibonacci
```cpp
template<int N>
struct Fibonacci {
    static constexpr int value = Fibonacci<N-1>::value + Fibonacci<N-2>::value;
};

template<>
struct Fibonacci<0> {
    static constexpr int value = 0;
};

template<>
struct Fibonacci<1> {
    static constexpr int value = 1;
};

int main() {
    constexpr int fib10 = Fibonacci<10>::value;  // 55
    
    return 0;
}
```

---

## constexpr Functions

### Modern Approach: constexpr
```cpp
// Much cleaner than template recursion!
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr int fibonacci(int n) {
    return n <= 1 ? n : fibonacci(n - 1) + fibonacci(n - 2);
}

int main() {
    constexpr int fact5 = factorial(5);    // Compile time
    constexpr int fib10 = fibonacci(10);   // Compile time
    
    int n = 5;
    int runtime_fact = factorial(n);       // Can also run at runtime!
    
    return 0;
}
```

### C++14: Relaxed constexpr
```cpp
// C++14: Can use loops and local variables
constexpr int factorial_iterative(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

constexpr int fibonacci_iterative(int n) {
    if (n <= 1) return n;
    
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}

int main() {
    constexpr int fact = factorial_iterative(10);  // 3628800
    constexpr int fib = fibonacci_iterative(20);   // 6765
    
    return 0;
}
```

### consteval (C++20): Immediate Functions
```cpp
// MUST be evaluated at compile time — no runtime fallback
consteval int square(int n) {
    return n * n;
}

int main() {
    constexpr int x = square(5);   // OK: compile time

    int n = 5;
    // int y = square(n);           // ERROR: n is not a constant expression
}
```

### constinit (C++20): Static Initialization
```cpp
#include <array>

// Guarantees compile-time (constant) initialization of static/thread-local
// storage. It does NOT make the variable const — it can be modified at runtime.
constinit int global_counter = 42;

// The initializer must be a constant expression. std::array is an aggregate
// and is constant-initializable; std::vector is NOT (it allocates), so
// `constinit std::vector<int> v = {1,2,3};` would FAIL to compile.
constinit std::array<int, 3> ids = {1, 2, 3};  // avoids static-init-order fiasco

int runtime_value();
// constinit int derived = runtime_value();  // ERROR: initializer not constant
```

### Comparison: constexpr vs consteval vs constinit
```
┌─────────────────────────────────────────────────────────────────────┐
│           constexpr vs consteval vs constinit                       │
├─────────────────────────────────────────────────────────────────────┤
│                constexpr      │  consteval    │  constinit          │
├───────────────────────────────┼───────────────┼─────────────────────┤
│ What           Function/      │  Function     │  Variable           │
│                variable       │  (immediate)  │  (static storage)   │
│ When evaluated Compile or     │  Compile only │  Init at compile    │
│                runtime        │               │  time; may mutate   │
│ Runtime call   Yes (if args   │  No           │  N/A                │
│                are constexpr) │               │                     │
└─────────────────────────────────────────────────────────────────────┘
```

💡 **Hunch**: Use `constexpr` by default; `consteval` when compile-time-only is a hard requirement; `constinit` for globals that must be initialized before `main` without dynamic static init.

---

## Type Traits

### Standard Type Traits

See [`<type_traits>` on cppreference](https://en.cppreference.com/w/cpp/header/type_traits). Prefer `*_v` variable templates (C++17) over `::value`.

```cpp
#include <type_traits>
#include <iostream>

template<typename T>
void analyze_type() {
    std::cout << std::boolalpha;
    std::cout << "is_integral: " << std::is_integral_v<T> << '\n';
    std::cout << "is_floating_point: " << std::is_floating_point_v<T> << '\n';
    std::cout << "is_pointer: " << std::is_pointer_v<T> << '\n';
    std::cout << "is_const: " << std::is_const_v<T> << '\n';
    std::cout << "is_reference: " << std::is_reference_v<T> << '\n';
    std::cout << "is_array: " << std::is_array_v<T> << '\n';
}

int main() {
    analyze_type<int>();
    analyze_type<const double*>();
    analyze_type<int&>();
    analyze_type<int[]>();
    
    return 0;
}
```

### Type Transformations
```cpp
#include <type_traits>

int main() {
    // Remove const/volatile
    using type1 = std::remove_const_t<const int>;        // int
    using type2 = std::remove_volatile_t<volatile int>;  // int
    using type3 = std::remove_cv_t<const volatile int>;  // int
    
    // Remove reference
    using type4 = std::remove_reference_t<int&>;         // int
    using type5 = std::remove_reference_t<int&&>;        // int
    
    // Remove pointer
    using type6 = std::remove_pointer_t<int*>;           // int
    
    // Add const/pointer/reference
    using type7 = std::add_const_t<int>;                 // const int
    using type8 = std::add_pointer_t<int>;               // int*
    using type9 = std::add_lvalue_reference_t<int>;      // int&
    
    // Decay (like function parameters)
    using type10 = std::decay_t<int&>;                   // int
    using type11 = std::decay_t<int[3]>;                 // int*
    using type12 = std::decay_t<int(int)>;               // int(*)(int)
    
    return 0;
}
```

### Custom Type Traits
```cpp
#include <type_traits>

// Check if type has size() method
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>>
    : std::true_type {};

// Helper variable template
template<typename T>
inline constexpr bool has_size_v = has_size<T>::value;

// Usage
int main() {
    static_assert(has_size_v<std::vector<int>>);     // true
    static_assert(!has_size_v<int>);                 // false
    
    return 0;
}
```

---

## if constexpr (C++17)

`if constexpr` evaluates its condition at **compile time** and **discards** the non-taken branches — ill-formed code in discarded branches is not compiled. This is essential for heterogeneous templates without runtime overhead.

```cpp
template<typename T>
auto get_value(T t) {
    if constexpr (std::is_pointer_v<T>) {
        return *t;  // Dereference if pointer
    } else {
        return t;   // Return as-is
    }
}

int main() {
    int x = 42;
    int* px = &x;
    
    auto v1 = get_value(x);    // Returns 42
    auto v2 = get_value(px);   // Returns 42 (dereferenced)
    
    return 0;
}
```

### Before vs After if constexpr
```cpp
// Before C++17: Complex SFINAE or tag dispatching
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
process(T value) {
    return value * 2;
}

template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
process(T value) {
    return value * 1.5;
}

// C++17: Much cleaner with if constexpr
template<typename T>
T process_modern(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 1.5;
    } else {
        static_assert(std::is_arithmetic_v<T>, "Must be arithmetic");
    }
}
```

### Visual: if constexpr Compilation
```
Template: process_modern<int>(5)
│
└─► Compile-time evaluation:
    if constexpr (std::is_integral_v<int>)  // true
        return value * 2;                   ✓ Keep this branch
    else if constexpr (...)                 ✗ Discard this branch
        ...
    else
        ...                                 ✗ Discard this branch

Generated code:
int process_modern(int value) {
    return value * 2;  // Only this branch exists in binary
}
```

### Complex if constexpr Example
```cpp
#include <type_traits>
#include <string>

template<typename T>
std::string to_string_custom(const T& value) {
    if constexpr (std::is_same_v<T, std::string>) {
        return value;
    } else if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(value);
    } else if constexpr (std::is_pointer_v<T>) {
        return "pointer: " + std::to_string(reinterpret_cast<uintptr_t>(value));
    } else {
        static_assert(std::is_arithmetic_v<T> || std::is_same_v<T, std::string>,
                      "Unsupported type for to_string_custom");
        return std::to_string(value);
    }
}
```

For new code, prefer [C++20 concepts](09_templates.md) over trait-based SFINAE — clearer errors and simpler signatures:

```cpp
template<std::integral T>
T process(T value) { return value * 2; }

template<std::floating_point T>
T process(T value) { return value * 1.5; }
```

---

## SFINAE (Substitution Failure Is Not An Error)

### Expression SFINAE
```cpp
#include <iostream>
#include <type_traits>
#include <vector>

// Detect if type has begin() and end()
template<typename T, typename = void>
struct is_iterable : std::false_type {};

template<typename T>
struct is_iterable<T, std::void_t<
    decltype(std::declval<T>().begin()),
    decltype(std::declval<T>().end())
>> : std::true_type {};

template<typename T>
inline constexpr bool is_iterable_v = is_iterable<T>::value;

// Use it
template<typename T>
std::enable_if_t<is_iterable_v<T>> print_all(const T& container) {
    for (const auto& elem : container) {
        std::cout << elem << ' ';
    }
}

int main() {
    std::vector<int> vec = {1, 2, 3};
    print_all(vec);  // OK
    
    // print_all(42);  // ERROR: int is not iterable
    
    return 0;
}
```

⚠️ **Gotchas**: SFINAE errors are often deep template instantiation traces. When constraints express intent, use [concepts](09_templates.md) instead of `std::enable_if_t`. The [detection idiom](https://en.cppreference.com/w/cpp/experimental/is_detected) (`std::void_t` + partial specialization) remains useful for type traits.

### std::void_t Trick
```cpp
// std::void_t always produces 'void', but SFINAE happens during substitution

template<typename...>
using void_t = void;

// Example: detect operator<<
template<typename T, typename = void>
struct has_ostream_operator : std::false_type {};

template<typename T>
struct has_ostream_operator<T, std::void_t<
    decltype(std::declval<std::ostream&>() << std::declval<T>())
>> : std::true_type {};

template<typename T>
void print_if_possible(const T& value) {
    if constexpr (has_ostream_operator<T>::value) {
        std::cout << value << '\n';
    } else {
        std::cout << "[unprintable type]\n";
    }
}
```

---

## Variadic Templates and Fold Expressions

### Pack Expansion
```cpp
// Print all arguments
template<typename... Args>
void print_all(Args... args) {
    ((std::cout << args << ' '), ...);  // Fold expression
    std::cout << '\n';
}

// Sum all arguments
template<typename... Args>
auto sum(Args... args) {
    return (... + args);  // Left fold
}

// All satisfy predicate
template<typename... Args>
bool all_positive(Args... args) {
    return ((args > 0) && ...);  // Fold with &&
}

int main() {
    print_all(1, 2, 3, "hello", 4.5);
    
    auto total = sum(1, 2, 3, 4, 5);  // 15
    
    bool all_pos = all_positive(1, 2, 3);    // true
    bool not_all = all_positive(1, -2, 3);   // false
    
    return 0;
}
```

### Type List Manipulation
```cpp
// Count types in pack
template<typename... Ts>
struct TypeList {
    static constexpr size_t size = sizeof...(Ts);
};

// Get type at index
template<size_t Index, typename... Ts>
struct TypeAt;

template<typename T, typename... Ts>
struct TypeAt<0, T, Ts...> {
    using type = T;
};

template<size_t Index, typename T, typename... Ts>
struct TypeAt<Index, T, Ts...> {
    using type = typename TypeAt<Index - 1, Ts...>::type;
};

// Helper
template<size_t Index, typename... Ts>
using TypeAt_t = typename TypeAt<Index, Ts...>::type;

int main() {
    using List = TypeList<int, double, char, std::string>;
    static_assert(List::size == 4);
    
    using FirstType = TypeAt_t<0, int, double, char>;  // int
    using SecondType = TypeAt_t<1, int, double, char>; // double
    
    return 0;
}
```

---

## Compile-Time Algorithms

### Compile-Time String
```cpp
template<size_t N>
struct CompileTimeString {
    char data[N];
    
    constexpr CompileTimeString(const char (&str)[N]) {
        for (size_t i = 0; i < N; ++i) {
            data[i] = str[i];
        }
    }
    
    constexpr size_t length() const {
        return N - 1;  // Exclude null terminator
    }
    
    constexpr char operator[](size_t i) const {
        return data[i];
    }
};

template<size_t N>
CompileTimeString(const char (&)[N]) -> CompileTimeString<N>;

constexpr auto hello = CompileTimeString("Hello");
static_assert(hello.length() == 5);
static_assert(hello[0] == 'H');
```

### Compile-Time Sorting
```cpp
#include <algorithm>
#include <array>

constexpr auto sort_array(std::array<int, 5> arr) {
    // C++20: constexpr algorithms
    std::sort(arr.begin(), arr.end());
    return arr;
}

int main() {
    constexpr auto unsorted = std::array{5, 2, 8, 1, 9};
    constexpr auto sorted = sort_array(unsorted);
    // sorted = {1, 2, 5, 8, 9} at compile time!
    
    return 0;
}
```

---

## Advanced Metaprogramming Techniques

### Expression Templates
```cpp
// Lazy evaluation for mathematical expressions
template<typename E>
class Expression {
public:
    double operator[](size_t i) const {
        return static_cast<const E&>(*this)[i];
    }
    
    size_t size() const {
        return static_cast<const E&>(*this).size();
    }
};

// Vector wrapper
class Vector : public Expression<Vector> {
    std::vector<double> data_;
public:
    Vector(size_t n) : data_(n) {}
    
    double& operator[](size_t i) { return data_[i]; }
    double operator[](size_t i) const { return data_[i]; }
    size_t size() const { return data_.size(); }
};

// Addition expression (lazy)
template<typename E1, typename E2>
class VectorSum : public Expression<VectorSum<E1, E2>> {
    const E1& u_;
    const E2& v_;
public:
    VectorSum(const E1& u, const E2& v) : u_(u), v_(v) {}
    
    double operator[](size_t i) const {
        return u_[i] + v_[i];  // Computed on demand
    }
    
    size_t size() const { return u_.size(); }
};

// Operator overload
template<typename E1, typename E2>
VectorSum<E1, E2> operator+(const Expression<E1>& u, const Expression<E2>& v) {
    return VectorSum<E1, E2>(
        static_cast<const E1&>(u),
        static_cast<const E2&>(v)
    );
}
```

### Type-Based Dispatch
```cpp
// Optimization based on iterator category
template<typename Iter>
void advance_impl(Iter& it, int n, std::random_access_iterator_tag) {
    it += n;  // O(1)
}

template<typename Iter>
void advance_impl(Iter& it, int n, std::bidirectional_iterator_tag) {
    if (n >= 0) {
        while (n--) ++it;
    } else {
        while (n++) --it;
    }
}

template<typename Iter>
void my_advance(Iter& it, int n) {
    advance_impl(it, n,
        typename std::iterator_traits<Iter>::iterator_category{});
}
```

---

## constexpr vs Macros

### Why constexpr is Better
```cpp
// BAD: Macro (no type safety, textual substitution)
#define SQUARE(x) x * x          // note: NO inner parentheses

int main() {
    int result1 = SQUARE(5);      // OK: 5 * 5 = 25
    int result2 = SQUARE(2 + 3);  // 11, NOT 25! expands to 2 + 3 * 2 + 3
    // Adding parentheses — #define SQUARE(x) ((x) * (x)) — fixes THIS bug,
    // but a macro still double-evaluates its argument: SQUARE(i++) is UB.
}

// GOOD: constexpr (type-safe)
constexpr int square(int x) {
    return x * x;
}

int main() {
    constexpr int result1 = square(5);      // 25
    constexpr int result2 = square(2 + 3);  // 25 (correct!)
}
```

---

## Practical Applications

### Compile-Time Unit Checking
```cpp
template<int M, int L, int T>  // Mass, Length, Time
struct Unit {
    static constexpr int mass = M;
    static constexpr int length = L;
    static constexpr int time = T;
};

template<typename U>
class Quantity {
    double value_;
public:
    constexpr explicit Quantity(double v) : value_(v) {}
    constexpr double value() const { return value_; }
};

// Type aliases
using Meter = Unit<0, 1, 0>;
using Second = Unit<0, 0, 1>;
using Velocity = Unit<0, 1, -1>;  // Length / Time

template<typename U1, typename U2>
using ProductUnit = Unit<
    U1::mass + U2::mass,
    U1::length + U2::length,
    U1::time + U2::time
>;

// Multiplication
template<typename U1, typename U2>
constexpr auto operator*(Quantity<U1> a, Quantity<U2> b) {
    return Quantity<ProductUnit<U1, U2>>(a.value() * b.value());
}

int main() {
    Quantity<Meter> distance(100.0);
    Quantity<Second> time(10.0);
    
    auto velocity = distance * time;  // Quantity<Unit<0,1,1>> - WRONG!
    // Type system catches unit errors at compile time!
    
    return 0;
}
```

### Compile-Time Configuration
```cpp
template<bool Debug, bool Logging>
struct Config {
    static constexpr bool debug = Debug;
    static constexpr bool logging = Logging;
};

template<typename Config>
class Application {
public:
    void run() {
        if constexpr (Config::debug) {
            std::cout << "[DEBUG] Starting application\n";
        }
        
        process();
        
        if constexpr (Config::logging) {
            log("Application finished");
        }
    }
    
private:
    void process() {
        // Main logic
    }
    
    void log(const std::string& msg) {
        std::cout << "[LOG] " << msg << '\n';
    }
};

int main() {
    using DevConfig = Config<true, true>;
    using ProdConfig = Config<false, false>;
    
    Application<DevConfig> dev_app;
    dev_app.run();  // Debug and logging code included
    
    Application<ProdConfig> prod_app;
    prod_app.run(); // Debug and logging code eliminated!
    
    return 0;
}
```

---

## Performance Impact

### Compile Time vs Runtime
```
┌────────────────────────────────────────────────────────┐
│         Metaprogramming Trade-offs                     │
├────────────────────────────────────────────────────────┤
│ Aspect         │ Pros              │ Cons              │
├────────────────┼───────────────────┼───────────────────┤
│ Performance    │ No runtime cost   │ Longer compilation│
│ Code size      │ Can be smaller*   │ Can be larger**   │
│ Debuggability  │ Compile errors    │ Complex errors    │
│ Maintainability│ Type-safe         │ Complex syntax    │
└────────────────────────────────────────────────────────┘

* With constexpr
** With template instantiation bloat
```

---

## Best Practices

### 1. Prefer constexpr Over Template Recursion
```cpp
// OLD: Template recursion
template<int N> struct Fib { ... };

// NEW: constexpr function
constexpr int fib(int n) { ... }
```

### 2. Use if constexpr for Clarity
```cpp
// Clear branching at compile time
if constexpr (condition) {
    // One path
} else {
    // Another path
}
```

### 3. Limit Template Instantiation Depth
```cpp
// Can cause compilation issues
// Set reasonable limits or use constexpr
```

### 4. Provide Clear Error Messages
```cpp
template<typename T>
void func(T value) {
    static_assert(std::is_arithmetic_v<T>, 
                  "T must be an arithmetic type");
}
```

---

## Common Pitfalls

### 1. ODR Violations with constexpr
```cpp
// Header file - OK: one shared object across all TUs (C++17)
inline constexpr int value = 42;

// Non-inline in a header is NOT an ODR violation: a namespace-scope
// constexpr variable is implicitly const and therefore has INTERNAL linkage,
// so every TU gets its own private copy. That wastes storage and means the
// address differs per TU — use `inline` to get a single shared definition.
// constexpr int value = 42;
```

### 2. Forgetting static_assert
```cpp
template<typename T>
class Container {
    // GOOD: Check requirements early
    static_assert(std::is_default_constructible_v<T>,
                  "T must be default constructible");
};
```

### 3. Excessive Template Bloat
```cpp
// Each instantiation creates new code
template<int N>
void process() {
    // Lots of code...
}

// Better: Extract non-dependent code
void process_impl() {
    // Common code
}

template<int N>
void process() {
    process_impl();
    // Only parameter-dependent code here
}
```

---

## Complete Practical Example: Compile-Time Expression Evaluator

Here's a comprehensive example demonstrating template metaprogramming, constexpr, type traits, and compile-time computation:

```cpp
#include <iostream>
#include <type_traits>
#include <array>
#include <string_view>

// 1. Compile-time mathematics
namespace CompileTimeMath {
    // Factorial
    constexpr int factorial(int n) {
        return n <= 1 ? 1 : n * factorial(n - 1);
    }
    
    // Power
    constexpr double power(double base, int exp) {
        if (exp == 0) return 1.0;
        if (exp < 0) return 1.0 / power(base, -exp);
        
        double result = 1.0;
        for (int i = 0; i < exp; ++i) {
            result *= base;
        }
        return result;
    }
    
    // Fibonacci
    constexpr int fibonacci(int n) {
        if (n <= 1) return n;
        
        int a = 0, b = 1;
        for (int i = 2; i <= n; ++i) {
            int temp = a + b;
            a = b;
            b = temp;
        }
        return b;
    }
    
    // Is prime
    constexpr bool is_prime(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        
        for (int i = 3; i * i <= n; i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
}

// 2. Type traits and SFINAE
namespace TypeTraits {
    // Check if type has begin() method
    template<typename T, typename = void>
    struct has_begin : std::false_type {};
    
    template<typename T>
    struct has_begin<T, std::void_t<decltype(std::declval<T>().begin())>> 
        : std::true_type {};
    
    template<typename T>
    inline constexpr bool has_begin_v = has_begin<T>::value;
    
    // Check if type is iterable
    template<typename T, typename = void>
    struct is_iterable : std::false_type {};
    
    template<typename T>
    struct is_iterable<T, std::void_t<
        decltype(std::declval<T>().begin()),
        decltype(std::declval<T>().end())
    >> : std::true_type {};
    
    template<typename T>
    inline constexpr bool is_iterable_v = is_iterable<T>::value;
    
    // Get element type of container
    template<typename T>
    struct container_value_type {
        using type = typename T::value_type;
    };
    
    template<typename T>
    using container_value_type_t = typename container_value_type<T>::type;
}

// 3. Compile-time string operations
class ConstString {
private:
    const char* data_;
    std::size_t size_;
    
public:
    template<std::size_t N>
    constexpr ConstString(const char (&str)[N]) 
        : data_(str), size_(N - 1) {}
    
    constexpr std::size_t size() const { return size_; }
    constexpr const char* data() const { return data_; }
    constexpr char operator[](std::size_t i) const { return data_[i]; }
    
    constexpr bool starts_with(char c) const {
        return size_ > 0 && data_[0] == c;
    }
    
    constexpr bool ends_with(char c) const {
        return size_ > 0 && data_[size_ - 1] == c;
    }
};

// 4. Compile-time array generation
template<std::size_t N>
constexpr auto generate_primes() {
    std::array<int, N> primes{};
    std::size_t count = 0;
    int num = 2;
    
    while (count < N) {
        if (CompileTimeMath::is_prime(num)) {
            primes[count++] = num;
        }
        num++;
    }
    
    return primes;
}

template<std::size_t N>
constexpr auto generate_fibonacci() {
    std::array<int, N> fib{};
    
    for (std::size_t i = 0; i < N; ++i) {
        fib[i] = CompileTimeMath::fibonacci(static_cast<int>(i));
    }
    
    return fib;
}

// 5. Compile-time configuration system
template<bool EnableLogging, bool EnableDebug, int MaxConnections>
struct Config {
    static constexpr bool logging = EnableLogging;
    static constexpr bool debug = EnableDebug;
    static constexpr int max_connections = MaxConnections;
    
    static constexpr const char* build_type() {
        if constexpr (debug) {
            return "Debug";
        } else {
            return "Release";
        }
    }
    
    static constexpr int buffer_size() {
        if constexpr (max_connections < 10) {
            return 1024;
        } else if constexpr (max_connections < 100) {
            return 4096;
        } else {
            return 16384;
        }
    }
};

// 6. Type-based dispatch with if constexpr
template<typename T>
void process_value(const T& value) {
    std::cout << "Processing: ";
    
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Integer " << value << "\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Float " << value << "\n";
    } else if constexpr (std::is_pointer_v<T>) {
        std::cout << "Pointer to " << *value << "\n";
    } else if constexpr (TypeTraits::is_iterable_v<T>) {
        std::cout << "Container [";
        bool first = true;
        for (const auto& item : value) {
            if (!first) std::cout << ", ";
            std::cout << item;
            first = false;
        }
        std::cout << "]\n";
    } else {
        std::cout << "Unknown type\n";
    }
}

// 7. Compile-time unit conversion
namespace Units {
    enum class Length { Meter, Kilometer, Mile };
    
    template<Length Unit>
    struct LengthValue {
        double value;
        
        constexpr LengthValue(double v) : value(v) {}
        
        // Conversion to meters
        constexpr double to_meters() const {
            if constexpr (Unit == Length::Meter) {
                return value;
            } else if constexpr (Unit == Length::Kilometer) {
                return value * 1000.0;
            } else if constexpr (Unit == Length::Mile) {
                return value * 1609.34;
            }
        }
        
        // Convert to another unit
        template<Length TargetUnit>
        constexpr LengthValue<TargetUnit> convert() const {
            double meters = to_meters();
            
            if constexpr (TargetUnit == Length::Meter) {
                return LengthValue<TargetUnit>(meters);
            } else if constexpr (TargetUnit == Length::Kilometer) {
                return LengthValue<TargetUnit>(meters / 1000.0);
            } else if constexpr (TargetUnit == Length::Mile) {
                return LengthValue<TargetUnit>(meters / 1609.34);
            }
        }
    };
}

// 8. Compile-time validation
template<int N>
struct ValidatedInt {
    static_assert(N >= 0, "Value must be non-negative");
    static_assert(N <= 100, "Value must be <= 100");
    static constexpr int value = N;
};

// 9. Type selector based on size
template<std::size_t Bytes>
struct IntegerSelector {
    using type = std::conditional_t<Bytes == 1, std::int8_t,
                 std::conditional_t<Bytes == 2, std::int16_t,
                 std::conditional_t<Bytes == 4, std::int32_t,
                 std::conditional_t<Bytes == 8, std::int64_t, void>>>>;
    
    static_assert(!std::is_void_v<type>, "Invalid byte size");
};

// Demonstrations
void demo_compile_time_math() {
    std::cout << "\n--- Compile-Time Mathematics ---\n";
    
    // All computed at compile time!
    constexpr int fact5 = CompileTimeMath::factorial(5);
    constexpr double pow_result = CompileTimeMath::power(2.0, 10);
    constexpr int fib10 = CompileTimeMath::fibonacci(10);
    constexpr bool is_17_prime = CompileTimeMath::is_prime(17);
    
    std::cout << "5! = " << fact5 << "\n";
    std::cout << "2^10 = " << pow_result << "\n";
    std::cout << "fib(10) = " << fib10 << "\n";
    std::cout << "is_prime(17) = " << std::boolalpha << is_17_prime << "\n";
}

void demo_compile_time_arrays() {
    std::cout << "\n--- Compile-Time Arrays ---\n";
    
    // Arrays generated at compile time
    constexpr auto primes = generate_primes<10>();
    constexpr auto fib_seq = generate_fibonacci<10>();
    
    std::cout << "First 10 primes: ";
    for (int p : primes) {
        std::cout << p << " ";
    }
    std::cout << "\n";
    
    std::cout << "First 10 Fibonacci: ";
    for (int f : fib_seq) {
        std::cout << f << " ";
    }
    std::cout << "\n";
}

void demo_type_dispatch() {
    std::cout << "\n--- Type-Based Dispatch ---\n";
    
    process_value(42);
    process_value(3.14);
    
    int x = 100;
    process_value(&x);
    
    std::vector<int> vec = {1, 2, 3, 4, 5};
    process_value(vec);
}

void demo_unit_conversion() {
    std::cout << "\n--- Compile-Time Unit Conversion ---\n";
    
    constexpr Units::LengthValue<Units::Length::Kilometer> km(5.0);
    constexpr auto meters = km.convert<Units::Length::Meter>();
    constexpr auto miles = km.convert<Units::Length::Mile>();
    
    std::cout << "5 km = " << meters.value << " m\n";
    std::cout << "5 km = " << miles.value << " miles\n";
}

void demo_compile_time_config() {
    std::cout << "\n--- Compile-Time Configuration ---\n";
    
    using DevConfig = Config<true, true, 10>;
    using ProdConfig = Config<false, false, 1000>;
    
    std::cout << "Dev Config:\n";
    std::cout << "  Build: " << DevConfig::build_type() << "\n";
    std::cout << "  Logging: " << DevConfig::logging << "\n";
    std::cout << "  Buffer: " << DevConfig::buffer_size() << " bytes\n";
    
    std::cout << "Prod Config:\n";
    std::cout << "  Build: " << ProdConfig::build_type() << "\n";
    std::cout << "  Logging: " << ProdConfig::logging << "\n";
    std::cout << "  Buffer: " << ProdConfig::buffer_size() << " bytes\n";
}

void demo_type_traits() {
    std::cout << "\n--- Custom Type Traits ---\n";
    
    std::cout << "std::vector has begin: " 
              << TypeTraits::has_begin_v<std::vector<int>> << "\n";
    std::cout << "int has begin: " 
              << TypeTraits::has_begin_v<int> << "\n";
    
    std::cout << "std::vector is iterable: " 
              << TypeTraits::is_iterable_v<std::vector<int>> << "\n";
    std::cout << "int is iterable: " 
              << TypeTraits::is_iterable_v<int> << "\n";
}

void demo_integer_selector() {
    std::cout << "\n--- Size-Based Type Selection ---\n";
    
    using Int1 = IntegerSelector<1>::type;  // int8_t
    using Int2 = IntegerSelector<2>::type;  // int16_t
    using Int4 = IntegerSelector<4>::type;  // int32_t
    using Int8 = IntegerSelector<8>::type;  // int64_t
    
    std::cout << "1 byte: " << sizeof(Int1) << " bytes\n";
    std::cout << "2 byte: " << sizeof(Int2) << " bytes\n";
    std::cout << "4 byte: " << sizeof(Int4) << " bytes\n";
    std::cout << "8 byte: " << sizeof(Int8) << " bytes\n";
}

int main() {
    std::cout << "=== Template Metaprogramming Demo ===\n";
    
    // 1. Compile-time mathematics
    demo_compile_time_math();
    
    // 2. Compile-time array generation
    demo_compile_time_arrays();
    
    // 3. Type-based dispatch
    demo_type_dispatch();
    
    // 4. Unit conversion
    demo_unit_conversion();
    
    // 5. Compile-time configuration
    demo_compile_time_config();
    
    // 6. Custom type traits
    demo_type_traits();
    
    // 7. Integer type selection
    demo_integer_selector();
    
    std::cout << "\n=== Demo Complete ===\n";
    std::cout << "\nNote: Most computations happened at compile time!\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **constexpr functions**: Compile-time computation
- **constexpr variables**: Compile-time constants
- **Compile-time arrays**: std::array generation
- **Custom type traits**: Detecting member functions
- **SFINAE**: std::void_t for trait detection
- **if constexpr**: Compile-time branching
- **std::conditional**: Type selection
- **static_assert**: Compile-time validation
- **Template specialization**: Type-specific behavior
- **Type deduction**: auto with constexpr
- **Compile-time strings**: ConstString class
- **Unit conversion**: Type-safe conversions
- **Configuration systems**: Compile-time settings
- **Type dispatching**: Different code paths per type

All expensive computations run at compile time - zero runtime cost!

---

## Related Topics

- [Templates](09_templates.md) — variadic templates, concepts, and template specialization underpin TMP
- [Modern Features](07_modern_features.md) — `constexpr` variables, `if constexpr`, fold expressions, designated init
- [Lambdas](10_lambdas.md) — `constexpr` lambdas; generic lambdas as lightweight compile-time helpers
- [Advanced Features](12_advanced_features.md) — `if constexpr` with value categories; `inline` variables (C++17)
- [Best Practices](13_best_practices.md) — prefer `constexpr` over macros; limit template bloat
- [Utility Containers](08_utility_containers.md) — `std::optional`, `std::variant` for type-safe compile-time-friendly APIs
- [Quick Reference](99_quick_reference.md) — common type traits and `constexpr` patterns

---

## Next Steps
- **Next**: [Advanced C++ Features →](12_advanced_features.md)
- **Previous**: [← Lambdas](10_lambdas.md)

---
*Chapter 11 — Template Metaprogramming*

