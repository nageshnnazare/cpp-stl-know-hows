# Modules (C++20)

## Overview

C++20 introduced modules as a modern alternative to header files. Modules can speed compilation, improve encapsulation via `export`, and reduce macro leakage — but **tooling is still maturing**. Build systems must compile module interface units before importers, and support varies across GCC 14/15, Clang, MSVC, and especially Apple Clang/libc++. This chapter covers interface vs implementation units, partitions, the global module fragment, `import std` (C++23), and migration strategies.

```
┌────────────────────────────────────────────────────────┐
│                   MODULES SYSTEM                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  DEFINITION     │  USAGE           │  PARTITIONS       │
│  ──────────     │  ─────           │  ──────────       │
│  • export       │  • import        │  • Submodules     │
│  • module       │  • No #include   │  • Private impl   │
│  • Interface    │  • No macros     │  • Organization   │
│                                                        │
│  BENEFITS:                                             │
│  • Faster compilation (no redundant parsing)           │
│  • Better encapsulation (export control)               │
│  • No header guards needed                             │
│  • No macro leakage                                    │
│  • Order-independent imports                           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Why Modules?

### Problems with Headers
```cpp
// Traditional header problems:

// 1. Redundant parsing
#include <vector>    // Parsed in every .cpp file
#include <string>    // Thousands of lines parsed repeatedly
#include <iostream>

// 2. Macro leakage
#include <windows.h>  // Defines 'min', 'max' macros
int min = 5;          // ERROR: min is a macro!

// 3. Header order matters
#include "b.h"
#include "a.h"  // Might fail if a.h needed before b.h

// 4. Multiple inclusion guards everywhere
#ifndef HEADER_H
#define HEADER_H
// ...
#endif

// 5. Can't hide implementation details
// Everything in header is visible
```

### Modules Advantages
```
┌───────────────────────────────────────────────────────┐
│         Headers vs Modules Comparison                 │
├───────────────────────────────────────────────────────┤
│ Feature        │ Headers      │ Modules               │
├────────────────┼──────────────┼───────────────────────┤
│ Compilation    │ Slow         │ Fast                  │
│ Encapsulation  │ Weak         │ Strong                │
│ Macros         │ Leak         │ Don't leak            │
│ Order          │ Matters      │ Doesn't matter        │
│ Guards         │ Required     │ Not needed            │
│ Isolation      │ No           │ Yes                   │
│                                                       │
│ Typical compilation speedup: 2-10x                    │
└───────────────────────────────────────────────────────┘
```

---

## Basic Module Syntax

### Creating a Module
```cpp
// math.cppm (module interface file)
export module math;

export int add(int a, int b) {
    return a + b;
}

export int multiply(int a, int b) {
    return a * b;
}

// Not exported, internal only
int helper() {
    return 42;
}
```

### Using a Module
```cpp
// main.cpp
import math;  // Import module

int main() {
    int sum = add(5, 3);           // OK: exported
    int product = multiply(4, 2);  // OK: exported
    
    // int x = helper();           // ERROR: not exported
    
    return 0;
}
```

### Compilation

**Order matters**: compile partitions → primary interface → importers. Flags differ by compiler:

```bash
# GCC 14+ (modules; -fmodules-ts on older GCC)
g++ -std=c++20 -c math.cppm -o math.o
g++ -std=c++20 main.cpp math.o -o program

# MSVC: /interface for .cppm, /reference for BMI
# Clang: -std=c++20 -fprebuilt-module-path=...
```

Use CMake 3.28+ `FILE_SET CXX_MODULES` when possible; verify your toolchain's module docs before adopting in production.

---

## Module Structure

### Module Declaration
```cpp
// shapes.cppm
export module shapes;  // Module declaration

// Everything below is part of module 'shapes'
// but only 'export' declarations are visible to importers
```

### Export Declarations
```cpp
export module geometry;

// Export single function
export double area_circle(double radius) {
    return 3.14159 * radius * radius;
}

// Export multiple declarations
export {
    double area_square(double side);
    double area_rectangle(double width, double height);
}

// Export class
export class Rectangle {
public:
    Rectangle(double w, double h);
    double area() const;
private:
    double width_, height_;
};

