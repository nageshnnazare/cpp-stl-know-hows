# Container Adaptors

## Overview

Container adaptors are not full container classes, but rather wrappers around existing [sequence containers](01_sequence_containers.md) (`deque`, `vector`, `list`). They provide a restricted interface by delegating to an underlying container.

⚠️ **Gotcha**: Adaptors expose **no iterators** — no `begin()`/`end()`, no range-`for`, and no direct use with STL algorithms. To traverse, pop elements into a temporary structure or use the underlying container directly. See [Iterators](05_iterators.md) for how algorithms expect iterator ranges.

```
┌──────────────────────────────────────────────────────────┐
│            CONTAINER ADAPTORS FAMILY                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  std::stack          LIFO (Last In, First Out)           │
│  std::queue          FIFO (First In, First Out)          │
│  std::priority_queue Max-heap (by default)               │
│                                                          │
│  All wrap underlying containers (deque, vector, etc.)    │
│  Provide restricted interface for specific use cases     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Adaptor vs Container
```
Regular Container:
- Full interface (begin, end, insert, erase, etc.)
- Access anywhere
- Multiple ways to use

Container Adaptor:
- Restricted interface (push, pop, top/front)
- Access only at specific positions
- No iterators (cannot range-for or pass to algorithms)
- Single, well-defined purpose

Default underlying containers ([cppreference](https://en.cppreference.com/w/cpp/container)):
- std::stack  → std::deque
- std::queue  → std::deque
- std::priority_queue → std::vector + std::less (max-heap; largest on top)
```

---

## 1. std::stack

### Description
LIFO (Last In, First Out) data structure. Elements are added and removed from the same end (top).

### Visual Representation
```
Stack Operations:

push(1):     push(2):     push(3):     pop():       pop():
┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
│     │      │     │      │  3  │ ← top│     │      │     │
├─────┤      ├─────┤      ├─────┤      ├─────┤      ├─────┤
│     │      │  2  │ ← top│  2  │      │  2  │ ← top│     │
├─────┤      ├─────┤      ├─────┤      ├─────┤      ├─────┤
│  1  │ ← top│  1  │      │  1  │      │  1  │      │  1  │ ← top
└─────┘      └─────┘      └─────┘      └─────┘      └─────┘

LIFO: Last element pushed is first element popped
```

### Internal Structure
```
Stack is typically implemented using deque:

std::stack<int> s;

Internal deque:
┌──────────────────────────────────┐
│ [1] [2] [3] [4] [5]              │ ← back = top
└──────────────────────────────────┘
  ▲
front

Operations:
- push() → deque.push_back()
- pop()  → deque.pop_back()
- top()  → deque.back()
```

### Basic Operations

```cpp
#include <stack>
#include <iostream>

int main() {
    // Construction (default uses std::deque)
    std::stack<int> s1;
    
    // Specify underlying container
    std::stack<int, std::vector<int>> s2;     // Using vector
    std::stack<int, std::deque<int>> s3;      // Using deque (default)
    // Can also use std::list
    
    std::stack<int> s;
    
    // Push elements - O(1)
    s.push(10);
    s.push(20);
    s.push(30);
    s.push(40);
    // Stack: [10, 20, 30, 40] (40 is top)
    
    // C++11: emplace (construct in-place)
    s.emplace(50);
    // Stack: [10, 20, 30, 40, 50]
    
    // Access top element - O(1)
    int top = s.top();
    std::cout << "Top: " << top << '\n';  // 50
    
    // Modify top
    s.top() = 99;
    std::cout << "New top: " << s.top() << '\n';  // 99
    
    // Pop (remove top) - O(1)
    s.pop();  // Removes 99
    std::cout << "After pop, top: " << s.top() << '\n';  // 40
    
    // Size
    std::cout << "Size: " << s.size() << '\n';  // 4
    
    // Check if empty
    if (!s.empty()) {
        std::cout << "Stack is not empty\n";
    }
    
    // Pop all elements
    while (!s.empty()) {
        std::cout << s.top() << ' ';
        s.pop();
    }
    // Output: 40 30 20 10
    
    return 0;
}
```

### Stack Interface Summary
```cpp
Member Function    | Description              | Complexity
-------------------|--------------------------|------------
push(value)        | Add element to top       | O(1)
emplace(args...)   | Construct element on top | O(1)
pop()              | Remove top element       | O(1)
top()              | Access top element       | O(1)
empty()            | Check if empty           | O(1)
size()             | Number of elements       | O(1)
swap(other)        | Swap with another stack  | O(1)

Note: pop() returns void! Use top() before pop() to get value.
No iterators — cannot traverse; see 05_iterators.md.
```

### Choosing Underlying Container
```cpp
// Default (deque) - balanced performance
std::stack<int> s1;

// Vector - better cache locality, may reallocate
std::stack<int, std::vector<int>> s2;

// List - no reallocation, but worse cache performance
std::stack<int, std::list<int>> s3;

Performance comparison for typical use:
┌─────────────┬───────┬────────┬──────┐
│ Container   │ Push  │ Pop    │ Top  │
├─────────────┼───────┼────────┼──────┤
│ deque       │ Fast  │ Fast   │ Fast │
│ vector      │ Fast* │ Fast   │ Fast │
│ list        │ Fast  │ Fast   │ Fast │
└─────────────┴───────┴────────┴──────┘
* May occasionally reallocate
```

### Common Use Cases

#### Use Case 1: Expression Evaluation
```cpp
#include <stack>
#include <string>
#include <cctype>

// Evaluate postfix expression
// Example: "2 3 + 5 *" = (2 + 3) * 5 = 25
int evaluate_postfix(const std::string& expr) {
    std::stack<int> operands;
    
    for (char c : expr) {
        if (std::isdigit(c)) {
            operands.push(c - '0');
        } else if (c == '+' || c == '-' || c == '*' || c == '/') {
            int b = operands.top(); operands.pop();
            int a = operands.top(); operands.pop();
            
            switch (c) {
                case '+': operands.push(a + b); break;
                case '-': operands.push(a - b); break;
                case '*': operands.push(a * b); break;
                case '/': operands.push(a / b); break;
            }
        }
    }
    
    return operands.top();
}
```

#### Use Case 2: Balanced Parentheses
```cpp
#include <stack>
#include <string>

bool is_balanced(const std::string& s) {
    std::stack<char> brackets;
    
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            brackets.push(c);
        } else if (c == ')' || c == ']' || c == '}') {
            if (brackets.empty()) return false;
            
            char top = brackets.top();
            brackets.pop();
            
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) {
                return false;
            }
        }
    }
    
    return brackets.empty();
}

