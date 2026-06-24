# Memory Management and Allocators

## Overview

Memory management is crucial for performance and correctness in C++. This chapter covers the [Allocator](https://en.cppreference.com/w/cpp/named_req/Allocator) requirements, `std::allocator_traits`, polymorphic memory resources (PMR), alignment, placement new, and pool strategies. Containers that accept allocators are covered in [Sequence Containers](01_sequence_containers.md) and [Utility Containers](08_utility_containers.md).

```
┌───────────────────────────────────────────────────────┐
│           MEMORY MANAGEMENT COMPONENTS                │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ALLOCATORS     │  PMR (C++17)     │  ALIGNMENT       │
│  ──────────     │  ───────────     │  ─────────       │
│  • std::alloc   │  • pmr::memory   │  • alignas       │
│  • Custom       │  • pool_resource │  • alignof       │
│  • Stateful     │  • monotonic     │  • aligned_alloc │
│                 │  • synchronized  │                  │
│                                                       │
│  PLACEMENT NEW  │  MEMORY POOLS    │  DEBUGGING       │
│  ─────────────  │  ────────────    │  ─────────       │
│  • new (ptr)    │  • Object pools  │  • Sanitizers    │
│  • Manual ctor  │  • Arena         │  • Valgrind      │
│  • Alignment    │  • Stack alloc   │  • Tools         │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

## Standard Allocators

### std::allocator and allocator_traits

Custom allocators should implement the Allocator named requirements; prefer `std::allocator_traits<Alloc>` to call `allocate`, `deallocate`, `construct`, and `destroy` so your allocator works with pre-C++17 and stateful allocators.

```cpp
#include <iostream>
#include <limits>
#include <memory>

int main() {
    std::allocator<int> alloc;
    
    // Allocate memory for 10 ints (uninitialized)
    int* p = alloc.allocate(10);
    
    // Construct objects
    for (int i = 0; i < 10; ++i) {
        std::allocator_traits<std::allocator<int>>::construct(alloc, &p[i], i);
    }
    
    // Use objects
    for (int i = 0; i < 10; ++i) {
        std::cout << p[i] << ' ';
    }
    std::cout << '\n';
    
    // Destroy objects
    for (int i = 0; i < 10; ++i) {
        std::allocator_traits<std::allocator<int>>::destroy(alloc, &p[i]);
    }
    
    // Deallocate memory
    alloc.deallocate(p, 10);
    
    return 0;
}
```

### Allocator Phases
```
Memory Management Phases:
┌────────────────────────────────────────────────────────┐
│                                                        │
│ 1. Allocation:    allocate() → raw memory              │
│                   ┌──────────────┐                     │
│                   │ [???][???]   │ (uninitialized)     │
│                   └──────────────┘                     │
│                                                        │
│ 2. Construction:  construct() → initialized objects    │
│                   ┌──────────────┐                     │
│                   │ [10] [20]    │ (valid objects)     │
│                   └──────────────┘                     │
│                                                        │
│ 3. Use:           Access objects                       │
│                                                        │
│ 4. Destruction:   destroy() → call destructors         │
│                   ┌──────────────┐                     │
│                   │ [???][???]   │ (memory still there)│
│                   └──────────────┘                     │
│                                                        │
│ 5. Deallocation:  deallocate() → return memory         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Custom Allocator
```cpp
#include <cstdlib>
#include <iostream>
#include <limits>
#include <memory>
#include <vector>

template<typename T>
class LoggingAllocator {
public:
    using value_type = T;
    
    LoggingAllocator() = default;
    
    template<typename U>
    LoggingAllocator(const LoggingAllocator<U>&) {}
    
    T* allocate(std::size_t n) {
        std::cout << "Allocating " << n << " objects of size " 
                  << sizeof(T) << '\n';
        
        if (n > std::numeric_limits<std::size_t>::max() / sizeof(T)) {
            throw std::bad_alloc();
        }
        
        void* p = std::malloc(n * sizeof(T));
        if (!p) throw std::bad_alloc();
        
        return static_cast<T*>(p);
    }
    
    void deallocate(T* p, std::size_t n) {
        std::cout << "Deallocating " << n << " objects\n";
        std::free(p);
    }
};

template<typename T, typename U>
bool operator==(const LoggingAllocator<T>&, const LoggingAllocator<U>&) {
    return true;
}

template<typename T, typename U>
bool operator!=(const LoggingAllocator<T>&, const LoggingAllocator<U>&) {
    return false;
}

int main() {
    std::vector<int, LoggingAllocator<int>> vec;
    vec.push_back(1);
    vec.push_back(2);
    vec.push_back(3);
    
    return 0;
}
```

### Stateful Allocators and Propagation

Some allocators carry state (arena pointer, pool identity). Container copy/move/swap behavior is controlled by traits:

| Trait | Effect when `true` |
|-------|-------------------|
| `propagate_on_container_copy_assignment` | Copying a container copies its allocator |
| `propagate_on_container_move_assignment` | Moving transfers allocator ownership |
| `propagate_on_container_swap` | `swap` also swaps allocators |
| `is_always_equal` | All instances interchangeable (e.g. `std::allocator`) |

Mismatching allocators on `deallocate` is undefined behavior — always pair `allocate`/`deallocate` through the same allocator instance.

---

## Polymorphic Memory Resources (PMR - C++17)

PMR decouples *what* allocates from *how* via `std::pmr::memory_resource`. `std::pmr::polymorphic_allocator<T>` is the type-erased allocator adapters use internally; pass a `memory_resource*` to PMR containers (`std::pmr::vector`, `std::pmr::string`).

### Memory Resources
```cpp
#include <iostream>
#include <memory_resource>
#include <vector>

int main() {
    // Default resource (global new/delete)
    std::pmr::memory_resource* default_resource = 
        std::pmr::new_delete_resource();
    
    // Or: std::pmr::get_default_resource() (may be redirected via set_default_resource)
    
    // Monotonic buffer resource (fast, no deallocation)
    char buffer[1024];
    std::pmr::monotonic_buffer_resource pool(buffer, sizeof(buffer));
    
    // Use with pmr containers
    std::pmr::vector<int> vec(&pool);
    
    for (int i = 0; i < 10; ++i) {
        vec.push_back(i);
    }
    
    // Memory comes from buffer, not heap!
    
    return 0;
}  // All allocations from pool released at once
```

### PMR Memory Resources
```
┌────────────────────────────────────────────────────────┐
│           PMR Memory Resource Types                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│ new_delete_resource:                                   │
│   • Uses global new/delete                             │
│   • Default resource                                   │
│                                                        │
│ monotonic_buffer_resource:                             │
│   • Fast allocation                                    │
│   • No per-object deallocation                         │
│   • Releases all at once                               │
│   • Good for: Temporary allocations                    │
│                                                        │
│ synchronized_pool_resource:                            │
│   • Thread-safe pooling                                │
│   • Reduces fragmentation                              │
│   • Good for: Multi-threaded apps                      │
│                                                        │
│ unsynchronized_pool_resource:                          │
│   • Single-threaded pooling                            │
│   • Faster than synchronized                           │
│   • Good for: Single-threaded perf                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### PMR Container Performance

Use [Time and Chrono](18_time_chrono.md) with `steady_clock` when benchmarking — PMR wins are often visible only under allocation-heavy workloads.

```cpp
#include <chrono>
#include <iostream>
#include <memory_resource>
#include <vector>

void benchmark_pmr() {
    using namespace std::chrono;
    
    // Standard allocator
    {
        auto start = steady_clock::now();
        std::vector<int> vec;
        for (int i = 0; i < 100000; ++i) {
            vec.push_back(i);
        }
        auto elapsed = steady_clock::now() - start;
        std::cout << "Standard: " 
                  << duration_cast<milliseconds>(elapsed).count() << "ms\n";
    }
    
    // PMR with monotonic buffer
    {
        char buffer[1000000];
        std::pmr::monotonic_buffer_resource pool(buffer, sizeof(buffer));
        
        auto start = steady_clock::now();
        std::pmr::vector<int> vec(&pool);
        for (int i = 0; i < 100000; ++i) {
            vec.push_back(i);
        }
        auto elapsed = steady_clock::now() - start;
        std::cout << "PMR: " 
                  << duration_cast<milliseconds>(elapsed).count() << "ms\n";
    }
}
```

---

## Alignment

Objects have natural alignment (`alignof(T)`). SIMD, atomics, and hardware interfaces may need stricter alignment via `alignas`. Over-aligned types (`alignas(N)` where `N > alignof(T)`) may require `operator new`/`delete` overloads or `std::aligned_alloc`.

### Memory Alignment Basics
```cpp
#include <cstddef>
#include <iostream>

int main() {
    // Query alignment
    std::cout << "int alignment: " << alignof(int) << '\n';
    std::cout << "double alignment: " << alignof(double) << '\n';
    std::cout << "max_align_t: " << alignof(std::max_align_t) << '\n';
    
    // Aligned struct
    struct alignas(16) AlignedStruct {
        int x;
        double y;
    };
    
    std::cout << "AlignedStruct alignment: " 
              << alignof(AlignedStruct) << '\n';  // 16
    std::cout << "AlignedStruct size: " 
              << sizeof(AlignedStruct) << '\n';    // Padded to 16
    
    // Aligned variable
    alignas(32) int aligned_var;
    
    return 0;
}
```

### Aligned Allocation and std::align
```cpp
#include <cstddef>
#include <cstdlib>
#include <new>

int main() {
    // C17/C++17: aligned_alloc — size must be a multiple of alignment
    void* p1 = std::aligned_alloc(64, 256);
    if (p1) std::free(p1);
    
    // std::align: bump a pointer within an existing buffer
    alignas(std::max_align_t) unsigned char buffer[128];
    void* ptr = buffer;
    std::size_t space = sizeof(buffer);
    void* aligned = std::align(32, 64, ptr, space);  // 64 bytes at 32-byte boundary
    
    struct alignas(64) OverAligned {
        char data[64];
    };
    
    OverAligned* p2 = new OverAligned;
    delete p2;
    
    return 0;
}
```

### Visual: Memory Alignment
```
Unaligned:
Address: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
         [int ][int ][int ][int ]
         0     4     8     12

Aligned to 8 bytes:
Address: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
         [int ][pad][int ][pad]
         0     4     8     12
         
Padding ensures aligned access (faster on most architectures)
```

---

## Placement New

Placement `new` constructs an object in *existing* storage — it does **not** allocate. You **must** call the destructor manually (`ptr->~T()`); there is no `delete` for placement-new objects.

⚠️ **Gotchas**: If you reconstruct over live storage, use `std::launder` (C++17) to obtain a valid pointer to the new object; without it, the compiler may assume the old object still lives ([Advanced Features](12_advanced_features.md)).

### Manual Object Construction
```cpp
#include <iostream>
#include <new>

class Widget {
    int value_;
public:
    Widget(int v) : value_(v) {
        std::cout << "Widget constructed with " << v << '\n';
    }
    
    ~Widget() {
        std::cout << "Widget destructed\n";
    }
    
    int get() const { return value_; }
};

int main() {
    // Allocate raw memory
    alignas(Widget) char buffer[sizeof(Widget)];
    
    // Construct object in memory (placement new)
    Widget* w = new (buffer) Widget(42);
    
    // Use object
    std::cout << "Value: " << w->get() << '\n';
    
    // Explicitly destroy (required!)
    w->~Widget();
    
    // Memory (buffer) still exists
    // No delete needed (buffer is on stack)
    
    return 0;
}
```

### Array Placement New
```cpp
#include <new>

int main() {
    const size_t N = 10;
    alignas(int) char buffer[sizeof(int) * N];
    
    // Construct array in buffer
    int* arr = new (buffer) int[N];
    
    for (size_t i = 0; i < N; ++i) {
        arr[i] = i;
    }
    
    // Destroy array (explicit destructor call for each element)
    for (size_t i = 0; i < N; ++i) {
        arr[i].~int();
    }
    
    // For trivial types, destructor call can be omitted
    
    return 0;
}
```

---

## Memory Pools

### Simple Object Pool
```cpp
#include <vector>
#include <memory>

template<typename T>
class ObjectPool {
    std::vector<std::unique_ptr<T>> pool_;
    std::vector<T*> available_;
    
public:
    ObjectPool(size_t initial_size = 10) {
        pool_.reserve(initial_size);
        available_.reserve(initial_size);
        
        for (size_t i = 0; i < initial_size; ++i) {
            auto obj = std::make_unique<T>();
            available_.push_back(obj.get());
            pool_.push_back(std::move(obj));
        }
    }
    
    T* acquire() {
        if (available_.empty()) {
            // Pool exhausted, create new object
            auto obj = std::make_unique<T>();
            T* ptr = obj.get();
            pool_.push_back(std::move(obj));
            return ptr;
        }
        
        T* obj = available_.back();
        available_.pop_back();
        return obj;
    }
    
    void release(T* obj) {
        // Reset object to default state if needed
        *obj = T();
        available_.push_back(obj);
    }
    
    size_t capacity() const { return pool_.size(); }
    size_t available() const { return available_.size(); }
};

// Usage
int main() {
    ObjectPool<std::string> pool(5);
    
    std::string* s1 = pool.acquire();
    *s1 = "Hello";
    
    std::string* s2 = pool.acquire();
    *s2 = "World";
    
    pool.release(s1);
    pool.release(s2);
    
    return 0;
}
```

### Visual: Object Pool
```
Object Pool:
┌────────────────────────────────────────────────────┐
│ Pool Storage:      [obj1][obj2][obj3][obj4][obj5]  │
│                      ▲     ▲                       │
│                      │     │                       │
│ Available List:    [ptr1][ptr2]                    │
│                                                    │
│ In Use:            [obj3][obj4][obj5]              │
└────────────────────────────────────────────────────┘

acquire() → Pops from available list
release() → Pushes back to available list

Benefits:
• Reduced allocation overhead
• Better cache locality
• Predictable performance
```

---

## Stack Allocators

### Fixed-Size Stack Allocator
```cpp
#include <array>
#include <cstddef>
#include <vector>

template<typename T, size_t N>
class StackAllocator {
    alignas(T) std::array<std::byte, sizeof(T) * N> buffer_;
    std::byte* current_;
    
public:
    using value_type = T;
    
    StackAllocator() : current_(buffer_.data()) {}
    
    T* allocate(std::size_t n) {
        if (current_ + n * sizeof(T) > buffer_.data() + buffer_.size()) {
            throw std::bad_alloc();
        }
        
        T* result = reinterpret_cast<T*>(current_);
        current_ += n * sizeof(T);
        return result;
    }
    
    void deallocate(T* p, std::size_t n) {
        // Stack allocator: no actual deallocation
        // Memory reused when allocator destroyed
    }
    
    template<typename U>
    struct rebind {
        using other = StackAllocator<U, N>;
    };
};

int main() {
    std::vector<int, StackAllocator<int, 100>> vec;
    vec.push_back(1);
    vec.push_back(2);
    // Uses stack memory, not heap!
    
    return 0;
}
```

---

## PMR Detailed Examples

### Monotonic Buffer Resource
```cpp
#include <memory_resource>
#include <vector>
#include <iostream>

void example_monotonic() {
    // Stack buffer
    std::array<std::byte, 10000> buffer;
    std::pmr::monotonic_buffer_resource pool{buffer.data(), buffer.size()};
    
    // Containers use pool
    std::pmr::vector<int> vec(&pool);
    std::pmr::vector<std::pmr::string> strings(&pool);
    
    for (int i = 0; i < 100; ++i) {
        vec.push_back(i);
    }
    
    strings.push_back("Hello");
    strings.push_back("World");
    
    std::cout << "Used stack memory, no heap allocations!\n";
    
    // All memory released when pool destroyed
}
```

### Pool Resources
```cpp
#include <memory_resource>
#include <vector>
#include <thread>

void example_pool() {
    // Unsynchronized pool (single-threaded)
    std::pmr::unsynchronized_pool_resource pool;
    std::pmr::vector<int> vec(&pool);
    
    for (int i = 0; i < 1000; ++i) {
        vec.push_back(i);
    }
    
    // Synchronized pool (thread-safe)
    std::pmr::synchronized_pool_resource sync_pool;
    
    auto worker = [&sync_pool]() {
        std::pmr::vector<int> local_vec(&sync_pool);
        for (int i = 0; i < 100; ++i) {
            local_vec.push_back(i);
        }
    };
    
    std::thread t1(worker);
    std::thread t2(worker);
    
    t1.join();
    t2.join();
}
```

### Chaining Memory Resources
```cpp
#include <memory_resource>

void example_chain() {
    // Create hierarchy of resources
    std::pmr::synchronized_pool_resource global_pool;
    
    // Monotonic buffer backed by pool
    std::array<std::byte, 1000> buffer;
    std::pmr::monotonic_buffer_resource local_buffer{
        buffer.data(), buffer.size(), &global_pool
    };
    // Uses buffer first, then falls back to global_pool
    
    std::pmr::vector<int> vec(&local_buffer);
    
    // First allocations from buffer
    // If exhausted, allocations from global_pool
}
```

---

## Memory Debugging

### Detecting Memory Leaks
```cpp
// Compile with sanitizers:
// g++ -fsanitize=address program.cpp

void leak_example() {
    int* p = new int(42);
    // Forgot to delete - AddressSanitizer detects this
}

// Use smart pointers to avoid leaks
void safe_example() {
    auto p = std::make_unique<int>(42);
    // Automatically deleted
}
```

### Custom Memory Tracking
```cpp
#include <cstdlib>
#include <iostream>

static size_t total_allocated = 0;
static size_t total_deallocated = 0;

void* operator new(std::size_t size) {
    total_allocated += size;
    std::cout << "Allocating " << size << " bytes\n";
    void* p = std::malloc(size);
    if (!p) throw std::bad_alloc();
    return p;
}

void operator delete(void* p) noexcept {
    std::cout << "Deallocating\n";
    total_deallocated++;
    std::free(p);
}

void report_memory() {
    std::cout << "Total allocated: " << total_allocated << '\n';
    std::cout << "Total deallocations: " << total_deallocated << '\n';
}
```

---

## Performance Tips

### 1. Reserve Capacity
```cpp
// BAD: Multiple reallocations
std::vector<int> vec;
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // May reallocate many times
}

