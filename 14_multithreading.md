# Multithreading and Concurrency

## Overview

C++11 introduced a standardized threading library, making concurrent programming portable across platforms. This chapter covers threads, synchronization, and thread safety. For higher-level async abstractions built on these primitives, see [Async and Futures](15_async_futures.md); for RAII patterns that make locks and resources exception-safe, see [Advanced Features](12_advanced_features.md) and [Exception Handling](17_exceptions.md).

```
┌─────────────────────────────────────────────────────────┐
│              MULTITHREADING COMPONENTS                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  THREADS          │  SYNCHRONIZATION  │  COMMUNICATION  │
│  ───────          │  ───────────────  │  ─────────────  │
│  • std::thread    │  • std::mutex     │  • Condition    │
│  • std::jthread   │  • Lock guards    │    variables    │
│  • Thread IDs     │  • Atomics        │  • Futures      │
│                   │  • Semaphores     │  • Promises     │
│                                                         │
│  C++11: Basic threading                                 │
│  C++14: Shared locks                                    │
│  C++17: Parallel algorithms                             │
│  C++20: jthread, stop_token, semaphores, barriers       │
│  C++23: Move-only function                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## std::thread Basics

### Creating Threads
```cpp
#include <thread>
#include <iostream>

void hello() {
    std::cout << "Hello from thread!\n";
}

void hello_with_arg(int n) {
    std::cout << "Hello " << n << " times!\n";
}

int main() {
    // Create thread from function
    std::thread t1(hello);
    
    // Create thread with arguments
    std::thread t2(hello_with_arg, 5);
    
    // Create thread from lambda
    std::thread t3([]() {
        std::cout << "Hello from lambda!\n";
    });
    
    // Must join or detach before destruction — otherwise std::terminate()
    t1.join();  // Wait for thread to finish
    t2.join();
    t3.join();
    
    return 0;
}
```

### Thread Lifecycle
```
┌────────────────────────────────────────────────────────┐
│                  Thread Lifecycle                      │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Created ──► Running ──► Finished                      │
│     │           │                                      │
│     │           └─► join() ──► Thread destroyed        │
│     │                                                  │
│     └─► detach() ──► Thread runs independently         │
│                                                        │
│  ⚠️ Destroying a joinable std::thread → std::terminate │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Join vs Detach
```cpp
#include <thread>
#include <chrono>

void work() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Work done\n";
}

int main() {
    // Join: Wait for thread to complete
    {
        std::thread t(work);
        t.join();  // Blocks until thread finishes
        std::cout << "Thread completed\n";
    }
    
    // Detach: Thread runs independently
    {
        std::thread t(work);
        t.detach();  // Thread continues running
        std::cout << "Thread detached\n";
        // Main may exit before thread finishes!
    }
    
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 0;
}
```

### Thread with Member Functions
```cpp
#include <thread>
#include <iostream>

class Worker {
public:
    void do_work(int n) {
        std::cout << "Working " << n << " times\n";
    }
    
    void operator()() {
        std::cout << "Operator() called\n";
    }
};

int main() {
    Worker w;
    
    // Method 1: Using member function pointer
    std::thread t1(&Worker::do_work, &w, 5);
    
    // Method 2: Using function object
    std::thread t2(w);  // Calls operator()
    
    // Method 3: Using lambda
    std::thread t3([&w]() {
        w.do_work(10);
    });
    
    t1.join();
    t2.join();
    t3.join();
    
    return 0;
}
```

### Thread Management
```cpp
#include <thread>
#include <iostream>

int main() {
    std::thread t([]() {
        std::cout << "Thread running\n";
    });
    
    // Get thread ID
    std::thread::id tid = t.get_id();
    std::cout << "Thread ID: " << tid << '\n';
    
    // Check if joinable
    if (t.joinable()) {
        std::cout << "Thread is joinable\n";
        t.join();
    }
    
    // After join, thread is no longer joinable
    if (!t.joinable()) {
        std::cout << "Thread is not joinable\n";
    }
    
    // Hardware concurrency
    unsigned int n = std::thread::hardware_concurrency();
    std::cout << "Hardware supports " << n << " concurrent threads\n";
    
    return 0;
}
```

---

## std::jthread (C++20)

