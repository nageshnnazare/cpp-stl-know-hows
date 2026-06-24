# Iterators

## Overview

Iterators are the glue between [containers](01_sequence_containers.md) and [algorithms](06_algorithms.md) in the STL. They provide a uniform way to traverse and access elements, similar to pointers but more abstract and type-safe.

💡 **Hunch**: Almost every STL algorithm takes a **half-open range** `[first, last)` — `first` is inclusive, `last` is one-past-the-end. Empty range ⇔ `first == last`. This is why `end()` is not dereferenceable.

```
┌──────────────────────────────────────────────────────────┐
│                  ITERATOR CONCEPT                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │
│  │ CONTAINER   │───►│  ITERATOR   │◄───│ ALGORITHM   │   │
│  │             │    │             │    │             │   │
│  │ Stores data │    │ Accesses    │    │ Processes   │   │
│  │             │    │ elements    │    │ data        │   │
│  └─────────────┘    └─────────────┘    └─────────────┘   │
│                                                          │
│  Abstraction: Algorithms don't need to know about        │
│  container internals, only iterator interface            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### The Power of Iterators
```
Without iterators:
- std::sort needs different code for vector, deque, array
- Each container needs its own set of algorithms

With iterators:
- std::sort works with ANY container that provides random-access iterators ([cppreference](https://en.cppreference.com/w/cpp/named_req/RandomAccessIterator))
- Write algorithm once, use everywhere
- Container adaptors ([stack/queue/priority_queue](04_container_adaptors.md)) deliberately omit iterators
```

---

## Iterator Categories

### Category Hierarchy

Input and output iterators are **separate** single-pass categories. Forward iterators combine multi-pass read/write; each higher category adds operations:

```
                    Input Iterator        Output Iterator
                    (single-pass read)    (single-pass write)
                           \               /
                            \             /
                            Forward Iterator
                         (multi-pass read/write)
                                  │
                         Bidirectional Iterator
                             (--it, it--)
                                  │
                        Random Access Iterator
                        (it+n, it[n], it-it, <)
                                  │
                       Contiguous Iterator (C++20)
                      (elements in contiguous storage;
                       std::to_address(it) is valid)

Higher categories include all capabilities of lower ones
(except input/output, which are not in a single line).
```

### 1. Input Iterator
```
Capabilities:
- Read from pointed element (once)
- Move forward (++it)
- Single-pass only

Operations:
- *it       (read)
- ++it, it++
- it1 == it2, it1 != it2

Example: std::istream_iterator
```

```cpp
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    // Reading from input stream
    std::istream_iterator<int> input_it(std::cin);
    std::istream_iterator<int> eof;  // End-of-stream iterator
    
    std::vector<int> numbers;
    while (input_it != eof) {
        numbers.push_back(*input_it);
        ++input_it;
    }
    
    // Or more concisely:
    std::vector<int> nums(
        std::istream_iterator<int>(std::cin),
        std::istream_iterator<int>()
    );
    
    return 0;
}
```

### 2. Output Iterator
```
Capabilities:
- Write to pointed element
- Move forward (++it)
- Single-pass only

Operations:
- *it = value (write)
- ++it, it++

Example: std::ostream_iterator, std::back_inserter
```

```cpp
#include <iostream>
#include <iterator>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    // Output to stream
    std::ostream_iterator<int> output_it(std::cout, " ");
    std::copy(numbers.begin(), numbers.end(), output_it);
    // Output: 1 2 3 4 5
    
    // Insert to container
    std::vector<int> dest;
    std::copy(numbers.begin(), numbers.end(), 
              std::back_inserter(dest));
    
    return 0;
}
```

### 3. Forward Iterator
```
Capabilities:
- Read and write
- Move forward multiple times (multi-pass)
- Can save and reuse position

Operations:
- All Input/Output operations
- Can iterate multiple times

Example: std::forward_list::iterator
```

```cpp
#include <forward_list>

int main() {
    std::forward_list<int> flist = {1, 2, 3, 4, 5};
    
    auto it1 = flist.begin();
    auto it2 = it1;  // Copy iterator
    
    ++it1;  // Move first iterator
    // it1 and it2 now point to different elements
    // Can iterate again from it2
    
    return 0;
}
```

### 4. Bidirectional Iterator
```
Capabilities:
- All Forward Iterator operations
- Move backward (--it)

Operations:
- All Forward operations
- --it, it--

Example: std::list::iterator, std::set::iterator
```

```cpp
#include <list>
#include <set>

int main() {
    std::list<int> lst = {1, 2, 3, 4, 5};
    
    auto it = lst.end();
    --it;  // Points to 5
    --it;  // Points to 4
    ++it;  // Points to 5 again
    
    // Works with sets too
    std::set<int> s = {10, 20, 30};
    auto sit = s.end();
    --sit;  // Points to 30
    
    return 0;
}
```

### 5. Random Access Iterator
```
Capabilities:
- All Bidirectional operations
- Jump to any position (it + n)
- Random access (it[n])
- Distance calculation (it2 - it1)
- Comparison (<, >, <=, >=)