// Export namespace
export namespace shapes {
    class Triangle { /* ... */ };
    class Circle { /* ... */ };
}

// Export type alias
export using Point = std::pair<double, double>;

// Export concept
export template<typename T>
concept Numeric = std::is_arithmetic_v<T>;
```

### Implementation Unit vs Interface Unit

- **Module interface unit**: begins with `export module name;` — defines the public API (BMI).
- **Module implementation unit**: begins with `module name;` (no `export`) — internal definitions linked with the interface.

```cpp
// geometry.cppm — interface unit
export module geometry;

export class Rectangle {
public:
    Rectangle(double w, double h);
    double area() const;
private:
    double width_, height_;
};

// geometry_impl.cpp — implementation unit
module geometry;

Rectangle::Rectangle(double w, double h) : width_(w), height_(h) {}
double Rectangle::area() const { return width_ * height_; }
```

Definitions may also live inline in the interface unit (common for small modules).

---

## Module Partitions

### Interface Partitions
```cpp
// math:basic.cppm
export module math:basic;  // Partition declaration

export int add(int a, int b) {
    return a + b;
}

export int subtract(int a, int b) {
    return a - b;
}
```

```cpp
// math:advanced.cppm
export module math:advanced;

export double power(double base, double exp) {
    return std::pow(base, exp);
}

export double sqrt(double x) {
    return std::sqrt(x);
}
```

```cpp
// math.cppm (primary module interface)
export module math;

// Re-export partitions
export import :basic;
export import :advanced;

// Or selectively export
// import :basic;
// import :advanced;
// export using math::basic::add;
```

### Implementation Partitions
```cpp
// math:internal.cppm (internal partition, not exported)
module math:internal;

// Helper functions, not visible outside module
int helper_function() {
    return 42;
}
```

```cpp
// math.cppm
export module math;

import :internal;  // Use internal partition

export int public_function() {
    return helper_function();  // Use internal helper
}
```

---

## Importing Standard Library

### Header Units (C++20)
```cpp
import <vector>;
import <string>;
import <iostream>;
```

Header units bridge legacy `#include` headers into the module system. They still pull in macro pollution from C headers — prefer true modules where available.

### `import std` (C++23)

C++23 standardizes `import std;` for the entire standard library as a module (MSVC ships `std.ixx`; GCC/libc++ support is evolving). Check `__cpp_lib_modules` and your compiler version.

```cpp
import std;  // C++23 — entire standard library

int main() {
    std::vector<int> vec = {1, 2, 3};
    std::cout << "Hello, modules!\n";
}
```

⚠️ **Gotchas**: Apple Clang often lags on `import std` and PMF modules; many projects still mix `import <vector>;` or `#include` via the global module fragment.

---

## Mixing Modules and Headers

### Global Module Fragment

The global module fragment (`module;` … `export module …;`) is the only place to `#include` legacy headers before the module purview begins. Headers included here are not exported unless you re-export symbols.

```cpp
module;  // global module fragment — #includes allowed here

#include <vector>
#include <string>

export module mymodule;

export class MyClass {
    std::vector<std::string> data_;
};
```

### Interoperability
```cpp
// Can import modules and include headers together

// main.cpp
#include <iostream>  // Traditional header
import mymodule;     // Modern module

int main() {
    // Use both
    std::cout << "Using both!\n";
    MyClass obj;
    
    return 0;
}
```

---

## Module Examples

### Simple Math Module
```cpp
// math.cppm
export module math;

import <cmath>;

export namespace math {
    double add(double a, double b) {
        return a + b;
    }
    
    double multiply(double a, double b) {
        return a * b;
    }
    
    double power(double base, double exp) {
        return std::pow(base, exp);
    }
}
```

### String Utilities Module
```cpp
// string_utils.cppm
export module string_utils;

import <string>;
import <algorithm>;
import <cctype>;

export namespace string_utils {
    std::string to_upper(std::string str) {
        std::transform(str.begin(), str.end(), str.begin(),
                      [](unsigned char c) { return std::toupper(c); });
        return str;
    }
    
    std::string to_lower(std::string str) {
        std::transform(str.begin(), str.end(), str.begin(),
                      [](unsigned char c) { return std::tolower(c); });
        return str;
    }
    
    bool starts_with(const std::string& str, const std::string& prefix) {
        return str.substr(0, prefix.size()) == prefix;
    }
}
```