### Automatic Joining Thread
```cpp
#include <thread>
#include <iostream>

void work() {
    std::cout << "Working...\n";
}

int main() {
    // OLD: std::thread requires explicit join/detach
    {
        std::thread t(work);
        t.join();  // Must call join or detach
    }
    
    // NEW: std::jthread joins automatically
    {
        std::jthread jt(work);
        // Automatically joins on destruction
    }
    
    return 0;
}
```

### Cooperative Cancellation with stop_token
```cpp
#include <thread>
#include <chrono>
#include <iostream>

void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::cout << "Working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    std::cout << "Stop requested, exiting\n";
}

int main() {
    std::jthread jt(worker);
    
    std::this_thread::sleep_for(std::chrono::seconds(2));
    
    // Request stop
    jt.request_stop();
    
    // jthread automatically joins on destruction
    return 0;
}
```

### Visual: std::thread vs std::jthread
```
std::thread:
┌──────────────────────────────────────┐
│ Create thread                        │
│ Do work                              │
│ Must explicitly join or detach       │ ← Can forget!
│ Destructor terminates if not joined  │ ← Danger!
└──────────────────────────────────────┘

std::jthread:
┌──────────────────────────────────────┐
│ Create thread                        │
│ Do work                              │
│ Destructor automatically joins       │ ← Safe!
│ Built-in cancellation support        │ ← Bonus!
└──────────────────────────────────────┘
```

💡 **Hunch**: Check `__cpp_lib_jthread` (C++20) before relying on `std::jthread`. Apple Clang/libc++ added it in recent Xcode releases; older toolchains need `std::thread` + explicit `join()`.

---

## Mutexes and Locks

`std::mutex` is **not recursive** — locking it twice from the same thread is undefined behavior. Use `std::recursive_mutex` only when re-entrant locking is genuinely required (e.g., callbacks that re-enter the same object); prefer redesigning the lock scope instead.

Always prefer RAII lock wrappers over manual `lock()`/`unlock()` — see [Advanced Features](12_advanced_features.md) and [Exception Handling](17_exceptions.md) for why RAII matters when exceptions unwind the stack.

### std::mutex
```cpp
#include <thread>
#include <mutex>
#include <iostream>

std::mutex mtx;
int shared_counter = 0;

void increment(int n) {
    for (int i = 0; i < n; ++i) {
        mtx.lock();
        ++shared_counter;  // Critical section
        mtx.unlock();
    }
}

int main() {
    std::thread t1(increment, 1000);
    std::thread t2(increment, 1000);
    
    t1.join();
    t2.join();
    
    std::cout << "Counter: " << shared_counter << '\n';  // 2000
    
    return 0;
}
```

### std::lock_guard
```cpp
#include <thread>
#include <mutex>

std::mutex mtx;
int shared_counter = 0;

void increment_safe(int n) {
    for (int i = 0; i < n; ++i) {
        std::lock_guard<std::mutex> lock(mtx);  // RAII lock
        ++shared_counter;
        // Automatically unlocked when lock goes out of scope
    }
}

int main() {
    std::thread t1(increment_safe, 1000);
    std::thread t2(increment_safe, 1000);
    
    t1.join();
    t2.join();
    
    std::cout << "Counter: " << shared_counter << '\n';
    
    return 0;
}
```

### std::unique_lock
```cpp
#include <thread>
#include <mutex>

std::mutex mtx;

void flexible_locking() {
    std::unique_lock<std::mutex> lock(mtx);
    
    // Do some work
    
    lock.unlock();  // Can manually unlock
    
    // Do work that doesn't need lock
    
    lock.lock();  // Can manually lock again
    
    // More critical section work
}  // Automatically unlocks on destruction

// Deferred locking
void deferred_lock() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // Mutex not locked yet
    
    // ... do preparatory work ...
    
    lock.lock();  // Lock when ready
    // Critical section
}

// Try lock
void try_locking() {
    std::unique_lock<std::mutex> lock(mtx, std::try_to_lock);
    
    if (lock.owns_lock()) {
        // Got the lock
    } else {
        // Didn't get lock, do something else
    }
}
```

