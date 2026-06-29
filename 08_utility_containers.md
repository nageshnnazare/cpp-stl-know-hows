# Utility Containers and Types

## Overview

Beyond the standard [sequence](01_sequence_containers.md) and [associative](02_associative_containers.md) containers, C++ provides several utility types for special purposes: strings, optional values, variants, tuples, and non-owning views like [`std::span`](07_modern_features.md).

```
┌──────────────────────────────────────────────────────────┐
│              UTILITY TYPES AND CONTAINERS                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  STRING TYPES     │  OPTIONAL TYPES      │  TUPLE TYPES  │
│  ────────────     │  ──────────────      │  ───────────  │
│  • string         │  • optional          │  • pair       │
│  • string_view    │  • variant           │  • tuple      │
│                   │  • any               │               │
│                                                          │
│  BIT MANIPULATION │  SMART WRAPPERS      │               │
│  ──────────────── │  ──────────────      │               │
│  • bitset         │  • reference_wrapper |               │
│                   │  • function          |               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## std::string

### Basic String Operations
```cpp
#include <string>
#include <iostream>

int main() {
    // Construction
    std::string s1;                         // Empty
    std::string s2 = "Hello";               // From literal
    std::string s3(5, 'a');                 // "aaaaa"
    std::string s4(s2);                     // Copy
    std::string s5(s2, 1, 3);               // Substring: "ell"
    
    // Assignment
    std::string s;
    s = "Hello";
    s = s2;
    s.assign("World");
    
    // Concatenation (s1 is still empty here, s2 == "Hello")
    std::string result = s1 + " " + s2;     // " Hello"
    s1 += " World";                         // Append -> s1 == " World"
    s1.append(" !");                        // Append method
    
    // Access
    char c1 = s1[0];                        // No bounds check
    char c2 = s1.at(1);                     // With bounds check
    char& first = s1.front();               // First char
    char& last = s1.back();                 // Last char
    const char* cstr = s1.c_str();          // C-string
    
    // Size
    size_t len = s1.size();                 // Length
    size_t length = s1.length();            // Same as size()
    size_t cap = s1.capacity();             // Allocated capacity
    bool empty = s1.empty();                // Check if empty
    
    // Modification
    s1.push_back('!');                      // Add char
    s1.pop_back();                          // Remove last char
    s1.insert(5, " there");                 // Insert at position
    s1.erase(5, 6);                         // Erase 6 chars from pos 5
    s1.replace(0, 5, "Hi");                 // Replace substring
    s1.clear();                             // Clear string
    
    return 0;
}
```

### String Search Operations
```cpp
#include <string>

int main() {
    std::string s = "Hello, World! Hello again!";
    
    // Find substring
    size_t pos1 = s.find("World");          // 7
    size_t pos2 = s.find("xyz");            // std::string::npos (not found)
    
    // Find character
    size_t pos3 = s.find('o');              // 4 (first occurrence)
    size_t pos4 = s.find('o', 5);           // 8 (starting from index 5)
    
    // Reverse find
    size_t pos5 = s.rfind("Hello");         // 14 (last occurrence)
    
    // Find first of any character
    size_t pos6 = s.find_first_of("aeiou"); // 1 ('e')
    
    // Find first not of
    size_t pos7 = s.find_first_not_of("Hello"); // 5 (',')
    
    // Find last of
    size_t pos8 = s.find_last_of("!");      // 25
    
    // Check if found
    if (pos1 != std::string::npos) {
        std::cout << "Found at position " << pos1 << '\n';
    }
    
    return 0;
}
```

### String Comparison
```cpp
#include <string>

int main() {
    std::string s1 = "apple";
    std::string s2 = "banana";
    
    // Comparison operators
    bool eq = (s1 == s2);                   // false
    bool ne = (s1 != s2);                   // true
    bool lt = (s1 < s2);                    // true (lexicographic)
    bool gt = (s1 > s2);                    // false
    
    // Compare method
    int cmp = s1.compare(s2);               // < 0 (s1 < s2)
    // Returns: < 0 if s1 < s2
    //          0 if s1 == s2
    //          > 0 if s1 > s2
    
    // Compare substring
    int cmp2 = s1.compare(0, 3, s2, 0, 3);  // Compare "app" with "ban"
    
    return 0;
}
```

### String Conversion
```cpp
#include <string>

int main() {
    // String to number
    std::string s_int = "42";
    int i = std::stoi(s_int);               // 42
    
    std::string s_long = "1234567890";
    long l = std::stol(s_long);
    
    std::string s_float = "3.14";
    float f = std::stof(s_float);
    
    std::string s_double = "2.71828";
    double d = std::stod(s_double);
    
    // Number to string
    std::string from_int = std::to_string(42);      // "42"
    std::string from_double = std::to_string(3.14); // "3.140000"
    
    // Custom formatting (C++20)
    #if __cplusplus >= 202002L
    std::string formatted = std::format("{:.2f}", 3.14159); // "3.14"
    #endif
    
    return 0;
}
```

### String Substrings
```cpp
#include <string>

