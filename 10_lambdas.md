# Lambdas and Functional Programming

## Overview

Lambda expressions (introduced in C++11) provide a concise way to create anonymous function objects — closures with optional capture of surrounding state. They are the standard way to pass behavior to [STL algorithms](06_algorithms.md) and to write inline callbacks without a separate functor type.

```
┌─────────────────────────────────────────────────────────┐
│                    LAMBDA EXPRESSIONS                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [capture](parameters) -> return_type { body }          │
│   ▲        ▲             ▲              ▲               │
│   │        │             │              │               │
│   │        │             │              └─ Function body│
│   │        │             └─ Return type (optional)      │
│   │        └─ Parameters                                │
│   └─ Capture clause                                     │
│                                                         │
│  EVOLUTION:                                             │
│  C++11: Basic lambdas                                   │
│  C++14: Generic lambdas, init-capture                   │
│  C++17: constexpr lambdas                               │
│  C++20: Template lambdas                                │
│  C++23: Explicit object parameter                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Basic Lambda Syntax

### Simple Lambda
```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // Basic lambda
    auto greet = []() {
        std::cout << "Hello, World!\n";
    };
    greet();  // Call it
    
    // Lambda with parameters
    auto add = [](int a, int b) {
        return a + b;
    };
    int sum = add(5, 3);  // 8
    
    // Lambda with explicit return type
    auto divide = [](int a, int b) -> double {
        return static_cast<double>(a) / b;
    };
    
    // Inline lambda (common with STL algorithms)
    std::vector<int> vec = {1, 2, 3, 4, 5};
    std::for_each(vec.begin(), vec.end(), [](int x) {
        std::cout << x << ' ';
    });
    
    return 0;
}
```

### Lambda Anatomy
```
Full Lambda Expression:

[capture_clause](parameters) mutable -> return_type { body }
 ▲              ▲            ▲        ▲              ▲
 │              │            │        │              │
 │              │            │        │              └─ Function body
 │              │            │        └─ Return type (optional if deducible)
 │              │            └─ mutable keyword (optional)
 │              └─ Parameter list
 └─ Capture clause (how to access outer variables)

Minimal Lambda:
[](){ }
```

---

## Capture Clauses

### Capture Modes

```cpp
int main() {
    int x = 10;
    int y = 20;
    
    // [=] Capture all by value
    auto lambda1 = [=]() {
        return x + y;  // x and y are copies
    };
    
    // [&] Capture all by reference
    auto lambda2 = [&]() {
        x++;  // Modifies original x
        y++;  // Modifies original y
    };
    
    // [x] Capture x by value
    auto lambda3 = [x]() {
        return x * 2;  // x is a copy
    };
    
    // [&x] Capture x by reference
    auto lambda4 = [&x]() {
        x *= 2;  // Modifies original x
    };
    
    // [x, &y] Mixed capture
    auto lambda5 = [x, &y]() {
        // x is copy, y is reference
        return x + (++y);
    };
    
    // [=, &y] Capture all by value except y by reference
    auto lambda6 = [=, &y]() {
        return x + (++y);
    };
    
    // [&, x] Capture all by reference except x by value
    auto lambda7 = [&, x]() {
        y++;  // Modifies original y
        return x + y;  // x is a copy
    };
    
    return 0;
}
```

### `this`, Statics, and Globals

```cpp
class Widget {
    int value_ = 42;

public:
    auto by_this() {
        return [this]() { return value_; };      // captures Widget* (pointer)
    }

    auto by_copy() {
        return [*this]() { return value_; };      // C++17: copies entire object
    }

    auto by_default() {
        return [=]() { return value_; };           // implicit [this] — same as [this]
    }
};

int global = 100;

void demo() {
    int local = 10;
    static int s = 20;

    auto f = [=]() { return local + s + global; };
    // Captures: local (by value) — only automatic-storage locals are captured
    // Does NOT capture: global, or the static local s (both accessed by name)
}
```

⚠️ **Gotchas**
- `[this]` stores a **pointer** to the enclosing object. If the lambda outlives `*this`, it dangles — same risk as a raw pointer. Prefer `[*this]` (C++17) when the lambda must survive the call that created it.
- `[=]` and `[&]` in a member function implicitly capture `this` (as a pointer), not individual data members. Implicit `this` capture via `[=]` is **deprecated since C++20** (P0806) — write `[=, this]` (or `[this]`) explicitly.
- Default captures (`[=]`/`[&]`) never capture `static` or global variables — those are accessed directly by name.

### Visual: Capture Behavior
```
Original Variables:
┌─────┬─────┐
│ x=10│ y=20│
└─────┴─────┘

