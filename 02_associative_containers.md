# Associative Containers

## Overview

Associative containers store elements in a sorted order based on keys. They use tree-based data structures (typically red-black trees) to provide logarithmic time complexity for most operations.

> If you don't need sorted order or range queries, the hash-based
> [unordered containers](03_unordered_containers.md) offer **O(1) average**
> lookups instead of **O(log n)**. The choice between them is one of the most
> common container decisions — see the
> [decision tree in the Quick Reference](99_quick_reference.md#container-selection-guide).

```
┌──────────────────────────────────────────────────────┐
│         ASSOCIATIVE CONTAINERS FAMILY                │
├──────────────────────────────────────────────────────┤
│                                                      │
│  std::set            Sorted unique elements          │
│  std::multiset       Sorted elements (duplicates)    │
│  std::map            Sorted key-value pairs (unique) │
│  std::multimap       Sorted key-value pairs (dupes)  │
│                                                      │
│  All use balanced binary search trees (Red-Black)    │
│  All maintain sorted order automatically             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 1. std::set

### Description
A sorted collection of unique elements. Elements are sorted by their values using a comparison function (default: `<` operator).

### Internal Structure (Red-Black Tree)
```
Red-Black Tree Example:

                    ┌───┐
                    │ 5 │ (Black)
                    └─┬─┘
              ┌───────┴───────┐
              ▼               ▼
            ┌───┐           ┌───┐
            │ 3 │ (Red)     │ 8 │ (Red)
            └─┬─┘           └─┬─┘
          ┌───┴───┐       ┌───┴───┐
          ▼       ▼       ▼       ▼
        ┌───┐   ┌───┐   ┌───┐   ┌───┐
        │ 1 │   │ 4 │   │ 7 │   │ 9 │ (All Black)
        └───┘   └───┘   └───┘   └───┘

Properties:
- Self-balancing
- Height = O(log n)
- In-order traversal gives sorted sequence
- Each node: Black or Red with specific rules
```

### Key Characteristics
- **Search**: O(log n)
- **Insertion**: O(log n)
- **Deletion**: O(log n)
- **Elements**: Always sorted, unique
- **Iterators**: Bidirectional (not random access)

### Basic Operations

```cpp
#include <set>
#include <iostream>

int main() {
    // Construction
    std::set<int> s1;                          // Empty set
    std::set<int> s2 = {5, 2, 8, 1, 9, 2};    // Initializer (2 appears once!)
    std::set<int> s3(s2);                      // Copy
    std::set<int, std::greater<int>> s4 = {1, 2, 3}; // Descending order
    
    // Insertion
    std::set<int> s;
    s.insert(5);                    // Returns pair<iterator, bool>
    s.insert(3);
    s.insert(7);
    s.insert(3);                    // Ignored - already exists
    // s = {3, 5, 7}
    
    auto [it, success] = s.insert(9);
    if (success) {
        std::cout << "Inserted: " << *it << '\n';
    }
    
    // Emplace (construct in-place)
    s.emplace(4);
    s.emplace_hint(s.begin(), 1);  // Hint for insertion position
    
    // Search
    auto found = s.find(5);
    if (found != s.end()) {
        std::cout << "Found: " << *found << '\n';
    }
    
    // C++20: contains()
    if (s.contains(7)) {
        std::cout << "Set contains 7\n";
    }
    
    // Count (returns 0 or 1 for set)
    if (s.count(3) > 0) {
        std::cout << "3 exists\n";
    }
    
    // Erase
    s.erase(5);                     // Erase by value
    s.erase(s.begin());             // Erase by iterator
    s.erase(s.find(7), s.end());    // Erase range
    
    // Size
    std::cout << "Size: " << s.size() << '\n';
    bool empty = s.empty();
    
    // Iteration (always in sorted order!)
    std::set<int> nums = {5, 2, 8, 1, 9};
    for (int n : nums) {
        std::cout << n << ' ';  // Prints: 1 2 5 8 9
    }
    
    return 0;
}
```

### Range Operations

```cpp
#include <set>

int main() {
    std::set<int> s = {1, 3, 5, 7, 9, 11, 13, 15};
    
    // Lower bound: first element >= value
    auto lb = s.lower_bound(7);   // Points to 7
    
    // Upper bound: first element > value
    auto ub = s.upper_bound(7);   // Points to 9
    
    // Equal range: [lower_bound, upper_bound)
    auto [first, last] = s.equal_range(7);
    // first points to 7, last points to 9
    
    // Example: Find all elements in range [5, 10)
    auto start = s.lower_bound(5);
    auto end = s.lower_bound(10);
    for (auto it = start; it != end; ++it) {
        std::cout << *it << ' ';  // Prints: 5 7 9
    }
    
    return 0;
}
```

### Visual Explanation of Bounds
```
Set: {1, 3, 5, 7, 9, 11, 13}

lower_bound(7):  Points to first element >= 7
                 ↓
         1  3  5  7  9  11  13
                 ▲
                 └─ Returns iterator here

upper_bound(7):  Points to first element > 7
                    ↓
         1  3  5  7  9  11  13
                    ▲
                    └─ Returns iterator here

equal_range(7):  [lower_bound, upper_bound)
                 [──────]
         1  3  5  7  9  11  13
                 ▲─▲
                 │ └─ upper_bound
                 └─ lower_bound
```

### Custom Comparator

```cpp
#include <set>
#include <string>

// Custom comparator struct
struct CaseInsensitiveCompare {
    bool operator()(const std::string& a, const std::string& b) const {
        return std::lexicographical_compare(
            a.begin(), a.end(),
            b.begin(), b.end(),
            [](char c1, char c2) {
                return std::tolower(c1) < std::tolower(c2);
            }
        );
    }
};

int main() {
    std::set<std::string, CaseInsensitiveCompare> words;
    words.insert("Hello");
    words.insert("HELLO");  // Considered duplicate!
    words.insert("World");
    // Set contains: {"Hello", "World"}
    
    // Lambda comparator (C++11)
    auto comp = [](int a, int b) { return a > b; }; // Descending
    std::set<int, decltype(comp)> desc_set(comp);
    desc_set.insert({1, 5, 3, 9, 2});
    // Set: {9, 5, 3, 2, 1}
    
    return 0;
}
```

### When to Use std::set
✓ Need sorted unique elements  
✓ Frequent searches required  
✓ Need to find ranges (lower_bound, etc.)  
✓ Want automatic sorting  
✗ Need duplicates (use multiset)  
✗ Order doesn't matter (use unordered_set - faster)  
✗ Need random access by index  

---

## 2. std::multiset

### Description
Similar to `std::set`, but allows duplicate elements. All duplicates are stored adjacently in sorted order.

### Visual Representation
```
Multiset: {1, 3, 3, 3, 5, 5, 7}

Internal Tree (simplified):
                ┌───┐
                │ 3 │
                └─┬─┘
          ┌───────┴───────┐
          ▼               ▼
        ┌───┐           ┌───┐
        │ 1 │           │ 5 │
        └───┘           └─┬─┘
                    ┌─────┴─────┐
                    ▼           ▼
                  ┌───┐       ┌───┐
                  │ 3 │       │ 7 │
                  └─┬─┘       └───┘
                    │
                    ▼
                  ┌───┐
                  │ 3 │
                  └───┘

Note: Implementation may vary, but duplicates are adjacent when iterating.
```

### Basic Operations

```cpp
#include <set>
#include <iostream>

int main() {
    std::multiset<int> ms;
    
    // Insert duplicates
    ms.insert(5);
    ms.insert(3);
    ms.insert(5);  // Allowed!
    ms.insert(3);
    ms.insert(5);
    // ms = {3, 3, 5, 5, 5}
    
    // Count occurrences
    std::cout << "Count of 5: " << ms.count(5) << '\n';  // 3
    std::cout << "Count of 3: " << ms.count(3) << '\n';  // 2
    
    // Find (returns iterator to any occurrence)
    auto it = ms.find(5);
    if (it != ms.end()) {
        std::cout << "Found: " << *it << '\n';
    }
    
    // Equal range (find all occurrences)
    auto [first, last] = ms.equal_range(5);
    std::cout << "All 5s: ";
    for (auto it = first; it != last; ++it) {
        std::cout << *it << ' ';  // 5 5 5
    }
    std::cout << '\n';
    
    // Erase
    ms.erase(5);              // Removes ALL occurrences of 5
    // ms = {3, 3}
    
    // Erase single occurrence
    auto found = ms.find(3);
    if (found != ms.end()) {
        ms.erase(found);      // Removes only one 3
    }
    // ms = {3}
    
    return 0;
}
```

### Multiset Use Cases

```cpp
// Example 1: Frequency tracking with automatic sorting
std::multiset<int> scores = {85, 92, 78, 92, 85, 88, 92};
std::cout << "92 appears " << scores.count(92) << " times\n";

// Example 2: Event scheduling (with duplicates)
std::multiset<std::string> events = {"Meeting", "Lunch", "Meeting", "Call"};
// Automatically sorted: {Call, Lunch, Meeting, Meeting}

// Example 3: Running median (with two multisets)
// Keep smaller half in max-heap, larger half in min-heap
```

### set vs multiset
```
Operation          | set              | multiset
-------------------|------------------|------------------
Duplicate elements | No               | Yes
insert() return    | pair<iter, bool> | iterator
count()            | 0 or 1           | 0 to n
erase(value)       | Removes 1        | Removes all
```

---

## 3. std::map

### Description
Sorted collection of key-value pairs. Keys are unique and sorted. Also called an associative array or dictionary.

### Internal Structure
```
Map<string, int> Example:
Keys:   {"apple", "banana", "cherry", "date"}
Values: {5,       3,        8,         2}

Internal Red-Black Tree:
                    ┌─────────────┐
                    │   "cherry"  │
                    │      8      │
                    └──────┬──────┘
              ┌────────────┴────────────┐
              ▼                         ▼
        ┌──────────┐              ┌──────────┐
        │  "banana"│              │  "date"  │
        │     3    │              │     2    │
        └────┬─────┘              └──────────┘
             │
             ▼
        ┌──────────┐
        │  "apple" │
        │     5    │
        └──────────┘

Each node stores a pair: {key, value}
```

### Basic Operations

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    // Construction
    std::map<std::string, int> m1;
    std::map<std::string, int> m2 = {
        {"Alice", 25},
        {"Bob", 30},
        {"Charlie", 35}
    };
    
    // Insertion methods
    std::map<std::string, int> ages;
    
    // Method 1: insert with pair
    ages.insert(std::make_pair("Alice", 25));
    ages.insert({"Bob", 30});  // C++11 brace initialization
    
    // Method 2: insert with value_type
    ages.insert(std::map<std::string, int>::value_type("Charlie", 35));
    
    // Method 3: operator[] (creates if doesn't exist)
    ages["David"] = 40;
    ages["Eve"] = 28;
    
    // Method 4: emplace (C++11)
    ages.emplace("Frank", 45);
    
    // Method 5: try_emplace (C++17 - doesn't overwrite)
    auto [it, inserted] = ages.try_emplace("Alice", 99);
    // Alice already exists, so value remains 25
    
    // Method 6: insert_or_assign (C++17)
    ages.insert_or_assign("Alice", 26);  // Updates to 26
    
    // Access
    int alice_age = ages["Alice"];     // Returns 26
    int bob_age = ages.at("Bob");      // With bounds checking (throws if not found)
    
    // operator[] creates element if it doesn't exist!
    int unknown = ages["Unknown"];     // Creates "Unknown" with value 0
    
    // Safe access
    if (ages.contains("Charlie")) {    // C++20
        std::cout << ages["Charlie"] << '\n';
    }
    
    // Or pre-C++20:
    if (ages.find("Charlie") != ages.end()) {
        std::cout << ages["Charlie"] << '\n';
    }
    
    // Erase
    ages.erase("Unknown");             // By key
    ages.erase(ages.find("Eve"));      // By iterator
    
    // Iteration
    for (const auto& [key, value] : ages) {  // C++17 structured bindings
        std::cout << key << ": " << value << '\n';
    }
    // Output in sorted order by key!
    
    return 0;
}
```

### Important: operator[] vs at() vs find()

```cpp
std::map<std::string, int> m = { {"A", 1}, {"B", 2} };

// operator[] - creates if doesn't exist
int x = m["C"];        // Creates "C" with value 0
// m = { {"A", 1}, {"B", 2}, {"C", 0} }

// at() - throws exception if doesn't exist
try {
    int y = m.at("D"); // Throws std::out_of_range
} catch (const std::out_of_range& e) {
    std::cout << "Key not found!\n";
}

// find() - returns iterator (safe check)
auto it = m.find("E");
if (it != m.end()) {
    int z = it->second;
} else {
    std::cout << "Key not found!\n";
}
```

### Visual Comparison
```
Initial map: {"A": 1, "B": 2}

After m["C"]:
┌───┬───┬───┐
│A:1│B:2│C:0│  ← "C" added with default value!
└───┴───┴───┘

m.at("D"):
┌───┬───┬───┐
│A:1│B:2│C:0│  ← No change, exception thrown
└───┴───┴───┘

m.find("D"):
┌───┬───┬───┐
│A:1│B:2│C:0│  ← No change, returns end()
└───┴───┴───┘
```

### Searching and Bounds

```cpp
std::map<int, std::string> m = {
    {1, "one"}, {3, "three"}, {5, "five"}, {7, "seven"}, {9, "nine"}
};

// Lower bound
auto lb = m.lower_bound(5);   // Points to {5, "five"}
auto lb2 = m.lower_bound(4);  // Points to {5, "five"} (next element)

// Upper bound
auto ub = m.upper_bound(5);   // Points to {7, "seven"}

// Equal range
auto [first, last] = m.equal_range(5);
// first points to {5, "five"}, last points to {7, "seven"}

// Extract keys or values
std::vector<int> keys;
for (const auto& [key, value] : m) {
    keys.push_back(key);
}

std::vector<std::string> values;
for (const auto& [key, value] : m) {
    values.push_back(value);
}
```

### C++17 Features

```cpp
// Node extraction (move nodes between maps without reallocation)
std::map<int, std::string> m1 = { {1, "a"}, {2, "b"} };
std::map<int, std::string> m2 = { {3, "c"} };

// Extract node
auto node = m1.extract(1);  // Removes {1, "a"} from m1
node.key() = 4;             // Change key
m2.insert(std::move(node)); // Insert into m2
// m1 = { {2, "b"} }, m2 = { {3, "c"}, {4, "a"} }

// Merge maps
m2.merge(m1);  // Move all elements from m1 to m2
// m1 becomes empty, m2 has all elements
```

### Custom Key Type

```cpp
struct Person {
    std::string name;
    int age;
    
    // Must define comparison for map
    bool operator<(const Person& other) const {
        if (name != other.name)
            return name < other.name;
        return age < other.age;
    }
};

int main() {
    std::map<Person, std::string> people;
    people[{"Alice", 25}] = "Engineer";
    people[{"Bob", 30}] = "Doctor";
    
    // Or use custom comparator
    auto comp = [](const Person& a, const Person& b) {
        return a.age < b.age;
    };
    std::map<Person, std::string, decltype(comp)> by_age(comp);
    
    return 0;
}
```

### When to Use std::map
✓ Need key-value associations  
✓ Need keys in sorted order  
✓ Frequent lookups by key  
✓ Need range queries on keys  
✗ Don't need sorting (use unordered_map - faster)  
✗ Need multiple values per key (use multimap)  

---

## 4. std::multimap

### Description
Like `std::map`, but allows multiple values for the same key. All values for a key are stored adjacently in sorted order.

### Visual Representation
```
Multimap Example: Student grades

Key (Student) → Values (Grades)
"Alice"       → {85, 92, 88}
"Bob"         → {78, 91}
"Charlie"     → {95}

Internal structure maintains sorted keys,
and all values for a key are adjacent.
```

### Basic Operations

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    std::multimap<std::string, int> grades;
    
    // Insert multiple values for same key
    grades.insert({"Alice", 85});
    grades.insert({"Alice", 92});
    grades.insert({"Alice", 88});
    grades.insert({"Bob", 78});
    grades.insert({"Bob", 91});
    grades.insert({"Charlie", 95});
    
    // NO operator[] - which value would it return?
    // Must use find() or equal_range()
    
    // Count values for a key
    std::cout << "Alice has " << grades.count("Alice") << " grades\n";  // 3
    
    // Find (returns iterator to first occurrence)
    auto it = grades.find("Alice");
    if (it != grades.end()) {
        std::cout << "First Alice grade: " << it->second << '\n';
    }
    
    // Get all values for a key
    auto [first, last] = grades.equal_range("Alice");
    std::cout << "All Alice's grades: ";
    for (auto it = first; it != last; ++it) {
        // 85 92 88 — the container sorts by KEY only. Among equal keys,
        // elements keep their INSERTION order (guaranteed since C++11),
        // so values are NOT sorted.
        std::cout << it->second << ' ';
    }
    std::cout << '\n';
    
    // Iterate all
    for (const auto& [student, grade] : grades) {
        std::cout << student << ": " << grade << '\n';
    }
    // Output (keys sorted; values in insertion order within each key):
    // Alice: 85
    // Alice: 92
    // Alice: 88
    // Bob: 78
    // Bob: 91
    // Charlie: 95
    
    // Erase
    grades.erase("Alice");  // Removes ALL of Alice's grades
    
    // Erase single value
    auto found = grades.find("Bob");
    if (found != grades.end()) {
        grades.erase(found);  // Removes only one Bob grade
    }
    
    return 0;
}
```

### Common Use Cases

```cpp
// Use Case 1: Word index (word → line numbers)
std::multimap<std::string, int> word_index;
word_index.insert({"hello", 1});
word_index.insert({"world", 1});
word_index.insert({"hello", 5});
word_index.insert({"hello", 12});
// "hello" appears on lines: 1, 5, 12