Operations:
- All Bidirectional operations
- it + n, it - n
- it += n, it -= n
- it1 - it2 (distance)
- it[n]
- it1 < it2, it1 > it2, etc.

Example: std::vector::iterator, std::deque::iterator
```

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    auto it = vec.begin();
    
    // Jump operations
    it += 5;           // Move forward 5 positions
    int val = it[2];   // Access element at offset 2
    it = it + 3;       // Jump forward 3
    it = it - 2;       // Jump backward 2
    
    // Distance
    auto distance = vec.end() - vec.begin();  // 10
    
    // Comparison
    auto it1 = vec.begin();
    auto it2 = vec.begin() + 5;
    bool less = it1 < it2;  // true
    
    // Random access enables sorting
    std::sort(vec.begin(), vec.end());
    
    return 0;
}
```

### 6. Contiguous Iterator (C++20)
```
Capabilities:
- All Random Access operations
- Elements guaranteed to be in contiguous memory
- Can safely convert to pointer

Operations:
- All Random Access operations
- std::to_address(it) → pointer

Example: std::vector::iterator, std::array::iterator, pointers
```

```cpp
#include <vector>
#include <span>
#include <memory>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    auto it = vec.begin();
    
    // Can safely get pointer (C++20)
    int* ptr = std::to_address(it);
    
    // Elements are contiguous
    ptr[0];  // Same as vec[0]
    ptr[1];  // Same as vec[1]
    
    // Enables std::span (C++20)
    std::span<int> s(vec.begin(), vec.end());
    
    return 0;
}
```

### Visual Comparison of Iterator Categories
```
Container / Iterator   | Category              | Key ops
-----------------------|-----------------------|---------------------------
istream_iterator       | Input                 | → read once
ostream_iterator       | Output                | → write once
forward_list           | Forward               | → multi-pass
unordered_set/map      | Forward (const)       | → (no --)
list, set, map         | Bidirectional         | ← →
deque                  | Random Access         | ←5→ [n]  (not contiguous*)
vector, array, string  | Contiguous (C++20)    | ←5→ [n] + contiguous ptr
stack/queue/pq         | (none)                | adaptors have no iterators

* deque iterators are random-access but elements are not contiguous in memory.
  See [Sequence Containers](01_sequence_containers.md).
```

---

## Iterator Operations

### Basic Iterator Operations

```cpp
#include <vector>
#include <iostream>
#include <iterator>

int main() {
    std::vector<int> vec = {10, 20, 30, 40, 50};
    
    // Get iterators
    auto begin = vec.begin();        // First element
    auto end = vec.end();            // Past-the-end
    auto rbegin = vec.rbegin();      // Reverse: last element
    auto rend = vec.rend();          // Reverse: before-first
    
    // Const iterators (cannot modify elements)
    auto cbegin = vec.cbegin();
    auto cend = vec.cend();
    auto crbegin = vec.crbegin();
    auto crend = vec.crend();
    
    // Dereference
    int value = *begin;              // 10
    
    // Arrow operator (for objects)
    struct Point { int x, y; };
    std::vector<Point> points = { {1, 2}, {3, 4} };
    int x = points.begin()->x;       // 1
    
    // Increment/Decrement
    ++begin;                         // Points to 20
    begin++;                         // Points to 30
    --begin;                         // Points to 20
    
    // Arithmetic (random access only)
    auto it = vec.begin() + 3;       // Points to 40
    it -= 2;                         // Points to 20
    
    // Subscript (random access only)
    int val = begin[2];              // Element at begin + 2
    
    // Distance
    auto dist = std::distance(vec.begin(), vec.end());  // 5
    
    // Advance
    auto iter = vec.begin();
    std::advance(iter, 3);           // Move forward 3 positions
    
    return 0;
}
```

### Visual Representation of begin/end
```
Vector: [10, 20, 30, 40, 50]

Forward iteration:
         begin()                  end()
           ↓                        ↓
         ┌────┬────┬────┬────┬────┬──┐
         │ 10 │ 20 │ 30 │ 40 │ 50 │  │
         └────┴────┴────┴────┴────┴──┘
           →    →    →    →    →
         
Reverse iteration:
         rend()                  rbegin()
           ↓                        ↓
         ┌──┬────┬────┬────┬────┬────┐
         │  │ 10 │ 20 │ 30 │ 40 │ 50 │
         └──┴────┴────┴────┴────┴────┘
              ←    ←    ←    ←    ←

Note: end() and rend() point PAST the actual element!
```