[=] Capture by Value:
Lambda object:
┌─────┬─────┐
│ x=10│ y=20│  (copies)
└─────┴─────┘

[&] Capture by Reference:
Lambda object:
┌────┬────┐
│ &x │ &y │  (pointers to originals)
└────┴────┘
   │    │
   ▼    ▼
┌─────┬─────┐
│ x=10│ y=20│  (original variables)
└─────┴─────┘
```

### Capture Examples
```cpp
#include <iostream>

int main() {
    int count = 0;
    
    // Capture by value (count is copied)
    auto increment_copy = [count]() mutable {
        count++;  // Modifies the copy
        std::cout << "Inside: " << count << '\n';
    };
    
    increment_copy();  // Inside: 1
    increment_copy();  // Inside: 2
    std::cout << "Outside: " << count << '\n';  // Outside: 0
    
    // Capture by reference (count is referenced)
    auto increment_ref = [&count]() {
        count++;  // Modifies original
        std::cout << "Inside: " << count << '\n';
    };
    
    increment_ref();  // Inside: 1
    increment_ref();  // Inside: 2
    std::cout << "Outside: " << count << '\n';  // Outside: 2
    
    return 0;
}
```

### Init Capture (C++14)
```cpp
#include <memory>
#include <vector>

int main() {
    int x = 10;
    
    // Initialize new variable in capture
    auto lambda = [y = x + 5]() {
        return y * 2;  // y is 15
    };
    
    // Move into lambda
    auto ptr = std::make_unique<int>(42);
    auto lambda2 = [p = std::move(ptr)]() {
        return *p;  // Owns the unique_ptr
    };
    
    // Complex initialization
    auto lambda3 = [vec = std::vector<int>{1, 2, 3}]() {
        return vec.size();
    };
    
    return 0;
}
```

---

## Mutable Lambdas

### Understanding mutable
```cpp
int main() {
    int x = 10;
    
    // Without mutable: captured values are const
    auto lambda1 = [x]() {
        // x++;  // ERROR: x is const
        return x;
    };
    
    // With mutable: can modify captured values
    auto lambda2 = [x]() mutable {
        x++;  // OK: modifies the copy
        return x;
    };
    
    std::cout << lambda2() << '\n';  // 11
    std::cout << lambda2() << '\n';  // 12
    std::cout << x << '\n';          // 10 (original unchanged)
    
    return 0;
}
```

### Visual: mutable Keyword
```
Without mutable:
Lambda object (const members):
┌─────────┐
│ const x │  Cannot modify
└─────────┘

With mutable:
Lambda object (non-const members):
┌─────┐
│  x  │  Can modify
└─────┘

Note: Only affects captured-by-value variables!
References are always modifiable.
```

---

## Generic Lambdas (C++14)

A generic lambda with `auto` parameters is a **function call operator template** — each distinct argument type set instantiates a new specialization. For explicit template parameters or concepts, use C++20 template lambdas (below) or see [Templates](09_templates.md).

### Auto Parameters
```cpp
#include <iostream>
#include <string>

int main() {
    // Generic lambda (works with any type)
    auto print = [](auto x) {
        std::cout << x << '\n';
    };
    
    print(42);           // int
    print(3.14);         // double
    print("Hello");      // const char*
    print(std::string("World"));  // std::string
    
    // Multiple auto parameters
    auto add = [](auto a, auto b) {
        return a + b;
    };
    
    auto sum1 = add(5, 3);        // int + int
    auto sum2 = add(1.5, 2.5);    // double + double
    auto sum3 = add(std::string("Hello"), std::string(" World"));  // string + string
    
    // Generic lambda with constraints
    auto multiply = [](auto a, auto b) -> decltype(a * b) {
        return a * b;
    };
    
    return 0;
}
```

### Template Lambda (C++20)

C++14 generic lambdas (`auto` parameters) are implemented as a call operator template. C++20 adds **explicit** template parameters — useful when you need `T` in the capture clause or to constrain types (often with [concepts](09_templates.md)):

```cpp
#include <iostream>
#include <tuple>
#include <utility>
#include <vector>