// Use Case 2: One-to-many relationships
std::multimap<std::string, std::string> author_books;
author_books.insert({"Tolkien", "The Hobbit"});
author_books.insert({"Tolkien", "LOTR: Fellowship"});
author_books.insert({"Tolkien", "LOTR: Two Towers"});
author_books.insert({"Tolkien", "LOTR: Return of King"});

// Use Case 3: Event log with timestamps
std::multimap<std::string, Event> timeline;
timeline.insert({"2024-01-01", Event{/* ... */} });
timeline.insert({"2024-01-01", Event{/* ... */} });
timeline.insert({"2024-01-02", Event{/* ... */} });
```

### Alternative to multimap

```cpp
// Instead of multimap, you can use map with vector:
std::map<std::string, std::vector<int>> grades_alt;
grades_alt["Alice"].push_back(85);
grades_alt["Alice"].push_back(92);
grades_alt["Alice"].push_back(88);

// Pros: Easy access to all values, can use []
// Cons: Extra indirection, values not automatically sorted
```

---

## Performance Comparison

### Time Complexity Table
```
Operation       | set/map       | multiset/multimap
----------------|---------------|------------------
Search          | O(log n)      | O(log n)
Insert          | O(log n)      | O(log n)
Delete          | O(log n)      | O(log n + k)*
Access by key   | O(log n)      | O(log n)
Range query     | O(log n + k)† | O(log n + k)†
Iteration       | O(n)          | O(n)