### Const Iterators
```cpp
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3};
    
    // Non-const iterator (can modify)
    auto it = vec.begin();
    *it = 10;  // OK
    
    // Const iterator (cannot modify)
    auto cit = vec.cbegin();
    // *cit = 10;  // ERROR: cannot modify
    int val = *cit;  // OK: can read
    
    // Const vector
    const std::vector<int> cvec = {1, 2, 3};
    auto it2 = cvec.begin();  // Returns const_iterator automatically
    // *it2 = 10;  // ERROR
    
    return 0;
}
```

### Reverse Iterators
```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Forward iteration
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        std::cout << *it << ' ';
    }
    std::cout << '\n';  // Output: 1 2 3 4 5
    
    // Reverse iteration
    for (auto rit = vec.rbegin(); rit != vec.rend(); ++rit) {
        std::cout << *rit << ' ';
    }
    std::cout << '\n';  // Output: 5 4 3 2 1
    
    // Convert reverse_iterator to iterator
    auto rit = vec.rbegin();
    ++rit;  // Points to 4
    auto it = rit.base();  // Convert to forward iterator
    // Note: base() points to element AFTER reverse iterator position!
    
    return 0;
}
```

### Reverse Iterator Quirk
```
Vector: [1, 2, 3, 4, 5]

Forward:  begin()        end()
            ↓              ↓
           [1][2][3][4][5][ ]
            ↑           ↑
         *begin()    *(end()-1)

Reverse:  rend()      rbegin()
            ↓              ↓
           [ ][1][2][3][4][5]
                           ↑
                     *rbegin()

Conversion:
rbegin().base() == end()
rend().base() == begin()

rit = rbegin() + 2;  // Points to 3
*rit == 3
rit.base() points to 4 (one past rit)!
```

---

## Iterator Adaptors

### 1. Insert Iterators

Insert iterators are **output iterators** that call container `insert`/`push_back`/`push_front` instead of writing through a fixed slot — the only way algorithms can grow a container. Requires `#include <iterator>`.

```cpp
#include <deque>
#include <vector>
#include <iterator>
#include <algorithm>

int main() {
    std::vector<int> src = {1, 2, 3};
    std::vector<int> dest;
    
    // back_inserter - calls push_back()
    std::copy(src.begin(), src.end(), 
              std::back_inserter(dest));
    // dest = {1, 2, 3}
    
    // front_inserter - calls push_front() (requires deque/list)
    std::deque<int> dq;
    std::copy(src.begin(), src.end(), 
              std::front_inserter(dq));
    // dq = {3, 2, 1} (reversed!)
    
    // inserter - calls insert() at position
    std::vector<int> vec = {1, 5, 9};
    auto it = vec.begin() + 1;
    std::copy(src.begin(), src.end(), 
              std::inserter(vec, it));
    // vec = {1, 1, 2, 3, 5, 9}
    
    return 0;
}
```

### Insert Iterator Visualization
```
back_inserter:
src:  [1][2][3]
dest: [ ][ ][ ]
       ↓  ↓  ↓
      [1][2][3]

front_inserter:
src:  [1][2][3]
dest: [ ][ ][ ]
       ↓  ↓  ↓
      [3][2][1]  (reversed!)

inserter (at position 1):
src:  [1][2][3]
vec:  [1][5][9]
         ↓ ↓ ↓
      [1][1][2][3][5][9]
```

### 2. Stream Iterators
```cpp
#include <iterator>
#include <iostream>
#include <sstream>
#include <vector>
#include <algorithm>

int main() {
    // Input stream iterator
    std::istringstream iss("1 2 3 4 5");
    std::vector<int> numbers(
        std::istream_iterator<int>(iss),
        std::istream_iterator<int>()  // EOF
    );
    
    // Output stream iterator
    std::copy(numbers.begin(), numbers.end(),
              std::ostream_iterator<int>(std::cout, ", "));
    // Output: 1, 2, 3, 4, 5,
    
    return 0;
}
```

### 3. Move Iterator (C++11)
```cpp
#include <vector>
#include <algorithm>
#include <iterator>
#include <string>

int main() {
    std::vector<std::string> src = {"hello", "world", "foo", "bar"};
    std::vector<std::string> dest;
    
    // Copy (copies strings)
    std::copy(src.begin(), src.end(), 
              std::back_inserter(dest));
    
    // Move (moves strings, src becomes empty strings)
    std::vector<std::string> src2 = {"hello", "world"};
    std::vector<std::string> dest2;
    std::copy(std::make_move_iterator(src2.begin()),
              std::make_move_iterator(src2.end()),
              std::back_inserter(dest2));
    // dest2 = {"hello", "world"}
    // src2 = {"", ""} (moved from)
    
    return 0;
}
```

---

## Iterator Traits

### What are Iterator Traits?
Iterator traits provide type information about iterators, enabling algorithms to optimize based on iterator capabilities.