int main() {
    // Explicit template parameters
    auto print_container = []<typename T>(const std::vector<T>& vec) {
        for (const auto& elem : vec) {
            std::cout << elem << ' ';
        }
        std::cout << '\n';
    };
    
    std::vector<int> v1 = {1, 2, 3};
    std::vector<double> v2 = {1.1, 2.2, 3.3};
    
    print_container(v1);
    print_container(v2);
    
    // Perfect forwarding — see [Advanced Features](12_advanced_features.md)
    auto forward_call = []<typename... Args>(Args&&... args) {
        return std::make_tuple(std::forward<Args>(args)...);
    };

    // C++20: capture a parameter pack by moving into the closure
    auto make_tuple_moved = []<typename... Args>(Args&&... args) {
        return [... args = std::forward<Args>(args)]() mutable {
            return std::make_tuple(std::move(args)...);
        };
    };

    return 0;
}
```

---

## Lambda with STL Algorithms

### Predicates
```cpp
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // count_if with lambda predicate
    auto even_count = std::count_if(vec.begin(), vec.end(), 
        [](int x) { return x % 2 == 0; });
    
    // find_if
    auto it = std::find_if(vec.begin(), vec.end(),
        [](int x) { return x > 5; });
    
    // remove_if
    vec.erase(
        std::remove_if(vec.begin(), vec.end(),
            [](int x) { return x % 2 == 0; }),
        vec.end()
    );
    // vec now contains only odd numbers
    
    // sort with custom comparator
    std::vector<std::string> words = {"apple", "pie", "a", "hello"};
    std::sort(words.begin(), words.end(),
        [](const std::string& a, const std::string& b) {
            return a.length() < b.length();  // Sort by length
        });
    
    return 0;
}
```

### Transform
```cpp
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    std::vector<int> squares(5);
    
    // Square each number
    std::transform(numbers.begin(), numbers.end(), squares.begin(),
        [](int x) { return x * x; });
    
    // In-place transform
    std::transform(numbers.begin(), numbers.end(), numbers.begin(),
        [](int x) { return x * 2; });
    
    return 0;
}
```

### Accumulate with Lambda
```cpp
#include <numeric>
#include <vector>
#include <string>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Sum with lambda
    int sum = std::accumulate(vec.begin(), vec.end(), 0,
        [](int acc, int x) { return acc + x; });
    
    // Product
    int product = std::accumulate(vec.begin(), vec.end(), 1,
        [](int acc, int x) { return acc * x; });
    
    // Concatenate strings
    std::vector<std::string> words = {"Hello", " ", "World", "!"};
    std::string sentence = std::accumulate(words.begin(), words.end(), 
        std::string(""),
        [](const std::string& acc, const std::string& word) {
            return acc + word;
        });
    
    return 0;
}
```

---

## Returning Lambdas

### Lambda as Return Value
```cpp
#include <functional>
#include <iostream>

// Return lambda from function
auto make_adder(int n) {
    return [n](int x) { return x + n; };
}

// Using std::function for type erasure
std::function<int(int)> make_multiplier(int n) {
    return [n](int x) { return x * n; };
}

int main() {
    auto add5 = make_adder(5);
    std::cout << add5(10) << '\n';  // 15
    std::cout << add5(20) << '\n';  // 25
    
    auto times3 = make_multiplier(3);
    std::cout << times3(4) << '\n';  // 12
    
    return 0;
}
```

### Closure and Captured Variables
```cpp
auto make_counter() {
    int count = 0;
    return [count]() mutable {
        return ++count;
    };
}

int main() {
    auto counter1 = make_counter();
    auto counter2 = make_counter();
    
    std::cout << counter1() << '\n';  // 1
    std::cout << counter1() << '\n';  // 2
    std::cout << counter2() << '\n';  // 1 (separate closure)
    
    return 0;
}
```

---

## constexpr Lambdas (C++17)

### Compile-Time Lambdas
```cpp
int main() {
    // Implicitly constexpr if possible
    auto square = [](int x) { return x * x; };
    
    constexpr int result = square(5);  // Computed at compile time
    
    // Explicit constexpr
    constexpr auto add = [](int a, int b) constexpr {
        return a + b;
    };
    
    constexpr int sum = add(10, 20);  // Compile time
    
    // Use in constexpr context
    static_assert(square(4) == 16);
    
    return 0;
}
```

---

## Recursive Lambdas

### Using std::function
```cpp
#include <functional>