* k = number of duplicates
† k = size of range
```

### Memory Overhead
```
Each node typically contains:
- Element/pair (data)
- Left child pointer
- Right child pointer
- Parent pointer (implementation-dependent)
- Color bit (Red-Black tree)

Memory per node ≈ 24-32 bytes + sizeof(element)

Example:
set<int>: ~28 bytes per element (vs ~4 for vector)
map<int, int>: ~32 bytes per pair (vs ~8 for vector of pairs)
```

### Associative vs Unordered Containers
```
                Associative         Unordered
                (Tree-based)        (Hash-based)
----------------------------------------------------------------
Search          O(log n)            O(1) average
Insert          O(log n)            O(1) average
Delete          O(log n)            O(1) average
Worst case      O(log n)            O(n)
Ordered         Yes                 No
Range queries   Efficient           Not supported
Memory          Higher              Moderate
```

---

## Advanced Topics

### 1. Custom Allocators

```cpp
#include <set>
#include <memory_resource>   // std::pmr facilities live here

// A set whose nodes are allocated through a PMR memory resource.
// (std::pmr::set is a ready-made alias for exactly this.)
template<typename T>
using MySet = std::set<T, std::less<T>,
                       std::pmr::polymorphic_allocator<T>>;

int main() {
    std::pmr::monotonic_buffer_resource pool;
    // Pass the resource via the allocator; brace-init is the clearest form.
    MySet<int> s{std::pmr::polymorphic_allocator<int>{&pool}};
    s.insert(42);
    return 0;
}
```

### 2. Transparent Comparators (C++14)

```cpp
#include <set>
#include <string>