// Usage:
// is_balanced("()[]{}") → true
// is_balanced("([)]")   → false
```

#### Use Case 3: Undo/Redo Functionality
```cpp
#include <stack>
#include <string>

class TextEditor {
    std::string text;
    std::stack<std::string> undo_stack;
    std::stack<std::string> redo_stack;
    
public:
    void type(const std::string& str) {
        undo_stack.push(text);
        redo_stack = std::stack<std::string>();  // Clear redo on new action
        text += str;
    }
    
    void undo() {
        if (!undo_stack.empty()) {
            redo_stack.push(text);
            text = undo_stack.top();
            undo_stack.pop();
        }
    }
    
    void redo() {
        if (!redo_stack.empty()) {
            undo_stack.push(text);
            text = redo_stack.top();
            redo_stack.pop();
        }
    }
    
    std::string get_text() const { return text; }
};
```

#### Use Case 4: Depth-First Search (DFS)
```cpp
#include <stack>
#include <vector>
#include <unordered_set>  // see 03_unordered_containers.md
#include <iostream>

void dfs_iterative(int start, const std::vector<std::vector<int>>& graph) {
    std::stack<int> to_visit;
    std::unordered_set<int> visited;
    
    to_visit.push(start);
    
    while (!to_visit.empty()) {
        int node = to_visit.top();
        to_visit.pop();
        
        if (visited.contains(node)) continue;
        
        visited.insert(node);
        std::cout << "Visiting: " << node << '\n';
        
        // Add neighbors to stack
        for (int neighbor : graph[node]) {
            if (!visited.contains(neighbor)) {
                to_visit.push(neighbor);
            }
        }
    }
}
```

---

## 2. std::queue

### Description
FIFO (First In, First Out) data structure. Elements are added at the back and removed from the front.

### Visual Representation
```
Queue Operations:

push(1):           push(2):           push(3):
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ [1]         │    │ [1] [2]     │    │ [1] [2] [3] │
└─────────────┘    └─────────────┘    └─────────────┘
  ▲                  ▲                  ▲         ▲
front              front              front     back

pop():             pop():
┌─────────────┐    ┌─────────────┐
│     [2] [3] │    │         [3] │
└─────────────┘    └─────────────┘
      ▲    ▲             ▲      ▲
    front back         front  back

FIFO: First element pushed is first element popped
```

### Internal Structure
```
Queue is typically implemented using deque:

std::queue<int> q;

Internal deque:
┌──────────────────────────────────┐
│ [1] [2] [3] [4] [5]              │
└──────────────────────────────────┘
  ▲                 ▲
front             back

Operations:
- push()  → deque.push_back()
- pop()   → deque.pop_front()
- front() → deque.front()
- back()  → deque.back()
```

### Basic Operations

```cpp
#include <queue>
#include <iostream>

int main() {
    // Construction
    std::queue<int> q1;
    std::queue<int, std::deque<int>> q2;  // Default
    std::queue<int, std::list<int>> q3;   // Using list
    // Cannot use vector (no pop_front)!
    
    std::queue<int> q;
    
    // Push elements (add to back) - O(1)
    q.push(10);
    q.push(20);
    q.push(30);
    q.push(40);
    // Queue: [10, 20, 30, 40]
    //         ▲           ▲
    //       front       back
    
    // Emplace (C++11)
    q.emplace(50);
    
    // Access front - O(1)
    std::cout << "Front: " << q.front() << '\n';  // 10
    
    // Access back - O(1)
    std::cout << "Back: " << q.back() << '\n';    // 50
    
    // Modify front/back
    q.front() = 99;
    q.back() = 88;
    
    // Pop (remove front) - O(1)
    q.pop();  // Removes 99 (was 10)
    std::cout << "After pop, front: " << q.front() << '\n';  // 20
    
    // Size
    std::cout << "Size: " << q.size() << '\n';
    
    // Process all elements (FIFO order)
    while (!q.empty()) {
        std::cout << q.front() << ' ';
        q.pop();
    }
    // Output: 20 30 40 88
    
    return 0;
}
```

### Queue Interface Summary
```cpp
Member Function    | Description              | Complexity
-------------------|--------------------------|------------
push(value)        | Add element to back      | O(1)
emplace(args...)   | Construct element at back| O(1)
pop()              | Remove front element     | O(1)
front()            | Access front element     | O(1)
back()             | Access back element      | O(1)
empty()            | Check if empty           | O(1)
size()             | Number of elements       | O(1)
swap(other)        | Swap with another queue  | O(1)
```

### Common Use Cases

#### Use Case 1: Breadth-First Search (BFS)
```cpp
#include <queue>
#include <vector>
#include <unordered_set>