int main() {
    std::string s = "Hello, World!";
    
    // Extract substring
    std::string sub1 = s.substr(7);         // "World!" (from pos 7 to end)
    std::string sub2 = s.substr(7, 5);      // "World" (5 chars from pos 7)
    
    // Copy to buffer
    char buffer[10];
    size_t copied = s.copy(buffer, 5, 0);   // Copy 5 chars from pos 0
    buffer[copied] = '\0';                  // Null-terminate
    
    return 0;
}
```

### String Performance
```cpp
#include <string>
#include <sstream>

int main() {
    // BAD: Multiple concatenations (many allocations)
    std::string s;
    for (int i = 0; i < 1000; ++i) {
        s += std::to_string(i) + " ";       // Many reallocations!
    }
    
    // BETTER: Reserve space
    std::string s2;
    s2.reserve(10000);                      // Pre-allocate
    for (int i = 0; i < 1000; ++i) {
        s2 += std::to_string(i) + " ";
    }
    
    // BEST: Use stringstream for many concatenations
    std::ostringstream oss;
    for (int i = 0; i < 1000; ++i) {
        oss << i << " ";
    }
    std::string s3 = oss.str();
    
    return 0;
}
```

---

## std::string_view (C++17)

### What is string_view?

Non-owning view of a string. Like a pointer + length, but safer.

```cpp
#include <string_view>
#include <string>

// BAD: Takes string by value (copies!)
void process_bad(std::string s) {
    // ...
}

// BETTER: Takes string by const reference
void process_better(const std::string& s) {
    // ...
}

// BEST: Takes string_view (works with any string-like type)
void process_best(std::string_view sv) {
    // Works with: std::string, const char*, string literals, etc.
    std::cout << sv << '\n';
}

int main() {
    std::string s = "Hello";
    const char* cs = "World";
    
    process_best(s);           // OK
    process_best(cs);          // OK
    process_best("Literal");   // OK
    
    // All without copying!
    
    return 0;
}
```

### string_view Operations
```cpp
#include <string_view>

int main() {
    std::string_view sv = "Hello, World!";
    
    // Access
    char c = sv[0];
    char c2 = sv.at(1);
    char first = sv.front();
    char last = sv.back();
    const char* data = sv.data();
    
    // Size
    size_t len = sv.size();
    size_t length = sv.length();
    bool empty = sv.empty();
    
    // Substring
    std::string_view sub = sv.substr(7, 5);  // "World"
    
    // Modify view (not underlying data!)
    sv.remove_prefix(7);                     // Skip first 7 chars
    // sv now views "World!"
    
    sv.remove_suffix(1);                     // Skip last char
    // sv now views "World"
    
    // Search (same as std::string)
    size_t pos = sv.find("or");
    
    // Comparison
    bool eq = (sv == "World");
    
    return 0;
}
```

### string_view Pitfalls
```cpp
#include <string_view>
#include <string>

// DANGER: Dangling string_view
std::string_view bad_function() {
    std::string s = "temporary";
    return s;  // BAD: Returns view to temporary!
}  // s destroyed, string_view dangles!

// SAFE: Return string (owns data)
std::string good_function() {
    std::string s = "temporary";
    return s;  // OK: Returns owned data
}

int main() {
    // DANGER: Temporary string
    std::string_view sv = std::string("temp");  // BAD!
    // Temporary destroyed, sv dangles
    
    // SAFE: String outlives view
    std::string s = "safe";
    std::string_view sv2 = s;  // OK: s outlives sv2
    
    return 0;
}
```

### When to Use string_view
```
✓ Function parameters (read-only string access)
✓ Parsing without allocations
✓ Substring operations without copying
✗ Storing strings (use std::string)
✗ When you need null-termination guarantee — data() is NOT guaranteed '\0'-terminated
✗ Returning from functions (usually) — same dangling risk as [views](07_modern_features.md)
```

⚠️ **Gotchas**: `sv.data()` points at the first character but `sv[size()]` is not required to be `'\0'`. APIs expecting C strings (`fopen`, legacy C libraries) need an owned `std::string` or a copied buffer. [`std::format`](07_modern_features.md) and most modern APIs accept `string_view` directly.

---

## std::optional (C++17)

### What is optional?

Represents a value that may or may not exist. Better than using pointers or special values like -1. For error signaling without a value, see [Exceptions](17_exceptions.md).

```cpp
#include <optional>
#include <string>

// BAD: Use special value for "not found"
int find_bad(const std::vector<int>& vec, int target) {
    // ...
    return -1;  // -1 means "not found", but what if -1 is valid?
}

// GOOD: Use optional
std::optional<int> find_good(const std::vector<int>& vec, int target) {
    for (size_t i = 0; i < vec.size(); ++i) {
        if (vec[i] == target) {
            return i;  // Return value
        }
    }
    return std::nullopt;  // Return "no value"
}

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    auto result = find_good(vec, 3);
    
    if (result) {  // or: result.has_value()
        std::cout << "Found at index: " << *result << '\n';
    } else {
        std::cout << "Not found\n";
    }
    
    return 0;
}
```

⚠️ **Gotchas**: `optional<T&>` (optional reference) is **not** part of standard C++ through C++23; it is proposed for C++26. Until then, use `T*` or `std::reference_wrapper<T>` when you need optional aliasing. Dereferencing an empty optional (`*opt`, `opt.value()`) is undefined behavior or throws `std::bad_optional_access`, respectively.

### optional Operations
```cpp
#include <optional>
#include <string>

