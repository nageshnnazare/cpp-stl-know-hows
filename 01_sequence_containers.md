# Sequence Containers

## Overview

Sequence containers store elements in a linear sequence. Each element has a specific position that depends on the time and place of insertion, not on the element's value.

```
┌────────────────────────────────────────────────────────────┐
│              SEQUENCE CONTAINERS FAMILY                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  std::vector      [■][■][■][■]                             │
│                   Dynamic array, random access             │
│                                                            │
│  std::deque       [■][■][■][■][■][■]                       │
│                   Double-ended queue                       │
│                                                            │
│  std::list        [■]↔[■]↔[■]↔[■]                          │
│                   Doubly-linked list                       │
│                                                            │
│  std::forward_list [■]→[■]→[■]→[■]                         │
│                   Singly-linked list                       │
│                                                            │
│  std::array       [■][■][■][■]                             │
│                   Fixed-size array                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 1. std::vector

### Description
A dynamic array that can grow and shrink automatically. It's the most commonly used STL container.

### Memory Layout
```
Vector Structure:
┌──────────────┐
│  Vector Obj  │
├──────────────┤
│ begin*       │───┐
│ end*         │───┼─┐
│ capacity*    │───┼─┼─┐
└──────────────┘   │ │ │
                   │ │ │
    Heap Memory:   │ │ │
    ┌──────────────▼─▼─▼─────────────┐
    │ [0] [1] [2] [3] [4] [?] [?] [?]│
    └────────────────────────────────┘
    │<─ size ─>│<─ unused capacity ─>│
    │<────── total capacity  ───────>│

* = pointer to heap memory
```

### Key Characteristics
- **Access**: O(1) random access
- **Insertion/Deletion at end**: O(1) amortized
- **Insertion/Deletion at middle**: O(n)
- **Memory**: Contiguous storage

### Basic Operations

```cpp
#include <vector>
#include <iostream>

int main() {
    // Construction
    std::vector<int> v1;                    // Empty vector
    std::vector<int> v2(5);                 // 5 elements (default: 0)
    std::vector<int> v3(5, 10);             // 5 elements, all 10
    std::vector<int> v4 = {1, 2, 3, 4, 5}; // Initializer list
    std::vector<int> v5(v4);                // Copy constructor
    std::vector<int> v6(std::move(v4));     // Move constructor
    
    // Adding elements
    std::vector<int> vec;
    vec.push_back(1);           // Add to end: [1]
    vec.push_back(2);           // [1, 2]
    vec.emplace_back(3);        // Construct in-place: [1, 2, 3]
    
    // Accessing elements
    int first = vec[0];         // No bounds checking
    int second = vec.at(1);     // With bounds checking (throws exception)
    int& front = vec.front();   // First element
    int& back = vec.back();     // Last element
    int* data = vec.data();     // Pointer to underlying array
    
    // Size information
    size_t size = vec.size();           // Number of elements
    size_t capacity = vec.capacity();   // Allocated capacity
    bool empty = vec.empty();           // Check if empty
    
    // Capacity management
    vec.reserve(100);           // Reserve space for 100 elements
    vec.shrink_to_fit();        // Reduce capacity to fit size
    vec.resize(10);             // Resize to 10 elements
    vec.resize(15, 42);         // Resize to 15, new elements = 42
    
    // Insertion — NOTE: each insert may reallocate and ALWAYS invalidates
    // iterators at/after the insertion point, so re-fetch begin() each time.
    vec.insert(vec.begin(), 99);             // Insert 99 at beginning
    vec.insert(vec.begin() + 2, 3, 88);      // Insert 3×88 at position 2
    vec.insert(vec.begin(), {7, 8, 9});      // Insert multiple elements
    
    // Deletion
    vec.pop_back();                     // Remove last element
    vec.erase(vec.begin());             // Remove first element
    vec.erase(vec.begin(), vec.begin() + 3); // Remove range
    vec.clear();                        // Remove all elements
    
    return 0;
}
```

### Vector Growth Strategy

When `push_back` exceeds the current capacity, the vector allocates a larger
block, **moves/copies** the existing elements into it, and frees the old block.
The exact sequence of capacities is *implementation-defined*. With the common
**2x growth factor** (libstdc++/GCC and libc++/Clang), the capacities grow
`1, 2, 4, 8, 16, ...`:

```
Initial: capacity=0, size=0
         []