void bfs(int start, const std::vector<std::vector<int>>& graph) {
    std::queue<int> to_visit;
    std::unordered_set<int> visited;
    
    to_visit.push(start);
    visited.insert(start);
    
    while (!to_visit.empty()) {
        int node = to_visit.front();
        to_visit.pop();
        
        std::cout << "Visiting: " << node << '\n';
        
        // Add unvisited neighbors
        for (int neighbor : graph[node]) {
            if (!visited.contains(neighbor)) {
                visited.insert(neighbor);
                to_visit.push(neighbor);
            }
        }
    }
}
```

#### Use Case 2: Level-Order Tree Traversal
```cpp
#include <queue>

struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

void level_order_traversal(TreeNode* root) {
    if (!root) return;
    
    std::queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int level_size = q.size();
        
        // Process entire level
        for (int i = 0; i < level_size; ++i) {
            TreeNode* node = q.front();
            q.pop();
            
            std::cout << node->val << ' ';
            
            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }
        std::cout << '\n';  // New level
    }
}
```

#### Use Case 3: Task Scheduling
```cpp
#include <queue>
#include <string>

struct Task {
    std::string name;
    int priority;
};

class TaskScheduler {
    std::queue<Task> tasks;
    
public:
    void add_task(const std::string& name, int priority) {
        tasks.push({name, priority});
    }
    
    void process_tasks() {
        while (!tasks.empty()) {
            Task task = tasks.front();
            tasks.pop();
            
            std::cout << "Processing: " << task.name << '\n';
            // Process task...
        }
    }
};
```

#### Use Case 4: Producer-Consumer Pattern

Thread-safe queue built on `std::queue`; synchronization primitives are covered in [Multithreading](14_multithreading.md).

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>

template<typename T>
class ThreadSafeQueue {
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cond_var_;
    
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(std::move(value));
        cond_var_.notify_one();
    }
    
    T pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_var_.wait(lock, [this] { return !queue_.empty(); });
        
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
};
```

---

## 3. std::priority_queue

### Description
A heap-based data structure that always keeps the extremal element at the top. The default is a **max-heap**: `std::priority_queue<T>` is `std::priority_queue<T, std::vector<T>, std::less<T>>`, so `top()` returns the **largest** element. Pass `std::greater<T>` as the third template argument for a min-heap.

### Visual Representation
```
Max-Heap (default priority_queue):

                    ┌────┐
                    │ 50 │ ← top (largest)
                    └──┬─┘
              ┌────────┴────────┐
              ▼                 ▼
            ┌────┐            ┌────┐
            │ 30 │            │ 40 │
            └──┬─┘            └──┬─┘
          ┌────┴────┐        ┌───┴───┐
          ▼         ▼        ▼       ▼
        ┌────┐   ┌────┐   ┌────┐  ┌────┐
        │ 10 │   │ 20 │   │ 15 │  │ 25 │
        └────┘   └────┘   └────┘  └────┘

Properties:
- Parent ≥ Children (max-heap)
- Complete binary tree
- Stored as array: [50, 30, 40, 10, 20, 15, 25]
```

### Internal Structure
```
priority_queue uses vector by default:

Array representation of heap:
Index:  0   1   2   3   4   5   6
Value: [50][30][40][10][20][15][25]
        ▲
       top

Parent of i: (i-1)/2
Left child:  2*i+1
Right child: 2*i+2

Max-heap property: parent ≥ children
```

### Basic Operations

```cpp
#include <queue>
#include <vector>
#include <iostream>

int main() {
    // Default: max-heap — vector + std::less<T>
    std::priority_queue<int> pq1;  // same as below
    std::priority_queue<int, std::vector<int>, std::less<int>> pq_max;
    
    // Underlying container must support random-access iterators (vector or deque)
    std::priority_queue<int, std::deque<int>> pq_deque;
    
    // Min-heap: std::greater puts smaller values at top()
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_pq;
    
    // Max-heap example
    std::priority_queue<int> pq;
    
    // Push elements - O(log n)
    pq.push(30);
    pq.push(10);
    pq.push(50);
    pq.push(20);
    pq.push(40);
    
    // Top is always the largest - O(1)
    std::cout << "Top: " << pq.top() << '\n';  // 50
    
    // Pop (remove largest) - O(log n)
    pq.pop();
    std::cout << "After pop: " << pq.top() << '\n';  // 40
    
    // Emplace (C++11)
    pq.emplace(45);
    
    // Size
    std::cout << "Size: " << pq.size() << '\n';
    
    // Process all elements (sorted order)
    while (!pq.empty()) {
        std::cout << pq.top() << ' ';
        pq.pop();
    }
    // Output: 45 40 30 20 10
    
    return 0;
}
```

### Min-Heap vs Max-Heap

