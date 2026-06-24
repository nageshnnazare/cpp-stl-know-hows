# Modern C++20/23 Features

## Overview

C++20 and C++23 introduced revolutionary changes to the STL, with [ranges](https://en.cppreference.com/w/cpp/ranges) being the most significant addition. Ranges provide a more composable, efficient, and expressive way to work with sequences — see also [Iterators](05_iterators.md) and [Algorithms](06_algorithms.md) for the pre-ranges foundation they build on.

```
┌──────────────────────────────────────────────────────────┐
│           C++20/23 MAJOR ADDITIONS TO STL                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  RANGES (C++20)     │  VIEWS (C++20)    │  NEW IN C++23  │
│  ───────────────    │  ──────────────   │  ───────────   │
│  • Concepts         │  • Lazy eval      │  • std::mdspan │
│  • Range algos      │  • Composable     │  • std::flat_* │
│  • Projections      │  • Zero-copy      │  • Ranges++    │
│  • Constrained      │  • Pipeable       │  • More views  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Ranges Library (C++20)

### What is a Range?

A range is anything that you can iterate over - it's a generalization of the iterator pair concept.

```cpp
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Traditional iterator pair
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        std::cout << *it << ' ';
    }
    
    // Range (simpler!)
    for (int x : vec) {  // vec is a range
        std::cout << x << ' ';
    }
    
    // Ranges namespace
    namespace rng = std::ranges;
    
    // Range algorithms
    rng::sort(vec);  // No need for .begin()/.end()!
    
    return 0;
}
```

### Range Concepts
```cpp
#include <ranges>
#include <vector>
#include <list>
#include <forward_list>

template<typename R>
void analyze_range() {
    static_assert(std::ranges::range<R>);  // Basic range
    
    // More specific concepts
    constexpr bool is_sized = std::ranges::sized_range<R>;
    constexpr bool is_common = std::ranges::common_range<R>;
    constexpr bool is_random = std::ranges::random_access_range<R>;
    constexpr bool is_contiguous = std::ranges::contiguous_range<R>;
}

int main() {
    // std::vector satisfies all range concepts
    static_assert(std::ranges::contiguous_range<std::vector<int>>);
    static_assert(std::ranges::random_access_range<std::vector<int>>);
    static_assert(std::ranges::sized_range<std::vector<int>>);
    
    // std::list is bidirectional but not random access
    static_assert(std::ranges::bidirectional_range<std::list<int>>);
    static_assert(!std::ranges::random_access_range<std::list<int>>);
    
    return 0;
}
```

### Range Algorithms
```cpp
#include <ranges>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> vec = {5, 2, 8, 1, 9, 3};
    
    // Traditional STL algorithm
    std::sort(vec.begin(), vec.end());
    
    // Ranges algorithm (cleaner!)
    std::ranges::sort(vec);
    
    // No more begin()/end() pairs!
    std::ranges::reverse(vec);
    
    // Find with ranges
    auto it = std::ranges::find(vec, 5);
    
    // Count with ranges
    auto count = std::ranges::count(vec, 5);
    
    // All algorithms work on ranges
    return 0;
}
```

⚠️ **Gotchas**: Not every range algorithm accepts every range category. `std::ranges::sort` requires a [`random_access_range`](https://en.cppreference.com/w/cpp/ranges/random_access_range) — it will not compile on `std::list` or `std::forward_list`. Use `std::list::sort` member function or copy into a random-access container first. Mutating algorithms that reorder elements (`sort`, `reverse`, `rotate`) generally need stronger iterator requirements than read-only ones (`find`, `count`, `all_of`).

💡 **Hunch**: Range algorithms take the range as the first argument and optional comparison/projection after — the same projection syntax shown later applies here.

---

## Views (C++20)

### What are Views?

Views are **lazy**, **composable** range adaptors (also called adaptors) that don't own data and have O(1) copy/move operations. Chain them with the pipe operator `|`; nothing runs until you iterate or call a consuming algorithm.

```
┌──────────────────────────────────────────────────────┐
│                    VIEW CONCEPT                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Original Container: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] │
│                           │                          │
│                           ▼                          │
│  View (filter even):  [2, 4, 6, 8, 10]               │
│                           │                          │
│                           ▼                          │
│  View (transform *2): [4, 8, 12, 16, 20]             │
│                           │                          │
│                           ▼                          │
│  View (take 3):       [4, 8, 12]                     │
│                                                      │
│  NO COPIES! Views are lazy!                          │
│  Computation happens only when iterating!            │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Basic Views
```cpp
#include <ranges>
#include <vector>
#include <iostream>

namespace rng = std::ranges;
namespace vw = std::views;

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // views::filter - keep elements matching predicate
    auto evens = vec | vw::filter([](int x) { return x % 2 == 0; });
    // evens is a VIEW, not a container
    // No copies made, no memory allocated!
    
    for (int x : evens) {
        std::cout << x << ' ';  // 2 4 6 8 10
    }
    std::cout << '\n';
    
    // views::transform - apply function to each element
    auto squares = vec | vw::transform([](int x) { return x * x; });
    for (int x : squares) {
        std::cout << x << ' ';  // 1 4 9 16 25 36 49 64 81 100
    }
    std::cout << '\n';
    
    // views::take - first N elements
    auto first_5 = vec | vw::take(5);
    for (int x : first_5) {
        std::cout << x << ' ';  // 1 2 3 4 5
    }
    std::cout << '\n';
    
    // views::drop - skip first N elements
    auto skip_5 = vec | vw::drop(5);
    for (int x : skip_5) {
        std::cout << x << ' ';  // 6 7 8 9 10
    }
    std::cout << '\n';
    
    return 0;
}
```