### Container Module with Template
```cpp
// container.cppm
export module container;

import <vector>;
import <stdexcept>;

export template<typename T>
class SafeVector {
    std::vector<T> data_;
    
public:
    void push_back(const T& value) {
        data_.push_back(value);
    }
    
    T& at(size_t index) {
        if (index >= data_.size()) {
            throw std::out_of_range("Index out of range");
        }
        return data_[index];
    }
    
    size_t size() const {
        return data_.size();
    }
};
```

---

## Private Module Fragment

### Hiding Implementation
```cpp
// widget.cppm
export module widget;

import <string>;

export class Widget {
public:
    Widget(std::string name);
    void display();
    
private:
    class Impl;  // Forward declaration
    Impl* pimpl_;
};

// Private module fragment (implementation hidden)
module :private;

class Widget::Impl {
public:
    std::string name_;
    int internal_data_;
    
    void internal_helper() {
        // Complex implementation
    }
};

Widget::Widget(std::string name) 
    : pimpl_(new Impl{std::move(name), 0}) {}

void Widget::display() {
    pimpl_->internal_helper();
}
```

---

## Module Visibility

### What Gets Exported?
```cpp
export module visibility;

// Exported: Visible to importers
export int public_var = 10;
export void public_func() {}
export class PublicClass {};

// Not exported: Module-internal
int private_var = 20;
void private_func() {}
class PrivateClass {};

// Exported function using private types
export void process() {
    PrivateClass obj;  // OK: Can use private types internally
    private_func();    // OK: Can call private functions
}

// ERROR: Can't export function with private type in signature
// export void bad(PrivateClass obj);  // ERROR!
```

### Transitive Visibility
```cpp
// a.cppm
export module a;

export struct A {
    int value;
};

// b.cppm
export module b;
import a;

export struct B {
    A a_member;  // A is visible in B's interface
};

// main.cpp
import b;

int main() {
    B obj;
    obj.a_member.value = 42;  // OK: A is transitively visible
    
    return 0;
}
```

---

## Compilation Model

### Build Process
```
Traditional Headers:
┌──────────────────────────────────────────────────┐
│ main.cpp    lib.cpp      util.cpp                │
│    ↓           ↓             ↓                   │
│ Preprocess  Preprocess   Preprocess              │
│ (parse all  (parse all   (parse all              │
│  headers    headers      headers                 │
│  again)     again)       again)   ← Redundant!   │
│    ↓           ↓             ↓                   │
│ Compile     Compile      Compile                 │
│    ↓           ↓             ↓                   │
│    └───────────┴─────────────┘                   │
│                ↓                                 │
│              Link                                │
└──────────────────────────────────────────────────┘

Modules:
┌──────────────────────────────────────────────────┐
│ mymodule.cppm                                    │
│    ↓                                             │
│ Compile once → BMI (Binary Module Interface)     │
│                 ↓                                │
│    ┌────────────┼─────────────┐                  │
│    ↓            ↓             ↓                  │
│ main.cpp    lib.cpp      util.cpp                │
│    ↓            ↓             ↓                  │
│ Compile     Compile       Compile                │
│ (import     (import       (import                │
│  BMI)       BMI)          BMI)    ← Fast!        │
│    ↓            ↓             ↓                  │
│    └────────────┴─────────────┘                  │
│                 ↓                                │
│               Link                               │
└──────────────────────────────────────────────────┘
```

---

## Best Practices

### 1. Module Names
```cpp
// GOOD: Descriptive, hierarchical
export module company.project.component;
export module myapp.utils.string;

// OK: Simple
export module math;

// BAD: Too generic
export module utils;
```

### 2. Organize with Partitions
```cpp
// Large module split into partitions
export module mylib;

export import :core;
export import :algorithms;
export import :io;

// Each partition in separate file
```

### 3. Export Judiciously
```cpp
// Only export what's needed
export module mymodule;

export class PublicAPI {
    // Public interface
};

// Internal helpers (not exported)
namespace internal {
    void helper() {}
}
```