### Lock Comparison
```
┌────────────────────────────────────────────────────────┐
│              Lock Types Comparison                     │
├────────────────────────────────────────────────────────┤
│              │ lock_guard │ unique_lock │ scoped_lock  │
├──────────────┼────────────┼─────────────┼──────────────┤
│ RAII         │ Yes        │ Yes         │ Yes          │
│ Manual lock  │ No         │ Yes         │ No           │
│ Manual unlock│ No         │ Yes         │ No           │
│ Try lock     │ No         │ Yes         │ No           │
│ Defer lock   │ No         │ Yes         │ No           │
│ Multi-mutex  │ No         │ No          │ Yes (C++17)  │
│ Overhead     │ Minimal    │ Small       │ Minimal      │
│                                                        │
│ Use lock_guard when: Simple RAII locking               │
│ Use unique_lock when: Need flexibility / condition_var │
│ Use scoped_lock when: Locking multiple mutexes         │
│ Use shared_lock when: Reader with shared_mutex (C++17) │
└────────────────────────────────────────────────────────┘
```

### std::scoped_lock (C++17)
```cpp
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;
int resource1 = 0, resource2 = 0;

void transfer(int amount) {
    // Locks both mutexes atomically (avoids deadlock)
    std::scoped_lock lock(mtx1, mtx2);
    
    resource1 -= amount;
    resource2 += amount;
}

int main() {
    std::thread t1(transfer, 10);
    std::thread t2(transfer, 20);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

### Deadlock Prevention
```cpp
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;

// BAD: Potential deadlock
void bad_transfer() {
    std::lock_guard<std::mutex> lock1(mtx1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> lock2(mtx2);
    // If another thread locks in opposite order: DEADLOCK!
}

// GOOD: std::lock (C++11) - locks multiple mutexes atomically
void good_transfer() {
    std::unique_lock<std::mutex> lock1(mtx1, std::defer_lock);
    std::unique_lock<std::mutex> lock2(mtx2, std::defer_lock);
    std::lock(lock1, lock2);  // Deadlock-free
}

// BEST: std::scoped_lock (C++17)
void best_transfer() {
    std::scoped_lock lock(mtx1, mtx2);  // Atomic, deadlock-free
}
```

### Visual: Deadlock Scenario
```
Thread 1              Thread 2
────────              ────────
Lock mtx1             Lock mtx2
  ✓                     ✓
Wait for mtx2  ◄────► Wait for mtx1
  ⏳                     ⏳
DEADLOCK!             DEADLOCK!

Solution with std::scoped_lock:
Thread 1              Thread 2
────────              ────────
Request mtx1, mtx2    Request mtx1, mtx2
One succeeds first    One succeeds first
  ✓                   Waits for release
Complete                ⏳
Release               Lock both
                        ✓
                      Complete
```

---

## Reader-Writer Locks

### std::shared_mutex (C++17)
```cpp
#include <shared_mutex>
#include <thread>
#include <vector>

class ThreadSafeCounter {
    mutable std::shared_mutex mtx_;
    int value_ = 0;
    
public:
    // Multiple readers can access simultaneously
    int get() const {
        std::shared_lock<std::shared_mutex> lock(mtx_);
        return value_;
    }
    
    // Only one writer can access
    void increment() {
        std::unique_lock<std::shared_mutex> lock(mtx_);
        ++value_;
    }
    
    void set(int value) {
        std::unique_lock<std::shared_mutex> lock(mtx_);
        value_ = value;
    }
};

int main() {
    ThreadSafeCounter counter;
    
    // Many readers
    std::vector<std::thread> readers;
    for (int i = 0; i < 10; ++i) {
        readers.emplace_back([&counter]() {
            for (int j = 0; j < 100; ++j) {
                int val = counter.get();  // Concurrent reads OK
            }
        });
    }
    
    // Few writers
    std::vector<std::thread> writers;
    for (int i = 0; i < 2; ++i) {
        writers.emplace_back([&counter]() {
            for (int j = 0; j < 100; ++j) {
                counter.increment();  // Exclusive write
            }
        });
    }
    
    for (auto& t : readers) t.join();
    for (auto& t : writers) t.join();
    
    return 0;
}
```

### Visual: Reader-Writer Lock
```
Shared Lock (read):
Reader 1 ──┐
Reader 2 ──┤──► All can read simultaneously
Reader 3 ──┘

Exclusive Lock (write):
Writer ──► Only one writer, blocks all readers

Timeline:
┌─────────────────────────────────────────────┐
│ R1 R2 R3 │ W1 │ R4 R5 │ W2 │ R6             │
├─────────────────────────────────────────────┤
│ ← Multiple readers can run together         │
│           ← Writer blocks all               │
│                 ← Readers together          │
│                        ← Writer blocks all  │
└─────────────────────────────────────────────┘
```

---

## Atomics

Concurrent unsynchronized access to the same non-atomic object from multiple threads is a **data race** → undefined behavior. `std::atomic<T>` (for trivially copyable `T` up to a platform-dependent size) provides lock-free or lock-based atomic operations. Default memory order is `memory_order_seq_cst` (strongest, easiest to reason about).

### std::atomic Basics
```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

std::atomic<int> counter(0);  // Lock-free atomic integer

void increment(int n) {
    for (int i = 0; i < n; ++i) {
        ++counter;  // Atomic increment, no mutex needed!
        // Equivalent to: counter.fetch_add(1);
    }
}

int main() {
    std::vector<std::thread> threads;
    
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(increment, 1000);
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << "Counter: " << counter << '\n';  // 10000
    
    return 0;
}
```

### Atomic Operations
```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> x(10);
    
    // Basic operations
    x = 20;                    // store
    int val = x;               // load
    
    // Arithmetic (for integral types)
    ++x;                       // 21
    x++;                       // 22
    x += 5;                    // 27
    x -= 3;                    // 24
    
    // Fetch and modify
    int old = x.fetch_add(5);  // old = 24, x = 29
    old = x.fetch_sub(4);      // old = 29, x = 25
    
    // Compare-and-swap (strong: no spurious failure; weak: may fail spuriously, use in loops)
    int expected = 25;
    bool success = x.compare_exchange_strong(expected, 30);
    // If x == expected, set x = 30, return true
    // Else, set expected = x, return false
    
    // Check if lock-free (may use internal mutex on some platforms)
    if (x.is_lock_free()) {
        std::cout << "Lock-free implementation\n";
    }
    
    return 0;
}
```

### Memory Ordering
```cpp
#include <atomic>
#include <thread>