### Composing Views (Pipeable!)
```cpp
#include <ranges>
#include <vector>
#include <iostream>

namespace vw = std::views;

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Compose views with pipe operator |
    auto result = vec 
        | vw::filter([](int x) { return x % 2 == 0; })  // Keep even
        | vw::transform([](int x) { return x * x; })    // Square them
        | vw::take(3);                                   // Take first 3
    
    for (int x : result) {
        std::cout << x << ' ';  // 4 16 36
    }
    std::cout << '\n';
    
    // All operations are LAZY!
    // Computation only happens when iterating
    
    // Complex pipeline
    auto complex = vec
        | vw::filter([](int x) { return x > 3; })
        | vw::transform([](int x) { return x * 2; })
        | vw::reverse
        | vw::take(4);
    
    for (int x : complex) {
        std::cout << x << ' ';  // 20 18 16 14
    }
    
    return 0;
}
```

### All Standard Views
```
┌────────────────────────────────────────────────────────┐
│              STANDARD VIEWS (C++20)                    │
├────────────────────────────────────────────────────────┤
│ View              | Description                        │
├───────────────────┼────────────────────────────────────┤
│ all               | Identity view                      │
│ filter            | Elements matching predicate        │
│ transform         | Apply function to each element     │
│ take              | First N elements                   │
│ take_while        | Elements while predicate is true   │
│ drop              | Skip first N elements              │
│ drop_while        | Skip while predicate is true       │
│ join              | Flatten nested ranges              │
│ split             | Split by delimiter                 │
│ reverse           | Reverse order                      │
│ iota              | Generate integer sequence          │
│ elements          | Extract Nth element of tuples      │
│ keys              | Extract keys from pairs            │
│ values            | Extract values from pairs          │
│ common            | Convert to common range            │
│ counted           | Create from iterator + count       │
├───────────────────┴────────────────────────────────────┤
│              NEW VIEWS (C++23)                         │
├───────────────────┬────────────────────────────────────┤
│ zip               | Combine multiple ranges            │
│ zip_transform     | Zip and transform                  │
│ adjacent          | Adjacent N elements                │
│ adjacent_transform| Transform adjacent elements        │
│ join_with         | Join with delimiter                │
│ slide             | Sliding window                     │
│ chunk             | Split into chunks                  │
│ stride            | Every Nth element                  │
│ cartesian_product | Cartesian product of ranges        │
└────────────────────────────────────────────────────────┘
```

`filter`, `transform`, `take`, `drop`, `take_while`, `drop_while`, and `reverse` are covered in **Basic Views** and **Composing Views** above. The sections below cover views with less obvious semantics.

### views::keys and views::values
```cpp
#include <ranges>
#include <map>

int main() {
    std::map<std::string, int> ages = {
        {"Alice", 25},
        {"Bob", 30},
        {"Charlie", 35}
    };
    
    // Extract just keys
    auto names = ages | std::views::keys;
    // Result: "Alice", "Bob", "Charlie"
    
    // Extract just values
    auto age_values = ages | std::views::values;
    // Result: 25, 30, 35
    
    // Useful for algorithms
    auto max_age = std::ranges::max(ages | std::views::values);
    // max_age = 35
    
    return 0;
}
```

### views::split (C++20)
```cpp
#include <ranges>
#include <string>
#include <iostream>

int main() {
    std::string text = "hello,world,foo,bar";
    
    // Split by delimiter
    auto words = text | std::views::split(',');
    
    for (auto word : words) {
        for (char c : word) {
            std::cout << c;
        }
        std::cout << '\n';
    }
    // Output:
    // hello
    // world
    // foo
    // bar
    
    return 0;
}
```