// GOOD: Single allocation
std::vector<int> vec;
vec.reserve(10000);
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // No reallocation
}
```

### 2. Use emplace for In-Place Construction
```cpp
std::vector<std::string> vec;

// BAD: Creates temporary
vec.push_back(std::string("hello"));

// GOOD: Constructs in-place
vec.emplace_back("hello");
```

### 3. Small String Optimization (SSO)
```cpp
// Most std::string implementations use SSO
std::string small = "short";  // No heap allocation
std::string large = "This is a very long string that exceeds SSO buffer";
// Heap allocation
```

### 4. Object Pools for Frequent Allocations
```cpp
// Use object pool when:
// • Creating/destroying many objects
// • Objects of uniform size
// • Performance critical

ObjectPool<GameObject> pool(100);
```

---

## Best Practices

### 1. Prefer Smart Pointers
```cpp
// GOOD: Automatic cleanup
auto p = std::make_unique<Widget>();

// BAD: Manual management
Widget* p = new Widget();
delete p;
```

### 2. Use PMR for Performance-Critical Code
```cpp
// Hot path with many allocations
std::array<std::byte, 100000> buffer;
std::pmr::monotonic_buffer_resource pool{buffer.data(), buffer.size()};
std::pmr::vector<Data> vec(&pool);
// Fast allocations from stack buffer
```

### 3. Align Data Appropriately
```cpp
// For SIMD operations
alignas(32) float data[8];  // AVX alignment