std::atomic<int> data(0);
std::atomic<bool> ready(false);

void producer() {
    data.store(42, std::memory_order_relaxed);
    ready.store(true, std::memory_order_release);  // Release
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // Acquire
        // Spin wait
    }
    int value = data.load(std::memory_order_relaxed);
    std::cout << "Data: " << value << '\n';
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

### Memory Order Guide
```
┌────────────────────────────────────────────────────────┐
│               Memory Order Semantics                   │
├────────────────────────────────────────────────────────┤
│ Order            │ Use                │ Synchronization│
├──────────────────┼────────────────────┼────────────────┤
│ relaxed          │ Counter, stats     │ None           │
│ acquire          │ Consume data       │ Read-acquire   │
│ release          │ Publish data       │ Write-release  │
│ acq_rel          │ Read-modify-write  │ Both           │
│ seq_cst (default)│ When unsure        │ Full ordering  │
│                                                        │
│ Relaxed: Fastest, no synchronization                   │
│ Acquire/Release: Common, publish-subscribe pattern     │
│ Sequential: Slowest, easiest to reason about           │
└────────────────────────────────────────────────────────┘
```

⚠️ **Gotchas**:
- **Release → acquire** pairing establishes *happens-before*: writes before `release` are visible to reads after a matching `acquire`.
- `relaxed` provides no cross-thread ordering — fine for independent counters, not for publishing shared data.
- `compare_exchange_weak` may spuriously fail; prefer it in retry loops, `strong` when you need a single attempt.

---

## Condition Variables

`std::condition_variable` requires `std::unique_lock<std::mutex>` (not `lock_guard`). Always call `wait` with a **predicate** to handle spurious wakeups. You must hold the lock when calling `wait`; `notify_one`/`notify_all` may be called with or without the lock held (notify-without-lock can reduce contention).

### Basic Wait and Notify
```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> data_queue;
bool done = false;

void producer() {
    for (int i = 0; i < 10; ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        
        {
            std::lock_guard<std::mutex> lock(mtx);
            data_queue.push(i);
            std::cout << "Produced: " << i << '\n';
        }
        cv.notify_one();  // Wake up one consumer
    }
    
    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;
    }
    cv.notify_all();  // Wake up all consumers
}

void consumer(int id) {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        
        // Wait until data available or done
        cv.wait(lock, []() {
            return !data_queue.empty() || done;
        });
        
        if (data_queue.empty()) {
            // Done and no more data
            break;
        }
        
        int value = data_queue.front();
        data_queue.pop();
        lock.unlock();
        
        std::cout << "Consumer " << id << " got: " << value << '\n';
    }
}

int main() {
    std::thread prod(producer);
    std::thread cons1(consumer, 1);
    std::thread cons2(consumer, 2);
    
    prod.join();
    cons1.join();
    cons2.join();
    
    return 0;
}
```