### views::join (C++20)
```cpp
#include <ranges>
#include <vector>
#include <string>

int main() {
    std::vector<std::vector<int>> nested = {
        {1, 2, 3},
        {4, 5},
        {6, 7, 8, 9}
    };
    
    // Flatten nested ranges
    auto flattened = nested | std::views::join;
    // Result: 1, 2, 3, 4, 5, 6, 7, 8, 9
    
    // Join strings
    std::vector<std::string> words = {"hello", "world"};
    auto chars = words | std::views::join;
    // Result: 'h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd'
    
    return 0;
}
```

### views::iota (Generate sequence)
```cpp
#include <ranges>
#include <iostream>

int main() {
    // Infinite sequence starting from 1
    auto infinite = std::views::iota(1);
    
    // Take first 10
    auto first_10 = std::views::iota(1) | std::views::take(10);
    // Result: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
    
    // Bounded iota
    auto range = std::views::iota(1, 11);  // [1, 11)
    // Result: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
    
    // Use in algorithms
    for (int i : std::views::iota(0, 5)) {
        std::cout << i << ' ';  // 0 1 2 3 4
    }
    
    return 0;
}
```

### C++23 Views: views::zip
```cpp
#include <ranges>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> names = {"Alice", "Bob", "Charlie"};
    std::vector<int> ages = {25, 30, 35};
    
    // Zip two ranges together
    auto zipped = std::views::zip(names, ages);
    
    for (auto [name, age] : zipped) {
        std::cout << name << " is " << age << " years old\n";
    }
    // Output:
    // Alice is 25 years old
    // Bob is 30 years old
    // Charlie is 35 years old
    
    return 0;
}
```

### C++23 Views: views::chunk
```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Split into chunks of 3
    auto chunks = vec | std::views::chunk(3);
    
    for (auto chunk : chunks) {
        for (int x : chunk) {
            std::cout << x << ' ';
        }
        std::cout << "| ";
    }
    // Output: 1 2 3 | 4 5 6 | 7 8 9 | 10 |
    
    return 0;
}
```

### C++23 Views: views::slide (Sliding Window)
```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Sliding window of size 3
    auto windows = vec | std::views::slide(3);
    
    for (auto window : windows) {
        for (int x : window) {
            std::cout << x << ' ';
        }
        std::cout << "| ";
    }
    // Output: 1 2 3 | 2 3 4 | 3 4 5 |
    
    return 0;
}
```

### C++23 Views: views::stride
```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Take every 3rd element
    auto every_3rd = vec | std::views::stride(3);
    // Result: 1, 4, 7, 10
    
    // Every 2nd element (odd indices)
    auto evens = vec | std::views::drop(1) | std::views::stride(2);
    // Result: 2, 4, 6, 8, 10
    
    return 0;
}
```

---

## Projections

Projections allow you to specify how to extract a value from an element before applying an operation.

```cpp
#include <ranges>
#include <vector>
#include <algorithm>

struct Person {
    std::string name;
    int age;
};

int main() {
    std::vector<Person> people = {
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35}
    };
    
    // Sort by age using projection
    std::ranges::sort(people, {}, &Person::age);
    // Equivalent to: sort(people, [](auto& a, auto& b) { return a.age < b.age; })
    
    // Find person with age 25
    auto it = std::ranges::find(people, 25, &Person::age);
    
    // Count people over 30
    auto count = std::ranges::count_if(people, 
        [](int age) { return age > 30; },
        &Person::age
    );
    
    return 0;
}
```

### Projection Syntax
```cpp
// Algorithm signature with projection
template<typename Range, typename Comp = less, typename Proj = identity>
void sort(Range&& r, Comp comp = {}, Proj proj = {});

// Usage examples:
std::ranges::sort(vec);                    // Default comparison, no projection
std::ranges::sort(vec, std::greater{});    // Custom comparison, no projection
std::ranges::sort(vec, {}, &Type::field);  // Default comparison, with projection
std::ranges::sort(vec, std::greater{}, &Type::field);  // Both
```

---

## Range Adaptors

### Creating Custom Range Adaptors
```cpp
#include <ranges>
#include <vector>

// Simple custom range adaptor
template<std::ranges::input_range R>
auto my_double_view(R&& range) {
    return std::forward<R>(range) 
        | std::views::transform([](auto x) { return x * 2; });
}

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    auto doubled = vec | my_double_view;
    // Result: 2, 4, 6, 8, 10
    
    return 0;
}
```