struct TransparentCompare {
    using is_transparent = void;  // Enable transparent comparison
    
    bool operator()(const std::string& a, const std::string& b) const {
        return a < b;
    }
    
    bool operator()(const std::string& a, const char* b) const {
        return a < b;
    }
    
    bool operator()(const char* a, const std::string& b) const {
        return a < b;
    }
};

int main() {
    std::set<std::string, TransparentCompare> words = {"hello", "world"};
    
    // Without transparent comparator: creates temporary string
    // With transparent comparator: no temporary!
    auto it = words.find("hello");  // Uses const char* directly!
    
    return 0;
}
```

### 3. Extract and Merge (C++17)

```cpp
std::set<int> s1 = {1, 2, 3};
std::set<int> s2 = {3, 4, 5};

// Extract node
auto node = s1.extract(2);
node.value() = 10;
s2.insert(std::move(node));
// s1 = {1, 3}, s2 = {3, 4, 5, 10}

// Merge sets
s2.merge(s1);  // Move unique elements from s1 to s2
// s1 = {3} (couldn't move - already in s2)
// s2 = {1, 3, 4, 5, 10}
```

---

## Common Patterns and Idioms

### Pattern 1: Counting Occurrences
```cpp
std::vector<int> data = {1, 2, 2, 3, 3, 3, 4, 4, 4, 4};
std::map<int, int> frequency;