### Visual: Condition Variable
```
Producer                        Consumer
────────                        ────────
Lock mutex                      Lock mutex
Add data to queue               Wait on cv (releases mutex)
Unlock mutex                    (sleeps)
Notify cv           ─────►      Wakes up (reacquires mutex)
                                Check condition (queue not empty)
                                Process data
                                Unlock mutex
```

### Spurious Wakeups
```cpp
// BAD: Not protected against spurious wakeups
void bad_consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);  // May wake up even if not notified!
    // Process data (might not be ready)
}

// GOOD: Always use predicate
void good_consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []() {
        return !data_queue.empty();  // Only proceed if true
    });
    // Process data (definitely ready)
}

// The predicate version is equivalent to:
void equivalent() {
    std::unique_lock<std::mutex> lock(mtx);
    while (!condition) {
        cv.wait(lock);
    }
}
```

---

## Thread-Safe Data Structures

### Thread-Safe Queue
```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>

template<typename T>
class ThreadSafeQueue {
private:
    mutable std::mutex mtx_;
    std::condition_variable cv_;
    std::queue<T> queue_;
    
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mtx_);
        queue_.push(std::move(value));
        cv_.notify_one();
    }
    
    std::optional<T> pop() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this]() { return !queue_.empty(); });
        
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
    
    std::optional<T> try_pop() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (queue_.empty()) {
            return std::nullopt;
        }
        
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
    
    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.empty();
    }
    
    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.size();
    }
};
```

---

## Thread Local Storage

### thread_local Variables
```cpp
#include <thread>
#include <iostream>

thread_local int tls_counter = 0;  // Each thread has its own copy

void increment() {
    ++tls_counter;
    std::cout << "Thread " << std::this_thread::get_id() 
              << ": " << tls_counter << '\n';
}

int main() {
    std::thread t1([]() {
        for (int i = 0; i < 3; ++i) increment();
    });
    
    std::thread t2([]() {
        for (int i = 0; i < 3; ++i) increment();
    });
    
    t1.join();
    t2.join();
    
    // Each thread's counter goes 1, 2, 3
    // They don't interfere with each other
    
    return 0;
}
```

---

## Common Patterns

### Double-Checked Locking (Singleton)
```cpp
#include <mutex>
#include <atomic>

class Singleton {
    static std::atomic<Singleton*> instance_;
    static std::mutex mtx_;
    
    Singleton() = default;
    
public:
    static Singleton* getInstance() {
        Singleton* tmp = instance_.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx_);
            tmp = instance_.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new Singleton;
                instance_.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }
};

// C++11: Better way using std::call_once
class BetterSingleton {
    static std::once_flag init_flag_;
    static BetterSingleton* instance_;
    
    BetterSingleton() = default;
    
public:
    static BetterSingleton* getInstance() {
        std::call_once(init_flag_, []() {
            instance_ = new BetterSingleton;
        });
        return instance_;
    }
};

// C++11: Best way using local static (thread-safe)
class BestSingleton {
    BestSingleton() = default;
    
public:
    static BestSingleton& getInstance() {
        static BestSingleton instance;  // Thread-safe in C++11+
        return instance;
    }
};
```

---

## Best Practices

### 1. Always Prefer RAII Locks
```cpp
// BAD — unlock skipped if an exception is thrown
mtx.lock();
// ... work ...
mtx.unlock();

// GOOD — lock released on scope exit, even during stack unwinding
{
    std::lock_guard<std::mutex> lock(mtx);
    // ... work ...
}  // Automatically unlocked
```
See [Exception Handling](17_exceptions.md) for how RAII interacts with exception safety guarantees.

### 2. Keep Critical Sections Small
```cpp
// BAD: Long critical section
{
    std::lock_guard<std::mutex> lock(mtx);
    expensive_computation();  // Holds lock unnecessarily
    shared_data++;
}

// GOOD: Minimal critical section
expensive_computation();  // Do this first
{
    std::lock_guard<std::mutex> lock(mtx);
    shared_data++;  // Quick operation under lock
}
```

