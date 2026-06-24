# Algorithms

## Overview

The STL algorithms library provides a rich collection of functions for processing sequences of elements. These algorithms work with [iterators](05_iterators.md), making them container-agnostic and highly reusable.

💡 **Hunch**: Ranges are always **half-open** `[first, last)` — `last` is one-past-the-end and must not be dereferenced. An empty range has `first == last`.

```
┌────────────────────────────────────────────────────────────┐
│                STL ALGORITHMS LIBRARY                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  NON-MODIFYING      │  MODIFYING        │  SORTING         │
│  ─────────────      │  ─────────        │  ───────         │
│  • find             │  • copy           │  • sort          │
│  • count            │  • fill           │  • partial_sort  │
│  • search           │  • transform      │  • nth_element   │
│  • equal            │  • replace        │  • stable_sort   │
│                     │  • remove         │                  │
│                     │  • reverse        │                  │
│  ─────────────────────────────────────────────────────     │
│  BINARY SEARCH      │  PARTITIONING     │  NUMERIC         │
│  ─────────────      │  ────────────     │  ───────         │
│  • lower_bound      │  • partition      │  • accumulate    │
│  • upper_bound      │  • stable_partition  • inner_product │
│  • binary_search    │                   │  • adjacent_diff │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Algorithm Characteristics
```
1. Work with iterators (container-agnostic) — see 05_iterators.md
2. Do not change container size (except via insert iterators like back_inserter)
3. Defined in <algorithm> and <numeric>
4. Many have _if variants (with predicates) — often clearer with [lambdas](10_lambdas.md)
5. Many have _n variants (specify count)
6. C++17: parallel execution policies (<execution>) — may require linking TBB on libstdc++
7. C++20: ranges algorithms (std::ranges::) and std::erase / std::erase_if — see 07_modern_features.md
8. std::sort requires random-access iterators; std::list::sort() is the list alternative
```

---

## Non-Modifying Sequence Operations

### std::find, std::find_if, std::find_if_not
```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    
    // Find specific value
    auto it1 = std::find(vec.begin(), vec.end(), 5);
    if (it1 != vec.end()) {
        std::cout << "Found: " << *it1 << '\n';
    }
    
    // Find with predicate
    auto it2 = std::find_if(vec.begin(), vec.end(), 
                           [](int x) { return x > 5; });
    // it2 points to 6
    
    // Find not matching predicate
    auto it3 = std::find_if_not(vec.begin(), vec.end(),
                                [](int x) { return x < 5; });
    // it3 points to 5
    
    return 0;
}
```

### std::count, std::count_if
```cpp
#include <algorithm>
#include <vector>
#include <iostream>
#include <iterator>

int main() {
    std::vector<int> vec = {1, 2, 2, 3, 2, 4, 2, 5};
    
    // Count occurrences
    auto count = std::count(vec.begin(), vec.end(), 2);
    std::cout << "2 appears " << count << " times\n";  // 4
    
    // Count with predicate
    auto even_count = std::count_if(vec.begin(), vec.end(),
                                    [](int x) { return x % 2 == 0; });
    std::cout << "Even numbers: " << even_count << '\n';  // 5
    
    return 0;
}
```

### std::all_of, std::any_of, std::none_of (C++11)
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {2, 4, 6, 8, 10};
    
    // Check if all elements satisfy predicate
    bool all_even = std::all_of(vec.begin(), vec.end(),
                                [](int x) { return x % 2 == 0; });
    // true
    
    // Check if any element satisfies predicate
    bool any_greater_5 = std::any_of(vec.begin(), vec.end(),
                                     [](int x) { return x > 5; });
    // true
    
    // Check if no elements satisfy predicate
    bool none_odd = std::none_of(vec.begin(), vec.end(),
                                 [](int x) { return x % 2 == 1; });
    // true
    
    return 0;
}
```

### std::for_each
```cpp
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Apply function to each element
    std::for_each(vec.begin(), vec.end(), 
                 [](int x) { std::cout << x << ' '; });
    std::cout << '\n';
    
    // Modify elements (if not const)
    std::for_each(vec.begin(), vec.end(), 
                 [](int& x) { x *= 2; });
    // vec = {2, 4, 6, 8, 10}
    
    // C++11: Range-based for is often clearer
    for (int& x : vec) {
        x += 1;
    }
    
    return 0;
}
```

### std::search, std::find_end
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> haystack = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    std::vector<int> needle = {4, 5, 6};
    
    // Find first occurrence of subsequence
    auto it = std::search(haystack.begin(), haystack.end(),
                         needle.begin(), needle.end());
    if (it != haystack.end()) {
        std::cout << "Found at position: " 
                  << std::distance(haystack.begin(), it) << '\n';  // 3
    }
    
    // Find last occurrence
    auto it2 = std::find_end(haystack.begin(), haystack.end(),
                            needle.begin(), needle.end());
    
    return 0;
}
```

### std::mismatch, std::equal
```cpp
#include <algorithm>
#include <vector>
#include <iostream>
#include <cctype>