for (int x : data) {
    frequency[x]++;  // operator[] creates with 0 if doesn't exist
}
// frequency = { {1, 1}, {2, 2}, {3, 3}, {4, 4} }
```

### Pattern 2: Unique Elements with Order
```cpp
std::vector<int> data = {5, 2, 8, 2, 1, 5, 9};
std::set<int> unique(data.begin(), data.end());
// unique = {1, 2, 5, 8, 9}

// Convert back to vector if needed
std::vector<int> sorted_unique(unique.begin(), unique.end());
```

### Pattern 3: Finding Top K Elements
```cpp
std::vector<int> data = {5, 2, 8, 1, 9, 3, 7};
std::multiset<int, std::greater<int>> top_k;  // Descending

for (int x : data) {
    top_k.insert(x);
    if (top_k.size() > 3) {
        top_k.erase(--top_k.end());  // Remove smallest
    }
}
// top_k = {9, 8, 7}
```

### Pattern 4: Interval Scheduling
```cpp
std::map<int, int> intervals;  // {start, end}

bool can_schedule(int start, int end) {
    auto next = intervals.lower_bound(start);
    
    // Check overlap with next interval
    if (next != intervals.end() && next->first < end) {
        return false;
    }
    
    // Check overlap with previous interval
    if (next != intervals.begin()) {
        auto prev = std::prev(next);
        if (prev->second > start) {
            return false;
        }
    }
    
    intervals[start] = end;
    return true;
}
```

---

## Practice Exercises

### Exercise 1: Word Frequency
```cpp
// Count word frequency in a text, print in sorted order
std::string text = "the quick brown fox jumps over the lazy dog the fox";
// Expected output:
// fox: 2
// the: 3
// ... (sorted by word)
```

### Exercise 2: Leaderboard
```cpp
// Implement a game leaderboard using multimap
// Support: add score, get top N scores, get player scores
```

### Exercise 3: Range Sum
```cpp
// Using map, implement efficient range sum queries
// Given operations: insert(key, value), sum(low, high)
```

---

## Common Pitfalls

### 1. Using operator[] on const map
```cpp
void print_age(const std::map<std::string, int>& ages) {
    // ERROR: operator[] is non-const (can modify map)
    // std::cout << ages["Alice"] << '\n';
    
    // CORRECT: Use at() or find()
    std::cout << ages.at("Alice") << '\n';
}
```

### 2. Modifying keys
```cpp
std::set<int> s = {1, 2, 3};
auto it = s.begin();
// *it = 10;  // ERROR: elements are const!