// Cache line alignment (avoid false sharing)
alignas(64) std::atomic<int> counter;
```

### 4. Profile Before Optimizing
```cpp
// Don't assume - measure!
// Use profilers: Valgrind, perf, custom timers
```

---

## Common Pitfalls

### 1. Forgetting to Destroy Placement New Objects
```cpp
// BAD
alignas(Widget) char buffer[sizeof(Widget)];
Widget* w = new (buffer) Widget();
// Forgot w->~Widget(); - leak!

// GOOD
w->~Widget();
```

### 2. Allocator Mismatch
```cpp
std::allocator<int> alloc;
int* p = alloc.allocate(10);
delete p;  // BAD: Must use alloc.deallocate!

alloc.deallocate(p, 10);  // GOOD
```

### 3. Using After Free
```cpp
int* p = new int(42);
delete p;
int x = *p;  // UB: Use after free!
```

### 4. std::launder After In-Place Reconstruction
```cpp
alignas(T) unsigned char buf[sizeof(T)];
T* p = new (buf) T(1);
p->~T();
T* q = new (buf) T(2);
// T* valid = std::launder(q);  // Required if optimizer could see stale p
```

---

## Complete Practical Example: Game Object Memory Manager

Here's a comprehensive example integrating allocators, PMR, object pools, and custom memory management:

```cpp
#include <iostream>
#include <memory>
#include <memory_resource>
#include <vector>
#include <list>
#include <string>
#include <chrono>
#include <array>