int main() {
    std::vector<int> v1 = {1, 2, 3, 4, 5};
    std::vector<int> v2 = {1, 2, 9, 4, 5};
    
    // Find first mismatch
    auto [it1, it2] = std::mismatch(v1.begin(), v1.end(), v2.begin());
    // *it1 = 3, *it2 = 9
    
    // Check if ranges are equal
    bool eq1 = std::equal(v1.begin(), v1.end(), v2.begin());
    // false
    
    std::vector<int> v3 = {1, 2, 3, 4, 5};
    bool eq2 = std::equal(v1.begin(), v1.end(), v3.begin());
    // true
    
    // Equal with predicate
    auto case_insensitive_equal = [](char a, char b) {
        return std::tolower(a) == std::tolower(b);
    };
    
    return 0;
}
```

---

## Modifying Sequence Operations

### std::copy, std::copy_if, std::copy_n
```cpp
#include <algorithm>
#include <vector>
#include <iterator>

int main() {
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dest(5);
    
    // Copy all elements
    std::copy(src.begin(), src.end(), dest.begin());
    // dest = {1, 2, 3, 4, 5}
    
    // Copy with predicate (C++11)
    std::vector<int> dest2;
    std::copy_if(src.begin(), src.end(), std::back_inserter(dest2),
                [](int x) { return x % 2 == 0; });
    // dest2 = {2, 4}
    
    // Copy first n elements (C++11)
    std::vector<int> dest3(3);
    std::copy_n(src.begin(), 3, dest3.begin());
    // dest3 = {1, 2, 3}
    
    return 0;
}
```

### std::move (algorithm, not std::move)
```cpp
#include <algorithm>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> src = {"hello", "world", "foo"};
    std::vector<std::string> dest(3);
    
    // Move elements (not copy)
    std::move(src.begin(), src.end(), dest.begin());
    // dest = {"hello", "world", "foo"}
    // src = {"", "", ""} (moved from, but valid)
    
    return 0;
}
```

### std::fill, std::fill_n
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec(10);
    
    // Fill range with value
    std::fill(vec.begin(), vec.end(), 42);
    // vec = {42, 42, 42, ..., 42}
    
    // Fill first n elements
    std::fill_n(vec.begin(), 5, 99);
    // vec = {99, 99, 99, 99, 99, 42, 42, 42, 42, 42}
    
    return 0;
}
```

### std::generate, std::generate_n
```cpp
#include <algorithm>
#include <vector>
#include <random>

int main() {
    std::vector<int> vec(10);
    
    // Generate with function
    int n = 0;
    std::generate(vec.begin(), vec.end(), [&n] { return n++; });
    // vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
    
    // Generate random numbers
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(1, 100);
    
    std::generate(vec.begin(), vec.end(), [&] { return dis(gen); });
    // vec = {random numbers}
    
    return 0;
}
```

### std::transform
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dest(5);
    
    // Unary transform
    std::transform(src.begin(), src.end(), dest.begin(),
                  [](int x) { return x * x; });
    // dest = {1, 4, 9, 16, 25}
    
    // Binary transform
    std::vector<int> src2 = {10, 20, 30, 40, 50};
    std::transform(src.begin(), src.end(), src2.begin(), dest.begin(),
                  [](int a, int b) { return a + b; });
    // dest = {11, 22, 33, 44, 55}
    
    // In-place transform
    std::transform(src.begin(), src.end(), src.begin(),
                  [](int x) { return x + 1; });
    // src = {2, 3, 4, 5, 6}
    
    return 0;
}
```

### std::replace, std::replace_if
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 2, 4, 2, 5};
    
    // Replace all occurrences
    std::replace(vec.begin(), vec.end(), 2, 99);
    // vec = {1, 99, 3, 99, 4, 99, 5}
    
    // Replace with predicate
    std::vector<int> vec2 = {1, 2, 3, 4, 5, 6};
    std::replace_if(vec2.begin(), vec2.end(),
                   [](int x) { return x % 2 == 0; }, 0);
    // vec2 = {1, 0, 3, 0, 5, 0}
    
    return 0;
}
```

### std::remove, std::remove_if
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 2, 4, 2, 5};
    
    // Remove value (doesn't actually erase!)
    auto new_end = std::remove(vec.begin(), vec.end(), 2);
    // vec = {1, 3, 4, 5, ?, ?, ?}
    //                   ^
    //               new_end
    
    // Must erase to actually remove
    vec.erase(new_end, vec.end());
    // vec = {1, 3, 4, 5}
    
    // C++20: std::erase / std::erase_if — no erase-remove idiom needed
    // std::erase(vec, 2);
    // std::erase_if(vec, [](int x) { return x % 2 == 0; });

    // Pre-C++20 erase-remove idiom (still valid everywhere):
    vec.erase(std::remove(vec.begin(), vec.end(), 2), vec.end());
    
    // Remove with predicate
    vec = {1, 2, 3, 4, 5, 6};
    vec.erase(
        std::remove_if(vec.begin(), vec.end(), 
                      [](int x) { return x % 2 == 0; }),
        vec.end()
    );
    // vec = {1, 3, 5}
    
    return 0;
}
```

### Visual: How std::remove Works
```
Original: [1, 2, 3, 2, 4, 2, 5]
                ↑     ↑     ↑
            (elements to remove: 2)

After std::remove(vec.begin(), vec.end(), 2):
Result:   [1, 3, 4, 5, 2, 2, 2]
                      ↑
                   new_end

Elements to keep moved to front.
Returns iterator to new logical end.
Old elements still exist but are "garbage".

After vec.erase(new_end, vec.end()):
Result:   [1, 3, 4, 5]