std::map<int, int> m = { {1, 10}, {2, 20} };
auto mit = m.begin();
// mit->first = 5;   // ERROR: key is const!
mit->second = 99;    // OK: value is mutable
```

### 3. Iterator Invalidation
```cpp
std::set<int> s = {1, 2, 3, 4, 5};
for (auto it = s.begin(); it != s.end(); ++it) {
    if (*it % 2 == 0) {
        s.erase(it);  // Invalidates it!
        // ++it will crash
    }
}

// CORRECT:
for (auto it = s.begin(); it != s.end(); ) {
    if (*it % 2 == 0) {
        it = s.erase(it);  // erase returns next valid iterator
    } else {
        ++it;
    }
}
```

### 4. Comparison Function Requirements
```cpp
// BAD: Inconsistent comparison
struct BadCompare {
    bool operator()(int a, int b) const {
        return true;  // Always true - violates strict weak ordering!
    }
};
// std::set<int, BadCompare> s;  // Undefined behavior!

// GOOD: Must satisfy strict weak ordering
// - Irreflexive: comp(a, a) = false
// - Asymmetric: if comp(a, b) then !comp(b, a)
// - Transitive: if comp(a, b) and comp(b, c) then comp(a, c)
```

---

## Complete Practical Example: Student Grade Management System

Here's a comprehensive example integrating `set`, `multiset`, `map`, and `multimap`:

```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>
#include <vector>
#include <iomanip>
#include <algorithm>   // std::find_if, std::count_if

struct Student {
    int id;
    std::string name;
    int year;  // 1-4
    
    bool operator<(const Student& other) const {
        return id < other.id;
    }
};

class GradeBook {
private:
    // set: Unique students by ID
    std::set<Student> students_;
    
    // multimap: Student ID -> Course grades (one student, many courses)
    std::multimap<int, std::pair<std::string, double>> student_grades_;
    
    // map: Course name -> Set of enrolled student IDs
    std::map<std::string, std::set<int>> course_enrollment_;
    
    // multiset: All grades (for statistics, allows duplicates)
    std::multiset<double, std::greater<double>> all_grades_sorted_;
    
    // map: Student ID -> GPA
    std::map<int, double> student_gpa_;
    
public:
    // Add new student (set ensures uniqueness by ID)
    bool add_student(int id, const std::string& name, int year) {
        auto [it, inserted] = students_.insert({id, name, year});
        
        if (inserted) {
            std::cout << "Added student: " << name << " (ID: " << id << ")\n";
            return true;
        } else {
            std::cout << "Student with ID " << id << " already exists\n";
            return false;
        }
    }
    
    // Enroll student in course
    void enroll_in_course(int student_id, const std::string& course) {
        // Check if student exists using set::find
        auto student_it = std::find_if(students_.begin(), students_.end(),
            [student_id](const Student& s) { return s.id == student_id; });
        
        if (student_it == students_.end()) {
            std::cout << "Student ID " << student_id << " not found\n";
            return;
        }
        
        // Add to course enrollment map
        course_enrollment_[course].insert(student_id);
        std::cout << "Enrolled student " << student_id 
                  << " in " << course << "\n";
    }
    
