# C++ Standard Template Library (STL) Tutorial

## Complete Guide to C++23 STL

```
┌─────────────────────────────────────────────────────────────┐
│                    C++ STL ECOSYSTEM                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │ CONTAINERS  │   │  ITERATORS  │   │ ALGORITHMS  │        │
│  │             │◄──┤             │◄──┤             │        │
│  │  Data       │   │  Access     │   │  Process    │        │
│  │  Storage    │   │  Interface  │   │  Data       │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│        │                   │                  │             │
│        └───────────────────┴──────────────────┘             │
│                            │                                │
│                     ┌──────▼──────┐                         │
│                     │  ALLOCATORS │                         │
│                     │  & FUNCTORS │                         │
│                     └─────────────┘                         │
└─────────────────────────────────────────────────────────────┘
```

## Overview

The C++ Standard Template Library (STL) is a powerful collection of template classes and functions that provide generic implementations of common data structures and algorithms. Its four core pillars are **containers**, **iterators**, **algorithms**, and **function objects** (with **allocators** underpinning memory management).

> **A note on terminology.** Strictly speaking, the "STL" refers to the containers/iterators/algorithms/functors design that Alexander Stepanov contributed to C++. The **C++ Standard Library** is the broader entity standardized by ISO, and it also includes strings, I/O streams, threading, smart pointers, `std::format`, and much more. This tutorial uses "STL" loosely in its title but actually covers the whole modern Standard Library plus the core language features (OOP, templates, move semantics, coroutines, modules) you need to use it well — from C++98 through C++23.