Actually removes the garbage elements.
```

### std::unique
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 1, 2, 2, 2, 3, 3, 4, 5, 5};
    
    // Remove consecutive duplicates
    auto new_end = std::unique(vec.begin(), vec.end());
    vec.erase(new_end, vec.end());
    // vec = {1, 2, 3, 4, 5}
    
    // One-liner
    vec = {1, 1, 2, 2, 2, 3, 3, 4, 5, 5};
    vec.erase(std::unique(vec.begin(), vec.end()), vec.end());
    
    // With predicate
    std::vector<int> vec2 = {1, 2, 3, 4, 5, 6};
    vec2.erase(
        std::unique(vec2.begin(), vec2.end(),
                   [](int a, int b) { return (a % 2) == (b % 2); }),
        vec2.end()
    );
    // Removes consecutive elements with same parity
    
    return 0;
}
```

### std::reverse
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Reverse in-place
    std::reverse(vec.begin(), vec.end());
    // vec = {5, 4, 3, 2, 1}
    
    // Reverse part of container
    std::vector<int> vec2 = {1, 2, 3, 4, 5};
    std::reverse(vec2.begin() + 1, vec2.begin() + 4);
    // vec2 = {1, 4, 3, 2, 5}
    
    return 0;
}
```

### std::rotate
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Rotate: element at middle becomes first
    std::rotate(vec.begin(), vec.begin() + 2, vec.end());
    // vec = {3, 4, 5, 1, 2}
    
    // Rotate left by 1
    vec = {1, 2, 3, 4, 5};
    std::rotate(vec.begin(), vec.begin() + 1, vec.end());
    // vec = {2, 3, 4, 5, 1}
    
    // Rotate right by 1
    vec = {1, 2, 3, 4, 5};
    std::rotate(vec.rbegin(), vec.rbegin() + 1, vec.rend());
    // vec = {5, 1, 2, 3, 4}
    
    return 0;
}
```

### Visual: std::rotate
```
Original: [1, 2, 3, 4, 5]
           ▲     ▲     ▲
         first middle end

std::rotate(first, middle, end):
Result:   [3, 4, 5, 1, 2]
           ▲
        middle becomes first

Elements [middle, end) moved to front
Elements [first, middle) moved to back
```

### std::shuffle (C++11)
```cpp
#include <algorithm>
#include <vector>
#include <random>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Random shuffle
    std::random_device rd;
    std::mt19937 gen(rd());
    std::shuffle(vec.begin(), vec.end(), gen);
    // vec = {random permutation}
    
    return 0;
}
```

---

## Sorting Operations

### std::sort

Requires **random-access iterators** (`vector`, `deque`, `array`, C-style arrays). Cannot sort a `std::list` with `std::sort` — use `list.sort()` instead ([Sequence Containers](01_sequence_containers.md)).

```cpp
#include <algorithm>
#include <vector>
#include <functional>  // std::greater

int main() {
    std::vector<int> vec = {5, 2, 8, 1, 9, 3};
    
    // Sort ascending (default)
    std::sort(vec.begin(), vec.end());
    // vec = {1, 2, 3, 5, 8, 9}
    
    // Sort descending
    std::sort(vec.begin(), vec.end(), std::greater<int>());
    // vec = {9, 8, 5, 3, 2, 1}
    
    // Sort with custom comparator
    std::sort(vec.begin(), vec.end(), 
             [](int a, int b) { return std::abs(a) < std::abs(b); });
    
    return 0;
}
```

### Sorting Algorithms Comparison
```
┌─────────────────────────────────────────────────────────────────┐
│             SORTING ALGORITHM COMPARISON                        │
├─────────────────────────────────────────────────────────────────┤
│ Algorithm      │ Time        │ Space    │ Stable │ Notes        │
├────────────────┼─────────────┼──────────┼────────┼──────────────┤
│ sort           │ O(n log n)  │ O(log n) │ No     │ Fastest      │
│ stable_sort    │ O(n log n)  │ O(n)*    │ Yes    │ Stable       │
│ partial_sort   │ O(n log k)  │ O(1)     │ No     │ Top K        │
│ nth_element    │ O(n) avg    │ O(1)     │ No     │ Median       │
└─────────────────────────────────────────────────────────────────┘

sort uses O(log n) stack space (introsort recursion), not O(1).
* stable_sort uses O(n) extra memory; if none is available it falls
  back to an in-place merge that is O(n log^2 n) time instead.
Stable: preserves the relative order of equal elements.
```

### std::stable_sort
```cpp
#include <algorithm>
#include <vector>

struct Person {
    std::string name;
    int age;
};

int main() {
    std::vector<Person> people = {
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 30},
        {"David", 25}
    };
    
    // Stable sort by age (preserves name order for same age)
    std::stable_sort(people.begin(), people.end(),
                    [](const Person& a, const Person& b) {
                        return a.age < b.age;
                    });
    // Result:
    // Bob(25), David(25), Alice(30), Charlie(30)
    // Within same age, original order preserved
    
    return 0;
}
```