// Game object base class
struct GameObject {
    int id;
    float x, y, z;
    std::string name;
    
    GameObject(int i, std::string n)
        : id(i), x(0), y(0), z(0), name(std::move(n)) {}
    
    virtual ~GameObject() = default;
    virtual void update() = 0;
};

struct Enemy : GameObject {
    int health;
    
    Enemy(int id, std::string name, int hp)
        : GameObject(id, std::move(name)), health(hp) {}
    
    void update() override {
        // Enemy AI update
    }
};

struct Projectile : GameObject {
    float velocity_x, velocity_y;
    
    Projectile(int id, float vx, float vy)
        : GameObject(id, "projectile")
        , velocity_x(vx), velocity_y(vy) {}
    
    void update() override {
        x += velocity_x;
        y += velocity_y;
    }
};

// 1. Custom allocator with logging
template<typename T>
class LoggingAllocator {
private:
    static inline size_t allocations_ = 0;
    static inline size_t deallocations_ = 0;
    static inline size_t total_bytes_ = 0;
    
public:
    using value_type = T;
    
    LoggingAllocator() = default;
    
    template<typename U>
    LoggingAllocator(const LoggingAllocator<U>&) noexcept {}
    
    T* allocate(std::size_t n) {
        allocations_++;
        total_bytes_ += n * sizeof(T);
        
        std::cout << "  [ALLOC] " << n << " × " << sizeof(T) 
                  << " bytes = " << (n * sizeof(T)) << " bytes\n";
        
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    
    void deallocate(T* p, std::size_t n) noexcept {
        deallocations_++;
        std::cout << "  [DEALLOC] " << (n * sizeof(T)) << " bytes\n";
        ::operator delete(p);
    }
    
    static void print_stats() {
        std::cout << "\n=== Allocator Statistics ===\n";
        std::cout << "Allocations: " << allocations_ << "\n";
        std::cout << "Deallocations: " << deallocations_ << "\n";
        std::cout << "Total allocated: " << total_bytes_ << " bytes\n";
        std::cout << "Leaks: " << (allocations_ - deallocations_) << "\n";
    }
};

template<typename T, typename U>
bool operator==(const LoggingAllocator<T>&, const LoggingAllocator<U>&) { return true; }

template<typename T, typename U>
bool operator!=(const LoggingAllocator<T>&, const LoggingAllocator<U>&) { return false; }

// 2. Object pool for frequent allocations
template<typename T>
class ObjectPool {
private:
    union Slot {
        T object;
        Slot* next;
        