push_back(1):  REALLOCATION (0 -> 1)
         [1]
         capacity=1, size=1

push_back(2):  REALLOCATION (1 -> 2)
         [1][2]
         capacity=2, size=2

push_back(3):  REALLOCATION (2 -> 4)
         [1][2][3][?]
         capacity=4, size=3

push_back(4):  (fits)
         [1][2][3][4]
         capacity=4, size=4

push_back(5):  REALLOCATION (4 -> 8)
         [1][2][3][4][5][?][?][?]
         capacity=8, size=5
```

**Growth factor by implementation:**
- libstdc++ (GCC) and libc++ (Clang): **2x**
- MSVC: **1.5x** (capacities grow `1, 2, 3, 4, 6, 9, ...`)

A factor below 2 lets freed blocks be reused by later allocations, which can
reduce memory fragmentation — that is the rationale behind MSVC's 1.5x.

> **Tip:** If you know the final size, call `reserve(n)` *once* up front. This
> turns N reallocations into a single allocation and prevents the iterator/
> pointer invalidation discussed under [Common Pitfalls](#common-pitfalls).

### Performance Analysis
```
Operation          | Time      | Why
-------------------|-----------|----------------------------------
Access (operator[])| O(1)      | Direct array indexing
push_back()        | O(1)*     | *Amortized (occasional realloc)
pop_back()         | O(1)      | Just decrement size
insert() at front  | O(n)      | Shift all elements
insert() at middle | O(n)      | Shift half elements
erase() at front   | O(n)      | Shift all elements
find()             | O(n)      | Linear search
```

### When to Use std::vector
✓ Need random access  
✓ Mostly add/remove at the end  
✓ Cache-friendly (contiguous memory)  
✓ Default choice for most cases  
✗ Frequent insertions in middle  
✗ Need stable iterators (iterators invalidate on reallocation)  

### C++20/23 Features
```cpp
#include <vector>
#include <ranges>   // std::views::iota, std::from_range (C++20/23)

// C++20: uniform container erasure (free functions, not members).
// These fix the classic "erase-remove" idiom footgun (see Common Pitfalls).
std::vector<int> v = {1, 2, 3, 4, 5, 6};
std::erase(v, 3);               // Remove all elements equal to 3
std::erase_if(v, [](int x) {    // Remove all even numbers
    return x % 2 == 0;
});

// C++23: construct directly from any range with the std::from_range_t tag
auto some_range = std::views::iota(0, 5);          // 0,1,2,3,4 (see Modern Features)
std::vector<int> v1(std::from_range, some_range);
```

> `std::vector` has no `contains()` member — use
> [`std::ranges::find`](06_algorithms.md) / `std::find` and compare against
> `end()`, or pick an [associative](02_associative_containers.md) /
> [unordered](03_unordered_containers.md) container if membership testing is a
> hot path.

---

## 2. std::deque

### Description
Double-ended queue - allows fast insertion and deletion at both ends.

### Memory Layout
```
Deque Structure (chunked array):

┌────────────┐
│  Map Array │ (pointers to chunks)
├────────────┤
│  nullptr   │
│  chunk1*   │───────┐
│  chunk2*   │───────┼────┐
│  chunk3*   │───────┼────┼────┐
│  nullptr   │       │    │    │
└────────────┘       │    │    │
                     │    │    │
    Heap:            ▼    ▼    ▼
    ┌──────────┐┌──────────┐┌──────────┐
    │[0][1][2] ││[3][4][5] ││[6][7][8] │
    └──────────┘└──────────┘└──────────┘
    
Advantage: No reallocation of existing elements!
```

### Key Characteristics
- **Access**: O(1) random access (with small overhead)
- **Insertion/Deletion at both ends**: O(1)
- **Insertion/Deletion at middle**: O(n)
- **Memory**: Non-contiguous chunks

### Basic Operations

```cpp
#include <deque>
#include <iostream>