The third template parameter is a **Compare** functor: `comp(a, b)` returns `true` if `a` is ordered **after** `b` (i.e., lower priority, deeper in the heap). `std::less` makes larger values float to `top()`; `std::greater` does the opposite.

```cpp
#include <queue>
#include <vector>
#include <functional>  // std::less, std::greater

// Max-heap (default) — std::less → largest on top
std::priority_queue<int> max_heap;
max_heap.push(3);
max_heap.push(1);
max_heap.push(5);
max_heap.push(2);
// max_heap.top() = 5

// Min-heap - smallest element on top
std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
min_heap.push(3);
min_heap.push(1);
min_heap.push(5);
min_heap.push(2);
// min_heap.top() = 1
```

### Visual Comparison
```
Same elements {1, 2, 3, 5}:

Max-Heap:                Min-Heap:
      5                        1
     / \                      / \
    3   2                    2   3
   /                        /
  1                        5

Output order:            Output order:
5, 3, 2, 1              1, 2, 3, 5
```

### Custom Comparator

```cpp
#include <queue>
#include <string>

struct Person {
    std::string name;
    int age;
};

// Method 1: Comparator struct
struct CompareAge {
    bool operator()(const Person& a, const Person& b) const {
        return a.age < b.age;  // Max-heap by age
    }
};

// Method 2: Lambda (requires decltype)
int main() {
    // Using struct comparator
    std::priority_queue<Person, std::vector<Person>, CompareAge> pq1;
    pq1.push({"Alice", 25});
    pq1.push({"Bob", 30});
    pq1.push({"Charlie", 20});
    // pq1.top() is Bob (30, oldest)
    
    // Using lambda
    auto comp = [](const Person& a, const Person& b) {
        return a.age > b.age;  // Min-heap by age
    };
    std::priority_queue<Person, std::vector<Person>, decltype(comp)> pq2(comp);
    pq2.push({"Alice", 25});
    pq2.push({"Bob", 30});
    pq2.push({"Charlie", 20});
    // pq2.top() is Charlie (20, youngest)
    
    return 0;
}
```

### Priority Queue Interface Summary
```cpp
Member Function    | Description              | Complexity
-------------------|--------------------------|------------
push(value)        | Add element              | O(log n)
emplace(args...)   | Construct element        | O(log n)
pop()              | Remove top element       | O(log n)
top()              | Access top element       | O(1)
empty()            | Check if empty           | O(1)
size()             | Number of elements       | O(1)
swap(other)        | Swap with another pq     | O(1)

Note: No iterators! Cannot iterate through elements.
```

### Heap Operations Visualization
```
Insert 45 into max-heap [50, 30, 40, 10, 20, 15, 25]:

Step 1: Add to end
[50, 30, 40, 10, 20, 15, 25, 45]
                            ▲
                         new element

Step 2: Bubble up (compare with parent)
        50                      50
       /  \                    /  \
      30   40                 45   40
     / \   / \               / \   / \
    10 20 15 25            10 20 15 25
           \                     \
           45                    30

Final: [50, 45, 40, 30, 20, 15, 25, 10]

Remove top (50):

Step 1: Replace top with last element
[10, 45, 40, 30, 20, 15, 25]
 ▲
last element moved to top

Step 2: Bubble down (compare with larger child)
        45                     
       /  \                   
      30   40                 
     / \   / \               
    10 20 15 25            

Final: [45, 30, 40, 10, 20, 15, 25]
```

### Common Use Cases

#### Use Case 1: Top K Elements
```cpp
#include <queue>
#include <vector>

std::vector<int> top_k_elements(const std::vector<int>& nums, int k) {
    // Min-heap of size k
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
    
    for (int num : nums) {
        min_heap.push(num);
        if (min_heap.size() > k) {
            min_heap.pop();  // Remove smallest
        }
    }
    
    // Extract k largest elements
    std::vector<int> result;
    while (!min_heap.empty()) {
        result.push_back(min_heap.top());
        min_heap.pop();
    }
    
    return result;
}

// Example: top_k_elements({3,2,1,5,6,4}, 2) → {5, 6}
```

#### Use Case 2: Merge K Sorted Arrays
```cpp
#include <queue>
#include <vector>

struct Element {
    int value;
    int array_idx;
    int element_idx;
    
    bool operator>(const Element& other) const {
        return value > other.value;
    }
};

std::vector<int> merge_k_arrays(const std::vector<std::vector<int>>& arrays) {
    std::priority_queue<Element, std::vector<Element>, std::greater<Element>> min_heap;
    
    // Add first element from each array
    for (int i = 0; i < arrays.size(); ++i) {
        if (!arrays[i].empty()) {
            min_heap.push({arrays[i][0], i, 0});
        }
    }
    
    std::vector<int> result;
    
    while (!min_heap.empty()) {
        Element elem = min_heap.top();
        min_heap.pop();
        
        result.push_back(elem.value);
        
        // Add next element from same array
        if (elem.element_idx + 1 < arrays[elem.array_idx].size()) {
            min_heap.push({
                arrays[elem.array_idx][elem.element_idx + 1],
                elem.array_idx,
                elem.element_idx + 1
            });
        }
    }
    
    return result;
}
```