        Slot() {}
        ~Slot() {}
    };
    
    std::vector<std::unique_ptr<Slot[]>> blocks_;
    Slot* free_list_ = nullptr;
    size_t block_size_;
    size_t active_objects_ = 0;
    
public:
    explicit ObjectPool(size_t block_size = 32)
        : block_size_(block_size) {
        allocate_block();
    }
    
    ~ObjectPool() {
        // Objects should be returned before destruction
        if (active_objects_ > 0) {
            std::cerr << "Warning: " << active_objects_ 
                      << " objects not returned to pool\n";
        }
    }
    
    template<typename... Args>
    T* acquire(Args&&... args) {
        if (!free_list_) {
            allocate_block();
        }
        
        Slot* slot = free_list_;
        free_list_ = slot->next;
        
        new (&slot->object) T(std::forward<Args>(args)...);
        active_objects_++;
        
        return &slot->object;
    }
    
    void release(T* obj) {
        if (!obj) return;
        
        obj->~T();
        
        Slot* slot = reinterpret_cast<Slot*>(obj);
        slot->next = free_list_;
        free_list_ = slot;
        active_objects_--;
    }
    
    size_t active_count() const { return active_objects_; }
    size_t capacity() const { return blocks_.size() * block_size_; }
    
private:
    void allocate_block() {
        auto block = std::make_unique<Slot[]>(block_size_);
        
        for (size_t i = 0; i < block_size_; ++i) {
            block[i].next = (i == block_size_ - 1) ? free_list_ : &block[i + 1];
        }
        
        free_list_ = &block[0];
        blocks_.push_back(std::move(block));
        
        std::cout << "  [POOL] Allocated block of " << block_size_ 
                  << " slots (" << (block_size_ * sizeof(T)) << " bytes)\n";
    }
};

// 3. PMR monotonic buffer allocator
class FrameAllocator {
private:
    std::array<std::byte, 1024 * 1024> buffer_;  // 1MB stack buffer
    std::pmr::monotonic_buffer_resource pool_;
    
public:
    FrameAllocator() : pool_(buffer_.data(), buffer_.size()) {}
    