```cpp
#include <iterator>
#include <vector>
#include <list>

template<typename Iterator>
void analyze_iterator() {
    using traits = std::iterator_traits<Iterator>;
    
    // Type aliases provided:
    typename traits::iterator_category;  // Input/Output/Forward/etc.
    typename traits::value_type;         // Element type
    typename traits::difference_type;    // Type for iterator difference
    typename traits::pointer;            // Pointer to element
    typename traits::reference;          // Reference to element
}

int main() {
    using VecIter = std::vector<int>::iterator;
    using ListIter = std::list<int>::iterator;
    
    // Check iterator category
    constexpr bool is_random_access = 
        std::is_same_v<
            std::iterator_traits<VecIter>::iterator_category,
            std::random_access_iterator_tag
        >;  // true
    
    constexpr bool list_is_random = 
        std::is_same_v<
            std::iterator_traits<ListIter>::iterator_category,
            std::random_access_iterator_tag
        >;  // false (bidirectional)
    
    return 0;
}
```

### Using Traits for Optimization
```cpp
#include <iterator>
#include <type_traits>

// Advance implementation - optimized based on iterator category

// For random access iterators - O(1)
template<typename RAIter>
void advance_impl(RAIter& it, int n, 
                  std::random_access_iterator_tag) {
    it += n;  // O(1)
}

// For bidirectional iterators - O(n)
template<typename BiIter>
void advance_impl(BiIter& it, int n,
                  std::bidirectional_iterator_tag) {
    if (n >= 0) {
        while (n--) ++it;
    } else {
        while (n++) --it;
    }
}

// For forward iterators - O(n), forward only
template<typename FwdIter>
void advance_impl(FwdIter& it, int n,
                  std::forward_iterator_tag) {
    while (n--) ++it;
}

// Main advance function
template<typename Iterator>
void my_advance(Iterator& it, int n) {
    advance_impl(it, n, 
        typename std::iterator_traits<Iterator>::iterator_category{});
}
```

---

## C++20 Iterator Concepts

### New Iterator Concepts
```cpp
#include <iterator>
#include <vector>
#include <list>
#include <concepts>

// C++20 provides concepts for iterators (see also 07_modern_features.md)
template<typename I>
concept my_input_iterator = std::input_iterator<I>;

template<typename I>
concept my_forward_iterator = std::forward_iterator<I>;

template<typename I>
concept my_bidirectional_iterator = std::bidirectional_iterator<I>;

template<typename I>
concept my_random_access_iterator = std::random_access_iterator<I>;

template<typename I>
concept my_contiguous_iterator = std::contiguous_iterator<I>;

// Use in function templates
template<std::random_access_iterator Iter>
void efficient_sort(Iter first, Iter last) {
    // Can use random access operations
}

// Compile-time check
int main() {
    std::vector<int> vec;
    static_assert(std::contiguous_iterator<decltype(vec.begin())>);
    
    std::list<int> lst;
    static_assert(std::bidirectional_iterator<decltype(lst.begin())>);
    static_assert(!std::random_access_iterator<decltype(lst.begin())>);
    
    return 0;
}
```

### Iterator Concepts Hierarchy (C++20)
```
┌─────────────────────────────────────────────┐
│  C++20 Iterator Concept Requirements        │
├─────────────────────────────────────────────┤
│                                             │
│  input_iterator:                            │
│    - Can read (*it)                         │
│    - Can increment (++it)                   │
│                                             │
│  output_iterator:                           │
│    - Can write (*it = val)                  │
│    - Can increment (++it)                   │
│                                             │
│  forward_iterator:                          │
│    - input_iterator                         │
│    - Multi-pass guarantee                   │
│    - Default constructible                  │
│                                             │
│  bidirectional_iterator:                    │
│    - forward_iterator                       │
│    - Can decrement (--it)                   │
│                                             │
│  random_access_iterator:                    │
│    - bidirectional_iterator                 │
│    - it + n, it - n, it[n]                  │
│    - it1 - it2                              │
│    - it1 < it2                              │
│                                             │
│  contiguous_iterator:                       │
│    - random_access_iterator                 │
│    - Elements in contiguous memory          │
│    - std::to_address(it) valid              │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Custom Iterators

### Creating a Simple Iterator
```cpp
#include <iterator>

template<typename T>
class SimpleVector {
    T* data_;
    size_t size_;
    
public:
    // Iterator class
    class Iterator {
        T* ptr_;
        
    public:
        // Iterator traits (required)
        using iterator_category = std::random_access_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        
        // Constructor
        Iterator(T* ptr) : ptr_(ptr) {}
        
        // Dereference
        reference operator*() const { return *ptr_; }
        pointer operator->() const { return ptr_; }
        
        // Increment/Decrement
        Iterator& operator++() { ++ptr_; return *this; }
        Iterator operator++(int) { Iterator tmp = *this; ++ptr_; return tmp; }
        Iterator& operator--() { --ptr_; return *this; }
        Iterator operator--(int) { Iterator tmp = *this; --ptr_; return tmp; }
        