    // Record grade (multimap allows multiple grades per student)
    void record_grade(int student_id, const std::string& course, double grade) {
        // Add to student grades multimap
        student_grades_.insert({student_id, {course, grade} });
        
        // Add to sorted grades multiset
        all_grades_sorted_.insert(grade);
        
        std::cout << "Recorded grade " << grade << " for student " 
                  << student_id << " in " << course << "\n";
        
        // Update GPA
        update_gpa(student_id);
    }
    
    // Calculate and update GPA
    void update_gpa(int student_id) {
        // Find all grades for this student using multimap::equal_range
        auto range = student_grades_.equal_range(student_id);
        
        if (range.first == range.second) {
            student_gpa_.erase(student_id);
            return;
        }
        
        double total = 0.0;
        int count = 0;
        
        for (auto it = range.first; it != range.second; ++it) {
            total += it->second.second;  // grade
            count++;
        }
        
        student_gpa_[student_id] = total / count;
    }
    
    // Get student's transcript (all grades)
    void print_transcript(int student_id) const {
        // Find student info
        auto student_it = std::find_if(students_.begin(), students_.end(),
            [student_id](const Student& s) { return s.id == student_id; });
        
        if (student_it == students_.end()) {
            std::cout << "Student not found\n";
            return;
        }
        
        std::cout << "\n=== Transcript for " << student_it->name 
                  << " (ID: " << student_id << ") ===\n";
        
        // Get all grades using multimap::equal_range
        auto range = student_grades_.equal_range(student_id);
        
        if (range.first == range.second) {
            std::cout << "No grades recorded\n";
            return;
        }
        
        std::cout << std::fixed << std::setprecision(2);
        for (auto it = range.first; it != range.second; ++it) {
            std::cout << "  " << std::setw(20) << std::left 
                      << it->second.first  // course
                      << ": " << it->second.second << "\n";  // grade
        }
        
        // Print GPA
        auto gpa_it = student_gpa_.find(student_id);
        if (gpa_it != student_gpa_.end()) {
            std::cout << "\nGPA: " << gpa_it->second << "\n";
        }
    }
    
    // Get students in a course (using map + set)
    void print_course_roster(const std::string& course) const {
        auto it = course_enrollment_.find(course);
        
        if (it == course_enrollment_.end() || it->second.empty()) {
            std::cout << "No students enrolled in " << course << "\n";
            return;
        }
        
        std::cout << "\n=== " << course << " Roster ===\n";
        
        // Iterate through set of student IDs
        for (int student_id : it->second) {
            auto student_it = std::find_if(students_.begin(), students_.end(),
                [student_id](const Student& s) { return s.id == student_id; });
            
            if (student_it != students_.end()) {
                std::cout << "  " << student_it->name 
                          << " (ID: " << student_id << ")\n";
            }
        }
    }
    
    // Dean's list: Students with GPA >= 3.5 (using map and algorithms)
    void print_deans_list() const {
        std::cout << "\n=== Dean's List (GPA >= 3.5) ===\n";
        
        // Use map to find high-GPA students
        for (const auto& [student_id, gpa] : student_gpa_) {
            if (gpa >= 3.5) {
                auto student_it = std::find_if(students_.begin(), 
                                              students_.end(),
                    [student_id](const Student& s) { 
                        return s.id == student_id; 
                    });
                
                if (student_it != students_.end()) {
                    std::cout << "  " << student_it->name 
                              << " (GPA: " << std::fixed 
                              << std::setprecision(2) << gpa << ")\n";
                }
            }
        }
    }
    
    // Grade distribution using multiset
    void print_grade_distribution() const {
        if (all_grades_sorted_.empty()) {
            std::cout << "No grades recorded\n";
            return;
        }
        
        std::cout << "\n=== Grade Distribution ===\n";
        
        // Count grades in ranges using multiset::count
        int a_count = std::count_if(all_grades_sorted_.begin(), 
                                    all_grades_sorted_.end(),
            [](double g) { return g >= 90; });
        
        int b_count = std::count_if(all_grades_sorted_.begin(), 
                                    all_grades_sorted_.end(),
            [](double g) { return g >= 80 && g < 90; });
        
        int c_count = std::count_if(all_grades_sorted_.begin(), 
                                    all_grades_sorted_.end(),
            [](double g) { return g >= 70 && g < 80; });
        
        int d_count = std::count_if(all_grades_sorted_.begin(), 
                                    all_grades_sorted_.end(),
            [](double g) { return g >= 60 && g < 70; });
        
        int f_count = std::count_if(all_grades_sorted_.begin(), 
                                    all_grades_sorted_.end(),
            [](double g) { return g < 60; });
        
        std::cout << "A (90-100): " << a_count << "\n";
        std::cout << "B (80-89):  " << b_count << "\n";
        std::cout << "C (70-79):  " << c_count << "\n";
        std::cout << "D (60-69):  " << d_count << "\n";
        std::cout << "F (0-59):   " << f_count << "\n";
        
        // Top 5 grades (multiset is already sorted)
        std::cout << "\nTop 5 grades: ";
        auto it = all_grades_sorted_.begin();
        for (int i = 0; i < 5 && it != all_grades_sorted_.end(); ++i, ++it) {
            std::cout << *it << " ";
        }
        std::cout << "\n";
    }
    