### 3. Use Atomics for Simple Counters
```cpp
// GOOD for simple counter
std::atomic<int> counter(0);
++counter;  // Lock-free

// Overkill for simple counter
std::mutex mtx;
int counter = 0;
{
    std::lock_guard<std::mutex> lock(mtx);
    ++counter;
}
```

### 4. Prefer jthread over thread (C++20)
```cpp
// OLD
std::thread t(work);
t.join();  // Must remember

// NEW
std::jthread jt(work);
// Automatically joins
```

---

## Common Pitfalls

### 1. Forgetting to Join/Detach
```cpp
// BAD: Thread destroyed without join/detach
void bad() {
    std::thread t(work);
    // Terminates program if t not joined!
}

// GOOD: Always join or detach
void good() {
    std::thread t(work);
    t.join();
}
```

### 2. Data Race
```cpp
int shared_var = 0;

// BAD: Data race
void increment() {
    ++shared_var;  // Not atomic!
}

// GOOD: Protected access
std::mutex mtx;
void increment_safe() {
    std::lock_guard<std::mutex> lock(mtx);
    ++shared_var;
}

// BETTER: Use atomic
std::atomic<int> atomic_var(0);
void increment_atomic() {
    ++atomic_var;  // Atomic operation
}
```

### 3. Deadlock
```cpp
// BAD: Can deadlock
std::mutex m1, m2;
void thread1() {
    std::lock_guard<std::mutex> lock1(m1);
    std::lock_guard<std::mutex> lock2(m2);
}
void thread2() {
    std::lock_guard<std::mutex> lock2(m2);
    std::lock_guard<std::mutex> lock1(m1);
}

// GOOD: Lock in same order or use scoped_lock
void safe() {
    std::scoped_lock lock(m1, m2);
}
```

---

## Complete Practical Example: Concurrent Web Server Simulator

Here's a comprehensive example integrating threads, synchronization, atomics, and thread-safe data structures:

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <shared_mutex>
#include <atomic>
#include <condition_variable>
#include <queue>
#include <vector>
#include <string>
#include <chrono>
#include <random>
#include <functional>

// Thread-safe request queue
class RequestQueue {
private:
    std::queue<std::string> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    std::atomic<bool> shutdown_{false};
    
public:
    void push(std::string request) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            queue_.push(std::move(request));
        }
        cv_.notify_one();  // Wake up one waiting thread
    }
    
    bool pop(std::string& request, std::chrono::milliseconds timeout) {
        std::unique_lock<std::mutex> lock(mutex_);
        
        // Wait for request or shutdown
        if (!cv_.wait_for(lock, timeout, [this] {
            return !queue_.empty() || shutdown_;
        })) {
            return false;  // Timeout
        }
        
        if (shutdown_ && queue_.empty()) {
            return false;
        }
        
        request = std::move(queue_.front());
        queue_.pop();
        return true;
    }
    
    void shutdown() {
        shutdown_ = true;
        cv_.notify_all();  // Wake all waiting threads
    }
    
    size_t size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.size();
    }
};

// Thread-safe statistics with atomics
class ServerStats {
private:
    std::atomic<uint64_t> total_requests_{0};
    std::atomic<uint64_t> successful_requests_{0};
    std::atomic<uint64_t> failed_requests_{0};
    std::atomic<uint64_t> total_processing_time_ms_{0};
    
public:
    void record_request(bool success, uint64_t processing_time_ms) {
        total_requests_.fetch_add(1, std::memory_order_relaxed);
        
        if (success) {
            successful_requests_.fetch_add(1, std::memory_order_relaxed);
        } else {
            failed_requests_.fetch_add(1, std::memory_order_relaxed);
        }
        
        total_processing_time_ms_.fetch_add(processing_time_ms, 
                                            std::memory_order_relaxed);
    }
    
    void print_stats() const {
        uint64_t total = total_requests_.load(std::memory_order_relaxed);
        uint64_t successful = successful_requests_.load(std::memory_order_relaxed);
        uint64_t failed = failed_requests_.load(std::memory_order_relaxed);
        uint64_t total_time = total_processing_time_ms_.load(std::memory_order_relaxed);
        
        std::cout << "\n=== Server Statistics ===\n";
        std::cout << "Total requests: " << total << "\n";
        std::cout << "Successful: " << successful << "\n";
        std::cout << "Failed: " << failed << "\n";
        
        if (total > 0) {
            std::cout << "Success rate: " 
                      << (100.0 * successful / total) << "%\n";
            std::cout << "Average processing time: " 
                      << (total_time / total) << "ms\n";
        }
    }
};