        // Arithmetic
        Iterator operator+(difference_type n) const { return Iterator(ptr_ + n); }
        Iterator operator-(difference_type n) const { return Iterator(ptr_ - n); }
        Iterator& operator+=(difference_type n) { ptr_ += n; return *this; }
        Iterator& operator-=(difference_type n) { ptr_ -= n; return *this; }
        
        difference_type operator-(const Iterator& other) const {
            return ptr_ - other.ptr_;
        }
        
        // Subscript
        reference operator[](difference_type n) const { return ptr_[n]; }
        
        // Comparison
        bool operator==(const Iterator& other) const { return ptr_ == other.ptr_; }
        bool operator!=(const Iterator& other) const { return ptr_ != other.ptr_; }
        bool operator<(const Iterator& other) const { return ptr_ < other.ptr_; }
        bool operator>(const Iterator& other) const { return ptr_ > other.ptr_; }
        bool operator<=(const Iterator& other) const { return ptr_ <= other.ptr_; }
        bool operator>=(const Iterator& other) const { return ptr_ >= other.ptr_; }
    };
    
    // Container methods
    Iterator begin() { return Iterator(data_); }
    Iterator end() { return Iterator(data_ + size_); }
};
```

### Range-based for Loop Requirements
```cpp
#include <iostream>

class Range {
    int start_, end_;
    
public:
    Range(int start, int end) : start_(start), end_(end) {}
    
    class Iterator {
        int current_;
    public:
        Iterator(int val) : current_(val) {}
        
        int operator*() const { return current_; }
        Iterator& operator++() { ++current_; return *this; }
        bool operator!=(const Iterator& other) const {
            return current_ != other.current_;
        }
    };
    
    Iterator begin() const { return Iterator(start_); }
    Iterator end() const { return Iterator(end_); }
};

int main() {
    // Use custom range in range-based for loop
    for (int i : Range(1, 6)) {
        std::cout << i << ' ';  // Output: 1 2 3 4 5
    }
    
    return 0;
}
```

---

## Iterator Utilities

### std::distance
```cpp
#include <iterator>
#include <vector>
#include <list>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Distance between iterators - O(1) for random access
    auto dist = std::distance(vec.begin(), vec.end());  // 5
    
    // For non-random access iterators - O(n)
    std::list<int> lst = {1, 2, 3, 4, 5};
    auto ldist = std::distance(lst.begin(), lst.end());  // 5 (counted)
    
    return 0;
}
```

### std::advance
```cpp
#include <iterator>
#include <vector>
#include <list>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto it = vec.begin();
    
    std::advance(it, 3);  // O(1) for random access
    // it points to 4
    
    std::list<int> lst = {1, 2, 3, 4, 5};
    auto lit = lst.begin();
    std::advance(lit, 3);  // O(n) for bidirectional
    
    return 0;
}
```

### std::next and std::prev (C++11)
```cpp
#include <iterator>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto it = vec.begin();
    
    // Get iterator n positions ahead (doesn't modify it)
    auto next_it = std::next(it, 3);  // Points to 4
    // it still points to 1
    
    // Get iterator n positions back
    auto prev_it = std::prev(vec.end(), 2);  // Points to 4
    
    // Equivalent to:
    auto it2 = vec.begin();
    std::advance(it2, 3);  // Modifies it2
    
    return 0;
}
```

### std::iter_swap
```cpp
#include <iterator>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    auto it1 = vec.begin();
    auto it2 = vec.begin() + 4;
    
    std::iter_swap(it1, it2);
    // vec = {5, 2, 3, 4, 1}
    
    return 0;
}
```

---

## Common Patterns and Idioms

### Pattern 1: Erase-Remove Idiom

`std::remove` / `std::remove_if` **do not erase** — they compact elements and return the new logical end. You must call `erase` on sequence containers. Full treatment in [Algorithms](06_algorithms.md).

```cpp
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> vec = {1, 2, 3, 2, 4, 2, 5};
    
    // C++20: std::erase / std::erase_if (no manual erase-remove)
    // std::erase(vec, 2);

    auto new_end = std::remove(vec.begin(), vec.end(), 2);
    vec.erase(new_end, vec.end());  // vec = {1, 3, 4, 5}
    
    return 0;
}
```

### Pattern 2: Finding with Iterators
```cpp
#include <vector>
#include <algorithm>
#include <iostream>
#include <iterator>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    // Find element
    auto it = std::find(vec.begin(), vec.end(), 3);
    if (it != vec.end()) {
        std::cout << "Found: " << *it << '\n';
        std::cout << "At position: " << std::distance(vec.begin(), it) << '\n';
    }
    
    // Find with predicate
    auto it2 = std::find_if(vec.begin(), vec.end(), 
                           [](int x) { return x > 3; });
    // it2 points to 4
    
    return 0;
}
```

### Pattern 3: Transform with Iterators
```cpp
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dest(5);
    
    // Transform: apply function to each element
    std::transform(src.begin(), src.end(), dest.begin(),
                   [](int x) { return x * 2; });
    // dest = {2, 4, 6, 8, 10}
    
    // In-place transform
    std::transform(src.begin(), src.end(), src.begin(),
                   [](int x) { return x + 1; });
    // src = {2, 3, 4, 5, 6}
    
    return 0;
}
```

### Pattern 4: Partitioning
```cpp
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    
    // Partition: even numbers first, odd numbers second
    auto partition_point = std::partition(vec.begin(), vec.end(),
                                         [](int x) { return x % 2 == 0; });
    
    // vec = {8, 2, 6, 4, 5, 3, 7, 1, 9}
    //                       ^
    //               partition_point
    
    // All even numbers are before partition_point
    // All odd numbers are after partition_point
    
    return 0;
}
```

---

## Common Pitfalls

### 1. Iterator Invalidation
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
auto it = vec.begin();

vec.push_back(6);  // May reallocate!
// it is now INVALID (dangling)

// CORRECT: Get iterator after modification
vec.push_back(6);
it = vec.begin();
```