#### Use Case 3: Dijkstra's Algorithm
```cpp
#include <queue>
#include <vector>
#include <limits>

// Classic graph algorithm; priority_queue drives the "open set"
struct Edge {
    int to;
    int weight;
    
    bool operator>(const Edge& other) const {
        return weight > other.weight;
    }
};

std::vector<int> dijkstra(const std::vector<std::vector<Edge>>& graph, int start) {
    int n = graph.size();
    std::vector<int> dist(n, std::numeric_limits<int>::max());
    std::priority_queue<Edge, std::vector<Edge>, std::greater<Edge>> pq;
    
    dist[start] = 0;
    pq.push({start, 0});
    
    while (!pq.empty()) {
        Edge current = pq.top();
        pq.pop();
        
        if (current.weight > dist[current.to]) continue;
        
        for (const Edge& edge : graph[current.to]) {
            int new_dist = dist[current.to] + edge.weight;
            if (new_dist < dist[edge.to]) {
                dist[edge.to] = new_dist;
                pq.push({edge.to, new_dist});
            }
        }
    }
    
    return dist;
}
```

#### Use Case 4: Median from Data Stream
```cpp
#include <queue>

class MedianFinder {
    std::priority_queue<int> max_heap;  // Lower half
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;  // Upper half
    
public:
    void add_num(int num) {
        // Add to appropriate heap
        if (max_heap.empty() || num <= max_heap.top()) {
            max_heap.push(num);
        } else {
            min_heap.push(num);
        }
        
        // Balance heaps
        if (max_heap.size() > min_heap.size() + 1) {
            min_heap.push(max_heap.top());
            max_heap.pop();
        } else if (min_heap.size() > max_heap.size()) {
            max_heap.push(min_heap.top());
            min_heap.pop();
        }
    }
    
    double find_median() {
        if (max_heap.size() == min_heap.size()) {
            return (max_heap.top() + min_heap.top()) / 2.0;
        }
        return max_heap.top();
    }
};

// Visual representation:
// max_heap (lower half) | min_heap (upper half)
//        [5, 3, 1]      |      [7, 9, 11]
//         ▲             |       ▲
//       max (5)         |     min (7)
//                       |
// Median = (5 + 7) / 2 = 6
```

---

## Performance Comparison

### Time Complexity Table
```
Container    | Push   | Pop    | Top/Front | Back  | Random Access
-------------|--------|--------|-----------|-------|---------------
stack        | O(1)   | O(1)   | O(1)      | -     | No
queue        | O(1)   | O(1)   | O(1)      | O(1)  | No
priority_queue| O(log n)| O(log n)| O(1)   | -     | No

Note: All adaptors have O(1) size() and empty()
```

### Space Complexity
```
All container adaptors have O(n) space complexity
Plus overhead from underlying container:
- deque: chunked allocation
- vector: contiguous allocation (may have extra capacity)
- list: node-based (pointer overhead per element)
```

---

## Common Patterns and Idioms

### Pattern 1: Monotonic Queue (Deque, Not std::queue)

For sliding-window min/max, `std::deque` beats `std::queue` because you need `pop_front` **and** `pop_back`. See the sliding-window example below; for a full deque treatment see [Sequence Containers](01_sequence_containers.md).

### Pattern 2: Monotonic Stack
```cpp
#include <stack>
#include <vector>

// Find next greater element for each element
std::vector<int> next_greater_element(const std::vector<int>& nums) {
    int n = nums.size();
    std::vector<int> result(n, -1);
    std::stack<int> indices;  // Monotonic decreasing stack
    
    for (int i = 0; i < n; ++i) {
        while (!indices.empty() && nums[i] > nums[indices.top()]) {
            result[indices.top()] = nums[i];
            indices.pop();
        }
        indices.push(i);
    }
    
    return result;
}

// Example: [4, 5, 2, 10] → [5, 10, 10, -1]
```

### Pattern 3: Sliding Window Maximum
```cpp
#include <deque>
#include <vector>

std::vector<int> max_sliding_window(const std::vector<int>& nums, int k) {
    std::deque<int> dq;  // Stores indices
    std::vector<int> result;
    
    for (int i = 0; i < nums.size(); ++i) {
        // Remove elements outside window
        if (!dq.empty() && dq.front() <= i - k) {
            dq.pop_front();
        }
        
        // Remove smaller elements (not useful)
        while (!dq.empty() && nums[dq.back()] < nums[i]) {
            dq.pop_back();
        }
        
        dq.push_back(i);
        
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    
    return result;
}
```

---

## Common Pitfalls

### 1. pop() Returns void
```cpp
std::stack<int> s = {1, 2, 3};

// BAD: This doesn't compile!
// int value = s.pop();  // ERROR: pop() returns void

// GOOD: Get value before popping
int value = s.top();
s.pop();
```

### 2. No Iteration Support
```cpp
#include <stack>
#include <iostream>

std::stack<int> s;
s.push(1); s.push(2); s.push(3);

// BAD: adaptors have no begin()/end() — not even with std::begin(s)
// for (int x : s) { }           // ERROR
// std::find(s.begin(), s.end()) // ERROR

// WORKAROUND: copy and drain (destroys order if you need the original)
std::stack<int> temp = s;
while (!temp.empty()) {
    std::cout << temp.top() << ' ';
    temp.pop();
}
// Or store data in a vector/deque and use stack/queue only as the access policy
```

### 3. priority_queue Comparison Direction
```cpp
#include <queue>
#include <vector>
#include <functional>

// Confusing but correct: std::greater → MIN-heap (smallest on top)
std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;

// Default max-heap uses std::less — largest on top
std::priority_queue<int> max_heap;  // std::less<int> by default

// Compare(a,b) == true  →  a ordered after b (sinks in heap)
// std::less:  larger values sink  → max at top()
// std::greater: smaller values sink → min at top()
```