int main() {
    // Construction
    std::optional<int> opt1;                // Empty
    std::optional<int> opt2 = std::nullopt; // Empty (explicit)
    std::optional<int> opt3 = 42;           // Contains 42
    std::optional<int> opt4{42};            // Contains 42
    
    // Assignment
    opt1 = 42;                              // Now contains 42
    opt1 = std::nullopt;                    // Now empty
    
    // Check if has value
    if (opt3.has_value()) {
        std::cout << "Has value\n";
    }
    
    if (opt3) {  // Implicit bool conversion
        std::cout << "Has value\n";
    }
    
    // Access value
    int val1 = *opt3;                       // Dereference (UB if empty!)
    int val2 = opt3.value();                // Throws if empty
    int val3 = opt3.value_or(0);            // Returns 0 if empty
    
    // Emplace (construct in-place)
    std::optional<std::string> opt_str;
    opt_str.emplace("Hello");               // Construct string in-place
    
    // Reset (make empty)
    opt3.reset();                           // Now empty
    opt3 = std::nullopt;                    // Same
    
    // Swap
    std::optional<int> a = 1, b = 2;
    a.swap(b);                              // a=2, b=1
    
    return 0;
}
```

### optional with Complex Types
```cpp
#include <optional>
#include <string>

struct Person {
    std::string name;
    int age;
};

std::optional<Person> find_person(const std::string& name) {
    if (name == "Alice") {
        return Person{"Alice", 25};
    }
    return std::nullopt;
}

int main() {
    auto person = find_person("Alice");
    
    if (person) {
        std::cout << person->name << " is " << person->age << '\n';
        // Arrow operator works!
    }
    
    // Transform optional
    auto age = person.and_then([](const Person& p) -> std::optional<int> {
        return p.age;
    });
    
    return 0;
}
```

### Monadic Operations (C++23)
```cpp
#include <optional>

int main() {
    std::optional<int> opt = 42;
    
    // and_then: Chain operations that return optional
    auto result1 = opt.and_then([](int x) -> std::optional<int> {
        return x > 0 ? std::optional{x * 2} : std::nullopt;
    });
    
    // transform: Apply function to value
    auto result2 = opt.transform([](int x) {
        return x * 2;  // Returns optional<int> automatically
    });
    
    // or_else: Provide alternative if empty
    auto result3 = opt.or_else([]() -> std::optional<int> {
        return 0;
    });
    
    return 0;
}
```

---

## std::variant (C++17)

### What is variant?

Type-safe union. Can hold one of several types at a time — unlike C `union`, it manages object lifetimes and works with non-trivial types like `std::string`. See [`std::any`](#stdany-c17) when the type set is unknown at compile time.

```cpp
#include <variant>
#include <string>

int main() {
    // Variant can hold int, double, or string
    std::variant<int, double, std::string> var;
    
    var = 42;                               // Holds int
    var = 3.14;                             // Now holds double
    var = "Hello";                          // Now holds string
    
    // Check which type it holds
    if (std::holds_alternative<int>(var)) {
        std::cout << "Holds int\n";
    }
    
    // Get index of current type
    size_t idx = var.index();               // 0=int, 1=double, 2=string
    
    // Access value (throws std::bad_variant_access if wrong type)
    int i = std::get<int>(var);             // Throws if not int
    
    // Safe access (returns pointer or nullptr)
    if (auto* p = std::get_if<std::string>(&var)) {
        std::cout << "String: " << *p << '\n';
    }
    
    return 0;
}
```

### std::monostate — Default-Constructible variant

A `std::variant` with no default-constructible alternative cannot be default-constructed. Insert `std::monostate` as the first alternative to represent "empty" or "none":

```cpp
#include <variant>

std::variant<std::monostate, int, std::string> v;  // Default: holds monostate
v = 42;
if (std::holds_alternative<std::monostate>(v)) { /* empty */ }
```

### Visiting Variants
```cpp
#include <variant>
#include <string>
#include <iostream>

int main() {
    std::variant<int, double, std::string> var = 3.14;
    
    // std::visit: Apply visitor to variant
    std::visit([](auto&& arg) {
        using T = std::decay_t<decltype(arg)>;
        if constexpr (std::is_same_v<T, int>) {
            std::cout << "int: " << arg << '\n';
        } else if constexpr (std::is_same_v<T, double>) {
            std::cout << "double: " << arg << '\n';
        } else if constexpr (std::is_same_v<T, std::string>) {
            std::cout << "string: " << arg << '\n';
        }
    }, var);
    
    // Cleaner with overloaded struct
    struct Visitor {
        void operator()(int i) { std::cout << "int: " << i << '\n'; }
        void operator()(double d) { std::cout << "double: " << d << '\n'; }
        void operator()(const std::string& s) { std::cout << "string: " << s << '\n'; }
    };
    
    std::visit(Visitor{}, var);
    
    return 0;
}
```

### Variant Use Cases
```cpp
#include <variant>
#include <vector>

// Example: Heterogeneous container
using Value = std::variant<int, double, std::string>;

int main() {
    std::vector<Value> values;
    values.push_back(42);
    values.push_back(3.14);
    values.push_back("Hello");
    
    for (const auto& val : values) {
        std::visit([](const auto& v) {
            std::cout << v << ' ';
        }, val);
    }
    
    return 0;
}
```

---

## std::any (C++17)

### What is any?

Type-safe container for any single value of any type.

```cpp
#include <any>
#include <string>
#include <iostream>