### 2. End Iterator
```cpp
std::vector<int> vec = {1, 2, 3};
auto it = vec.end();

// BAD: end() doesn't point to valid element!
// int x = *it;  // Undefined behavior!

// GOOD: Check before dereferencing
if (it != vec.end()) {
    int x = *it;
}
```

### 3. Modifying While Iterating
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// BAD: Iterator invalidated by erase
for (auto it = vec.begin(); it != vec.end(); ++it) {
    if (*it % 2 == 0) {
        vec.erase(it);  // Invalidates it!
    }
}

// GOOD: Use erase return value
for (auto it = vec.begin(); it != vec.end(); ) {
    if (*it % 2 == 0) {
        it = vec.erase(it);  // Get new valid iterator
    } else {
        ++it;
    }
}
```

### 4. Comparing Iterators from Different Containers

Iterators from different containers (or even different elements of the same `deque`) may compare only if they refer to the same sequence. Comparing `vec1.begin()` to `vec2.begin()` is undefined behavior.

```cpp
#include <vector>

std::vector<int> vec1 = {1, 2, 3};
std::vector<int> vec2 = {4, 5, 6};

// BAD: undefined behavior
// if (vec1.begin() == vec2.begin()) { }

// OK: same container, valid range
if (vec1.begin() == vec1.end()) { /* empty */ }
```

---

## Practice Exercises

### Exercise 1: Implement find_if
```cpp
// Implement your own find_if using iterators
template<typename Iterator, typename Predicate>
Iterator my_find_if(Iterator first, Iterator last, Predicate pred) {
    // Your implementation
}
```

### Exercise 2: Custom Iterator
```cpp
// Create an iterator for a simple linked list
template<typename T>
struct Node {
    T data;
    Node* next;
};

// Implement Iterator class with proper traits
```

### Exercise 3: Iterator Adapter
```cpp
// Create a "filter iterator" that skips elements not matching a predicate
template<typename Iterator, typename Predicate>
class FilterIterator {
    // Your implementation
};
```

---

## Complete Practical Example: Custom Data Structure with Iterators

Here's a comprehensive example showing custom iterators, iterator adapters, and all iterator categories:

```cpp
#include <iostream>
#include <iterator>
#include <vector>
#include <algorithm>
#include <memory>
#include <numeric>   // std::accumulate

// 1. Custom linked list with bidirectional iterator
template<typename T>
class LinkedList {
private:
    struct Node {
        T data;
        Node* next;
        Node* prev;
        
        Node(const T& d) : data(d), next(nullptr), prev(nullptr) {}
    };
    
    Node* head_;
    Node* tail_;
    size_t size_;
    
public:
    LinkedList() : head_(nullptr), tail_(nullptr), size_(0) {}
    
    ~LinkedList() {
        clear();
    }
    
    void clear() {
        while (head_) {
            Node* temp = head_;
            head_ = head_->next;
            delete temp;
        }
        tail_ = nullptr;
        size_ = 0;
    }
    
    void push_back(const T& value) {
        Node* new_node = new Node(value);
        
        if (!tail_) {
            head_ = tail_ = new_node;
        } else {
            tail_->next = new_node;
            new_node->prev = tail_;
            tail_ = new_node;
        }
        size_++;
    }
    
    size_t size() const { return size_; }
    
    // Bidirectional Iterator
    class Iterator {
    private:
        Node* current_;
        
    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;
        
        Iterator(Node* node) : current_(node) {}
        
        reference operator*() const { return current_->data; }
        pointer operator->() const { return &current_->data; }
        