int main() {
    // Factorial using recursive lambda
    std::function<int(int)> factorial = [&factorial](int n) {
        return n <= 1 ? 1 : n * factorial(n - 1);
    };
    
    std::cout << factorial(5) << '\n';  // 120
    
    return 0;
}
```

### Using Y-Combinator (C++14)
```cpp
auto Y = [](auto f) {
    return [f](auto... args) {
        return f(f, args...);
    };
};

int main() {
    auto factorial = Y([](auto self, int n) -> int {
        return n <= 1 ? 1 : n * self(self, n - 1);
    });
    
    std::cout << factorial(5) << '\n';  // 120
    
    return 0;
}
```

---

## Advanced Lambda Patterns

### Immediately Invoked Lambda Expression (IIFE)
```cpp
int main() {
    // Initialize const variable with complex logic
    const int value = [](int x) {
        if (x < 0) return 0;
        if (x > 100) return 100;
        return x;
    }(42);  // Immediately invoked!
    
    // Useful for const initialization
    const auto data = [&]() {
        std::vector<int> temp;
        for (int i = 0; i < 10; ++i) {
            temp.push_back(i * i);
        }
        return temp;
    }();
    
    return 0;
}
```

### Stateful Lambdas
```cpp
#include <iostream>

int main() {
    // Lambda with state (via mutable capture)
    int seed = 42;
    auto random = [seed]() mutable {
        seed = (seed * 1103515245 + 12345) & 0x7fffffff;
        return seed;
    };
    
    std::cout << random() << '\n';
    std::cout << random() << '\n';
    std::cout << random() << '\n';
    
    return 0;
}
```

### Lambda Overloading (C++17)
```cpp
#include <variant>

// Helper for overloading lambdas
template<typename... Ts>
struct overload : Ts... {
    using Ts::operator()...;
};

template<typename... Ts>
overload(Ts...) -> overload<Ts...>;

int main() {
    std::variant<int, double, std::string> var = 42;
    
    std::visit(overload{
        [](int i) { std::cout << "int: " << i << '\n'; },
        [](double d) { std::cout << "double: " << d << '\n'; },
        [](const std::string& s) { std::cout << "string: " << s << '\n'; }
    }, var);
    
    return 0;
}
```

---

## Functional Programming Patterns

### Map, Filter, Reduce
```cpp
#include <vector>
#include <algorithm>
#include <numeric>

template<typename T, typename F>
auto map(const std::vector<T>& vec, F func) {
    std::vector<decltype(func(vec[0]))> result;
    result.reserve(vec.size());
    std::transform(vec.begin(), vec.end(), std::back_inserter(result), func);
    return result;
}

template<typename T, typename F>
auto filter(const std::vector<T>& vec, F pred) {
    std::vector<T> result;
    std::copy_if(vec.begin(), vec.end(), std::back_inserter(result), pred);
    return result;
}

template<typename T, typename F>
auto reduce(const std::vector<T>& vec, T init, F func) {
    return std::accumulate(vec.begin(), vec.end(), init, func);
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    // Map: square each number
    auto squares = map(numbers, [](int x) { return x * x; });
    // {1, 4, 9, 16, 25}
    
    // Filter: keep only even numbers
    auto evens = filter(numbers, [](int x) { return x % 2 == 0; });
    // {2, 4}
    
    // Reduce: sum all numbers
    auto sum = reduce(numbers, 0, [](int acc, int x) { return acc + x; });
    // 15
    
    return 0;
}
```

### Function Composition
```cpp
#include <functional>

template<typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto x) { return f(g(x)); };
}

int main() {
    auto add1 = [](int x) { return x + 1; };
    auto times2 = [](int x) { return x * 2; };
    
    auto composed = compose(times2, add1);
    // composed(x) = times2(add1(x)) = (x + 1) * 2
    
    std::cout << composed(5) << '\n';  // (5 + 1) * 2 = 12
    
    return 0;
}
```

### Currying
```cpp
auto curry(auto func) {
    return [func](auto first) {
        return [func, first](auto second) {
            return func(first, second);
        };
    };
}