int main() {
    std::any a;
    
    // Can hold any type
    a = 42;
    a = 3.14;
    a = std::string("Hello");
    a = std::vector<int>{1, 2, 3};
    
    // Check if has value
    if (a.has_value()) {
        std::cout << "Has value\n";
    }
    
    // Get type
    std::cout << "Type: " << a.type().name() << '\n';
    
    // Access value (throws if wrong type)
    try {
        int i = std::any_cast<int>(a);      // Throws: holds vector
    } catch (const std::bad_any_cast& e) {
        std::cout << "Wrong type!\n";
    }
    
    // Safe access
    if (auto* p = std::any_cast<std::vector<int>>(&a)) {
        std::cout << "Vector size: " << p->size() << '\n';
    }
    
    // Reset
    a.reset();                              // Now empty
    
    return 0;
}
```

### any vs variant
```
┌────────────────────────────────────────────────────────┐
│              std::any vs std::variant                  │
├────────────────────────────────────────────────────────┤
│                  std::any    │    std::variant         │
├──────────────────────────────┼─────────────────────────┤
│ Types            Any type    │    Fixed set of types   │
│ Type safety      Runtime     │    Compile-time         │
│ Performance      Slower      │    Faster               │
│ Memory           Heap alloc* │    Stack                │
│ Visitation       No          │    Yes (std::visit)     │
│                                                        │
│ Use any when: Unknown types at compile time            │
│ Use variant when: Known set of types                   │
└────────────────────────────────────────────────────────┘
* Small object optimization may avoid heap
```

---

## std::pair and std::tuple

### std::pair
```cpp
#include <utility>
#include <string>
#include <map>
#include <iostream>

int main() {
    // Construction
    std::pair<int, std::string> p1(42, "answer");
    std::pair<int, std::string> p2 = {42, "answer"};
    auto p3 = std::make_pair(42, "answer");
    
    // Access
    int first = p1.first;
    std::string second = p1.second;
    
    // C++17: Structured bindings
    auto [number, text] = p1;
    std::cout << number << ": " << text << '\n';
    
    // Comparison (lexicographic)
    std::pair<int, int> a(1, 2), b(1, 3);
    bool less = a < b;  // true (compares first, then second)
    
    // Common use: map iterators — see Associative Containers
    std::map<int, std::string> m = { {1, "one"}, {2, "two"} };
    for (const auto& [key, value] : m) {  // Structured binding
        std::cout << key << ": " << value << '\n';
    }
    
    return 0;
}
```

### std::tuple
```cpp
#include <tuple>
#include <string>

int main() {
    // Construction
    std::tuple<int, double, std::string> t1(42, 3.14, "hello");
    auto t2 = std::make_tuple(42, 3.14, "hello");
    
    // Access by index
    int i = std::get<0>(t1);                // 42
    double d = std::get<1>(t1);             // 3.14
    std::string s = std::get<2>(t1);        // "hello"
    
    // Access by type (if unique)
    int i2 = std::get<int>(t1);             // 42
    std::string s2 = std::get<std::string>(t1); // "hello"
    
    // C++17: Structured bindings
    auto [num, pi, text] = t1;
    
    // Size
    constexpr size_t size = std::tuple_size_v<decltype(t1)>;  // 3
    
    // Concatenate tuples
    auto t3 = std::tuple_cat(t1, std::make_tuple(true));
    // t3 is tuple<int, double, string, bool>
    
    // Apply function to tuple
    auto sum = std::apply([](int a, double b, const std::string&) {
        return a + b;
    }, t1);  // 45.14
    
    return 0;
}
```

### Tuple Use Cases
```cpp
#include <tuple>

// Return multiple values
std::tuple<bool, int, std::string> divide(int a, int b) {
    if (b == 0) {
        return {false, 0, "Division by zero"};
    }
    return {true, a / b, "Success"};
}

int main() {
    auto [success, result, message] = divide(10, 2);
    
    if (success) {
        std::cout << "Result: " << result << '\n';
    } else {
        std::cout << "Error: " << message << '\n';
    }
    
    return 0;
}
```

---

## std::bitset

### Basic Bitset Operations
```cpp
#include <bitset>
#include <iostream>

int main() {
    // Construction
    std::bitset<8> b1;                      // 00000000
    std::bitset<8> b2(42);                  // 00101010
    std::bitset<8> b3("10101010");          // From string
    
    // Set/reset bits
    b1.set();                               // Set all: 11111111
    b1.set(3);                              // Set bit 3: 11111111
    b1.reset();                             // Clear all: 00000000
    b1.reset(3);                            // Clear bit 3
    b1.flip();                              // Flip all bits
    b1.flip(3);                             // Flip bit 3
    
    // Access bits
    bool bit = b2[0];                       // LSB (rightmost)
    bool test = b2.test(3);                 // Throws if out of range
    
    // Bitwise operations
    std::bitset<8> a(0b11110000);
    std::bitset<8> b(0b10101010);
    
    auto and_result = a & b;                // 10100000
    auto or_result = a | b;                 // 11111010
    auto xor_result = a ^ b;                // 01011010
    auto not_result = ~a;                   // 00001111
    
    // Shift
    auto left = a << 2;                     // 11000000
    auto right = a >> 2;                    // 00111100
    
    // Count
    size_t count = b2.count();              // Number of 1s
    size_t size = b2.size();                // Total bits (8)
    bool all = b2.all();                    // All bits set?
    bool any = b2.any();                    // Any bit set?
    bool none = b2.none();                  // No bits set?
    
    // Convert
    unsigned long ul = b2.to_ulong();       // 42
    unsigned long long ull = b2.to_ullong();
    std::string str = b2.to_string();       // "00101010"
    
    return 0;
}
```

### Bitset Use Cases
```cpp
#include <bitset>