### 4. Use Global Module Fragment for Legacy Headers
```cpp
module;

// Legacy headers that need macros/includes
#include <legacy_api.h>

export module wrapper;

// Wrap legacy code in modern module
```

---

## Migration from Headers

### Gradual Migration
```cpp
// Step 1: Wrap existing header via global module fragment
module;
#include "header.h"
export module mymodule;
export using ::Widget;

// Step 2: Importers switch
import mymodule;

// Step 3: Move definitions into the module; delete the header
```

### Conversion Example
```cpp
// Before (header):
// math.h
#ifndef MATH_H
#define MATH_H

namespace math {
    int add(int a, int b);
    int multiply(int a, int b);
}

#endif

// math.cpp
#include "math.h"

int math::add(int a, int b) { return a + b; }
int math::multiply(int a, int b) { return a * b; }

// After (module):
// math.cppm
export module math;

export namespace math {
    int add(int a, int b) { return a + b; }
    int multiply(int a, int b) { return a * b; }
}
```

---

## Compiler Support

### As of 2025
```
┌────────────────────────────────────────────────────────┐
│              Compiler Support Status                   │
├────────────────────────────────────────────────────────┤
│ Compiler      │ Version  │ Notes                       │
├───────────────┼──────────┼─────────────────────────────┤
│ GCC           │ 14–15    │ Native modules improving    │
│ Clang         │ 16+      │ Good; Apple Clang lags      │
│ MSVC          │ 19.28+   │ Strong std module support   │
│                                                        │
│ Build: CMake 3.28+ FILE_SET CXX_MODULES                │
│ Pitfall: compilation ORDER — interfaces before users   │
│ Pitfall: mixing TUs that import modules and #include   │
│          the same declarations → ODR violations        │
└────────────────────────────────────────────────────────┘
```