        // Pre-increment
        Iterator& operator++() {
            if (current_) current_ = current_->next;
            return *this;
        }
        
        // Post-increment
        Iterator operator++(int) {
            Iterator tmp = *this;
            ++(*this);
            return tmp;
        }
        
        // Pre-decrement
        Iterator& operator--() {
            if (current_) current_ = current_->prev;
            return *this;
        }
        
        // Post-decrement
        Iterator operator--(int) {
            Iterator tmp = *this;
            --(*this);
            return tmp;
        }
        
        bool operator==(const Iterator& other) const {
            return current_ == other.current_;
        }
        
        bool operator!=(const Iterator& other) const {
            return current_ != other.current_;
        }
        
        friend class LinkedList;
    };
    
    Iterator begin() { return Iterator(head_); }
    Iterator end() { return Iterator(nullptr); }
    
    const Iterator begin() const { return Iterator(head_); }
    const Iterator end() const { return Iterator(nullptr); }
};

// 2. Range generator with input iterator
class RangeIterator {
private:
    int current_;
    
public:
    using iterator_category = std::input_iterator_tag;
    using value_type = int;
    using difference_type = std::ptrdiff_t;
    using pointer = int*;
    using reference = int&;
    
    RangeIterator(int start) : current_(start) {}
    
    int operator*() const { return current_; }
    
    RangeIterator& operator++() {
        ++current_;
        return *this;
    }
    
    RangeIterator operator++(int) {
        RangeIterator tmp = *this;
        ++current_;
        return tmp;
    }
    
    bool operator==(const RangeIterator& other) const {
        return current_ == other.current_;
    }
    
    bool operator!=(const RangeIterator& other) const {
        return current_ != other.current_;
    }
};

class Range {
    int start_, end_;
    
public:
    Range(int start, int end) : start_(start), end_(end) {}
    
    RangeIterator begin() const { return RangeIterator(start_); }
    RangeIterator end() const { return RangeIterator(end_); }
};

// 3. Filter iterator adapter
template<typename Iterator, typename Predicate>
class FilterIterator {
private:
    Iterator current_;
    Iterator end_;
    Predicate pred_;
    
    void advance_to_next_valid() {
        while (current_ != end_ && !pred_(*current_)) {
            ++current_;
        }
    }
    
public:
    using iterator_category = std::forward_iterator_tag;
    using value_type = typename std::iterator_traits<Iterator>::value_type;
    using difference_type = typename std::iterator_traits<Iterator>::difference_type;
    using pointer = typename std::iterator_traits<Iterator>::pointer;
    using reference = typename std::iterator_traits<Iterator>::reference;
    
    FilterIterator(Iterator current, Iterator end, Predicate pred)
        : current_(current), end_(end), pred_(pred) {
        advance_to_next_valid();
    }
    
    reference operator*() const { return *current_; }
    pointer operator->() const { return &(*current_); }
    
    FilterIterator& operator++() {
        if (current_ != end_) {
            ++current_;
            advance_to_next_valid();
        }
        return *this;
    }
    
    FilterIterator operator++(int) {
        FilterIterator tmp = *this;
        ++(*this);
        return tmp;
    }
    
    bool operator==(const FilterIterator& other) const {
        return current_ == other.current_;
    }
    
    bool operator!=(const FilterIterator& other) const {
        return current_ != other.current_;
    }
};

template<typename Container, typename Predicate>
class FilterView {
    Container& container_;
    Predicate pred_;
    
public:
    FilterView(Container& c, Predicate p) : container_(c), pred_(p) {}
    
    auto begin() {
        return FilterIterator(container_.begin(), container_.end(), pred_);
    }
    
    auto end() {
        return FilterIterator(container_.end(), container_.end(), pred_);
    }
};

// 4. Transform iterator adapter
template<typename Iterator, typename Func>
class TransformIterator {
private:
    Iterator current_;
    Func func_;
    
public:
    using iterator_category = std::forward_iterator_tag;
    using value_type = decltype(func_(*current_));
    using difference_type = typename std::iterator_traits<Iterator>::difference_type;
    using pointer = value_type*;
    using reference = value_type;
    
    TransformIterator(Iterator it, Func f) : current_(it), func_(f) {}
    
    value_type operator*() const { return func_(*current_); }
    
    TransformIterator& operator++() {
        ++current_;
        return *this;
    }
    
    TransformIterator operator++(int) {
        TransformIterator tmp = *this;
        ++current_;
        return tmp;
    }
    
    bool operator==(const TransformIterator& other) const {
        return current_ == other.current_;
    }
    
    bool operator!=(const TransformIterator& other) const {
        return current_ != other.current_;
    }
};