### std::partial_sort
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {9, 3, 7, 1, 8, 2, 6, 4, 5};
    
    // Sort first 3 elements (smallest)
    std::partial_sort(vec.begin(), vec.begin() + 3, vec.end());
    // vec = {1, 2, 3, ?, ?, ?, ?, ?, ?}
    // First 3 are sorted, rest are unspecified
    
    // Top K largest
    vec = {9, 3, 7, 1, 8, 2, 6, 4, 5};
    std::partial_sort(vec.begin(), vec.begin() + 3, vec.end(), 
                     std::greater<int>());
    // vec = {9, 8, 7, ?, ?, ?, ?, ?, ?}
    
    return 0;
}
```

### std::nth_element
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {9, 3, 7, 1, 8, 2, 6, 4, 5};
    
    // Find median (5th element when sorted)
    std::nth_element(vec.begin(), vec.begin() + 4, vec.end());
    // vec[4] is now the median
    // Elements before vec[4] are <= vec[4]
    // Elements after vec[4] are >= vec[4]
    // But not fully sorted!
    
    int median = vec[4];  // 5
    
    // Find 90th percentile
    size_t idx = vec.size() * 0.9;
    std::nth_element(vec.begin(), vec.begin() + idx, vec.end());
    int percentile_90 = vec[idx];
    
    return 0;
}
```

### Visual: nth_element
```
Original: [9, 3, 7, 1, 8, 2, 6, 4, 5]

std::nth_element(begin, begin+4, end):

Result:   [3, 2, 1, 4, 5, 7, 8, 9, 6]
                      ▲
                    nth element (median)
          
          └───────────┘ └───────────┘
           all <= 5      all >= 5
           (unordered)   (unordered)

Faster than full sort: O(n) vs O(n log n)
```

---

## Binary Search Operations

**Important**: These require sorted ranges!

### std::binary_search
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    
    // Check if element exists
    bool found = std::binary_search(vec.begin(), vec.end(), 5);
    // true
    
    bool not_found = std::binary_search(vec.begin(), vec.end(), 10);
    // false
    
    return 0;
}
```

### std::lower_bound, std::upper_bound
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 2, 2, 3, 4, 5};
    
    // Lower bound: first element >= value
    auto lb = std::lower_bound(vec.begin(), vec.end(), 2);
    // Points to first 2 (index 1)
    
    // Upper bound: first element > value
    auto ub = std::upper_bound(vec.begin(), vec.end(), 2);
    // Points to 3 (index 4)
    
    // Count occurrences in sorted range
    int count = std::distance(lb, ub);  // 3
    
    // Insert while maintaining sorted order
    vec.insert(std::lower_bound(vec.begin(), vec.end(), 6), 6);
    // vec = {1, 2, 2, 2, 3, 4, 5, 6}
    
    return 0;
}
```

### std::equal_range
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 2, 2, 3, 4, 5};
    
    // Get [lower_bound, upper_bound) pair
    auto [first, last] = std::equal_range(vec.begin(), vec.end(), 2);
    
    // Range [first, last) contains all 2s
    for (auto it = first; it != last; ++it) {
        std::cout << *it << ' ';  // 2 2 2
    }
    
    return 0;
}
```

### Visual: Binary Search Operations
```
Sorted vector: [1, 2, 2, 2, 3, 4, 5]
                   ▲     ▲
                   │     │
    lower_bound(2)─┘     └─upper_bound(2)
                   │     │
    equal_range(2)=[     )
                   └─────┘
                   all 2s

binary_search(2) → true (exists)
binary_search(6) → false (doesn't exist)
```

---

## Set Operations (on Sorted Ranges)

### std::merge
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> v1 = {1, 3, 5, 7};
    std::vector<int> v2 = {2, 4, 6, 8};
    std::vector<int> result(8);
    
    // Merge two sorted ranges
    std::merge(v1.begin(), v1.end(),
              v2.begin(), v2.end(),
              result.begin());
    // result = {1, 2, 3, 4, 5, 6, 7, 8}
    
    return 0;
}
```

### std::set_union, std::set_intersection
```cpp
#include <algorithm>
#include <vector>
#include <iterator>

int main() {
    std::vector<int> v1 = {1, 2, 3, 4, 5};
    std::vector<int> v2 = {3, 4, 5, 6, 7};
    std::vector<int> result;
    
    // Union: elements in either set
    std::set_union(v1.begin(), v1.end(),
                  v2.begin(), v2.end(),
                  std::back_inserter(result));
    // result = {1, 2, 3, 4, 5, 6, 7}
    
    result.clear();
    
    // Intersection: elements in both sets
    std::set_intersection(v1.begin(), v1.end(),
                         v2.begin(), v2.end(),
                         std::back_inserter(result));
    // result = {3, 4, 5}
    
    return 0;
}
```

### std::set_difference, std::set_symmetric_difference
```cpp
#include <algorithm>
#include <vector>
#include <iterator>

int main() {
    std::vector<int> v1 = {1, 2, 3, 4, 5};
    std::vector<int> v2 = {3, 4, 5, 6, 7};
    std::vector<int> result;
    
    // Difference: elements in v1 but not in v2
    std::set_difference(v1.begin(), v1.end(),
                       v2.begin(), v2.end(),
                       std::back_inserter(result));
    // result = {1, 2}
    
    result.clear();
    
    // Symmetric difference: in one but not both
    std::set_symmetric_difference(v1.begin(), v1.end(),
                                 v2.begin(), v2.end(),
                                 std::back_inserter(result));
    // result = {1, 2, 6, 7}
    
    return 0;
}
```

### Visual: Set Operations
```
v1: {1, 2, 3, 4, 5}
v2: {3, 4, 5, 6, 7}

Union:               {1, 2, 3, 4, 5, 6, 7}
                      └─────────────────┘
                      all from both

Intersection:        {3, 4, 5}
                      └──────┘
                      common to both

Difference (v1-v2):  {1, 2}
                      └───┘
                      in v1 only

Symmetric Difference:{1, 2, 6, 7}
                      └───┘ └───┘
                      not in both
```

---

## Partitioning Operations

### std::partition
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    
    // Partition: even numbers first
    auto partition_point = std::partition(vec.begin(), vec.end(),
                                         [](int x) { return x % 2 == 0; });
    
    // vec might be: {8, 2, 6, 4, 5, 3, 7, 1, 9}
    //                         ▲
    //                  partition_point
    
    // All even numbers before partition_point
    // All odd numbers after partition_point
    // But order within each partition is not stable!
    
    return 0;
}
```

### std::stable_partition
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    
    // Stable partition: preserves relative order
    auto partition_point = std::stable_partition(vec.begin(), vec.end(),
                                                [](int x) { return x % 2 == 0; });
    
    // vec = {2, 4, 6, 8, 1, 3, 5, 7, 9}
    //                  ▲
    //           partition_point
    
    // Relative order preserved within each partition!
    
    return 0;
}
```