---

## Concepts and Constraints (C++20)

Concepts name compile-time requirements on template parameters. They largely replace [SFINAE](09_templates.md) (`std::enable_if`) with readable constraints and far clearer error messages. Full template mechanics live in [Templates](09_templates.md); here we focus on using concepts with ranges and modern APIs.

```cpp
#include <concepts>
#include <ranges>
#include <vector>

// Define a concept with 'concept' keyword
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// Constrained template parameter (abbreviated syntax)
template<Numeric T>
T clamp_nonneg(T value) {
    return value < T{0} ? T{0} : value;
}

// requires clause on template
template<typename T>
requires std::ranges::random_access_range<T>
void sort_in_place(T& range) {
    std::ranges::sort(range);
}

// requires expression — check that an operation is valid
template<typename T>
concept HasSize = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

// Constrained 'auto' parameter (abbreviated function template)
void print_size(HasSize auto const& container) {
    std::cout << container.size() << '\n';
}

// Standard library type constraint (no custom concept needed)
template<std::integral T>
T double_value(T x) { return x * 2; }

int main() {
    std::vector<int> v = {3, 1, 2};
    sort_in_place(v);           // OK: vector is random_access_range
    print_size(v);
    // sort_in_place(std::list{1, 2, 3});  // ERROR: list fails random_access_range
    return 0;
}
```

| Mechanism | Example | Purpose |
|-----------|---------|---------|
| `concept` definition | `concept C = requires(T t) { ... };` | Name a requirement set |
| `requires` clause | `template<typename T> requires C<T>` | Constrain a template |
| `requires` expression | `requires { t.size(); }` | Check validity of expressions |
| Abbreviated syntax | `template<std::integral T>` or `C auto& x` | Shorthand constraints |

⚠️ **Gotchas**: Concepts are checked at the template interface — they do not replace runtime validation. `std::ranges::sort` still requires `random_access_range` even if your function template compiles for weaker ranges.

---

## std::span (C++20)

### What is span?