If you only read one page, make it the [Quick Reference cheat sheet](99_quick_reference.md). For a guided path, see the [Learning Path](#learning-path) below.

## Tutorial Structure

### Part 0: Foundations

#### 0. [Object-Oriented Programming Concepts](00_oop_concepts.md)
- Classes and Objects
- Encapsulation (data hiding and access control)
- Inheritance (single and multiple)
- Polymorphism (compile-time and runtime)
- Abstraction (interfaces and abstract classes)
- Virtual functions and dynamic binding
- Constructors and destructors
- Complete practical banking system example
- **Important gotchas and hunches for each OOP concept**

### Part I: STL Fundamentals

#### 1. [Sequence Containers](01_sequence_containers.md)
- `std::vector` - Dynamic array
- `std::deque` - Double-ended queue
- `std::list` - Doubly-linked list
- `std::forward_list` - Singly-linked list
- `std::array` - Fixed-size array

#### 2. [Associative Containers](02_associative_containers.md)
- `std::set` / `std::multiset` - Ordered unique/multiple elements
- `std::map` / `std::multimap` - Ordered key-value pairs

#### 3. [Unordered Containers](03_unordered_containers.md)
- `std::unordered_set` / `std::unordered_multiset` - Hash-based sets
- `std::unordered_map` / `std::unordered_multimap` - Hash-based maps

#### 4. [Container Adaptors](04_container_adaptors.md)
- `std::stack` - LIFO data structure
- `std::queue` - FIFO data structure
- `std::priority_queue` - Heap-based priority queue

#### 5. [Iterators](05_iterators.md)
- Iterator categories
- Iterator operations
- Custom iterators
- C++20 ranges and iterators

#### 6. [Algorithms](06_algorithms.md)
- Non-modifying sequence operations
- Modifying sequence operations
- Sorting and searching
- Numeric algorithms
- C++20 ranges algorithms

#### 7. [C++20/23 Modern Features](07_modern_features.md)
- Ranges library
- Views and adaptors
- Concepts and constraints
- `std::span`, `std::mdspan` (C++23)
- New algorithms and utilities

#### 8. [Utility Containers](08_utility_containers.md)
- `std::string` and `std::string_view`
- `std::bitset`
- `std::span` (C++20)
- `std::optional`, `std::variant`, `std::any`
- `std::tuple` and `std::pair`

### Part II: Advanced C++ Features

#### 9. [Templates](09_templates.md)
- Function templates
- Class templates
- Variadic templates
- Template specialization
- SFINAE and Concepts (C++20)
- Template metaprogramming basics

#### 10. [Lambdas and Functional Programming](10_lambdas.md)
- Lambda expressions and captures
- Generic lambdas (C++14)
- Template lambdas (C++20)
- Functional programming patterns
- Map, filter, reduce
- Function composition

#### 11. [Template Metaprogramming](11_metaprogramming.md)
- Compile-time computation
- `constexpr` and `consteval`
- `if constexpr` (C++17)
- Type traits
- SFINAE techniques
- Advanced metaprogramming

#### 12. [Advanced Features](12_advanced_features.md)
- Move semantics and perfect forwarding
- Smart pointers (`unique_ptr`, `shared_ptr`, `weak_ptr`)
- RAII principles
- Rule of Five / Rule of Zero
- Value categories
- Attributes and inline variables

#### 13. [Best Practices and Idioms](13_best_practices.md)
- Modern C++ idioms
- Performance optimization
- Code organization
- Error handling
- Testing and debugging
- Common anti-patterns to avoid

### Part III: Concurrent Programming

#### 14. [Multithreading and Concurrency](14_multithreading.md)
- `std::thread` and `std::jthread` (C++20)
- Mutexes and locks (`lock_guard`, `unique_lock`, `scoped_lock`)
- Reader-writer locks (`shared_mutex`)
- Atomics and memory ordering
- Condition variables
- Thread-safe data structures
- Common concurrency patterns

#### 15. [Asynchronous Programming and Futures](15_async_futures.md)
- `std::async` and launch policies
- `std::future` and `std::promise`
- `std::packaged_task` and `std::shared_future`
- Parallel algorithms (C++17)
- Thread pools
- Coroutines (C++20 basics)
- Best practices for async programming

### Part IV: I/O and System Programming

#### 16. [I/O, Filesystem, and Formatting](16_io_filesystem.md)
- Stream I/O (`iostream`, `fstream`, `sstream`)
- File operations and binary I/O
- `std::filesystem` (C++17)
- Directory iteration and path operations
- `std::format` (C++20) and `std::print` (C++23)
- Modern string formatting

#### 17. [Exception Handling and Error Management](17_exceptions.md)
- Exception basics and standard exceptions
- Custom exception classes
- Exception safety guarantees
- `noexcept` specification
- `std::error_code` and `std::system_error`
- `std::expected` (C++23)
- RAII and exception safety

#### 18. [Time and Chrono Library](18_time_chrono.md)
- Durations and time points
- Clocks (`system_clock`, `steady_clock`)
- Calendar types (C++20)
- Timezone support (C++20)
- Time formatting
- Performance timing and benchmarking

#### 19. [Memory Management and Allocators](19_memory_allocators.md)
- Standard allocators
- Polymorphic Memory Resources (PMR - C++17)
- Custom allocators
- Memory alignment
- Placement new
- Object pools and stack allocators

#### 20. [Regular Expressions](20_regex.md)
- Regex syntax and patterns
- `std::regex_match` and `std::regex_search`
- `std::regex_replace` and substitution
- Capture groups
- Regex iterators
- Performance considerations

#### 21. [Modules (C++20)](21_modules.md)
- Module basics and syntax
- Module interface and implementation
- Module partitions
- Importing standard library
- Module visibility and encapsulation
- Migration from headers

#### 22. [Coroutines (C++20)](22_coroutines.md)
- Coroutine basics (`co_await`, `co_yield`, `co_return`)
- Generator pattern
- Async task pattern
- Promise types and awaitable objects
- Practical examples (lazy evaluation, pipelines)
- Performance considerations
- Best practices and common pitfalls

## Key Concepts

### Time Complexity Notation
- `O(1)` - Constant time
- `O(log n)` - Logarithmic time
- `O(n)` - Linear time
- `O(n log n)` - Linearithmic time
- `O(n²)` - Quadratic time

### Memory Layout
```
┌─────────────────────────────────────────┐
│ Stack Memory (automatic storage)        │
│ - Small, fixed-size containers          │
│ - std::array                            │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Heap Memory (dynamic storage)           │
│ - Dynamic containers                    │
│ - std::vector, std::map, etc.           │
│ - Managed via allocators                │
└─────────────────────────────────────────┘
```

## Prerequisites
- Basic C++ knowledge (variables, functions, control flow)
- Familiarity with pointers and references
- A C++11 or later compiler (C++23 for the latest features); see [Templates](09_templates.md) if generics are new to you

## Compiler Support

C++23 support is still rolling out across toolchains. As a rule of thumb:

| Toolchain | Recommended version | C++23 status |
|-----------|--------------------|--------------|
| GCC (libstdc++) | 14+ | Most library features; `import std;` from GCC 15 |
| Clang (libc++) | 18+ | Broad core + library support; some library bits trail |
| MSVC (STL) | VS 2022 17.10+ | Strong, growing C++23 coverage |
| Apple Clang | Xcode 16+ | Lags upstream Clang; check feature-test macros |

Compile examples with the right standard flag, e.g. `g++ -std=c++23 file.cpp` or `clang++ -std=c++23 file.cpp` (`/std:c++latest` on MSVC). When in doubt about a specific feature, check the [feature-test macros](https://en.cppreference.com/w/cpp/feature_test) (e.g. `__cpp_lib_format`) and [cppreference's compiler support table](https://en.cppreference.com/w/cpp/compiler_support).

## How to Use This Tutorial
1. Start with sequence containers if you're new to STL
2. Each file contains detailed explanations with ASCII diagrams
3. Code examples are provided for each concept
4. Practice exercises at the end of each section
5. Build up to advanced topics like ranges and views

## Quick Start Example

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // Create a vector (dynamic array)
    std::vector<int> numbers = {5, 2, 8, 1, 9};

    // Sort using STL algorithm
    std::sort(numbers.begin(), numbers.end());

    // Print using range-based for loop (C++11)
    for (int num : numbers) {
        std::cout << num << " ";
    }
    // Output: 1 2 5 8 9

    return 0;
}
```

## STL Philosophy

**Generic Programming**: Write code once, use with any type
```
Template Function/Class
        │
        ├─► Works with int
        ├─► Works with double
        ├─► Works with custom types
        └─► Works with any compatible type
```

**Separation of Concerns**:
- Containers know how to store data
- Iterators know how to traverse data
- Algorithms know how to process data
- Each component is independent and reusable

## Benefits of STL
1. ✓ **Reusability**: Don't reinvent the wheel
2. ✓ **Efficiency**: Highly optimized implementations
3. ✓ **Type Safety**: Compile-time type checking
4. ✓ **Flexibility**: Works with any compatible type
5. ✓ **Standardization**: Portable across platforms
6. ✓ **Modern C++**: Constantly evolving with new standards

## Learning Path

### For Beginners
1. **Start with**: [OOP Concepts](00_oop_concepts.md) - Essential foundation
2. Then learn: [Sequence Containers](01_sequence_containers.md)
3. Learn [Algorithms](06_algorithms.md)
4. Understand [Iterators](05_iterators.md)
5. Explore [Utility Containers](08_utility_containers.md)

### For Intermediate Developers
1. Master [Associative](02_associative_containers.md) and [Unordered Containers](03_unordered_containers.md)
2. Study [Modern C++20/23 Features](07_modern_features.md)
3. Learn [Lambdas](10_lambdas.md)
4. Understand [Templates](09_templates.md)

### For Advanced Users
1. Deep dive into [Metaprogramming](11_metaprogramming.md)
2. Master [Advanced Features](12_advanced_features.md)
3. Study [Best Practices](13_best_practices.md)
4. Learn [Multithreading](14_multithreading.md)
5. Master [Async Programming](15_async_futures.md)
6. Learn [Coroutines](22_coroutines.md) for elegant async code
7. Explore [I/O and Filesystem](16_io_filesystem.md)
8. Study [Exception Handling](17_exceptions.md)
9. Understand [Memory Management](19_memory_allocators.md)

## Navigation

### 🚀 Getting Started
- **Start Here**: [OOP Concepts →](00_oop_concepts.md) - **Essential Foundation**
- **Start STL**: [Sequence Containers →](01_sequence_containers.md)

### 📚 Quick Access
- **⭐ Quick Reference**: [Cheat Sheet →](99_quick_reference.md) - **Bookmark This!**
- **Jump to Advanced**: [Templates →](09_templates.md)
- **Jump to Concurrency**: [Multithreading →](14_multithreading.md)
- **Jump to I/O**: [I/O and Filesystem →](16_io_filesystem.md)
- **Jump to Coroutines**: [Coroutines (C++20) →](22_coroutines.md)

---

## 📖 Additional Resources

### [Quick Reference Guide](99_quick_reference.md) ⭐
**Condensed cheat sheet for daily development:**
- Container selection decision tree
- Time complexity table for all containers
- Most common algorithms with syntax
- C++11/14/17/20/23 feature lookup
- Common design patterns
- Performance tips
- Complete gotchas checklist
- Container operations reference
- Lambda syntax guide
- Smart pointer reference

---