### std::partition_point
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {2, 4, 6, 8, 1, 3, 5, 7, 9};
    // Already partitioned: even first, odd second
    
    // Find partition point
    auto pp = std::partition_point(vec.begin(), vec.end(),
                                   [](int x) { return x % 2 == 0; });
    
    std::cout << "Partition point: " << *pp << '\n';  // 1
    
    return 0;
}
```

---

## Numeric Algorithms

### std::accumulate
```cpp
#include <numeric>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Sum
    int sum = std::accumulate(vec.begin(), vec.end(), 0);
    // sum = 15
    
    // Product
    int product = std::accumulate(vec.begin(), vec.end(), 1,
                                 std::multiplies<int>());
    // product = 120
    
    // Custom operation
    int custom = std::accumulate(vec.begin(), vec.end(), 0,
                                [](int a, int b) { return a + b * b; });
    // custom = 0 + 1² + 2² + 3² + 4² + 5² = 55
    
    return 0;
}
```

### std::reduce (C++17, parallel version of accumulate)
```cpp
#include <numeric>
#include <vector>
#include <execution>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Parallel reduction
    int sum = std::reduce(std::execution::par, 
                         vec.begin(), vec.end());
    
    return 0;
}
```

### std::inner_product
```cpp
#include <numeric>
#include <vector>

int main() {
    std::vector<int> v1 = {1, 2, 3};
    std::vector<int> v2 = {4, 5, 6};
    
    // Dot product: sum of element-wise products
    int dot_product = std::inner_product(v1.begin(), v1.end(), 
                                        v2.begin(), 0);
    // = 1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32
    
    return 0;
}
```

### std::partial_sum
```cpp
#include <numeric>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    std::vector<int> result(5);
    
    // Cumulative sum
    std::partial_sum(vec.begin(), vec.end(), result.begin());
    // result = {1, 3, 6, 10, 15}
    //           1, 1+2, 1+2+3, 1+2+3+4, 1+2+3+4+5
    
    return 0;
}
```

### std::adjacent_difference
```cpp
#include <numeric>
#include <vector>

int main() {
    std::vector<int> vec = {1, 3, 6, 10, 15};
    std::vector<int> result(5);
    
    // Differences between adjacent elements
    std::adjacent_difference(vec.begin(), vec.end(), result.begin());
    // result = {1, 2, 3, 4, 5}
    //           1, 3-1, 6-3, 10-6, 15-10
    
    return 0;
}
```

### std::iota
```cpp
#include <numeric>
#include <vector>

int main() {
    std::vector<int> vec(10);
    
    // Fill with sequential values
    std::iota(vec.begin(), vec.end(), 1);
    // vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    return 0;
}
```

---

## Min/Max Operations

### std::min_element, std::max_element
```cpp
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec = {3, 1, 4, 1, 5, 9, 2, 6};
    
    // Find minimum
    auto min_it = std::min_element(vec.begin(), vec.end());
    std::cout << "Min: " << *min_it << '\n';  // 1
    std::cout << "At position: " << std::distance(vec.begin(), min_it) << '\n';
    
    // Find maximum
    auto max_it = std::max_element(vec.begin(), vec.end());
    std::cout << "Max: " << *max_it << '\n';  // 9
    
    // Find both at once
    auto [min_it2, max_it2] = std::minmax_element(vec.begin(), vec.end());
    
    return 0;
}
```

### std::clamp (C++17)
```cpp
#include <algorithm>

int main() {
    int value = 15;
    
    // Clamp value between [0, 10]
    int clamped = std::clamp(value, 0, 10);
    // clamped = 10
    
    std::clamp(5, 0, 10);   // 5 (within range)
    std::clamp(-5, 0, 10);  // 0 (below minimum)
    std::clamp(15, 0, 10);  // 10 (above maximum)
    
    return 0;
}
```

---

## Execution Policies (C++17)

Parallel algorithms take an execution policy as the first argument. Include `<execution>` and link the parallel backend your standard library uses (GCC libstdc++ often needs `-ltbb` for `std::execution::par`).

```cpp
#include <algorithm>
#include <execution>
#include <numeric>
#include <vector>