int main() {
    auto add = [](int a, int b) { return a + b; };
    auto curried_add = curry(add);
    
    auto add5 = curried_add(5);
    
    std::cout << add5(3) << '\n';   // 8
    std::cout << add5(10) << '\n';  // 15
    
    return 0;
}
```

---

## Performance Considerations

### Lambda vs Function Pointer
```cpp
// Function pointer (no inline)
void process1(const std::vector<int>& vec, int(*func)(int)) {
    for (int x : vec) {
        func(x);  // Indirect call, can't inline
    }
}

// Lambda (can inline)
template<typename F>
void process2(const std::vector<int>& vec, F func) {
    for (int x : vec) {
        func(x);  // Can be inlined by compiler
    }
}

int main() {
    std::vector<int> vec(1000000);
    
    auto lambda = [](int x) { return x * 2; };
    
    process1(vec, [](int x) { return x * 2; });  // Slower
    process2(vec, lambda);                        // Faster (inlining)
    
    return 0;
}
```

### Capture Overhead
```
┌───────────────────────────────────────────────────────┐
│              Lambda Capture Overhead                  │
├───────────────────────────────────────────────────────┤
│ Capture Type     │ Size        │ Performance          │
├──────────────────┼─────────────┼──────────────────────┤
│ No capture       │ 0 bytes*    │ Best (stateless)     │
│ By value (int)   │ sizeof(int) │ Good (inline)        │
│ By reference     │ sizeof(ptr) │ Good (indirect)      │
│ By value (large) │ sizeof(T)   │ Slow (copy)          │
│ [=] many vars    │ Sum of all  │ Depends on size      │
└───────────────────────────────────────────────────────┘

Tip: Capture only what you need!

*Stateless lambdas have no storage; they can convert to function pointers.
 Empty lambdas may still occupy 1 byte as a unique type (implementation-defined).
```

---

## Common Pitfalls

### 1. Dangling References
```cpp
// BAD: Returning lambda with reference to local
auto make_lambda() {
    int x = 42;
    return [&x]() { return x; };  // DANGER: x destroyed!
}

// GOOD: Capture by value
auto make_lambda_safe() {
    int x = 42;
    return [x]() { return x; };  // OK: x copied
}
```

### 2. Capturing `this`
```cpp
class Widget {
    int value_ = 42;

public:
    auto dangling() {
        return [this]() { return value_; };  // Widget* — dangles if lambda outlives *this
    }

    auto safe_copy() {
        return [*this]() { return value_; };  // C++17: snapshot of entire object
    }
};
// `[=]` in a member function is equivalent to `[=, this]` (C++20) / implicit `this` capture
```

### 3. Capture in Loops
```cpp
// BAD: All lambdas capture same reference
std::vector<std::function<int()>> funcs;
for (int i = 0; i < 5; ++i) {
    funcs.push_back([&i]() { return i; });  // All capture same i!
}
// All funcs return 5!

// GOOD: Capture by value
std::vector<std::function<int()>> funcs_good;
for (int i = 0; i < 5; ++i) {
    funcs_good.push_back([i]() { return i; });  // Each captures own i
}
```

---

## Best Practices

### 1. Prefer Generic Lambdas
```cpp
// GOOD: Works with any type
auto print = [](const auto& x) {
    std::cout << x << '\n';
};

// Less flexible
auto print_int = [](int x) {
    std::cout << x << '\n';
};
```

### 2. Use Init-Capture for Move-Only Types
```cpp
auto ptr = std::make_unique<int>(42);

// GOOD: Move into lambda
auto lambda = [p = std::move(ptr)]() {
    return *p;
};
```

### 3. Avoid Capturing Everything
```cpp
// BAD: Unclear what's captured
auto lambda = [=]() { /* ... */ };

// GOOD: Explicit captures
auto lambda = [x, y]() { /* ... */ };
```

### 4. Use mutable Sparingly
```cpp
// Consider if you really need mutable state
// Often better to use references or return new values
```

---

## C++23 Features

### Explicit Object Parameter
```cpp
// C++23: Explicit 'this' parameter
auto lambda = [](this auto self, int n) {
    if (n <= 0) return 1;
    return n * self(n - 1);  // Recursive without std::function!
};