### 4. Using Wrong Underlying Container
```cpp
// BAD: vector doesn't have push_front!
// std::queue<int, std::vector<int>> q;  // ERROR

// GOOD: Use deque or list
std::queue<int, std::deque<int>> q1;   // OK
std::queue<int, std::list<int>> q2;    // OK
```

---

## Decision Guide

### Which Adaptor to Use?
```
Need LIFO (Last In, First Out)?
└─→ std::stack
    Use cases: undo/redo, expression evaluation, DFS

Need FIFO (First In, First Out)?
└─→ std::queue
    Use cases: BFS, task scheduling, buffering

Need priority-based access?
└─→ std::priority_queue
    Use cases: Dijkstra's, scheduling, top-K problems
```

### Stack vs Queue Decision Tree
```
Processing order = insertion order?
├─ Reverse order → stack
└─ Same order → queue

Example:
- Web browser back/forward → stack (LIFO)
- Print queue → queue (FIFO)
- Call stack (recursion) → stack (LIFO)
```

---

## Practice Exercises

### Exercise 1: Valid Parentheses
```cpp
// Implement function to check if parentheses are balanced
// "(())" → true, "(()" → false
```

### Exercise 2: Implement Min Stack
```cpp
// Design a stack that supports push, pop, top, and getting min in O(1)
class MinStack {
    // Your implementation
};
```

### Exercise 3: Reverse Polish Notation
```cpp
// Evaluate RPN expression: ["2", "1", "+", "3", "*"] → 9
// (2 + 1) * 3 = 9
```

### Exercise 4: Kth Largest Element
```cpp
// Find Kth largest element in array using priority_queue
// nums = [3,2,1,5,6,4], k = 2 → 5
```

---

## Complete Practical Example: Operating System Process Scheduler

Here's a comprehensive example integrating stack, queue, and priority_queue in a realistic process scheduling system:

```cpp
#include <iostream>
#include <stack>
#include <queue>
#include <string>
#include <vector>
#include <chrono>
#include <iomanip>

// Process structure
struct Process {
    int pid;
    std::string name;
    int priority;  // 0=highest, 9=lowest
    int burst_time;  // CPU time needed
    std::chrono::system_clock::time_point arrival_time;
    
    Process(int id, std::string n, int prio, int burst)
        : pid(id), name(std::move(n)), priority(prio), burst_time(burst)
        , arrival_time(std::chrono::system_clock::now()) {}
};

// Comparator for priority queue (lower priority number = higher priority)
struct ProcessComparator {
    bool operator()(const Process& a, const Process& b) const {
        if (a.priority != b.priority) {
            return a.priority > b.priority;  // Lower number = higher priority
        }
        return a.pid > b.pid;  // Earlier PID breaks ties
    }
};

// 1. Call Stack Simulator using std::stack
class CallStack {
private:
    struct StackFrame {
        std::string function_name;
        std::vector<std::string> local_vars;
        int line_number;
    };
    
    std::stack<StackFrame> frames_;
    
public:
    void push_frame(const std::string& func, int line) {
        frames_.push({func, {}, line});
        std::cout << "  [CALL] " << func << " at line " << line << "\n";
    }
    
    void pop_frame() {
        if (!frames_.empty()) {
            std::cout << "  [RETURN] " << frames_.top().function_name << "\n";
            frames_.pop();
        }
    }
    
    void add_local_var(const std::string& var) {
        if (!frames_.empty()) {
            frames_.top().local_vars.push_back(var);
        }
    }
    
    void print_stack_trace() const {
        std::cout << "\n=== Stack Trace ===" << (frames_.empty() ? " (empty)" : "") << "\n";
        
        auto temp = frames_;
        int depth = 0;
        
        while (!temp.empty()) {
            const auto& frame = temp.top();
            std::cout << "  #" << depth++ << " " << frame.function_name 
                      << " (line " << frame.line_number << ")\n";
            
            if (!frame.local_vars.empty()) {
                std::cout << "     Locals: ";
                for (const auto& var : frame.local_vars) {
                    std::cout << var << " ";
                }
                std::cout << "\n";
            }
            
            temp.pop();
        }
    }
    
    size_t depth() const { return frames_.size(); }
};

// 2. Ready Queue using std::queue (FIFO scheduling)
class ReadyQueue {
private:
    std::queue<Process> queue_;
    
public:
    void enqueue(const Process& process) {
        queue_.push(process);
        std::cout << "  [ENQUEUE] Process " << process.pid << ": " 
                  << process.name << "\n";
    }
    
    Process dequeue() {
        if (queue_.empty()) {
            throw std::runtime_error("Queue is empty");
        }
        
        Process p = queue_.front();
        queue_.pop();
        
        std::cout << "  [DEQUEUE] Process " << p.pid << ": " << p.name << "\n";
        return p;
    }
    
    bool empty() const { return queue_.empty(); }
    size_t size() const { return queue_.size(); }
    
    void print() const {
        std::cout << "\nReady Queue (FIFO): ";
        if (queue_.empty()) {
            std::cout << "(empty)\n";
            return;
        }
        
        auto temp = queue_;
        while (!temp.empty()) {
            std::cout << "P" << temp.front().pid << " ";
            temp.pop();
        }
        std::cout << "\n";
    }
};

// 3. Priority Scheduler using std::priority_queue
class PriorityScheduler {
private:
    std::priority_queue<Process, std::vector<Process>, ProcessComparator> pq_;
    std::vector<Process> completed_;
    
public:
    void add_process(const Process& process) {
        pq_.push(process);
        std::cout << "  [ADD] Process " << process.pid << ": " << process.name 
                  << " (priority " << process.priority << ")\n";
    }
    
    Process get_next() {
        if (pq_.empty()) {
            throw std::runtime_error("No processes in queue");
        }
        
        Process p = pq_.top();
        pq_.pop();
        
        std::cout << "  [SCHEDULE] Process " << p.pid << ": " << p.name 
                  << " (priority " << p.priority << ")\n";
        
        return p;
    }
    
    void complete_process(const Process& p) {
        completed_.push_back(p);
        std::cout << "  [COMPLETE] Process " << p.pid << ": " << p.name << "\n";
    }
    
    bool empty() const { return pq_.empty(); }
    size_t pending_count() const { return pq_.size(); }
    size_t completed_count() const { return completed_.size(); }
    
    void print_queue() const {
        std::cout << "\nPriority Queue (by priority): ";
        if (pq_.empty()) {
            std::cout << "(empty)\n";
            return;
        }
        
        // Copy to print (priority_queue doesn't have iterators)
        auto temp = pq_;
        while (!temp.empty()) {
            const auto& p = temp.top();
            std::cout << "P" << p.pid << "(pri:" << p.priority << ") ";
            temp.pop();
        }
        std::cout << "\n";
    }
    
    void print_completed() const {
        std::cout << "\nCompleted Processes:\n";
        for (const auto& p : completed_) {
            std::cout << "  P" << p.pid << ": " << p.name 
                      << " (burst: " << p.burst_time << "ms)\n";
        }
    }
};

// 4. Undo/Redo Manager using dual stacks
class UndoRedoManager {
private:
    std::stack<std::string> undo_stack_;
    std::stack<std::string> redo_stack_;
    
public:
    void execute_action(const std::string& action) {
        undo_stack_.push(action);
        
        // Clear redo stack on new action
        while (!redo_stack_.empty()) {
            redo_stack_.pop();
        }
        
        std::cout << "  [ACTION] " << action << "\n";
    }
    
    void undo() {
        if (undo_stack_.empty()) {
            std::cout << "  [UNDO] Nothing to undo\n";
            return;
        }
        
        std::string action = undo_stack_.top();
        undo_stack_.pop();
        redo_stack_.push(action);
        
        std::cout << "  [UNDO] " << action << "\n";
    }
    
    void redo() {
        if (redo_stack_.empty()) {
            std::cout << "  [REDO] Nothing to redo\n";
            return;
        }
        
        std::string action = redo_stack_.top();
        redo_stack_.pop();
        undo_stack_.push(action);
        
        std::cout << "  [REDO] " << action << "\n";
    }
    
    void print_status() const {
        std::cout << "\n=== Undo/Redo Status ===\n";
        std::cout << "Undo stack size: " << undo_stack_.size() << "\n";
        std::cout << "Redo stack size: " << redo_stack_.size() << "\n";
    }
};

// 5. BFS Queue for Process Tree Traversal
class ProcessTree {
private:
    struct TreeNode {
        Process process;
        std::vector<TreeNode*> children;
        
        ~TreeNode() {
            for (auto* child : children) {
                delete child;
            }
        }
    };
    
    TreeNode* root_;
    
public:
    ProcessTree() : root_(nullptr) {}
    
    ~ProcessTree() {
        delete root_;
    }
    
    void set_root(const Process& p) {
        root_ = new TreeNode{p, { } };
    }
    
    void add_child(int parent_pid, const Process& child) {
        // Simplified: only adds to root for demo
        if (root_ && root_->process.pid == parent_pid) {
            root_->children.push_back(new TreeNode{child, { } });
        }
    }
    
    void bfs_traversal() const {
        if (!root_) return;
        
        std::cout << "\n=== Process Tree (BFS) ===\n";
        std::queue<TreeNode*> q;
        q.push(root_);
        
        while (!q.empty()) {
            TreeNode* node = q.front();
            q.pop();
            
            std::cout << "  P" << node->process.pid << ": " 
                      << node->process.name << "\n";
            
            for (auto* child : node->children) {
                q.push(child);
            }
        }
    }
};

int main() {
    std::cout << "=== Operating System Process Scheduler ===\n\n";
    
    // 1. Call Stack Demo
    std::cout << "--- Call Stack Demo ---\n";
    CallStack call_stack;
    
    call_stack.push_frame("main()", 1);
    call_stack.add_local_var("x");
    call_stack.push_frame("process_data()", 45);
    call_stack.add_local_var("buffer");
    call_stack.push_frame("validate()", 123);
    call_stack.add_local_var("result");
    
    call_stack.print_stack_trace();
    
    call_stack.pop_frame();  // return from validate()
    call_stack.pop_frame();  // return from process_data()
    call_stack.print_stack_trace();
    std::cout << "\n";
    
    // 2. Ready Queue (FIFO) Demo
    std::cout << "--- FIFO Ready Queue Demo ---\n";
    ReadyQueue ready_queue;
    
    ready_queue.enqueue(Process{1, "Browser", 5, 100});
    ready_queue.enqueue(Process{2, "Editor", 5, 150});
    ready_queue.enqueue(Process{3, "Terminal", 5, 80});
    
    ready_queue.print();
    
    std::cout << "\nProcessing in FIFO order:\n";
    while (!ready_queue.empty()) {
        Process p = ready_queue.dequeue();
    }
    std::cout << "\n";
    
    // 3. Priority Scheduler Demo
    std::cout << "--- Priority Scheduler Demo ---\n";
    PriorityScheduler scheduler;
    
    scheduler.add_process(Process{10, "Background Task", 7, 200});
    scheduler.add_process(Process{11, "System Service", 0, 50});
    scheduler.add_process(Process{12, "User App", 5, 100});
    scheduler.add_process(Process{13, "Critical System", 0, 30});
    scheduler.add_process(Process{14, "Low Priority", 9, 150});
    
    scheduler.print_queue();
    
    std::cout << "\nExecuting by priority (highest first):\n";
    while (!scheduler.empty()) {
        Process p = scheduler.get_next();
        // Simulate execution
        scheduler.complete_process(p);
    }
    
    scheduler.print_completed();
    std::cout << "\n";
    
    // 4. Undo/Redo Demo
    std::cout << "--- Undo/Redo Manager Demo ---\n";
    UndoRedoManager undo_mgr;
    
    undo_mgr.execute_action("Create file 'document.txt'");
    undo_mgr.execute_action("Write 'Hello World'");
    undo_mgr.execute_action("Format text");
    
    undo_mgr.print_status();
    
    std::cout << "\nUndo operations:\n";
    undo_mgr.undo();
    undo_mgr.undo();
    
    undo_mgr.print_status();
    
    std::cout << "\nRedo operations:\n";
    undo_mgr.redo();
    
    undo_mgr.print_status();
    std::cout << "\n";
    
    // 5. Process Tree BFS Demo
    std::cout << "--- Process Tree Traversal ---\n";
    ProcessTree tree;
    tree.set_root(Process{1, "init", 0, 0});
    tree.add_child(1, Process{100, "systemd", 0, 0});
    tree.add_child(1, Process{200, "shell", 5, 0});
    tree.add_child(1, Process{300, "daemon", 3, 0});
    
    tree.bfs_traversal();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Output (Sample):
```
=== Operating System Process Scheduler ===