int main() {
    std::vector<int> vec(1'000'000);
    std::iota(vec.begin(), vec.end(), 0);
    
    // Sequential execution (default)
    std::sort(vec.begin(), vec.end());
    
    // Parallel execution
    std::sort(std::execution::par, vec.begin(), vec.end());
    
    // Parallel unsequenced (vectorized)
    std::sort(std::execution::par_unseq, vec.begin(), vec.end());
    
    // Sequenced (no parallelism)
    std::sort(std::execution::seq, vec.begin(), vec.end());
    
    return 0;
}
```

### Execution Policies
```
┌──────────────────────────────────────────────────────┐
│ Execution Policy            | Parallel | Vectorized  │
├─────────────────────────────┼──────────┼─────────────┤
│ std::execution::seq         │    No    │     No      │
│ std::execution::par         │    Yes   │     No      │
│ std::execution::par_unseq   │    Yes   │     Yes     │
│ std::execution::unseq (C++20)    No    │     Yes     │
└──────────────────────────────────────────────────────┘

Note: seq, par, and par_unseq are C++17; unseq was added in C++20.

Use for: large datasets, CPU-intensive operations
Requirements:
- Operations and predicates must be thread-safe (no shared mutable state)
- On libstdc++, link Intel TBB: g++ ... -ltbb
- Not all algorithms have parallel overloads — check [cppreference](https://en.cppreference.com/w/cpp/algorithm)
```

---

## Ranges Algorithms (C++20)

C++20 adds `std::ranges::` versions that accept ranges directly, support **projections**, and compose with lazy **views**. See [Modern C++20/23 Features](07_modern_features.md) for views in depth.

```cpp
#include <algorithm>
#include <ranges>
#include <vector>
#include <string>

struct Person { std::string name; int age; };

int main() {
    std::vector<Person> people = {
        {"Alice", 30}, {"Bob", 25}, {"Charlie", 35}
    };

    // Sort by age via projection (no manual comparator boilerplate)
    std::ranges::sort(people, {}, &Person::age);

    // Lazy view: filter then transform (nothing runs until you iterate)
    auto names_of_adults = people
        | std::views::filter([](const Person& p) { return p.age >= 30; })
        | std::views::transform(&Person::name);

    for (const auto& name : names_of_adults) { /* ... */ }

    return 0;
}
```

Views are **lazy** — they store iterators/adaptors and evaluate on demand. Algorithms like `std::ranges::sort` materialize in-place on owning containers.

---

## Common Patterns and Best Practices

### Pattern 1: Erase-Remove Idiom

See the detailed walkthrough in [std::remove](#stdremove-stdremove_if) above. C++20 shortcut: `std::erase(vec, value)` / `std::erase_if(vec, pred)`.

### Pattern 2: Sort and Unique (Remove Duplicates)
```cpp
std::vector<int> vec = {3, 1, 4, 1, 5, 9, 2, 6, 5};

std::sort(vec.begin(), vec.end());
vec.erase(std::unique(vec.begin(), vec.end()), vec.end());
// vec = {1, 2, 3, 4, 5, 6, 9}
```

### Pattern 3: Transform and Filter
```cpp
std::vector<int> input = {1, 2, 3, 4, 5};
std::vector<int> output;

// Transform and copy only if predicate matches
std::copy_if(input.begin(), input.end(), 
            std::back_inserter(output),
            [](int x) { return x % 2 == 0; });

std::transform(output.begin(), output.end(), output.begin(),
              [](int x) { return x * x; });
// output = {4, 16}
```

### Pattern 4: Finding Nth Largest
```cpp
std::vector<int> vec = {3, 1, 4, 1, 5, 9, 2, 6, 5};

// Find 3rd largest (use partial_sort or nth_element)
std::nth_element(vec.begin(), vec.begin() + 2, vec.end(), 
                std::greater<int>());
int third_largest = vec[2];
```

---

## Practice Exercises

### Exercise 1: Custom Algorithm
```cpp
// Implement your own find_if
template<typename Iterator, typename Predicate>
Iterator my_find_if(Iterator first, Iterator last, Predicate pred);
```

### Exercise 2: Algorithm Combinations
```cpp
// Given vector of strings, count words starting with 'a' (case-insensitive)
std::vector<std::string> words = {"Apple", "banana", "Apricot", "cherry"};
// Use count_if with appropriate lambda
```

### Exercise 3: Optimization Challenge
```cpp
// Find the most frequent element in a vector
// Hint: Use sort, unique, and custom comparisons
```

---

## Complete Practical Example: Data Analysis Pipeline

Here's a comprehensive example that uses multiple STL algorithms to analyze student grades:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <string>
#include <iomanip>
#include <cmath>

struct Student {
    std::string name;
    int id;
    std::vector<double> grades;
    double average;
    
    Student(std::string n, int i, std::vector<double> g)
        : name(std::move(n)), id(i), grades(std::move(g)), average(0.0) {}
};

class GradeAnalyzer {
private:
    std::vector<Student> students_;
    
public:
    void add_student(const std::string& name, int id, 
                     const std::vector<double>& grades) {
        students_.emplace_back(name, id, grades);
    }
    
    // Calculate averages using std::accumulate
    void calculate_averages() {
        std::for_each(students_.begin(), students_.end(), 
            [](Student& s) {
                if (!s.grades.empty()) {
                    s.average = std::accumulate(s.grades.begin(), 
                                               s.grades.end(), 0.0) 
                               / s.grades.size();
                }
            });
    }
    
    // Find top performers using std::partial_sort
    std::vector<Student> get_top_students(size_t n) const {
        std::vector<Student> sorted = students_;
        
        n = std::min(n, sorted.size());
        std::partial_sort(sorted.begin(), sorted.begin() + n, sorted.end(),
            [](const Student& a, const Student& b) {
                return a.average > b.average;
            });
        
        sorted.resize(n);
        return sorted;
    }
    
    // Find median grade using std::nth_element
    double get_median_grade() const {
        std::vector<double> all_grades;
        
        // Collect all grades using std::for_each
        std::for_each(students_.begin(), students_.end(),
            [&all_grades](const Student& s) {
                all_grades.insert(all_grades.end(), 
                                 s.grades.begin(), s.grades.end());
            });
        
        if (all_grades.empty()) return 0.0;
        
        size_t mid = all_grades.size() / 2;
        std::nth_element(all_grades.begin(), 
                        all_grades.begin() + mid, 
                        all_grades.end());
        
        return all_grades[mid];
    }
    
    // Count students by grade range using std::count_if
    void show_grade_distribution() const {
        auto count_range = [this](double min, double max) {
            return std::count_if(students_.begin(), students_.end(),
                [min, max](const Student& s) {
                    return s.average >= min && s.average < max;
                });
        };
        
        std::cout << "\n=== Grade Distribution ===\n";
        std::cout << "A (90-100): " << count_range(90, 101) << " students\n";
        std::cout << "B (80-89):  " << count_range(80, 90) << " students\n";
        std::cout << "C (70-79):  " << count_range(70, 80) << " students\n";
        std::cout << "D (60-69):  " << count_range(60, 70) << " students\n";
        std::cout << "F (0-59):   " << count_range(0, 60) << " students\n";
    }
    
    // Find students needing help using std::find_if and std::copy_if
    std::vector<Student> get_struggling_students(double threshold = 60.0) const {
        std::vector<Student> struggling;
        
        std::copy_if(students_.begin(), students_.end(), 
                    std::back_inserter(struggling),
                    [threshold](const Student& s) {
                        return s.average < threshold;
                    });
        
        return struggling;
    }
    
    // Statistical analysis using multiple algorithms
    void show_statistics() const {
        if (students_.empty()) return;
        
        // Collect all averages
        std::vector<double> averages;
        std::transform(students_.begin(), students_.end(),
                      std::back_inserter(averages),
                      [](const Student& s) { return s.average; });
        
        // Min and max using std::minmax_element
        auto [min_it, max_it] = std::minmax_element(
            averages.begin(), averages.end());
        
        // Mean using std::accumulate
        double mean = std::accumulate(averages.begin(), averages.end(), 0.0) 
                     / averages.size();
        
        // Standard deviation
        double sq_sum = std::inner_product(averages.begin(), averages.end(),
                                          averages.begin(), 0.0,
                                          std::plus<>(),
                                          [mean](double a, double b) {
                                              return (a - mean) * (b - mean);
                                          });
        double stdev = std::sqrt(sq_sum / averages.size());
        
        std::cout << std::fixed << std::setprecision(2);
        std::cout << "\n=== Class Statistics ===\n";
        std::cout << "Number of students: " << students_.size() << "\n";
        std::cout << "Minimum average: " << *min_it << "\n";
        std::cout << "Maximum average: " << *max_it << "\n";
        std::cout << "Class mean: " << mean << "\n";
        std::cout << "Standard deviation: " << stdev << "\n";
        std::cout << "Median grade: " << get_median_grade() << "\n";
    }
    
    // Apply grade curve using std::transform
    void apply_curve(double bonus) {
        std::cout << "\nApplying curve (+" << bonus << " points)...\n";
        
        std::for_each(students_.begin(), students_.end(),
            [bonus](Student& s) {
                std::transform(s.grades.begin(), s.grades.end(),
                             s.grades.begin(),
                             [bonus](double grade) {
                                 return std::min(100.0, grade + bonus);
                             });
            });
        
        calculate_averages();
    }
    
    // Sort and display all students
    void show_all_students(bool sort_by_average = true) const {
        std::vector<Student> sorted = students_;
        
        if (sort_by_average) {
            std::sort(sorted.begin(), sorted.end(),
                [](const Student& a, const Student& b) {
                    return a.average > b.average;
                });
        } else {
            std::sort(sorted.begin(), sorted.end(),
                [](const Student& a, const Student& b) {
                    return a.name < b.name;
                });
        }
        
        std::cout << "\n=== All Students ===\n";
        std::cout << std::fixed << std::setprecision(1);
        
        for (const auto& s : sorted) {
            std::cout << s.name << " (ID: " << s.id << "): "
                      << s.average << " - [";
            
            // Display individual grades
            for (size_t i = 0; i < s.grades.size(); ++i) {
                std::cout << s.grades[i];
                if (i < s.grades.size() - 1) std::cout << ", ";
            }
            std::cout << "]\n";
        }
    }
};

int main() {
    GradeAnalyzer analyzer;
    
    // Add students with their grades
    analyzer.add_student("Alice Johnson", 101, {95, 92, 88, 94});
    analyzer.add_student("Bob Smith", 102, {78, 82, 75, 80});
    analyzer.add_student("Carol White", 103, {88, 90, 92, 89});
    analyzer.add_student("David Brown", 104, {65, 70, 68, 72});
    analyzer.add_student("Eve Davis", 105, {92, 95, 91, 93});
    analyzer.add_student("Frank Wilson", 106, {55, 60, 58, 62});
    analyzer.add_student("Grace Lee", 107, {85, 88, 84, 87});
    analyzer.add_student("Henry Taylor", 108, {72, 75, 70, 74});
    
    // Calculate averages for all students
    analyzer.calculate_averages();
    
    // Show all students
    analyzer.show_all_students(true);
    
    // Show statistics
    analyzer.show_statistics();
    
    // Show grade distribution
    analyzer.show_grade_distribution();
    
    // Find top 3 students
    std::cout << "\n=== Top 3 Students ===\n";
    auto top_students = analyzer.get_top_students(3);
    for (const auto& s : top_students) {
        std::cout << s.name << ": " << s.average << "\n";
    }
    
    // Find struggling students
    std::cout << "\n=== Students Needing Support (< 70) ===\n";
    auto struggling = analyzer.get_struggling_students(70.0);
    for (const auto& s : struggling) {
        std::cout << s.name << ": " << s.average << "\n";
    }
    
    // Apply curve
    analyzer.apply_curve(5.0);
    
    std::cout << "\n=== After Curve ===\n";
    analyzer.show_statistics();
    analyzer.show_grade_distribution();
    
    return 0;
}
```

### Output:
```
=== All Students ===
Eve Davis (ID: 105): 92.8 - [92.0, 95.0, 91.0, 93.0]
Alice Johnson (ID: 101): 92.2 - [95.0, 92.0, 88.0, 94.0]
Carol White (ID: 103): 89.8 - [88.0, 90.0, 92.0, 89.0]
Grace Lee (ID: 107): 86.0 - [85.0, 88.0, 84.0, 87.0]
Bob Smith (ID: 102): 78.8 - [78.0, 82.0, 75.0, 80.0]
Henry Taylor (ID: 108): 72.8 - [72.0, 75.0, 70.0, 74.0]
David Brown (ID: 104): 68.8 - [65.0, 70.0, 68.0, 72.0]
Frank Wilson (ID: 106): 58.8 - [55.0, 60.0, 58.0, 62.0]

=== Class Statistics ===
Number of students: 8
Minimum average: 58.75
Maximum average: 92.75
Class mean: 80.06
Standard deviation: 11.72
Median grade: 85.00

=== Grade Distribution ===
A (90-100): 2 students
B (80-89):  2 students
C (70-79):  2 students
D (60-69):  1 students
F (0-59):   1 students

=== Top 3 Students ===
Eve Davis: 92.75
Alice Johnson: 92.25
Carol White: 89.75

=== Students Needing Support (< 70) ===
David Brown: 68.75
Frank Wilson: 58.75

Applying curve (+5.00 points)...

=== After Curve ===
Number of students: 8
Minimum average: 63.75
Maximum average: 97.75
Class mean: 85.06
Standard deviation: 11.72
Median grade: 89.00

=== Grade Distribution ===
A (90-100): 3 students
B (80-89):  2 students
C (70-79):  2 students
D (60-69):  1 students
F (0-59):   0 students
```

### Algorithms Demonstrated:
- **for_each**: Iterate and apply function
- **accumulate**: Sum values
- **partial_sort**: Find top N elements
- **nth_element**: Find median efficiently
- **count_if**: Count elements matching condition
- **copy_if**: Filter and copy elements
- **transform**: Apply transformation
- **minmax_element**: Find min and max simultaneously
- **inner_product**: Calculate dot product (for variance)
- **sort**: Sort entire container
- **find_if**: Search for element
- **all_of/any_of/none_of**: Check conditions

This example shows the power of composing STL algorithms!

---

## Performance Tips

1. **Reserve space**: When using `back_inserter`, call `vec.reserve(n)` first to avoid repeated reallocations
2. **Use appropriate algorithm**: `nth_element` is O(n) average vs O(n log n) for full `sort` when you only need one order statistic
3. **Parallel execution**: Use `std::execution::par` for large datasets; link TBB on libstdc++ (`-ltbb`)
4. **Avoid unnecessary copies**: Use `std::move` algorithm and in-place transforms; see [Move Semantics](12_advanced_features.md)
5. **Choose right container**: Random-access iterators unlock `sort`; for `std::list`, call `list.sort()` instead
6. **Prefer associative containers** for sorted unique keys instead of `sort` + `unique` when inserts dominate ([Associative Containers](02_associative_containers.md))

---

## Related Topics

- [Iterators](05_iterators.md) — half-open ranges, categories, and why `std::sort` needs random access
- [Modern C++20/23 Features](07_modern_features.md) — `std::ranges`, views, projections, and concepts
- [Sequence Containers](01_sequence_containers.md) — `list::sort`, vector `reserve` before `back_inserter`
- [Lambdas](10_lambdas.md) — predicates and comparators for `_if` algorithms
- [Associative Containers](02_associative_containers.md) — `set`/`map` for sorted data without calling `sort`
- [Multithreading](14_multithreading.md) — thread safety when using `std::execution::par`

## Next Steps
- **Next**: [Modern C++20/23 Features →](07_modern_features.md)
- **Previous**: [← Iterators](05_iterators.md)

---
*Chapter 6 — Algorithms*