// Flags
enum class Permissions : uint8_t {
    Read = 0,
    Write = 1,
    Execute = 2
};

using PermissionSet = std::bitset<3>;

int main() {
    PermissionSet perms;
    
    // Set permissions
    perms.set(static_cast<size_t>(Permissions::Read));
    perms.set(static_cast<size_t>(Permissions::Write));
    
    // Check permission
    if (perms.test(static_cast<size_t>(Permissions::Read))) {
        std::cout << "Has read permission\n";
    }
    
    // Sieve of Eratosthenes (prime numbers)
    const int N = 100;
    std::bitset<N> is_prime;
    is_prime.set();  // Assume all prime initially
    is_prime[0] = is_prime[1] = 0;  // 0 and 1 not prime
    
    for (int i = 2; i * i < N; ++i) {
        if (is_prime[i]) {
            for (int j = i * i; j < N; j += i) {
                is_prime[j] = 0;  // Mark multiples as not prime
            }
        }
    }
    
    std::cout << "Primes < 100: " << is_prime.count() << '\n';
    
    return 0;
}
```

---

## std::span (C++20)

[`std::span`](https://en.cppreference.com/w/cpp/container/span) is a **non-owning** view over contiguous memory — the numeric/array analogue of `string_view`. Full coverage of ranges views and `mdspan` is in [Modern C++20/23 Features](07_modern_features.md); here is the utility-type perspective.

```cpp
#include <span>
#include <vector>
#include <array>

void scale(std::span<int> data, int factor) {
    for (int& x : data) x *= factor;
}

int main() {
    std::vector<int> vec = {1, 2, 3};
    std::array<int, 3> arr = {4, 5, 6};
    int raw[] = {7, 8, 9};

    scale(vec, 2);   // Works with vector, array, C array
    scale(arr, 2);
    scale(raw, 2);

    std::span s(vec);
    auto tail = s.subspan(1);  // Non-owning subrange — vec must stay alive
    return 0;
}
```

| Type | Owns data? | Contiguous? | Typical use |
|------|------------|-------------|-------------|
| `std::string` | Yes | Yes | Store text |
| `std::string_view` | No | Yes | Read-only string parameter |
| `std::span<T>` | No | Yes | Read/write buffer parameter |
| `std::vector<T>` | Yes | Yes | Growable owning storage |

⚠️ **Gotchas**: A `span` does not extend the lifetime of its source. Returning `span` over a local `vector` dangling is the same mistake as returning `string_view` over a temporary string.

---

## std::reference_wrapper

### What is reference_wrapper?

Allows storing references in containers (containers can't store raw references).

```cpp
#include <functional>
#include <vector>
#include <iostream>

int main() {
    int a = 1, b = 2, c = 3;
    
    // Can't do this:
    // std::vector<int&> refs;  // ERROR: Can't store references
    
    // Can do this:
    std::vector<std::reference_wrapper<int>> refs;
    refs.push_back(std::ref(a));
    refs.push_back(std::ref(b));
    refs.push_back(std::ref(c));
    
    // Modify through references
    for (int& r : refs) {
        r *= 2;
    }
    
    std::cout << a << ' ' << b << ' ' << c << '\n';  // 2 4 6
    
    // Use with algorithms
    std::vector<int> values = {10, 20, 30};
    std::sort(refs.begin(), refs.end());  // Sort references
    
    return 0;
}
```

---

## std::function

### What is std::function?

Type-erased wrapper for any callable (function, [lambda](10_lambdas.md), functor). Prefer a concrete function type or template when you do not need type erasure — `std::function` may allocate.

```cpp
#include <functional>
#include <iostream>

int add(int a, int b) {
    return a + b;
}