    // Summary statistics
    void print_summary() const {
        std::cout << "\n=== System Summary ===\n";
        std::cout << "Total students: " << students_.size() << "\n";
        std::cout << "Total courses: " << course_enrollment_.size() << "\n";
        std::cout << "Total grades recorded: " << all_grades_sorted_.size() << "\n";
        
        if (!all_grades_sorted_.empty()) {
            double sum = 0;
            for (double grade : all_grades_sorted_) {
                sum += grade;
            }
            std::cout << "Average grade: " << std::fixed 
                      << std::setprecision(2) << (sum / all_grades_sorted_.size()) 
                      << "\n";
        }
    }
};

int main() {
    GradeBook gradebook;
    
    // Add students (set)
    gradebook.add_student(101, "Alice Johnson", 2);
    gradebook.add_student(102, "Bob Smith", 3);
    gradebook.add_student(103, "Carol White", 2);
    gradebook.add_student(104, "David Brown", 4);
    gradebook.add_student(101, "Duplicate Alice", 1);  // Rejected (duplicate ID)
    
    // Enroll students in courses (map + set)
    gradebook.enroll_in_course(101, "CS101");
    gradebook.enroll_in_course(101, "MATH201");
    gradebook.enroll_in_course(102, "CS101");
    gradebook.enroll_in_course(102, "PHYS101");
    gradebook.enroll_in_course(103, "CS101");
    gradebook.enroll_in_course(103, "MATH201");
    gradebook.enroll_in_course(104, "CS101");
    
    // Record grades (multimap + multiset)
    gradebook.record_grade(101, "CS101", 95.0);
    gradebook.record_grade(101, "MATH201", 92.0);
    gradebook.record_grade(102, "CS101", 88.0);
    gradebook.record_grade(102, "PHYS101", 85.0);
    gradebook.record_grade(103, "CS101", 78.0);
    gradebook.record_grade(103, "MATH201", 82.0);
    gradebook.record_grade(104, "CS101", 91.0);
    
    // Print transcripts
    gradebook.print_transcript(101);
    gradebook.print_transcript(102);
    
    // Print course roster
    gradebook.print_course_roster("CS101");
    
    // Print Dean's list
    gradebook.print_deans_list();
    
    // Print grade distribution
    gradebook.print_grade_distribution();
    
    // Print summary
    gradebook.print_summary();
    
    return 0;
}
```

### Concepts Demonstrated:
- **std::set**: Unique students, automatic sorting by ID
- **std::multimap**: Multiple grades per student
- **std::map**: Course enrollment tracking
- **std::multiset**: Sorted grades with duplicates
- **equal_range()**: Finding all entries with same key
- **find()**: Searching for specific elements
- **insert()**: Adding elements with uniqueness checking
- **Ordered iteration**: Automatic sorting
- **Custom comparators**: Student comparison by ID

This example shows real-world use of all associative containers!

---

## Related Topics
- [Unordered Containers](03_unordered_containers.md) — the hash-based counterparts; use them when you don't need ordering.
- [Iterators](05_iterators.md) — these containers expose *bidirectional* iterators and the erase-while-iterating idiom shown above.
- [Algorithms](06_algorithms.md) — `lower_bound`/`upper_bound`/`equal_range` also exist as free algorithms; prefer the *member* versions on these containers (O(log n) vs O(n)).
- [Lambdas](10_lambdas.md) — custom comparators are usually written as lambdas or function objects.
- [Memory Management & Allocators](19_memory_allocators.md) — the PMR allocator used in "Advanced Topics" is covered in depth here.
- [Utility Containers](08_utility_containers.md) — `std::pair`/`std::tuple` underpin `map`/`multimap` values.
- [Quick Reference](99_quick_reference.md) — complexity table and container decision tree.

## Next Steps
- **Next**: [Unordered Containers →](03_unordered_containers.md)
- **Previous**: [← Sequence Containers](01_sequence_containers.md)

---
*Chapter 2 — Associative Containers*