int main() {
    std::deque<int> dq;
    
    // Adding elements at both ends
    dq.push_back(3);        // [3]
    dq.push_back(4);        // [3, 4]
    dq.push_front(2);       // [2, 3, 4]
    dq.push_front(1);       // [1, 2, 3, 4]
    
    // Removing from both ends
    dq.pop_front();         // [2, 3, 4]
    dq.pop_back();          // [2, 3]
    
    // Access (same as vector)
    int first = dq[0];
    int last = dq.at(dq.size() - 1);
    
    // Other operations similar to vector
    dq.insert(dq.begin() + 1, 99);
    dq.erase(dq.begin());
    
    return 0;
}
```

### Vector vs Deque Comparison
```
                    std::vector         std::deque
push_back()         O(1) amortized      O(1)
push_front()        N/A                 O(1)
pop_back()          O(1)                O(1)
pop_front()         N/A                 O(1)
Random access       O(1) faster         O(1) slower
Memory overhead     Lower               Higher
Iterator stability  Can invalidate      More stable
```

### When to Use std::deque
✓ Need to insert/remove at both ends  
✓ Need random access  
✓ Don't need contiguous memory  
✓ Iterator stability important  
✗ Need minimal memory overhead  
✗ Need maximum access speed  

---

## 3. std::list

### Description
Doubly-linked list - each element points to next and previous elements.

### Memory Layout
```
List Structure:

┌─────────┐
│ List Obj│
├─────────┤
│ head*   │────┐
│ tail*   │────┼───┐
│ size    │    │   │
└─────────┘    │   │
               │   │
    Heap:      ▼   │
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │  Node 1      │   │  Node 2      │   │  Node 3      │
    ├──────────────┤   ├──────────────┤   ├──────────────┤
    │ prev: nullptr│◄──│ prev*        │◄──│ prev*        │
    │ data: 10     │   │ data: 20     │   │ data: 30     │
    │ next*        │──►│ next*        │──►│ next: nullptr│
    └──────────────┘   └──────────────┘   └──────────────┘
                                                          ▲
                                                          │
                                                       tail*
```

### Key Characteristics
- **Access**: O(n) - must traverse from beginning or end
- **Insertion/Deletion anywhere**: O(1) if you have iterator
- **Finding position**: O(n)
- **Memory**: Non-contiguous, higher overhead per element

### Basic Operations

```cpp
#include <list>
#include <iostream>

int main() {
    std::list<int> lst = {1, 2, 3, 4, 5};
    
    // Adding elements
    lst.push_front(0);      // [0, 1, 2, 3, 4, 5]
    lst.push_back(6);       // [0, 1, 2, 3, 4, 5, 6]
    
    // Insertion (at iterator position)
    auto it = lst.begin();
    std::advance(it, 3);    // Move iterator 3 positions
    lst.insert(it, 99);     // O(1) insertion!
    
    // Removal
    lst.pop_front();
    lst.pop_back();
    lst.remove(99);         // Remove all elements with value 99
    lst.remove_if([](int x) { return x % 2 == 0; }); // Remove even
    
    // Unique list operations
    std::list<int> lst2 = {1, 1, 2, 3, 3, 4};
    lst2.unique();          // [1, 2, 3, 4] - remove consecutive duplicates
    
    lst2.sort();            // Sort the list
    lst2.reverse();         // Reverse the list
    
    // Splicing (move elements between lists)
    std::list<int> lst3 = {10, 20, 30};
    std::list<int> lst4 = {40, 50};
    lst3.splice(lst3.end(), lst4); // Move all of lst4 to end of lst3
    // lst3: [10, 20, 30, 40, 50], lst4: []
    
    return 0;
}
```

### List-Specific Operations
```
Operation              | Time  | Description
-----------------------|-------|----------------------------------
splice()               | O(1)* | Transfer elements between lists
merge()                | O(n)  | Merge two sorted lists
unique()               | O(n)  | Remove consecutive duplicates
sort()                 | O(n log n) | Sort the list
reverse()              | O(n)  | Reverse element order