// Shared resource cache with reader-writer lock
class ResourceCache {
private:
    std::map<std::string, std::string> cache_;
    mutable std::shared_mutex mutex_;
    std::atomic<uint64_t> hits_{0};
    std::atomic<uint64_t> misses_{0};
    
public:
    // Multiple readers can access simultaneously
    bool get(const std::string& key, std::string& value) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            value = it->second;
            hits_.fetch_add(1, std::memory_order_relaxed);
            return true;
        }
        
        misses_.fetch_add(1, std::memory_order_relaxed);
        return false;
    }
    
    // Only one writer at a time
    void put(const std::string& key, std::string value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        cache_[key] = std::move(value);
    }
    
    void print_cache_stats() const {
        uint64_t hits = hits_.load(std::memory_order_relaxed);
        uint64_t misses = misses_.load(std::memory_order_relaxed);
        uint64_t total = hits + misses;
        
        std::cout << "\n=== Cache Statistics ===\n";
        std::cout << "Hits: " << hits << "\n";
        std::cout << "Misses: " << misses << "\n";
        
        if (total > 0) {
            std::cout << "Hit rate: " << (100.0 * hits / total) << "%\n";
        }
    }
};

// Worker thread class
class WorkerThread {
private:
    std::thread thread_;
    int id_;
    RequestQueue& queue_;
    ServerStats& stats_;
    ResourceCache& cache_;
    std::atomic<bool>& running_;
    
    void process_request(const std::string& request) {
        auto start = std::chrono::steady_clock::now();
        
        // Check cache first
        std::string cached_response;
        if (cache_.get(request, cached_response)) {
            std::cout << "Worker " << id_ << ": Cache hit for " 
                      << request << "\n";
        } else {
            // Simulate processing
            std::this_thread::sleep_for(
                std::chrono::milliseconds(10 + (std::rand() % 40))
            );
            
            cached_response = "Response for " + request;
            cache_.put(request, cached_response);
            
            std::cout << "Worker " << id_ << ": Processed " 
                      << request << "\n";
        }
        
        auto end = std::chrono::steady_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
            end - start
        ).count();
        
        // Randomly fail some requests (10%)
        bool success = (std::rand() % 10) != 0;
        stats_.record_request(success, duration);
    }
    
    void run() {
        while (running_.load(std::memory_order_relaxed)) {
            std::string request;
            
            if (queue_.pop(request, std::chrono::milliseconds(100))) {
                try {
                    process_request(request);
                }
                catch (const std::exception& e) {
                    std::cerr << "Worker " << id_ << " error: " 
                              << e.what() << "\n";
                    stats_.record_request(false, 0);
                }
            }
        }
        
        std::cout << "Worker " << id_ << " shutting down\n";
    }
    
public:
    WorkerThread(int id, RequestQueue& queue, ServerStats& stats,
                ResourceCache& cache, std::atomic<bool>& running)
        : id_(id), queue_(queue), stats_(stats)
        , cache_(cache), running_(running) {
        
        thread_ = std::thread(&WorkerThread::run, this);
    }
    
    ~WorkerThread() {
        if (thread_.joinable()) {
            thread_.join();
        }
    }
    
    // Delete copy, allow move
    WorkerThread(const WorkerThread&) = delete;
    WorkerThread& operator=(const WorkerThread&) = delete;
    
    WorkerThread(WorkerThread&& other) noexcept
        : thread_(std::move(other.thread_))
        , id_(other.id_)
        , queue_(other.queue_)
        , stats_(other.stats_)
        , cache_(other.cache_)
        , running_(other.running_) {}
};

// Main server class
class WebServer {
private:
    RequestQueue queue_;
    ServerStats stats_;
    ResourceCache cache_;
    std::vector<WorkerThread> workers_;
    std::atomic<bool> running_{true};
    
public:
    void start(int num_workers) {
        std::cout << "Starting server with " << num_workers << " workers\n";
        
        // Create worker threads
        for (int i = 0; i < num_workers; ++i) {
            workers_.emplace_back(i, queue_, stats_, cache_, running_);
        }
    }
    