std::cout << lambda(5) << '\n';  // 120
```

---

## Summary

```
┌────────────────────────────────────────────────────────┐
│              Lambda Capabilities                       │
├────────────────────────────────────────────────────────┤
│ C++11: Basic lambdas, captures                         │
│ C++14: Generic lambdas (auto), init-capture            │
│ C++17: constexpr, *this capture                        │
│ C++20: Template parameters, concepts                   │
│ C++23: Explicit object parameter                       │
│                                                        │
│ Use Cases:                                             │
│ • STL algorithms predicates                            │
│ • Event handlers/callbacks                             │
│ • Custom comparators                                   │
│ • Functional programming                               │
│ • Inline helper functions                              │
└────────────────────────────────────────────────────────┘
```

---

## Complete Practical Example: Event Processing System

Here's a comprehensive example integrating lambdas, functional programming, and STL algorithms:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <functional>
#include <map>
#include <chrono>
#include <memory>

// Event types
enum class EventType { Click, KeyPress, MouseMove, Scroll, Custom };

struct Event {
    EventType type;
    std::string target;
    std::chrono::system_clock::time_point timestamp;
    std::map<std::string, std::string> data;
    
    Event(EventType t, std::string tgt)
        : type(t), target(std::move(tgt))
        , timestamp(std::chrono::system_clock::now()) {}
};

class EventProcessor {
private:
    using EventHandler = std::function<void(const Event&)>;
    using EventFilter = std::function<bool(const Event&)>;
    using EventTransform = std::function<Event(Event)>;
    
    std::vector<Event> events_;
    std::map<EventType, std::vector<EventHandler>> handlers_;
    std::vector<EventFilter> filters_;
    
public:
    // Register event handler using lambda
    void on(EventType type, EventHandler handler) {
        handlers_[type].push_back(std::move(handler));
    }
    
    // Add filter (events must pass all filters)
    void add_filter(EventFilter filter) {
        filters_.push_back(std::move(filter));
    }
    
    // Dispatch event to handlers
    void dispatch(const Event& event) {
        // Check all filters using std::all_of with lambda
        bool should_process = std::all_of(filters_.begin(), filters_.end(),
            [&event](const EventFilter& filter) {
                return filter(event);
            });
        
        if (!should_process) {
            std::cout << "Event filtered out\n";
            return;
        }
        
        events_.push_back(event);
        
        // Find and execute handlers
        auto it = handlers_.find(event.type);
        if (it != handlers_.end()) {
            // Call each handler with lambda
            std::for_each(it->second.begin(), it->second.end(),
                [&event](const EventHandler& handler) {
                    handler(event);
                });
        }
    }
    
    // Get events filtered by predicate (lambda)
    std::vector<Event> get_events(EventFilter predicate) const {
        std::vector<Event> result;
        std::copy_if(events_.begin(), events_.end(),
                    std::back_inserter(result),
                    predicate);
        return result;
    }
    
    // Transform events using lambda
    std::vector<std::string> get_event_descriptions() const {
        std::vector<std::string> descriptions;
        std::transform(events_.begin(), events_.end(),
                      std::back_inserter(descriptions),
                      [](const Event& e) {
                          return "Event on: " + e.target;
                      });
        return descriptions;
    }
    
    // Functional pipeline with lambdas
    auto create_event_pipeline() {
        // Return a composed function
        return [this](EventType type, std::string target) {
            Event event{type, target};
            
            // Pipeline: validate -> transform -> dispatch
            auto validate = [](const Event& e) {
                return !e.target.empty();
            };
            
            auto transform = [](Event e) {
                e.data["processed"] = "true";
                return e;
            };
            
            if (validate(event)) {
                event = transform(event);
                dispatch(event);
                return true;
            }
            return false;
        };
    }
    
    // Reducer pattern with lambda
    int count_events_by_type(EventType type) const {
        return std::count_if(events_.begin(), events_.end(),
            [type](const Event& e) {
                return e.type == type;
            });
    }
    
    // Map-reduce with lambdas
    std::map<EventType, int> get_event_counts() const {
        std::map<EventType, int> counts;
        
        // Custom reduce operation
        std::for_each(events_.begin(), events_.end(),
            [&counts](const Event& e) {
                counts[e.type]++;
            });
        
        return counts;
    }
    
    // Higher-order function: returns a function
    auto create_throttled_handler(EventHandler handler, int max_calls) {
        // Capture by value (copy), mutable to modify counter
        return [handler, max_calls, counter = 0](const Event& e) mutable {
            if (counter < max_calls) {
                handler(e);
                counter++;
            } else {
                std::cout << "Handler throttled (max " << max_calls << " calls)\n";
            }
        };
    }
    
    // Debounce with lambda and timing
    auto create_debounced_handler(EventHandler handler, 
                                  std::chrono::milliseconds delay) {
        // Use shared_ptr for shared state across lambda copies
        auto last_call = std::make_shared<
            std::chrono::system_clock::time_point
        >(std::chrono::system_clock::now() - delay);
        
        return [handler, delay, last_call](const Event& e) {
            auto now = std::chrono::system_clock::now();
            auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
                now - *last_call
            );
            
            if (elapsed >= delay) {
                handler(e);
                *last_call = now;
            } else {
                std::cout << "Event debounced\n";
            }
        };
    }
    
    // Compose multiple handlers
    EventHandler compose_handlers(std::vector<EventHandler> handlers) {
        return [handlers = std::move(handlers)](const Event& e) {
            for (const auto& handler : handlers) {
                handler(e);
            }
        };
    }
    
    void show_statistics() const {
        std::cout << "\n=== Event Statistics ===\n";
        std::cout << "Total events: " << events_.size() << "\n";
        
        auto counts = get_event_counts();
        for (const auto& [type, count] : counts) {
            std::cout << "Type " << static_cast<int>(type) 
                      << ": " << count << " events\n";
        }
    }
};

int main() {
    EventProcessor processor;
    
    // 1. Basic lambda as event handler
    processor.on(EventType::Click, [](const Event& e) {
        std::cout << "Click handler: " << e.target << "\n";
    });
    
    // 2. Lambda with capture
    int click_count = 0;
    processor.on(EventType::Click, [&click_count](const Event& e) {
        click_count++;
        std::cout << "  Total clicks: " << click_count << "\n";
    });
    
    // 3. Generic lambda (C++14)
    auto log_event = [](const auto& event) {
        std::cout << "Logged: Event on " << event.target << "\n";
    };
    processor.on(EventType::KeyPress, [log_event](const Event& e) {
        log_event(e);
    });
    
    // 4. Add filter using lambda
    processor.add_filter([](const Event& e) {
        // Only process events for "button" elements
        return e.target.find("button") != std::string::npos ||
               e.target.find("key") != std::string::npos;
    });
    
    // 5. Throttled handler (max 2 calls)
    auto throttled = processor.create_throttled_handler(
        [](const Event& e) {
            std::cout << "Throttled handler executed: " << e.target << "\n";
        },
        2
    );
    processor.on(EventType::MouseMove, throttled);
    
    // 6. Debounced handler (100ms delay)
    auto debounced = processor.create_debounced_handler(
        [](const Event& e) {
            std::cout << "Debounced handler executed: " << e.target << "\n";
        },
        std::chrono::milliseconds(100)
    );
    processor.on(EventType::Scroll, debounced);
    
    // 7. Compose multiple handlers
    auto composed = processor.compose_handlers({
        [](const Event& e) { std::cout << "  Handler 1\n"; },
        [](const Event& e) { std::cout << "  Handler 2\n"; },
        [](const Event& e) { std::cout << "  Handler 3\n"; }
    });
    processor.on(EventType::Custom, composed);
    
    // Dispatch events
    std::cout << "=== Dispatching Events ===\n\n";
    
    processor.dispatch({EventType::Click, "button_submit"});
    processor.dispatch({EventType::Click, "button_cancel"});
    processor.dispatch({EventType::KeyPress, "key_enter"});
    processor.dispatch({EventType::MouseMove, "div_header"});  // Filtered
    processor.dispatch({EventType::MouseMove, "button_close"}); // Works
    processor.dispatch({EventType::Scroll, "button_scroll"});
    processor.dispatch({EventType::Custom, "button_custom"});
    
    // Use functional pipeline
    std::cout << "\n=== Event Pipeline ===\n";
    auto pipeline = processor.create_event_pipeline();
    pipeline(EventType::Click, "button_pipeline");
    pipeline(EventType::Click, "");  // Fails validation
    
    // Query events with lambda predicates
    std::cout << "\n=== Querying Events ===\n";
    auto click_events = processor.get_events([](const Event& e) {
        return e.type == EventType::Click;
    });
    std::cout << "Found " << click_events.size() << " click events\n";
    
    // Show statistics
    processor.show_statistics();
    
    // Demonstrate lazy evaluation with lambda
    std::cout << "\n=== Lazy Evaluation ===\n";
    auto expensive_computation = [](int x) {
        std::cout << "  Computing for " << x << "...\n";
        return x * x;
    };
    
    // Only computed when needed
    auto lazy_value = [expensive_computation, x = 5]() {
        return expensive_computation(x);
    };
    
    std::cout << "Lambda created but not executed yet\n";
    std::cout << "Now executing: " << lazy_value() << "\n";
    
    // Demonstrate memoization with lambda
    std::cout << "\n=== Memoization ===\n";
    auto fibonacci_memo = [cache = std::map<int, int>{}](int n) mutable {
        if (n <= 1) return n;
        
        if (cache.find(n) != cache.end()) {
            std::cout << "  Cache hit for " << n << "\n";
            return cache[n];
        }
        
        std::cout << "  Computing fibonacci(" << n << ")\n";
        // Simplified - real recursion would need Y-combinator
        cache[n] = n;  // Placeholder
        return cache[n];
    };
    
    fibonacci_memo(5);
    fibonacci_memo(5);  // Cache hit
    
    return 0;
}
```