* O(1) for single element, O(n) for range
```

### Splicing Example
```
Before splice:
list1: [1]→[2]→[3]→[4]
list2: [A]→[B]→[C]

// std::list iterators are bidirectional, so advance with std::next
// (there is no operator+ for them):
list1.splice(std::next(list1.begin(), 2), list2);

After splice:
list1: [1]→[2]→[A]→[B]→[C]→[3]→[4]
list2: (empty)

No copying! Just pointer manipulation:
[2]→next = [A]
[C]→next = [3]
```

### When to Use std::list
✓ Frequent insertions/deletions in middle  
✓ Need to splice elements between containers  
✓ Iterator stability critical (never invalidated)  
✓ Don't need random access  
✗ Need fast element access  
✗ Memory overhead is concern  
✗ Cache performance matters  

---

## 4. std::forward_list

### Description
Singly-linked list (C++11) - each element only points to next element. More memory efficient than `std::list`.

### Memory Layout
```
Forward List Structure:

┌─────────┐
│ FList   │
├─────────┤
│ head*   │────┐
└─────────┘    │
               │
    Heap:      ▼
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │  Node 1      │   │  Node 2      │   │  Node 3      │
    ├──────────────┤   ├──────────────┤   ├──────────────┤
    │ data: 10     │   │ data: 20     │   │ data: 30     │
    │ next*        │──►│ next*        │──►│ next: nullptr│
    └──────────────┘   └──────────────┘   └──────────────┘

Note: No prev pointer! Only forward traversal.
```

### Key Characteristics
- **Access**: O(n) - forward only
- **No size() method!** (would require O(n) or extra storage)
- **Insertion/Deletion after position**: O(1)
- **Memory**: Smallest overhead of linked lists

### Basic Operations

```cpp
#include <forward_list>
#include <iostream>

int main() {
    std::forward_list<int> flst = {1, 2, 3, 4, 5};
    
    // Adding elements
    flst.push_front(0);     // [0, 1, 2, 3, 4, 5]
    // No push_back()! (would be O(n))
    
    // Insert after position
    auto it = flst.begin();
    flst.insert_after(it, 99);  // Insert 99 after first element
    
    // Erase after position
    flst.erase_after(flst.begin()); // Erase element after first
    
    // No size() method!
    // Use std::distance(flst.begin(), flst.end()) if needed
    
    // Other operations
    flst.remove(3);         // Remove all 3s
    flst.remove_if([](int x) { return x > 10; });
    flst.sort();
    flst.reverse();
    flst.unique();
    
    // Splice after position
    std::forward_list<int> flst2 = {10, 20};
    flst.splice_after(flst.begin(), flst2);
    
    return 0;
}
```

### Special Iterator: before_begin()
```cpp
std::forward_list<int> flst = {1, 2, 3};

// To insert at beginning, use before_begin()
flst.insert_after(flst.before_begin(), 0); // [0, 1, 2, 3]

// Why? Because insert_after needs position BEFORE insertion point
```

```
before_begin()    begin()
      │              │
      ▼              ▼
   [dummy]────►[1]────►[2]────►[3]────►nullptr
   
insert_after(before_begin(), 0):
   [dummy]────►[0]────►[1]────►[2]────►[3]────►nullptr
                ▲
             begin()
```

### When to Use std::forward_list
✓ Minimum memory overhead needed  
✓ Only forward traversal needed  
✓ Frequent insertions after known position  
✗ Need bidirectional traversal  
✗ Need to know size quickly  
✗ Need push_back()  

---

## 5. std::array

### Description
Fixed-size array (C++11) - compile-time size, stack allocated by default.

### Memory Layout
```
Array on Stack:

┌────────────────┐
│  std::array<>  │
├────────────────┤
│  [0] = 10      │
│  [1] = 20      │
│  [2] = 30      │
│  [3] = 40      │
│  [4] = 50      │
└────────────────┘