    void handle_request(const std::string& request) {
        queue_.push(request);
    }
    
    void stop() {
        std::cout << "\nShutting down server...\n";
        running_ = false;
        queue_.shutdown();
        
        // Wait for all workers
        workers_.clear();
        
        std::cout << "Server stopped\n";
        stats_.print_stats();
        cache_.print_cache_stats();
    }
    
    void wait_until_empty() {
        while (queue_.size() > 0) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
        // Give some extra time for last requests
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
};

// Load generator
void generate_load(WebServer& server, int num_requests) {
    std::vector<std::string> endpoints = {
        "/api/users",
        "/api/products",
        "/api/orders",
        "/api/users",  // Duplicate for cache hits
        "/api/products",  // Duplicate for cache hits
        "/health",
        "/status"
    };
    
    std::cout << "\nGenerating " << num_requests << " requests...\n";
    
    for (int i = 0; i < num_requests; ++i) {
        std::string endpoint = endpoints[std::rand() % endpoints.size()];
        std::string request = endpoint + "?id=" + std::to_string(i);
        
        server.handle_request(request);
        
        // Small delay between requests
        std::this_thread::sleep_for(std::chrono::milliseconds(5));
    }
    
    std::cout << "Finished generating requests\n";
}

int main() {
    std::srand(std::time(nullptr));
    
    WebServer server;
    
    // Start server with 4 worker threads
    server.start(4);
    
    // Simulate client load
    std::thread load_generator([&server]() {
        generate_load(server, 50);
    });
    
    // Let server run
    load_generator.join();
    
    // Wait for queue to empty
    server.wait_until_empty();
    
    // Shutdown
    server.stop();
    
    return 0;
}
```

### Output (Sample):
```
Starting server with 4 workers

Generating 50 requests...
Worker 0: Processed /api/users?id=0
Worker 1: Processed /api/products?id=1
Worker 2: Processed /health?id=2
Worker 3: Processed /api/orders?id=3
Worker 0: Cache hit for /api/users?id=4
Worker 1: Processed /status?id=5
...
Finished generating requests

Shutting down server...
Worker 0 shutting down
Worker 1 shutting down
Worker 2 shutting down
Worker 3 shutting down
Server stopped

=== Server Statistics ===
Total requests: 50
Successful: 45
Failed: 5
Success rate: 90%
Average processing time: 18ms

=== Cache Statistics ===
Hits: 22
Misses: 28
Hit rate: 44%
```

### Concepts Demonstrated:
- **std::thread**: Worker thread pool
- **std::mutex**: Protecting shared queue
- **std::condition_variable**: Thread signaling
- **std::shared_mutex**: Reader-writer lock for cache
- **std::atomic**: Lock-free statistics counters
- **std::unique_lock**: Flexible locking with condition variables
- **std::shared_lock**: Multiple concurrent readers
- **std::lock_guard**: RAII-based locking
- **Thread-safe queue**: Producer-consumer pattern
- **Graceful shutdown**: Coordinating thread termination
- **Memory ordering**: Relaxed ordering for counters
- **Move semantics**: Moving threads between containers
- **RAII**: Automatic thread joining in destructor

This example shows a real-world use case for multithreading!

---

## Related Topics

- [Async and Futures](15_async_futures.md) — `std::async`, futures, and thread pools as higher-level alternatives to raw `std::thread`
- [Advanced Features](12_advanced_features.md) — move semantics, smart pointers, and RAII patterns that underpin safe locking
- [Exception Handling](17_exceptions.md) — why RAII locks matter during stack unwinding; `noexcept` on destructors
- [Best Practices](13_best_practices.md) — const-correctness and design choices that reduce shared-state bugs
- [Modern C++20/23 Features](07_modern_features.md) — `std::jthread`, `std::stop_token`, and other concurrency-related C++20 additions
- [Coroutines](22_coroutines.md) — structured concurrency and async I/O as an alternative to thread-per-task models
- [Quick Reference](99_quick_reference.md) — concurrency primitives at a glance

## Next Steps
- **Next**: [Async and Futures →](15_async_futures.md)
- **Previous**: [← Best Practices](13_best_practices.md)

---
*Chapter 14 — Multithreading and Concurrency*