int main() {
    // Store function pointer
    std::function<int(int, int)> f1 = add;
    std::cout << f1(2, 3) << '\n';  // 5
    
    // Store lambda
    std::function<int(int, int)> f2 = [](int a, int b) {
        return a * b;
    };
    std::cout << f2(2, 3) << '\n';  // 6
    
    // Store functor
    struct Multiplier {
        int operator()(int a, int b) const { return a * b; }
    };
    std::function<int(int, int)> f3 = Multiplier{};
    std::cout << f3(2, 3) << '\n';  // 6
    
    // Store member function (with std::bind)
    struct Calculator {
        int add(int a, int b) { return a + b; }
    };
    Calculator calc;
    std::function<int(int, int)> f4 = 
        [&calc](int a, int b) { return calc.add(a, b); };
    
    // Check if callable
    if (f1) {
        std::cout << "f1 is callable\n";
    }
    
    return 0;
}
```

---

## Comparison Table

```
┌────────────────────────────────────────────────────────────────┐
│                  UTILITY TYPE COMPARISON                       │
├────────────────────────────────────────────────────────────────┤
│ Type             │ Purpose            │ When to Use            │
├──────────────────┼────────────────────┼────────────────────────┤
│ string           │ Owned text         │ Store strings          │
│ string_view      │ String reference   │ Function parameters    │
│ span<T>          │ Buffer reference   │ Contiguous data params │
│ optional<T>      │ Maybe value        │ Optional returns       │
│ variant<Ts...>   │ One of types       │ Known type set         │
│ any              │ Any type           │ Unknown types          │
│ pair<T1,T2>      │ Two values         │ Simple tuples          │
│ tuple<Ts...>     │ Multiple values    │ Multiple returns       │
│ bitset<N>        │ Fixed bits         │ Flags, bit operations  │
│ reference_wrapper│ Store references   │ Refs in containers     │
│ function<Sig>    │ Callable wrapper   │ Callbacks, delegates   │
└────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### 1. Use string_view for Parameters
```cpp
// GOOD: Accepts any string-like type without copying
void process(std::string_view sv) {
    // ...
}

// BAD: Forces copy or forces std::string
void process(std::string s);        // Copy
void process(const char* s);        // Only C-strings
```

### 2. Use optional for Optional Returns
```cpp
// GOOD: Clear intent
std::optional<int> find(const std::vector<int>& vec, int target);

// BAD: Magic values
int find(const std::vector<int>& vec, int target);  // Returns -1 on failure?
```

### 3. Use variant Instead of Union
```cpp
// BAD: Not type-safe
union Data {
    int i;
    double d;
    // Can't have std::string!
};

// GOOD: Type-safe, works with any type
std::variant<int, double, std::string> data;
```

### 4. Use Structured Bindings
```cpp
// Modern and clear
auto [key, value] = *map.find("key");
auto [success, result, error] = try_operation();

// Old style (verbose)
auto it = map.find("key");
auto key = it->first;
auto value = it->second;
```

---

## Common Pitfalls

### 1. string_view Lifetime
```cpp
// DANGER!
std::string_view get_name() {
    std::string temp = "Name";
    return temp;  // Returns view to destroyed string!
}
```

### 2. Accessing Empty optional
```cpp
std::optional<int> opt;
int x = *opt;  // UB! opt is empty
int y = opt.value();  // Throws std::bad_optional_access
```

### 3. Wrong variant Type
```cpp
std::variant<int, double> var = 3.14;
int i = std::get<int>(var);  // Throws std::bad_variant_access
// Use std::get_if or std::visit for safe dispatch
```

---

## Practice Exercises

### Exercise 1: Parse Configuration
```cpp
// Use optional to represent optional config values
struct Config {
    std::string name;
    std::optional<int> port;
    std::optional<std::string> host;
};
```

### Exercise 2: Expression Evaluator
```cpp
// Use variant for different expression types
using Expr = std::variant<int, double, std::string>;
// Implement evaluation with std::visit
```

### Exercise 3: String Processing
```cpp
// Implement efficient string splitting using string_view
std::vector<std::string_view> split(std::string_view sv, char delim);
```

---

## Conclusion

Utility containers and types provide essential building blocks for modern C++ programming:

- **string/string_view**: Efficient text handling
- **optional**: Clear optional value semantics
- **variant**: Type-safe unions
- **tuple/pair**: Multiple value returns
- **bitset**: Efficient bit manipulation

Use these types to write clearer, safer, and more efficient code!

---

## Complete Practical Example: Configuration System

Here's a comprehensive example using all utility containers to build a flexible configuration and data processing system:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <optional>
#include <variant>
#include <any>
#include <tuple>
#include <utility>
#include <bitset>
#include <functional>
#include <vector>
#include <map>
#include <sstream>

// 1. Configuration value that can hold different types (variant)
using ConfigValue = std::variant<int, double, std::string, bool>;

// 2. Parse result using optional
std::optional<ConfigValue> parse_config_value(std::string_view input) {
    // Try parsing as int
    try {
        size_t pos;
        int i_val = std::stoi(std::string(input), &pos);
        if (pos == input.size()) {
            return i_val;
        }
    } catch (...) {}
    
    // Try parsing as double
    try {
        size_t pos;
        double d_val = std::stod(std::string(input), &pos);
        if (pos == input.size()) {
            return d_val;
        }
    } catch (...) {}
    
    // Try parsing as bool
    if (input == "true" || input == "True") return true;
    if (input == "false" || input == "False") return false;
    
    // Default to string
    if (!input.empty()) {
        return std::string(input);
    }
    
    return std::nullopt;  // Parse failed
}

// 3. Configuration manager using multiple utility types
class ConfigManager {
private:
    std::map<std::string, ConfigValue> config_;
    std::bitset<32> feature_flags_;  // Up to 32 boolean flags
    
public:
    // Set config using string_view (efficient parameter passing)
    bool set(std::string_view key, std::string_view value) {
        if (auto parsed = parse_config_value(value)) {
            config_[std::string(key)] = *parsed;
            return true;
        }
        return false;
    }
    
    // Get config with optional return
    std::optional<ConfigValue> get(std::string_view key) const {
        auto it = config_.find(std::string(key));
        if (it != config_.end()) {
            return it->second;
        }
        return std::nullopt;
    }
    