No heap allocation!
Size is part of the type: std::array<int, 5>
```

### Key Characteristics
- **Access**: O(1)
- **Size**: Fixed at compile time
- **Memory**: Stack allocated (can be heap if object is on heap)
- **No overhead**: Same as C array, but with STL benefits

### Basic Operations

```cpp
#include <array>
#include <iostream>

int main() {
    // Declaration (size must be compile-time constant)
    std::array<int, 5> arr1;                    // Uninitialized
    std::array<int, 5> arr2 = {};              // All zeros
    std::array<int, 5> arr3 = {1, 2, 3, 4, 5}; // Initialized
    std::array<int, 5> arr4{1, 2, 3};          // Partial init (rest = 0)
    
    // Access
    int first = arr3[0];
    int second = arr3.at(1);    // With bounds checking
    int& front = arr3.front();
    int& back = arr3.back();
    int* data = arr3.data();
    
    // Size (compile-time constant)
    constexpr size_t size = arr3.size();
    bool empty = arr3.empty();  // Always false for size > 0
    
    // Fill
    arr1.fill(42);  // All elements = 42
    
    // Swap
    arr1.swap(arr2);  // O(n) - swaps all elements
    
    // Iterators work as expected
    std::sort(arr3.begin(), arr3.end());
    
    // C++17: Template argument deduction
    std::array arr5 = {1, 2, 3, 4, 5};  // Type deduced as std::array<int, 5>
    
    return 0;
}
```

### std::array vs C Array
```cpp
// C Array
int c_arr[5] = {1, 2, 3, 4, 5};
// - Cannot be copied with =
// - Decays to pointer
// - No bounds checking
// - No size() method

// std::array
std::array<int, 5> arr = {1, 2, 3, 4, 5};
// ✓ Can be copied with =
// ✓ No pointer decay
// ✓ Bounds checking with at()
// ✓ Has size() method
// ✓ Works with STL algorithms
```

### Comparison with std::vector
```
                    std::array          std::vector
Size                Fixed               Dynamic
Memory              Stack/In-place      Heap
Reallocation        Never               Sometimes
Overhead            None                Pointer + capacity
Copy cost           O(n)                O(n)
Size in type        Yes (template)      No
```

### When to Use std::array
✓ Size known at compile time  
✓ Want stack allocation  
✓ Need maximum performance  
✓ Interfacing with C APIs (use .data())  
✗ Size determined at runtime  
✗ Need to grow/shrink  

---

## Container Comparison Summary

### Quick Reference Table
```
Container        | Access | Insert End | Insert Mid | Memory      | Use Case
-----------------|--------|------------|------------|-------------|------------------
vector           | O(1)   | O(1)*      | O(n)       | Contiguous  | General purpose
deque            | O(1)   | O(1)       | O(n)       | Chunked     | Double-ended ops
list             | O(n)   | O(1)       | O(1)†      | Nodes       | Frequent mid ins.
forward_list     | O(n)   | N/A        | O(1)†      | Nodes       | Memory critical
array            | O(1)   | N/A        | N/A        | Contiguous  | Fixed size

* Amortized
† If iterator to position is known
```

### Decision Tree
```
Need dynamic size?
├─ No  ──→ std::array
└─ Yes
   └─ Need front/back operations?
      ├─ Both ──→ std::deque
      └─ Back only
         └─ Frequent middle insertions?
            ├─ Yes
            │  └─ Need bidirectional?
            │     ├─ Yes ──→ std::list
            │     └─ No  ──→ std::forward_list
            └─ No ──→ std::vector (default choice)
```

### Memory Layout Visual Summary
```
VECTOR (contiguous):
┌──┬──┬──┬──┬──┐
│ 1│ 2│ 3│ 4│ 5│
└──┴──┴──┴──┴──┘

DEQUE (chunked):
┌──┬──┐  ┌──┬──┐  ┌──┬──┐
│ 1│ 2│  │ 3│ 4│  │ 5│ 6│
└──┴──┘  └──┴──┘  └──┴──┘

LIST (doubly-linked):
┌──┐  ┌──┐  ┌──┐  ┌──┐
│ 1│◄►│ 2│◄►│ 3│◄►│ 4│
└──┘  └──┘  └──┘  └──┘