    std::pmr::memory_resource* get_resource() {
        return &pool_;
    }
    
    void reset() {
        pool_.release();
        std::cout << "  [FRAME] Reset frame allocator\n";
    }
    
    size_t bytes_allocated() const {
        // Approximate (PMR doesn't expose this directly)
        return buffer_.size();
    }
};

// 4. Stack allocator for small, short-lived objects
template<typename T, size_t N>
class StackAllocator {
private:
    alignas(T) std::array<std::byte, sizeof(T) * N> buffer_;
    std::byte* current_;
    size_t allocated_ = 0;
    
public:
    using value_type = T;
    
    StackAllocator() : current_(buffer_.data()) {}
    
    T* allocate(std::size_t n) {
        if (current_ + n * sizeof(T) > buffer_.data() + buffer_.size()) {
            throw std::bad_alloc();
        }
        
        T* result = reinterpret_cast<T*>(current_);
        current_ += n * sizeof(T);
        allocated_ += n;
        
        std::cout << "  [STACK] Allocated " << n << " objects on stack\n";
        return result;
    }
    
    void deallocate(T*, std::size_t) noexcept {
        // Stack allocator doesn't deallocate individual objects
    }
    
    size_t allocated_count() const { return allocated_; }
    
    template<typename U>
    struct rebind {
        using other = StackAllocator<U, N>;
    };
};

// Game memory manager
class GameMemoryManager {
private:
    ObjectPool<Projectile> projectile_pool_;
    ObjectPool<Enemy> enemy_pool_;
    FrameAllocator frame_allocator_;
    
public:
    GameMemoryManager()
        : projectile_pool_(64)  // Pool of 64 projectiles
        , enemy_pool_(32) {}     // Pool of 32 enemies
    