    // Get with default value
    template<typename T>
    T get_or(std::string_view key, T default_value) const {
        if (auto val = get(key)) {
            if (std::holds_alternative<T>(*val)) {
                return std::get<T>(*val);
            }
        }
        return default_value;
    }
    
    // Visit variant to print
    void print_config() const {
        std::cout << "\n=== Configuration ===\n";
        for (const auto& [key, value] : config_) {
            std::cout << key << " = ";
            
            std::visit([](const auto& v) {
                std::cout << v;
            }, value);
            
            std::cout << " (type: ";
            std::visit([](const auto& v) {
                using T = std::decay_t<decltype(v)>;
                if constexpr (std::is_same_v<T, int>) {
                    std::cout << "int";
                } else if constexpr (std::is_same_v<T, double>) {
                    std::cout << "double";
                } else if constexpr (std::is_same_v<T, std::string>) {
                    std::cout << "string";
                } else if constexpr (std::is_same_v<T, bool>) {
                    std::cout << "bool";
                }
            }, value);
            std::cout << ")\n";
        }
    }
    
    // Feature flags using bitset
    void set_flag(size_t index, bool value) {
        if (index < 32) {
            feature_flags_[index] = value;
        }
    }
    
    bool get_flag(size_t index) const {
        return index < 32 && feature_flags_[index];
    }
    
    void print_flags() const {
        std::cout << "\nFeature flags: " << feature_flags_ << "\n";
        std::cout << "Enabled count: " << feature_flags_.count() << "\n";
    }
};

// 4. Function registry using std::function
class FunctionRegistry {
private:
    std::map<std::string, std::function<void()>> callbacks_;
    
public:
    void register_callback(std::string_view name, std::function<void()> callback) {
        callbacks_[std::string(name)] = std::move(callback);
    }
    
    void execute(std::string_view name) {
        auto it = callbacks_.find(std::string(name));
        if (it != callbacks_.end() && it->second) {
            std::cout << "Executing: " << name << "\n";
            it->second();
        } else {
            std::cout << "Callback not found: " << name << "\n";
        }
    }
    
    void execute_all() {
        for (const auto& [name, callback] : callbacks_) {
            if (callback) {
                std::cout << "Executing: " << name << "\n";
                callback();
            }
        }
    }
};

// 5. Data structure with tuple
struct Person {
    std::string name;
    int age;
    std::optional<std::string> email;  // Email is optional
    std::optional<std::string> phone;  // Phone is optional
    
    // Return tuple for multiple values
    std::tuple<bool, std::string> validate() const {
        if (name.empty()) {
            return {false, "Name is required"};
        }
        if (age < 0 || age > 150) {
            return {false, "Invalid age"};
        }
        return {true, "Valid"};
    }
    
    // Get contact info as pair
    std::pair<bool, std::string> get_primary_contact() const {
        if (email) {
            return {true, *email};
        }
        if (phone) {
            return {true, *phone};
        }
        return {false, ""};
    }
};

// 6. Type-erased storage using std::any
class DataStore {
private:
    std::map<std::string, std::any> data_;
    
public:
    template<typename T>
    void store(std::string_view key, T value) {
        data_[std::string(key)] = std::move(value);
    }
    
    template<typename T>
    std::optional<T> retrieve(std::string_view key) const {
        auto it = data_.find(std::string(key));
        if (it != data_.end()) {
            try {
                return std::any_cast<T>(it->second);
            } catch (const std::bad_any_cast&) {
                return std::nullopt;
            }
        }
        return std::nullopt;
    }
    
    void print_types() const {
        std::cout << "\n=== Data Store Types ===\n";
        for (const auto& [key, value] : data_) {
            std::cout << key << ": " << value.type().name() << "\n";
        }
    }
};

// 7. String processing with string_view
std::vector<std::string_view> split(std::string_view sv, char delim) {
    std::vector<std::string_view> result;
    
    while (!sv.empty()) {
        size_t pos = sv.find(delim);
        if (pos == std::string_view::npos) {
            result.push_back(sv);
            break;
        }
        result.push_back(sv.substr(0, pos));
        sv.remove_prefix(pos + 1);
    }
    
    return result;
}

std::string_view trim(std::string_view sv) {
    // Remove leading whitespace
    while (!sv.empty() && std::isspace(sv.front())) {
        sv.remove_prefix(1);
    }
    
    // Remove trailing whitespace
    while (!sv.empty() && std::isspace(sv.back())) {
        sv.remove_suffix(1);
    }
    
    return sv;
}

// Demonstrations
void demo_config_manager() {
    std::cout << "\n--- Configuration Manager ---\n";
    
    ConfigManager config;
    
    config.set("port", "8080");
    config.set("host", "localhost");
    config.set("timeout", "30.5");
    config.set("debug", "true");
    config.set("max_connections", "100");
    
    config.print_config();
    
    // Get with default
    int port = config.get_or<int>("port", 80);
    std::string host = config.get_or<std::string>("host", "0.0.0.0");
    bool debug = config.get_or<bool>("debug", false);
    
    std::cout << "\nExtracted values:\n";
    std::cout << "  Port: " << port << "\n";
    std::cout << "  Host: " << host << "\n";
    std::cout << "  Debug: " << (debug ? "enabled" : "disabled") << "\n";
    
    // Feature flags
    config.set_flag(0, true);   // Feature A
    config.set_flag(1, false);  // Feature B
    config.set_flag(5, true);   // Feature F
    
    config.print_flags();
}