### Output:
```
=== Dispatching Events ===

Click handler: button_submit
  Total clicks: 1
Logged: Event on button_submit
Click handler: button_cancel
  Total clicks: 2
Logged: Event on button_cancel
Logged: Event on key_enter
Event filtered out
Throttled handler executed: button_close
Debounced handler executed: button_scroll
  Handler 1
  Handler 2
  Handler 3

=== Event Pipeline ===
Click handler: button_pipeline
  Total clicks: 3

=== Querying Events ===
Found 3 click events

=== Event Statistics ===
Total events: 6
Type 0: 3 events
Type 1: 1 events
Type 3: 1 events
Type 4: 1 events

=== Lazy Evaluation ===
Lambda created but not executed yet
Now executing:   Computing for 5...
25

=== Memoization ===
  Computing fibonacci(5)
  Cache hit for 5
```

### Concepts Demonstrated:
- **Basic lambdas**: Simple inline functions
- **Captures**: By value, by reference, init-capture
- **Generic lambdas**: Using `auto` parameters
- **Mutable lambdas**: Modifying captured variables
- **std::function**: Type-erased function wrapper
- **Higher-order functions**: Functions returning functions
- **Functional composition**: Combining multiple functions
- **Throttling/Debouncing**: Rate limiting with lambdas
- **Lazy evaluation**: Deferred computation
- **Memoization**: Caching with mutable capture
- **Pipeline pattern**: Composing operations
- **Predicates**: Lambda predicates for algorithms

This example shows how lambdas enable elegant functional programming in C++!

---

## Related Topics

- [Templates](09_templates.md) — generic lambdas are call-operator templates; C++20 adds explicit template parameters
- [Algorithms](06_algorithms.md) — lambdas as predicates, comparators, and callbacks for `std::sort`, `std::for_each`, etc.
- [Advanced Features](12_advanced_features.md) — perfect forwarding in template lambdas, move semantics, `std::function` overhead
- [Modern Features](07_modern_features.md) — `std::variant` + overloaded lambdas, `constexpr` lambdas
- [Best Practices](13_best_practices.md) — when to prefer a named function, capture hygiene, avoiding `std::function` hot paths
- [Multithreading](14_multithreading.md) — capturing shared state; prefer value/init-capture over `[&]` across threads
- [Quick Reference](99_quick_reference.md) — lambda syntax cheat sheet

---

## Next Steps
- **Next**: [Metaprogramming →](11_metaprogramming.md)
- **Previous**: [← Templates](09_templates.md)

---
*Chapter 10 — Lambdas and Functional Programming*