[`std::span`](https://en.cppreference.com/w/cpp/container/span) is a **non-owning** view over a contiguous sequence of elements — essentially a pointer plus a length. Note that `operator[]` is **not** bounds-checked (out-of-range access is undefined behavior, just like a raw array); C++20 `span` has no `at()` member (one is added in C++26). It does not own memory; the underlying storage must outlive the span (same lifetime rules as [views](#views-c20) and [`std::string_view`](08_utility_containers.md)). Prefer `std::span` over raw pointer + size pairs in function parameters.

```cpp
#include <span>
#include <vector>
#include <array>

void process(std::span<int> data) {
    // Works with any contiguous container!
    for (int& x : data) {
        x *= 2;
    }
}

int main() {
    // Works with vector
    std::vector<int> vec = {1, 2, 3, 4, 5};
    process(vec);
    
    // Works with array
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    process(arr);
    
    // Works with C array
    int c_arr[] = {1, 2, 3, 4, 5};
    process(c_arr);
    
    // Works with subrange
    process(std::span(vec).subspan(1, 3));  // Elements [1, 4)
    
    return 0;
}
```

### span Features
```cpp
#include <span>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::span<int> s(vec);
    
    // Access
    int first = s[0];
    int last = s[s.size() - 1];
    int* data = s.data();
    
    // Size
    size_t size = s.size();
    size_t bytes = s.size_bytes();
    bool empty = s.empty();
    
    // Subspans
    auto first_5 = s.first(5);      // First 5 elements
    auto last_5 = s.last(5);        // Last 5 elements
    auto middle = s.subspan(3, 4);  // 4 elements starting at index 3
    
    // Iteration
    for (int x : s) {
        std::cout << x << ' ';
    }
    
    return 0;
}
```

### Fixed vs Dynamic Extent
```cpp
#include <span>
#include <array>

int main() {
    // Dynamic extent (size known at runtime)
    std::vector<int> vec = {1, 2, 3};
    std::span<int> dynamic_span(vec);  // std::span<int, std::dynamic_extent>
    
    // Fixed extent (size known at compile time)
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    std::span<int, 5> fixed_span(arr);
    
    // Can convert fixed to dynamic, but not vice versa
    std::span<int> dyn = fixed_span;  // OK
    // std::span<int, 5> fixed = dynamic_span;  // ERROR
    
    return 0;
}
```

---

## std::mdspan (C++23)

[`std::mdspan`](https://en.cppreference.com/w/cpp/container/mdspan) is a **non-owning** multi-dimensional view over contiguous storage — the N-dimensional analogue of `std::span`. Layout and extents are part of the type; the backing buffer must outlive the `mdspan`.

```cpp
#include <mdspan>
#include <vector>

int main() {
    // Create 2D matrix data
    std::vector<int> data(3 * 4);  // 3x4 matrix
    
    // Create mdspan view
    std::mdspan<int, std::extents<size_t, 3, 4>> matrix(data.data());
    
    // Access elements
    matrix[0, 0] = 1;
    matrix[1, 2] = 5;
    
    // Iterate
    for (size_t i = 0; i < matrix.extent(0); ++i) {
        for (size_t j = 0; j < matrix.extent(1); ++j) {
            std::cout << matrix[i, j] << ' ';
        }
        std::cout << '\n';
    }
    
    return 0;
}
```

---

## Flat Containers (C++23)

### std::flat_set, std::flat_map

Sorted containers backed by contiguous storage (like [vector](01_sequence_containers.md)) instead of trees — see [Associative Containers](02_associative_containers.md) for `std::set`/`std::map` semantics they mirror.

```cpp
#include <flat_set>
#include <flat_map>

int main() {
    // flat_set: vector-backed sorted unique elements
    std::flat_set<int> fset = {5, 2, 8, 1, 9};
    // Internally uses sorted vector
    // Better cache locality than std::set
    // Slower insertion/deletion, faster iteration
    
    // flat_map: vector-backed sorted key-value pairs
    std::flat_map<std::string, int> fmap = {
        {"Alice", 25},
        {"Bob", 30}
    };
    
    // Same interface as set/map
    fset.insert(3);
    fmap["Charlie"] = 35;
    
    return 0;
}
```

### Performance Comparison
```
┌────────────────────────────────────────────────────┐
│           std::set vs std::flat_set                │
├────────────────────────────────────────────────────┤
│ Operation    │ std::set   │ std::flat_set          │
├──────────────┼────────────┼────────────────────────┤
│ Search       │ O(log n)   │ O(log n) (faster)      │
│ Insert       │ O(log n)   │ O(n) (slower)          │
│ Delete       │ O(log n)   │ O(n) (slower)          │
│ Iteration    │ O(n)       │ O(n) (much faster)     │
│ Memory       │ Higher     │ Lower                  │
│ Cache        │ Poor       │ Excellent              │
└────────────────────────────────────────────────────┘

Use flat_* when:
✓ More reads than writes
✓ Cache locality important
✓ Memory efficiency matters
```

---

## Practical Examples

### Example 1: Processing Pipeline
```cpp
#include <ranges>
#include <vector>
#include <iostream>

struct Student {
    std::string name;
    int score;
    bool passed;
};

int main() {
    std::vector<Student> students = {
        {"Alice", 85, true},
        {"Bob", 45, false},
        {"Charlie", 92, true},
        {"David", 55, false},
        {"Eve", 88, true}
    };
    
    // Find top 3 passing students
    auto top_students = students
        | std::views::filter(&Student::passed)
        | std::views::transform([](auto& s) { return std::pair{s.name, s.score}; })
        | std::views::take(3);
    
    for (auto [name, score] : top_students) {
        std::cout << name << ": " << score << '\n';
    }
    
    return 0;
}
```

### Example 2: Lazy Evaluation
```cpp
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Create view - NO computation yet!
    auto processed = vec
        | std::views::filter([](int x) { 
            std::cout << "Filtering " << x << '\n'; 
            return x % 2 == 0; 
          })
        | std::views::transform([](int x) { 
            std::cout << "Transforming " << x << '\n'; 
            return x * x; 
          });
    
    std::cout << "View created, no computation yet!\n\n";
    
    // Computation happens NOW, as we iterate
    for (int x : processed | std::views::take(2)) {
        std::cout << "Result: " << x << "\n\n";
    }
    
    // Output shows lazy evaluation:
    // Only processes elements until take(2) is satisfied
    
    return 0;
}
```

### Example 3: View Materialization
```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Create view (lazy)
    auto view = vec | std::views::filter([](int x) { return x % 2 == 0; });
    
    // Convert view to concrete container (materialize)
    std::vector<int> materialized(view.begin(), view.end());
    // materialized = {2, 4}
    
    // Or use ranges::to (C++23)
    auto materialized2 = view | std::ranges::to<std::vector>();
    
    return 0;
}
```

### Example 4: Cartesian Product (C++23)
```cpp
#include <ranges>
#include <vector>

int main() {
    std::vector<int> x = {1, 2, 3};
    std::vector<char> y = {'a', 'b'};
    
    // Cartesian product
    auto product = std::views::cartesian_product(x, y);
    
    for (auto [num, ch] : product) {
        std::cout << "(" << num << ", " << ch << ") ";
    }
    // Output: (1, a) (1, b) (2, a) (2, b) (3, a) (3, b)
    
    return 0;
}
```

---

## Performance Considerations

### Views are Cheap to Copy
```cpp
// View is just a pointer + size (or similar small state)
auto view = vec | std::views::filter(pred);
auto view2 = view;  // O(1) copy!

// Compare to copying vector
auto vec2 = vec;    // O(n) copy
```

### Avoid Unnecessary Materialization
```cpp
// BAD: Materializes intermediate results
std::vector<int> temp1 = filter_evens(vec);
std::vector<int> temp2 = square_all(temp1);
std::vector<int> result = take_first_5(temp2);

// GOOD: Use views (no intermediate vectors)
auto result = vec
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * x; })
    | std::views::take(5)
    | std::ranges::to<std::vector>();  // Materialize only at end
```

---

## Common Pitfalls

### 1. Dangling References
```cpp
// BAD: View outlives container
auto get_evens() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    return vec | std::views::filter([](int x) { return x % 2 == 0; });
}  // vec destroyed, view dangles!

// BAD: View over a temporary rvalue
auto evens = std::vector{1, 2, 3, 4, 5}
    | std::views::filter([](int x) { return x % 2 == 0; });
// Temporary vector destroyed at end of full-expression — evens dangles!

// GOOD: Return owned data or materialize before the temporary dies
std::vector<int> get_evens() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto view = vec | std::views::filter([](int x) { return x % 2 == 0; });
    return std::vector<int>(view.begin(), view.end());
}
```

Views borrow from their source range. Returning a view from a function that owns the source, or piping a temporary container without materializing, produces a **dangling view** — the same class of bug as dangling [`std::string_view`](08_utility_containers.md).

### 2. Modifying Through Views
```cpp
std::vector<int> vec = {1, 2, 3};
auto view = vec | std::views::take(2);

// Can modify through view
for (int& x : view) {
    x *= 2;  // OK: modifies vec
}
// vec = {2, 4, 3}
```

### 3. View Invalidation
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
auto view = vec | std::views::filter([](int x) { return x % 2 == 0; });

vec.push_back(6);  // May invalidate view's iterators!
// Using view after this is dangerous
```

---

## Best Practices

1. **Use ranges algorithms**: Cleaner than iterator pairs
2. **Compose views**: Build complex pipelines
3. **Materialize at the end**: Keep data lazy as long as possible
4. **Watch lifetimes**: Views don't own data
5. **Use projections**: Cleaner than lambdas for simple extractions
6. **Prefer constexpr**: Many range operations work at compile time
7. **Use span for parameters**: More flexible than specific containers

---

## Complete Practical Example: Data Processing Pipeline with Ranges

Here's a comprehensive example integrating ranges, views, concepts, span, and C++20/23 features:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <algorithm>
#include <string>
#include <span>
#include <concepts>
#include <numeric>

namespace rng = std::ranges;
namespace vws = std::views;

// 1. Concepts for type constraints
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<typename T>
concept Printable = requires(T t) {
    { std::cout << t } -> std::convertible_to<std::ostream&>;
};

// 2. Data structures
struct Product {
    int id;
    std::string name;
    double price;
    int stock;
    bool active;
    
    auto operator<=>(const Product&) const = default;  // C++20
};

struct Order {
    int order_id;
    int product_id;
    int quantity;
    double total;
};

// 3. Generic print function using concepts
template<rng::range R>
void print_range(const R& range, const std::string& label = "") {
    if (!label.empty()) {
        std::cout << label << ": ";
    }
    
    for (const auto& item : range) {
        if constexpr (Printable<decltype(item)>) {
            std::cout << item << " ";
        }
    }
    std::cout << "\n";
}

// 4. Process data using std::span (view of contiguous data)
template<Numeric T>
double calculate_average(std::span<const T> data) {
    if (data.empty()) return 0.0;
    
    T sum = std::accumulate(data.begin(), data.end(), T{0});
    return static_cast<double>(sum) / data.size();
}

template<Numeric T>
void normalize_data(std::span<T> data, T max_value) {
    for (auto& value : data) {
        value = std::min(value, max_value);
    }
}

// 5. Product inventory manager using ranges
class InventoryManager {
private:
    std::vector<Product> products_;
    std::vector<Order> orders_;
    
public:
    void add_product(int id, std::string name, double price, int stock) {
        products_.push_back({id, std::move(name), price, stock, true});
    }
    
    void add_order(int order_id, int product_id, int quantity) {
        auto it = rng::find_if(products_, [product_id](const Product& p) {
            return p.id == product_id;
        });
        
        if (it != products_.end()) {
            double total = it->price * quantity;
            orders_.push_back({order_id, product_id, quantity, total});
            it->stock -= quantity;
        }
    }
    
    // Get active products using views
    auto get_active_products() const {
        return products_ 
            | vws::filter([](const Product& p) { return p.active; });
    }
    
    // Get products by price range
    auto get_products_in_range(double min_price, double max_price) const {
        return products_
            | vws::filter([min_price, max_price](const Product& p) {
                return p.price >= min_price && p.price <= max_price && p.active;
              });
    }
    
    // Get low stock products
    auto get_low_stock(int threshold) const {
        return products_
            | vws::filter([threshold](const Product& p) {
                return p.stock < threshold && p.active;
              })
            | vws::transform([](const Product& p) {
                return std::pair{p.name, p.stock};
              });
    }
    
    // Calculate total inventory value using ranges algorithm
    double calculate_inventory_value() const {
        return rng::fold_left(
            products_
                | vws::filter([](const Product& p) { return p.active; })
                | vws::transform([](const Product& p) {
                    return p.price * p.stock;
                  }),
            0.0,
            std::plus<>()
        );
    }
    
    // Get top selling products
    auto get_top_sellers(size_t n) const {
        // Count product IDs in orders
        std::vector<std::pair<int, int>> product_counts;  // {product_id, count}
        
        for (int product_id : products_ | vws::transform(&Product::id)) {
            int count = rng::count_if(orders_, [product_id](const Order& o) {
                return o.product_id == product_id;
            });
            
            if (count > 0) {
                product_counts.push_back({product_id, count});
            }
        }
        
        // Sort by count
        rng::sort(product_counts, std::greater<>{}, &std::pair<int, int>::second);
        
        // Return top N product names
        return product_counts
            | vws::take(n)
            | vws::transform([this](const auto& p) {
                auto it = rng::find(products_, p.first, &Product::id);
                return it != products_.end() ? it->name : "Unknown";
              });
    }
    
    // Print inventory with custom projection
    void print_inventory() const {
        std::cout << "\n=== Inventory ===\n";
        
        // Sort by price descending
        auto sorted = products_
            | vws::filter(&Product::active)
            | std::ranges::to<std::vector>();  // Materialize
        
        rng::sort(sorted, std::greater<>{}, &Product::price);
        
        for (const auto& product : sorted) {
            std::cout << "ID: " << product.id 
                      << ", Name: " << product.name
                      << ", Price: $" << product.price
                      << ", Stock: " << product.stock << "\n";
        }
    }
    
    // Advanced pipeline: Find products to reorder
    void print_reorder_report() const {
        std::cout << "\n=== Reorder Report ===\n";
        
        auto reorder_items = products_
            | vws::filter([](const Product& p) { 
                return p.active && p.stock < 10; 
              })
            | vws::transform([](const Product& p) {
                return std::tuple{p.name, p.stock, 50 - p.stock};  // name, current, needed
              });
        
        for (const auto& [name, current, needed] : reorder_items) {
            std::cout << name << ": Current=" << current 
                      << ", Reorder=" << needed << " units\n";
        }
    }
};

// Demonstration functions
void demo_ranges_basics() {
    std::cout << "--- Ranges Basics ---\n";
    
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Filter and transform
    auto result = numbers
        | vws::filter([](int x) { return x % 2 == 0; })
        | vws::transform([](int x) { return x * x; })
        | vws::take(3);
    
    print_range(result, "Even squares (first 3)");
}

void demo_span_usage() {
    std::cout << "\n--- std::span Usage ---\n";
    
    std::vector<int> vec = {10, 20, 30, 40, 50};
    int arr[] = {5, 15, 25, 35, 45};
    
    // span works with both
    auto avg1 = calculate_average(std::span(vec));
    auto avg2 = calculate_average(std::span(arr));
    
    std::cout << "Vector average: " << avg1 << "\n";
    std::cout << "Array average: " << avg2 << "\n";
    
    // Modify through span
    std::span span_vec(vec);
    normalize_data(span_vec, 35);
    print_range(vec, "After normalization");
}

void demo_projections() {
    std::cout << "\n--- Projections ---\n";
    
    std::vector<Product> products = {
        {1, "Laptop", 999.99, 5, true},
        {2, "Mouse", 29.99, 50, true},
        {3, "Keyboard", 79.99, 20, true}
    };
    
    // Sort by price using projection
    rng::sort(products, {}, &Product::price);
    
    std::cout << "Sorted by price:\n";
    for (const auto& p : products) {
        std::cout << "  " << p.name << ": $" << p.price << "\n";
    }
    
    // Find max price product
    auto max_it = rng::max_element(products, {}, &Product::price);
    std::cout << "Most expensive: " << max_it->name << "\n";
}

void demo_views_composition() {
    std::cout << "\n--- View Composition ---\n";
    
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Complex pipeline
    auto pipeline = numbers
        | vws::filter([](int x) { return x > 3; })          // {4,5,6,7,8,9,10}
        | vws::transform([](int x) { return x * 2; })        // {8,10,12,14,16,18,20}
        | vws::drop(2)                                       // {12,14,16,18,20}
        | vws::take(3);                                      // {12,14,16}
    
    print_range(pipeline, "Pipeline result");
    
    // All operations are lazy - computed as we iterate!
}

int main() {
    std::cout << "=== Modern C++20/23 Features Demo ===\n\n";
    
    // 1. Ranges basics
    demo_ranges_basics();
    
    // 2. std::span
    demo_span_usage();
    
    // 3. Projections
    demo_projections();
    
    // 4. View composition
    demo_views_composition();
    
    // 5. Full inventory system
    std::cout << "\n--- Inventory Management System ---\n";
    InventoryManager inventory;
    
    inventory.add_product(101, "Laptop", 1299.99, 15);
    inventory.add_product(102, "Mouse", 29.99, 5);
    inventory.add_product(103, "Keyboard", 89.99, 8);
    inventory.add_product(104, "Monitor", 399.99, 20);
    inventory.add_product(105, "Headset", 149.99, 3);
    
    // Add orders
    inventory.add_order(1, 101, 2);
    inventory.add_order(2, 102, 1);
    inventory.add_order(3, 101, 1);
    inventory.add_order(4, 103, 2);
    
    // Print inventory
    inventory.print_inventory();
    
    // Low stock report
    std::cout << "\n=== Low Stock Alert (< 10) ===\n";
    for (const auto& [name, stock] : inventory.get_low_stock(10)) {
        std::cout << name << ": " << stock << " units left\n";
    }
    
    // Price range query
    std::cout << "\n=== Products $50-$200 ===\n";
    for (const auto& product : inventory.get_products_in_range(50.0, 200.0)) {
        std::cout << product.name << ": $" << product.price << "\n";
    }
    
    // Top sellers
    std::cout << "\n=== Top 3 Sellers ===\n";
    for (const auto& name : inventory.get_top_sellers(3)) {
        std::cout << "  " << name << "\n";
    }
    
    // Total value
    std::cout << "\nTotal inventory value: $" 
              << inventory.calculate_inventory_value() << "\n";
    
    // Reorder report
    inventory.print_reorder_report();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **Ranges algorithms**: `rng::sort`, `rng::find_if`, `rng::count_if`
- **Views**: `filter`, `transform`, `take`, `drop`
- **View composition**: Chaining multiple views
- **Lazy evaluation**: Computing only when needed
- **std::span**: Generic view of contiguous data
- **Concepts**: Constraining template parameters
- **Projections**: Sorting/finding by member
- **ranges::to**: Materializing views (C++23)
- **fold_left**: Accumulating with ranges
- **Spaceship operator**: `operator<=>`
- **Structured bindings**: Unpacking tuples/pairs
- **Lambda projections**: Member access in algorithms
- **Pipeline operator**: `|` for composing views

This example shows the power of modern C++ ranges and views!

---

## Related Topics

- [Algorithms](06_algorithms.md) — iterator-pair algorithms that ranges generalize; many have `std::ranges::` equivalents.
- [Iterators](05_iterators.md) — iterator categories (`random_access`, `bidirectional`, …) determine which range algorithms compile.
- [Templates](09_templates.md) — concepts, SFINAE, and variadic templates underpin constrained generic code.
- [Lambdas](10_lambdas.md) — predicates and transforms passed to `views::filter`, `views::transform`, and range algorithms.
- [Utility Containers](08_utility_containers.md) — `std::string_view` and `std::span` share the same non-owning lifetime model as views.
- [Sequence Containers](01_sequence_containers.md) — `vector`/`array` are ideal backing stores for views and spans.
- [Best Practices](13_best_practices.md) — when to materialize views vs keep pipelines lazy.

---

## Next Steps
- **Next**: [Utility Containers →](08_utility_containers.md)
- **Previous**: [← Algorithms](06_algorithms.md)

---
*Chapter 7 — Modern C++20/23 Features*