// Demonstrations
void demo_custom_linked_list() {
    std::cout << "\n--- Custom Linked List ---\n";
    
    LinkedList<int> list;
    list.push_back(10);
    list.push_back(20);
    list.push_back(30);
    list.push_back(40);
    
    // Forward iteration
    std::cout << "Forward: ";
    for (auto it = list.begin(); it != list.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    
    // Use with STL algorithms
    auto it = std::find(list.begin(), list.end(), 30);
    if (it != list.end()) {
        std::cout << "Found: " << *it << "\n";
    }
    
    // Count elements
    int count = std::count_if(list.begin(), list.end(),
        [](int x) { return x > 20; });
    std::cout << "Count > 20: " << count << "\n";
}

void demo_range_generator() {
    std::cout << "\n--- Range Generator ---\n";
    
    Range range(1, 11);
    
    std::cout << "Range [1, 11): ";
    for (int value : range) {
        std::cout << value << " ";
    }
    std::cout << "\n";
    
    // Use with STL algorithms
    auto sum = std::accumulate(range.begin(), range.end(), 0);
    std::cout << "Sum: " << sum << "\n";
}

void demo_filter_iterator() {
    std::cout << "\n--- Filter Iterator ---\n";
    
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Filter even numbers
    FilterView evens_view(numbers, [](int x) { return x % 2 == 0; });
    
    std::cout << "Even numbers: ";
    for (int value : evens_view) {
        std::cout << value << " ";
    }
    std::cout << "\n";
    
    // Filter numbers > 5
    FilterView large_view(numbers, [](int x) { return x > 5; });
    
    std::cout << "Numbers > 5: ";
    for (int value : large_view) {
        std::cout << value << " ";
    }
    std::cout << "\n";
}

void demo_transform_iterator() {
    std::cout << "\n--- Transform Iterator ---\n";
    
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    auto square = [](int x) { return x * x; };
    
    std::cout << "Squared: ";
    for (auto it = TransformIterator(numbers.begin(), square);
         it != TransformIterator(numbers.end(), square);
         ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
}

void demo_iterator_adapters() {
    std::cout << "\n--- Iterator Adapters ---\n";
    
    std::vector<int> source = {1, 2, 3, 4, 5};
    std::vector<int> dest;
    
    // back_inserter
    std::cout << "Using back_inserter:\n";
    std::copy(source.begin(), source.end(), std::back_inserter(dest));
    std::cout << "  Dest size: " << dest.size() << "\n";
    
    // reverse_iterator
    std::cout << "Reverse iteration: ";
    for (auto it = source.rbegin(); it != source.rend(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    
    // move_iterator
    std::vector<std::string> strings = {"hello", "world", "test"};
    std::vector<std::string> moved;
    
    std::copy(std::make_move_iterator(strings.begin()),
              std::make_move_iterator(strings.end()),
              std::back_inserter(moved));
    
    std::cout << "After move: strings[0] = '" << strings[0] << "' (moved from)\n";
    std::cout << "            moved[0] = '" << moved[0] << "'\n";
}

int main() {
    std::cout << "=== Iterators Demo ===\n";
    
    // 1. Custom linked list with bidirectional iterator
    demo_custom_linked_list();
    
    // 2. Range generator with input iterator
    demo_range_generator();
    
    // 3. Filter iterator adapter
    demo_filter_iterator();
    
    // 4. Transform iterator adapter
    demo_transform_iterator();
    
    // 5. Standard iterator adapters
    demo_iterator_adapters();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **Custom iterators**: Implementing all required operations
- **Iterator traits**: Defining iterator_category, value_type, etc.
- **Bidirectional iterator**: Forward and backward traversal
- **Input iterator**: Single-pass forward iteration (Range generator)
- **Iterator adapters**: Filter and transform views
- **back_inserter**: Inserting while iterating
- **reverse_iterator**: Backward iteration
- **move_iterator**: Moving elements
- **std::distance**: Iterator arithmetic
- **std::advance**: Moving iterators
- **Iterator invalidation**: Proper handling
- **STL algorithm integration**: Using custom iterators

This example shows how to create reusable, STL-compatible iterators!

---

## Related Topics

- [Algorithms](06_algorithms.md) — the primary consumers of `[first, last)` iterator ranges
- [Sequence Containers](01_sequence_containers.md) — `vector`/`deque`/`list` iterator categories and invalidation rules
- [Container Adaptors](04_container_adaptors.md) — `stack`, `queue`, and `priority_queue` intentionally provide no iterators
- [Modern C++20/23 Features](07_modern_features.md) — `std::ranges`, views, and iterator concepts in the ranges library
- [Move Semantics & RAII](12_advanced_features.md) — `std::make_move_iterator` and efficient transfers
- [Templates](09_templates.md) — writing iterator-based algorithm templates with traits/concepts

## Next Steps
- **Next**: [Algorithms →](06_algorithms.md)
- **Previous**: [← Container Adaptors](04_container_adaptors.md)

---
*Chapter 5 — Iterators*