    // Allocate projectile from pool
    Projectile* create_projectile(int id, float vx, float vy) {
        return projectile_pool_.acquire(id, vx, vy);
    }
    
    void destroy_projectile(Projectile* proj) {
        projectile_pool_.release(proj);
    }
    
    // Allocate enemy from pool
    Enemy* create_enemy(int id, const std::string& name, int health) {
        return enemy_pool_.acquire(id, name, health);
    }
    
    void destroy_enemy(Enemy* enemy) {
        enemy_pool_.release(enemy);
    }
    
    // Get frame allocator for temporary data
    std::pmr::memory_resource* get_frame_resource() {
        return frame_allocator_.get_resource();
    }
    
    void end_frame() {
        frame_allocator_.reset();
    }
    
    void print_stats() const {
        std::cout << "\n=== Game Memory Stats ===\n";
        std::cout << "Projectiles: " << projectile_pool_.active_count()
                  << "/" << projectile_pool_.capacity() << "\n";
        std::cout << "Enemies: " << enemy_pool_.active_count()
                  << "/" << enemy_pool_.capacity() << "\n";
    }
};

// Demo functions
void demo_custom_allocator() {
    std::cout << "\n--- Custom Allocator ---\n";
    
    std::vector<int, LoggingAllocator<int>> vec;
    vec.reserve(10);
    
    for (int i = 0; i < 5; ++i) {
        vec.push_back(i);
    }
    
    LoggingAllocator<int>::print_stats();
}

void demo_object_pool() {
    std::cout << "\n--- Object Pool ---\n";
    
    ObjectPool<Projectile> pool(8);
    
    std::vector<Projectile*> projectiles;
    
    // Spawn projectiles
    for (int i = 0; i < 12; ++i) {
        auto* proj = pool.acquire(i, 1.0f, 0.5f);
        projectiles.push_back(proj);
        std::cout << "  Spawned projectile " << i << "\n";
    }
    
    std::cout << "\nActive: " << pool.active_count() << "\n";
    
    // Destroy some
    for (size_t i = 0; i < 6; ++i) {
        pool.release(projectiles[i]);
        std::cout << "  Destroyed projectile " << i << "\n";
    }
    
    std::cout << "Active after cleanup: " << pool.active_count() << "\n";
    
    // Cleanup remaining
    for (size_t i = 6; i < projectiles.size(); ++i) {
        pool.release(projectiles[i]);
    }
}

void demo_pmr_allocator() {
    std::cout << "\n--- PMR Allocator ---\n";
    
    std::array<std::byte, 4096> buffer;
    std::pmr::monotonic_buffer_resource pool{buffer.data(), buffer.size()};
    
    std::pmr::vector<int> vec(&pool);
    std::pmr::string str(&pool);
    
    std::cout << "  Using stack buffer for PMR allocations\n";
    
    for (int i = 0; i < 100; ++i) {
        vec.push_back(i);
    }
    
    str = "Hello from PMR!";
    
    std::cout << "  Vector size: " << vec.size() << "\n";
    std::cout << "  String: " << str << "\n";
    std::cout << "  All allocated from stack buffer!\n";
}

void demo_game_memory() {
    std::cout << "\n--- Game Memory Manager ---\n";
    
    GameMemoryManager memory_mgr;
    
    // Create some enemies
    std::vector<Enemy*> enemies;
    for (int i = 0; i < 5; ++i) {
        enemies.push_back(memory_mgr.create_enemy(
            i, "Enemy" + std::to_string(i), 100));
    }
    
    // Create projectiles
    std::vector<Projectile*> projectiles;
    for (int i = 0; i < 10; ++i) {
        projectiles.push_back(memory_mgr.create_projectile(
            i, 1.0f, 0.5f));
    }
    
    memory_mgr.print_stats();
    
    // Frame-local allocations
    auto* frame_resource = memory_mgr.get_frame_resource();
    std::pmr::vector<int> temp_data(frame_resource);
    
    for (int i = 0; i < 100; ++i) {
        temp_data.push_back(i);
    }
    
    std::cout << "\nFrame temp data size: " << temp_data.size() << "\n";
    
    // End frame (resets frame allocator)
    memory_mgr.end_frame();
    
    // Cleanup
    for (auto* enemy : enemies) {
        memory_mgr.destroy_enemy(enemy);
    }
    for (auto* proj : projectiles) {
        memory_mgr.destroy_projectile(proj);
    }
    
    memory_mgr.print_stats();
}

void demo_stack_allocator() {
    std::cout << "\n--- Stack Allocator ---\n";
    
    std::vector<int, StackAllocator<int, 100>> vec;
    
    for (int i = 0; i < 10; ++i) {
        vec.push_back(i);
    }
    
    std::cout << "  Vector uses stack memory only!\n";
}

int main() {
    std::cout << "=== Memory Management Demo ===\n";
    
    // 1. Custom allocator with logging
    demo_custom_allocator();
    
    // 2. Object pool
    demo_object_pool();
    
    // 3. PMR allocator
    demo_pmr_allocator();
    
    // 4. Stack allocator
    demo_stack_allocator();
    
    // 5. Game memory manager (integrates everything)
    demo_game_memory();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **allocator_traits**: Portable allocator interface
- **PMR / polymorphic_allocator**: Type-erased memory resources
- **Object pools**: Reusing memory for frequent allocations
- **Monotonic buffer resource**: Fast frame-local allocations
- **Placement new / manual destroy**: In-buffer construction
- **Alignment**: `alignas`, `std::align`, over-aligned types

---

## Related Topics

- [Sequence Containers](01_sequence_containers.md) — `vector` allocator template parameter
- [Utility Containers](08_utility_containers.md) — `optional`, `variant` storage semantics
- [Templates](09_templates.md) — writing reusable allocator-aware templates
- [Best Practices](13_best_practices.md) — smart pointers, RAII, and sanitizers
- [Multithreading](14_multithreading.md) — `synchronized_pool_resource` for concurrent pools
- [Time and Chrono](18_time_chrono.md) — benchmarking allocation strategies
- [Quick Reference](99_quick_reference.md) — allocator and PMR API summary

## Next Steps
- **Next**: [Regular Expressions →](20_regex.md)
- **Previous**: [← Time and Chrono](18_time_chrono.md)

---
*Chapter 19 — Memory Management and Allocators*