FORWARD_LIST (singly-linked):
┌──┐  ┌──┐  ┌──┐  ┌──┐
│ 1│─►│ 2│─►│ 3│─►│ 4│
└──┘  └──┘  └──┘  └──┘

ARRAY (fixed contiguous):
┌──┬──┬──┬──┬──┐
│ 1│ 2│ 3│ 4│ 5│  (fixed size)
└──┴──┴──┴──┴──┘
```

---

## Practice Exercises

### Exercise 1: Vector Capacity
```cpp
// Experiment with vector growth
std::vector<int> v;
for (int i = 0; i < 100; ++i) {
    v.push_back(i);
    std::cout << "Size: " << v.size() 
              << " Capacity: " << v.capacity() << '\n';
}
// Observe the growth pattern
```

### Exercise 2: Performance Comparison
```cpp
// Compare insertion performance
#include <chrono>

// Test vector vs list for middle insertions
// Which is faster for 10,000 insertions?
```

### Exercise 3: When to Use Each Container
Given these scenarios, choose the best container:
1. Store coordinates (x, y, z) for 3D point - always 3 values
2. Implement a web browser's back/forward navigation
3. Implement a text editor with undo/redo
4. Store game high scores that need sorting
5. LRU cache implementation

**Answers:**
1. `std::array<double, 3>` — fixed size, no heap allocation.
2. `std::deque<Page>` or two `std::stack`s — you push/pop at both ends (or use back/forward stacks). A `std::vector` works too if you only grow one side.
3. `std::vector<State>` with an index, or `std::list<State>` — undo/redo is mostly push/pop at one end, so a `vector` is usually the cache-friendlier choice; reach for `list` only if you must splice from the middle.
4. `std::vector<int>` — contiguous storage sorts fastest (see [Algorithms](06_algorithms.md)).
5. **LRU cache** needs *two* structures working together: a `std::list<std::pair<Key,Value>>` for recency order (move-to-front in O(1) via `splice`) **plus** a `std::unordered_map<Key, list::iterator>` for O(1) lookup. A list alone gives O(n) lookups. See [Unordered Containers](03_unordered_containers.md).

---

## Common Pitfalls

### 1. Vector Reallocation
```cpp
std::vector<int> v = {1, 2, 3};
int* ptr = &v[0];
v.push_back(4);  // May reallocate!
// ptr is now INVALID - dangling pointer!
```

### 2. Deque Iterator Invalidation
```cpp
std::deque<int> dq = {1, 2, 3};
auto it = dq.begin();
dq.push_front(0);  // Invalidates ALL iterators...
// it is now INVALID
// ...but inserting at *either end* keeps references and pointers to
// existing elements valid (only middle insertion invalidates those).
```

> Iterator-invalidation rules differ per container and per operation. The
> [Iterators](05_iterators.md) chapter has the full table; when in doubt,
> re-fetch the iterator after any modifying call.

### 3. List Inefficient Access
```cpp
std::list<int> lst = {1, 2, 3, 4, 5};
// DON'T: lst[2] doesn't exist!
// DO: 
auto it = lst.begin();
std::advance(it, 2);
int value = *it;
```

### 4. Array Size Type
```cpp
template<typename T, size_t N>
void process(std::array<T, N>& arr) { /* ... */ }