### CMake Example
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)
project(ModuleExample CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Module target
add_library(mymodule)
target_sources(mymodule
    PUBLIC
        FILE_SET CXX_MODULES FILES
            mymodule.cppm
)

# Executable using module
add_executable(main main.cpp)
target_link_libraries(main PRIVATE mymodule)
```

---

## Common Pitfalls

### 1. Circular Dependencies
```cpp
// BAD: Circular module imports
// a.cppm
export module a;
import b;  // ERROR if b imports a

// Use forward declarations or redesign
```

### 2. Macro Issues
```cpp
// BAD: Macros don't cross module boundaries
export module mymodule;

#define MY_MACRO 42  // Not visible to importers!

// GOOD: Use constexpr
export constexpr int MY_CONSTANT = 42;
```

### 3. ODR Violations During Migration
```cpp
// BAD: same symbols from header AND module in one link
#include "math.h"
import math;  // duplicate definitions at link time
```

### 4. Forgetting `export`
```cpp
export module mymodule;
class MyClass {};        // module-internal only
export class Public {};  // visible to importers
```

---

## Future Directions

### Finer-Grained Standard Modules
```cpp
import std.io;          // Proposed/partitioned std modules
import std.containers;
```

Module macro handling and broader libc++ module coverage remain active standardization and implementation work.

---

## Resources

- **C++20 Standard**: ISO/IEC 14882:2020
- **Compiler Documentation**: Check GCC, Clang, MSVC docs
- **Build System Support**: CMake, Ninja, others

---

## Complete Practical Example: Math Library Module

Here's a comprehensive example showing module definition, partitions, and usage:

```cpp
// ==================== math.cppm ====================
// Primary module interface

export module math;

export import :core;       // Re-export core partition
export import :advanced;   // Re-export advanced partition

export namespace math {
    constexpr double PI = 3.14159265358979323846;
    constexpr double E  = 2.71828182845904523536;
}

// ==================== math_core.cppm ====================
// Core partition

export module math:core;

import <stdexcept>;   // needed for std::invalid_argument (no #include in a module purview)

export namespace math {
    // Basic operations
    inline int add(int a, int b) {
        return a + b;
    }
    
    inline int multiply(int a, int b) {
        return a * b;
    }
    
    inline double divide(double a, double b) {
        if (b == 0.0) {
            throw std::invalid_argument("Division by zero");
        }
        return a / b;
    }
}

// ==================== math_advanced.cppm ====================
// Advanced partition

export module math:advanced;

import <cmath>;

export namespace math {
    inline double power(double base, double exp) {
        return std::pow(base, exp);
    }
    
    inline double sqrt(double x) {
        return std::sqrt(x);
    }
    
    inline double sin(double x) {
        return std::sin(x);
    }
    
    inline double cos(double x) {
        return std::cos(x);
    }
}

// ==================== main.cpp ====================
// Using the module

import math;
import <iostream>;

int main() {
    std::cout << "=== Math Module Demo ===\n\n";
    
    // Use core functions
    std::cout << "Basic operations:\n";
    std::cout << "5 + 3 = " << math::add(5, 3) << "\n";
    std::cout << "5 * 3 = " << math::multiply(5, 3) << "\n";
    std::cout << "10 / 2 = " << math::divide(10.0, 2.0) << "\n\n";
    
    // Use advanced functions  
    std::cout << "Advanced operations:\n";
    std::cout << "2^8 = " << math::power(2, 8) << "\n";
    std::cout << "sqrt(16) = " << math::sqrt(16) << "\n";
    std::cout << "sin(PI/2) = " << math::sin(math::PI / 2) << "\n";
    std::cout << "cos(0) = " << math::cos(0) << "\n\n";
    
    // Use constants
    std::cout << "Constants:\n";
    std::cout << "PI = " << math::PI << "\n";
    std::cout << "E = " << math::E << "\n";
    
    return 0;
}
```

### Compilation (GCC example):

> The exact flags are compiler- and version-specific and still evolving.
> Older GCC used `-fmodules-ts`; recent GCC (14/15) uses `-fmodules` and may
> need a module mapper. Clang uses `-fmodules`/precompiled module interfaces,
> and MSVC uses `/interface` / `/reference`. Always check your compiler's
> current module docs.

```bash
# Compile module partitions first
g++ -std=c++20 -fmodules-ts -c math_core.cppm
g++ -std=c++20 -fmodules-ts -c math_advanced.cppm

# Compile primary module interface
g++ -std=c++20 -fmodules-ts -c math.cppm

# Compile and link main
g++ -std=c++20 -fmodules-ts main.cpp math.o math_core.o math_advanced.o -o program

# Run
./program
```

### CMakeLists.txt (C++20 modules support):
```cmake
cmake_minimum_required(VERSION 3.28)
project(MathModule CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Module library
add_library(math_module)
target_sources(math_module
    PUBLIC
        FILE_SET CXX_MODULES FILES
            math_core.cppm
            math_advanced.cppm
            math.cppm
)

# Executable using module
add_executable(main main.cpp)
target_link_libraries(main PRIVATE math_module)
```

### Key Concepts Demonstrated:
1. **Interface vs implementation unit**: `export module` vs `module`
2. **Partitions**: `:core`, `:advanced` with `export import`
3. **Global module fragment**: `module;` + `#include` before `export module`
4. **Compilation order**: partitions → primary → consumers
5. **`import std`**: C++23 standard library module (where supported)

---

## Related Topics

- [Templates](09_templates.md) — exporting templates and concepts from modules
- [Metaprogramming](11_metaprogramming.md) — constexpr interfaces in module units
- [Advanced Features](12_advanced_features.md) — `consteval`, concepts with modules
- [Best Practices](13_best_practices.md) — incremental migration and ODR safety
- [OOP Concepts](00_oop_concepts.md) — encapsulation benefits vs headers
- [Modern C++ Features](07_modern_features.md) — C++20 language baseline for modules
- [Quick Reference](99_quick_reference.md) — module syntax cheat sheet

## Next Steps
- **Next**: [Coroutines (C++20) →](22_coroutines.md)
- **Previous**: [← Regular Expressions](20_regex.md)
- **Back to**: [Table of Contents](README.md)

---
*Chapter 21 — Modules (C++20)*

**Note**: Module support varies by compiler and build system. Check feature-test macros (`__cpp_modules`, `__cpp_lib_modules`) and your toolchain docs before adopting.