--- Call Stack Demo ---
  [CALL] main() at line 1
  [CALL] process_data() at line 45
  [CALL] validate() at line 123

=== Stack Trace ===
  #0 validate() (line 123)
     Locals: result 
  #1 process_data() (line 45)
     Locals: buffer 
  #2 main() (line 1)
     Locals: x 

  [RETURN] validate()
  [RETURN] process_data()

=== Stack Trace ===
  #0 main() (line 1)
     Locals: x 

--- FIFO Ready Queue Demo ---
  [ENQUEUE] Process 1: Browser
  [ENQUEUE] Process 2: Editor
  [ENQUEUE] Process 3: Terminal

Ready Queue (FIFO): P1 P2 P3 

Processing in FIFO order:
  [DEQUEUE] Process 1: Browser
  [DEQUEUE] Process 2: Editor
  [DEQUEUE] Process 3: Terminal

--- Priority Scheduler Demo ---
  [ADD] Process 10: Background Task (priority 7)
  [ADD] Process 11: System Service (priority 0)
  [ADD] Process 12: User App (priority 5)
  [ADD] Process 13: Critical System (priority 0)
  [ADD] Process 14: Low Priority (priority 9)

Priority Queue (by priority): P11(pri:0) P13(pri:0) P12(pri:5) P10(pri:7) P14(pri:9) 

Executing by priority (highest first):
  [SCHEDULE] Process 11: System Service (priority 0)
  [COMPLETE] Process 11: System Service
  [SCHEDULE] Process 13: Critical System (priority 0)
  [COMPLETE] Process 13: Critical System
  ...
```

### Concepts Demonstrated:
- **std::stack**: 
  - Call stack management (LIFO)
  - Stack trace generation
  - Undo/redo pattern with dual stacks
- **std::queue**: 
  - FIFO process scheduling
  - BFS traversal of tree
  - Ready queue management
- **std::priority_queue**: 
  - Priority-based scheduling
  - Custom comparator
  - Heap-based ordering
- **Container adaptors**:
  - Restricted interfaces
  - Underlying container abstraction
  - Real-world use cases

This example shows how container adaptors solve real OS/application problems!

---

## Related Topics

- [Sequence Containers](01_sequence_containers.md) — underlying `deque`, `vector`, and `list` that adaptors wrap
- [Iterators](05_iterators.md) — why adaptors cannot supply iterator ranges to algorithms
- [Algorithms](06_algorithms.md) — heap algorithms (`std::push_heap`, `make_heap`) on iterator ranges as an alternative to `priority_queue`
- [Unordered Containers](03_unordered_containers.md) — `unordered_set` for visited tracking in BFS/DFS examples
- [Multithreading](14_multithreading.md) — mutex/condition_variable patterns behind thread-safe queues
- [Modern C++20/23 Features](07_modern_features.md) — `std::ranges` views when you need lazy, iterable pipelines instead of adaptors

## Next Steps
- **Next**: [Iterators →](05_iterators.md)
- **Previous**: [← Unordered Containers](03_unordered_containers.md)

---
*Chapter 4 — Container Adaptors*