std::array<int, 5> arr1;
std::array<int, 10> arr2;
// process() with arr1 and arr2 are DIFFERENT functions!
// Size is part of type!
```

---

## Complete Practical Example: Task Manager

Here's a comprehensive example that integrates multiple sequence containers to build a simple task management system:

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <array>
#include <string>
#include <algorithm>
#include <chrono>

// Task structure
struct Task {
    int id;
    std::string description;
    int priority;
    std::chrono::system_clock::time_point created;
    
    Task(int i, std::string desc, int prio)
        : id(i), description(std::move(desc)), priority(prio)
        , created(std::chrono::system_clock::now()) {}
};

class TaskManager {
private:
    // vector: Dynamic list of all tasks (frequent random access)
    std::vector<Task> all_tasks_;
    
    // deque: Task queue (add to both ends, process from front)
    std::deque<int> pending_queue_;  // Stores task IDs
    
    // list: Recently completed (frequent insertions/deletions)
    std::list<int> completed_history_;
    
    // array: Priority levels (fixed size)
    std::array<int, 5> priority_counts_ = {0, 0, 0, 0, 0};
    
    int next_id_ = 1;
    
public:
    // Add new task (uses vector::push_back and deque::push_back)
    int add_task(const std::string& description, int priority = 2) {
        if (priority < 0 || priority > 4) {
            throw std::invalid_argument("Priority must be 0-4");
        }
        
        int id = next_id_++;
        all_tasks_.emplace_back(id, description, priority);
        pending_queue_.push_back(id);
        priority_counts_[priority]++;
        
        std::cout << "Added task #" << id << ": " << description 
                  << " (priority " << priority << ")\n";
        return id;
    }
    
    // Add urgent task to front (uses deque::push_front)
    int add_urgent_task(const std::string& description) {
        int id = add_task(description, 4);
        // Move to front of queue
        pending_queue_.pop_back();  // Remove from back
        pending_queue_.push_front(id);  // Add to front
        
        std::cout << "  → Moved to front of queue (URGENT)\n";
        return id;
    }
    
    // Process next task (uses deque::front and list::push_back)
    void process_next_task() {
        if (pending_queue_.empty()) {
            std::cout << "No pending tasks!\n";
            return;
        }
        
        int task_id = pending_queue_.front();
        pending_queue_.pop_front();
        
        // Find task in vector (random access)
        auto it = std::find_if(all_tasks_.begin(), all_tasks_.end(),
            [task_id](const Task& t) { return t.id == task_id; });
        
        if (it != all_tasks_.end()) {
            std::cout << "Processing task #" << it->id << ": " 
                      << it->description << "\n";
            
            // Add to completed history (list allows efficient insertion)
            completed_history_.push_back(task_id);
            
            // Keep only last 10 completed tasks
            if (completed_history_.size() > 10) {
                completed_history_.pop_front();
            }
        }
    }
    
    // Get tasks by priority (uses vector iteration and array indexing)
    std::vector<Task> get_tasks_by_priority(int priority) const {
        std::vector<Task> result;
        result.reserve(priority_counts_[priority]);
        
        for (const auto& task : all_tasks_) {
            if (task.priority == priority) {
                result.push_back(task);
            }
        }
        
        return result;
    }
    
    // Display statistics (uses all containers)
    void show_statistics() const {
        std::cout << "\n=== Task Manager Statistics ===\n";
        std::cout << "Total tasks: " << all_tasks_.size() << "\n";
        std::cout << "Pending: " << pending_queue_.size() << "\n";
        std::cout << "Recently completed: " << completed_history_.size() << "\n";
        
        std::cout << "\nPriority breakdown:\n";
        for (size_t i = 0; i < priority_counts_.size(); ++i) {
            if (priority_counts_[i] > 0) {
                std::cout << "  Priority " << i << ": " 
                          << priority_counts_[i] << " tasks\n";
            }
        }
        
        if (!completed_history_.empty()) {
            std::cout << "\nRecent completions (IDs): ";
            for (int id : completed_history_) {
                std::cout << id << " ";
            }
            std::cout << "\n";
        }
    }
    
    // Display pending queue (uses deque iteration)
    void show_queue() const {
        std::cout << "\n=== Pending Queue ===\n";
        if (pending_queue_.empty()) {
            std::cout << "(empty)\n";
            return;
        }
        
        for (size_t i = 0; i < pending_queue_.size(); ++i) {
            int task_id = pending_queue_[i];  // Random access in deque
            
            auto it = std::find_if(all_tasks_.begin(), all_tasks_.end(),
                [task_id](const Task& t) { return t.id == task_id; });
            
            if (it != all_tasks_.end()) {
                std::cout << i + 1 << ". Task #" << it->id 
                          << " [P" << it->priority << "]: " 
                          << it->description << "\n";
            }
        }
    }
};

int main() {
    TaskManager tm;
    
    // Add various tasks (demonstrates vector and deque)
    tm.add_task("Write documentation", 2);
    tm.add_task("Fix bug #123", 3);
    tm.add_task("Review pull request", 1);
    tm.add_task("Update dependencies", 1);
    
    // Add urgent task (demonstrates deque::push_front).
    // add_urgent_task() takes only a description; it sets priority 4 internally.
    tm.add_urgent_task("Critical security patch!");
    
    tm.show_queue();
    tm.show_statistics();
    
    // Process some tasks (demonstrates list)
    std::cout << "\n=== Processing Tasks ===\n";
    for (int i = 0; i < 3; ++i) {
        tm.process_next_task();
    }
    
    tm.show_queue();
    tm.show_statistics();
    
    // Query by priority (demonstrates vector filtering)
    std::cout << "\n=== High Priority Tasks (3+) ===\n";
    auto high_priority = tm.get_tasks_by_priority(3);
    for (const auto& task : high_priority) {
        std::cout << "Task #" << task.id << ": " << task.description << "\n";
    }
    
    return 0;
}
```