void demo_function_registry() {
    std::cout << "\n--- Function Registry ---\n";
    
    FunctionRegistry registry;
    
    // Register different callable types
    registry.register_callback("startup", []() {
        std::cout << "  System starting up...\n";
    });
    
    int counter = 0;
    registry.register_callback("increment", [&counter]() {
        std::cout << "  Counter: " << ++counter << "\n";
    });
    
    registry.register_callback("shutdown", []() {
        std::cout << "  System shutting down...\n";
    });
    
    // Execute specific callbacks
    registry.execute("startup");
    registry.execute("increment");
    registry.execute("increment");
    registry.execute("shutdown");
}

void demo_optional_handling() {
    std::cout << "\n--- Optional Handling ---\n";
    
    Person p1{"Alice", 30, "alice@example.com", std::nullopt};
    Person p2{"Bob", 25, std::nullopt, "555-1234"};
    Person p3{"Invalid", -5, std::nullopt, std::nullopt};
    
    for (const auto& person : {p1, p2, p3}) {
        auto [valid, message] = person.validate();
        std::cout << person.name << ": " << message << "\n";
        
        if (valid) {
            auto [has_contact, contact] = person.get_primary_contact();
            if (has_contact) {
                std::cout << "  Contact: " << contact << "\n";
            } else {
                std::cout << "  No contact info\n";
            }
        }
    }
}

void demo_type_erased_storage() {
    std::cout << "\n--- Type-Erased Storage ---\n";
    
    DataStore store;
    
    // Store different types
    store.store("count", 42);
    store.store("pi", 3.14159);
    store.store("message", std::string("Hello"));
    store.store("items", std::vector<int>{1, 2, 3, 4, 5});
    
    store.print_types();
    
    // Retrieve with type checking
    if (auto count = store.retrieve<int>("count")) {
        std::cout << "\nCount: " << *count << "\n";
    }
    
    if (auto items = store.retrieve<std::vector<int>>("items")) {
        std::cout << "Items: ";
        for (int x : *items) {
            std::cout << x << " ";
        }
        std::cout << "\n";
    }
    
    // Wrong type - returns nullopt
    if (!store.retrieve<double>("count")) {
        std::cout << "count is not a double\n";
    }
}

void demo_string_processing() {
    std::cout << "\n--- String Processing with string_view ---\n";
    
    std::string data = "apple,banana,cherry,date";
    
    std::cout << "Original: " << data << "\n";
    std::cout << "Split: ";
    for (auto part : split(data, ',')) {
        std::cout << "[" << part << "] ";
    }
    std::cout << "\n";
    
    // Efficient trimming without allocation
    std::string padded = "  hello world  ";
    auto trimmed = trim(padded);
    std::cout << "Trimmed: [" << trimmed << "]\n";
    std::cout << "Original: [" << padded << "]\n";
}

int main() {
    std::cout << "=== Utility Containers Demo ===\n";
    
    // 1. Configuration with variant, optional, bitset
    demo_config_manager();
    
    // 2. Function callbacks with std::function
    demo_function_registry();
    
    // 3. Optional values and tuple returns
    demo_optional_handling();
    
    // 4. Type-erased storage with std::any
    demo_type_erased_storage();
    
    // 5. Efficient string processing with string_view
    demo_string_processing();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **std::string_view**: Efficient string parameter passing
- **std::optional**: Representing optional values
- **std::variant**: Type-safe union for multiple types
- **std::any**: Type-erased storage
- **std::tuple**: Multiple return values
- **std::pair**: Two-value returns
- **std::bitset**: Compact boolean flag storage
- **std::function**: Storing callable objects
- **std::visit**: Pattern matching on variants
- **std::holds_alternative**: Variant type checking
- **std::get**: Extracting variant values
- **Structured bindings**: Unpacking tuples/pairs
- **String trimming**: Efficient string_view manipulation
- **String splitting**: Zero-copy string parsing

This example demonstrates real-world usage of utility containers!

---

## Related Topics

- [Modern C++20/23 Features](07_modern_features.md) — `std::span`, `mdspan`, and range views share the non-owning lifetime model with `string_view`.
- [Sequence Containers](01_sequence_containers.md) — `std::string` and `std::vector` are typical backing stores for views and spans.
- [Templates](09_templates.md) — `std::variant`, `std::optional`, and `std::tuple` are class templates with specialization rules.
- [Lambdas](10_lambdas.md) — visitors for `std::visit`, callbacks stored in `std::function`.
- [Exceptions](17_exceptions.md) — `std::bad_optional_access`, `std::bad_variant_access`, `std::bad_any_cast`.
- [Algorithms](06_algorithms.md) — `std::apply` for tuples; `std::get` patterns mirror variant access.
- [Best Practices](13_best_practices.md) — when to choose `variant` vs inheritance vs `any`.

---

## Next Steps
- **Previous**: [← Modern C++20/23 Features](07_modern_features.md)
- **Back to**: [Main README](README.md)

---
*Chapter 8 — Utility Containers and Types*