### Output:
```
Added task #1: Write documentation (priority 2)
Added task #2: Fix bug #123 (priority 3)
Added task #3: Review pull request (priority 1)
Added task #4: Update dependencies (priority 1)
Added task #5: Critical security patch! (priority 4)
  → Moved to front of queue (URGENT)

=== Pending Queue ===
1. Task #5 [P4]: Critical security patch!
2. Task #1 [P2]: Write documentation
3. Task #2 [P3]: Fix bug #123
4. Task #3 [P1]: Review pull request
5. Task #4 [P1]: Update dependencies

=== Task Manager Statistics ===
Total tasks: 5
Pending: 5
Recently completed: 0

Priority breakdown:
  Priority 1: 2 tasks
  Priority 2: 1 tasks
  Priority 3: 1 tasks
  Priority 4: 1 tasks

=== Processing Tasks ===
Processing task #5: Critical security patch!
Processing task #1: Write documentation
Processing task #2: Fix bug #123

=== Pending Queue ===
1. Task #3 [P1]: Review pull request
2. Task #4 [P1]: Update dependencies

=== Task Manager Statistics ===
Total tasks: 5
Pending: 2
Recently completed: 3

Priority breakdown:
  Priority 1: 2 tasks
  Priority 2: 1 tasks
  Priority 3: 1 tasks
  Priority 4: 1 tasks

Recent completions (IDs): 5 1 2
```

### Concepts Demonstrated:
- **vector**: Dynamic task storage with random access
- **deque**: Queue with efficient push/pop from both ends
- **list**: Completed history with frequent insertions/deletions
- **array**: Fixed-size priority counters
- **emplace_back**: In-place construction
- **push_front/push_back**: Adding to both ends
- **Random access**: Indexing into containers
- **Iteration**: Range-based for loops
- **STL algorithms**: `std::find_if` for searching

This example shows why you'd choose different containers for different use cases!

---

## Related Topics
- [Iterators](05_iterators.md) — how to traverse these containers and the invalidation rules.
- [Algorithms](06_algorithms.md) — `sort`, `find`, `erase`-`remove`, and friends that operate on sequences.
- [Container Adaptors](04_container_adaptors.md) — `stack`/`queue`/`priority_queue` are built on top of `deque`/`vector`.
- [Modern Features](07_modern_features.md) — `std::span` gives a non-owning view over contiguous sequences; ranges/views compose lazily.
- [Memory Management & Allocators](19_memory_allocators.md) — customize how these containers allocate.
- [Advanced Features](12_advanced_features.md) — move semantics explain why `emplace_back` and reallocation behave the way they do.
- [Quick Reference](99_quick_reference.md) — complexity table and the container decision tree at a glance.

## Next Steps
- **Next**: [Associative Containers →](02_associative_containers.md)
- **Previous**: [← Main README](README.md)

---
*Chapter 1 — Sequence Containers*

