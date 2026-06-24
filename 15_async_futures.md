# Asynchronous Programming and Futures

## Overview

C++ provides high-level abstractions for asynchronous programming through futures, promises, and `std::async`. These enable concurrent code without directly managing threads — though thread pools and `std::async` still depend on the primitives in [Multithreading](14_multithreading.md).

```
┌────────────────────────────────────────────────────────┐
│           ASYNCHRONOUS PROGRAMMING TOOLS               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ASYNC          │  FUTURES/PROMISES │  PARALLEL        │
│  ──────         │  ──────────────── │  ────────        │
│  • std::async   │  • std::future    │  • par           │
│  • Policies     │  • std::promise   │  • par_unseq     │
│  • Fire-forget  │  • packaged_task  │  • Algorithms    │
│                 │  • shared_future  │                  │
│                                                        │
│  COROUTINES (C++20)  │  THREAD POOLS                   │
│  ──────────────────  │  ────────────                   │
│  • co_await          │  • Custom pools                 │
│  • co_return         │  • Work stealing                │
│  • co_yield          │  • Task queues                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## std::async

### Basic Usage
```cpp
#include <future>
#include <thread>
#include <chrono>
#include <iostream>

int calculate() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return 42;
}

int main() {
    // Launch asynchronously
    std::future<int> result = std::async(calculate);
    
    // Do other work while calculation runs
    std::cout << "Calculating...\n";
    
    // Get result (blocks if not ready)
    int value = result.get();
    std::cout << "Result: " << value << '\n';
    
    return 0;
}
```

### Launch Policies
```cpp
#include <future>

int work() { return 42; }

int main() {
    // async: Run in separate thread (guaranteed)
    auto f1 = std::async(std::launch::async, work);
    
    // deferred: Run when get() is called (lazy evaluation)
    auto f2 = std::async(std::launch::deferred, work);
    
    // async | deferred: Let implementation choose (default)
    auto f3 = std::async(std::launch::async | std::launch::deferred, work);
    
    // Default policy — same as async | deferred
    auto f4 = std::async(work);
    
    return 0;
}
```

⚠️ **Gotcha**: The `std::future` returned by `std::async` **blocks in its destructor** if the shared state is still asynchronous (i.e., you never called `get()` or `wait()`). This can stall shutdown unexpectedly. Always `get()`/`wait()` before letting the future go out of scope, or use `std::launch::deferred` if you control when work runs.

### Visual: async vs deferred
```
std::launch::async:
main thread:  ───────────►get()──────►
async thread:      ├──work──┤
                   └─ spawns immediately

std::launch::deferred:
main thread:  ───────────►get()──────►
                         └work┤
                   work runs in main thread

std::launch::async | deferred (default):
Implementation chooses — may defer to get() or spawn a thread
```

💡 **Hunch**: For real parallelism, pass `std::launch::async` explicitly. The default policy is a common source of "my code isn't running concurrently" bugs.

### async with Arguments
```cpp
#include <future>
#include <string>

int process(int x, const std::string& s) {
    return x + s.length();
}

int main() {
    // Pass arguments directly
    auto f1 = std::async(process, 10, "hello");
    
    // With lambda
    auto f2 = std::async([](int x) {
        return x * 2;
    }, 21);
    
    // With member function
    class Calculator {
    public:
        int add(int a, int b) { return a + b; }
    };
    
    Calculator calc;
    auto f3 = std::async(&Calculator::add, &calc, 5, 10);
    
    std::cout << f1.get() << '\n';  // 15
    std::cout << f2.get() << '\n';  // 42
    std::cout << f3.get() << '\n';  // 15
    
    return 0;
}
```

---

## std::future

### Future States
```
┌────────────────────────────────────────────────────────┐
│                  Future States                         │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Invalid ──► Valid ──► Ready ──► Retrieved            │
│     │          │         │          │                  │
│     │          │         │          └─ get() called    │
│     │          │         └─ Result available           │
│     │          └─ Waiting for result                   │
│     └─ Default constructed or moved from               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Future Operations
```cpp
#include <future>
#include <chrono>

int slow_calculation() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

int main() {
    auto future = std::async(std::launch::async, slow_calculation);
    
    // Check if valid
    if (future.valid()) {
        std::cout << "Future is valid\n";
    }
    
    // Wait for result
    future.wait();
    std::cout << "Calculation complete\n";
    
    // Wait with timeout
    auto status = future.wait_for(std::chrono::seconds(1));
    
    switch (status) {
        case std::future_status::ready:
            std::cout << "Result ready\n";
            break;
        case std::future_status::timeout:
            std::cout << "Timeout\n";
            break;
        case std::future_status::deferred:
            std::cout << "Deferred (not started)\n";
            break;
    }
    
    // Get result (blocks if not ready)
    int result = future.get();  // Can only call once!
    
    // Future is now invalid
    if (!future.valid()) {
        std::cout << "Future is invalid after get()\n";
    }
    
    return 0;
}
```

### Exception Handling with Futures
```cpp
#include <future>
#include <stdexcept>

int may_throw() {
    throw std::runtime_error("Error in async task");
    return 42;
}

int main() {
    auto future = std::async(may_throw);
    
    try {
        int result = future.get();  // Exception re-thrown here
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }
    
    return 0;
}
```

---

## std::promise

### Promise-Future Pair
```cpp
#include <future>
#include <thread>
#include <chrono>

void calculate(std::promise<int> prom) {
    // Do work
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    // Set result
    prom.set_value(42);
}

int main() {
    // Create promise
    std::promise<int> prom;
    
    // Get associated future
    std::future<int> fut = prom.get_future();
    
    // Start thread with promise
    std::thread t(calculate, std::move(prom));
    
    // Wait for result
    int result = fut.get();
    std::cout << "Result: " << result << '\n';
    
    t.join();
    
    return 0;
}
```

### Visual: Promise-Future Communication
```
Thread 1 (producer):        Thread 2 (consumer):
┌──────────────┐           ┌──────────────┐
│  Promise     │           │   Future     │
│              │           │              │
│ set_value(42)├──────────►│ get() ───► 42│
│              │  transfers│              │
└──────────────┘    value  └──────────────┘

Promise writes → Shared state ← Future reads
```

### Promise with Exceptions
```cpp
#include <future>
#include <thread>

void calculate_or_throw(std::promise<int> prom, bool should_throw) {
    try {
        if (should_throw) {
            throw std::runtime_error("Calculation failed");
        }
        prom.set_value(42);
    } catch (...) {
        // Set exception instead of value
        prom.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();
    
    std::thread t(calculate_or_throw, std::move(prom), true);
    
    try {
        int result = fut.get();  // Exception re-thrown
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << '\n';
    }
    
    t.join();
    
    return 0;
}
```

---

## std::packaged_task

### Wrapping Function for Async Execution
```cpp
#include <future>
#include <thread>
#include <iostream>

int calculate(int x) {
    return x * x;
}

int main() {
    // Create packaged task
    std::packaged_task<int(int)> task(calculate);
    
    // Get future before running task
    std::future<int> result = task.get_future();
    
    // Run task in thread
    std::thread t(std::move(task), 10);
    
    // Get result
    std::cout << "Result: " << result.get() << '\n';  // 100
    
    t.join();
    
    return 0;
}
```

### packaged_task vs async vs promise
```
┌────────────────────────────────────────────────────────┐
│        async vs packaged_task vs promise               │
├────────────────────────────────────────────────────────┤
│                                                        │
│ std::async:                                            │
│   • Simplest interface                                 │
│   • Automatic thread management                        │
│   • Returns future directly                            │
│   Use when: Quick parallel task                        │
│                                                        │
│ std::packaged_task:                                    │
│   • Wraps callable                                     │
│   • Manual thread management                           │
│   • Can store and execute later                        │
│   Use when: Task queues, thread pools                  │
│                                                        │
│ std::promise:                                          │
│   • Low-level primitive                                │
│   • Manual value setting                               │
│   • Most flexible                                      │
│   Use when: Complex communication patterns             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## std::shared_future

### Multiple Readers
```cpp
#include <future>
#include <thread>
#include <vector>

int calculate() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return 42;
}

int main() {
    // Regular future: single consumer
    std::future<int> fut = std::async(calculate);
    
    // Convert to shared_future: multiple consumers
    std::shared_future<int> shared_fut = fut.share();
    
    // Multiple threads can get the value
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back([shared_fut, i]() {
            int value = shared_fut.get();  // All can call get()
            std::cout << "Thread " << i << " got: " << value << '\n';
        });
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    return 0;
}
```

### Visual: future vs shared_future
```
std::future:
Producer ──► Future ──► Consumer
                        (single get())

std::shared_future:
                    ┌──► Consumer 1
                    │
Producer ──► Future ├──► Consumer 2
                    │
                    └──► Consumer 3
                         (multiple get())
```

---

## Parallel Algorithms (C++17)

Execution policies live in `<execution>`. `std::execution::par` and `par_unseq` may run operations concurrently — the callable must not introduce data races, deadlocks, or blocking I/O.

On **libstdc++** (GCC), parallel policies require linking against Intel oneTBB (`-ltbb`) or the bundled libstdc++ parallel backend. libc++ support varies by version. If policies are unavailable, use feature-test macro `__cpp_lib_parallel_algorithm` or fall back to `std::execution::seq`.

### Execution Policies
```cpp
#include <algorithm>
#include <execution>
#include <vector>

int main() {
    std::vector<int> vec(1000000);
    
    // Sequential (default)
    std::sort(vec.begin(), vec.end());
    
    // Parallel execution
    std::sort(std::execution::par, vec.begin(), vec.end());
    
    // Parallel unsequenced (vectorized)
    std::sort(std::execution::par_unseq, vec.begin(), vec.end());
    
    // Sequential (explicit)
    std::sort(std::execution::seq, vec.begin(), vec.end());
    
    return 0;
}
```

### Parallel Algorithm Examples
```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <numeric>

int main() {
    std::vector<int> data(10000000);
    std::iota(data.begin(), data.end(), 0);
    
    // Parallel for_each
    std::for_each(std::execution::par, data.begin(), data.end(),
        [](int& x) { x = x * x; });
    
    // Parallel transform
    std::vector<int> result(data.size());
    std::transform(std::execution::par, 
                   data.begin(), data.end(), result.begin(),
                   [](int x) { return x * 2; });
    
    // Parallel reduce
    int sum = std::reduce(std::execution::par, 
                         data.begin(), data.end());
    
    // Parallel transform_reduce (map-reduce)
    int sum_of_squares = std::transform_reduce(
        std::execution::par,
        data.begin(), data.end(),
        0,  // initial value
        std::plus<>(),  // reduction operation
        [](int x) { return x * x; }  // transformation
    );
    
    return 0;
}
```

### Execution Policy Performance
```
┌───────────────────────────────────────────────────────┐
│            Execution Policy Guidelines                │
├───────────────────────────────────────────────────────┤
│ Policy     │ Parallelism │ Vectorization │ Use when   │
├────────────┼─────────────┼───────────────┼────────────┤
│ seq        │ No          │ No            │ Debug      │
│ par        │ Yes         │ No            │ Safe ops   │
│ par_unseq  │ Yes         │ Yes           │ Fast, safe │
│ unseq      │ No          │ Yes           │ Sequential │
│                                                       │
│ Requirements for par_unseq:                           │
│ • No data races                                       │
│ • No deadlocks                                        │
│ • No blocking operations                              │
│ • Safe for vectorization                              │
└───────────────────────────────────────────────────────┘
```

---

## Thread Pools

`std::async` creates a new thread (or defers) per call — expensive for many small tasks. A thread pool reuses worker threads and is the standard pattern for task queues. See [Multithreading](14_multithreading.md) for the mutex/condition-variable primitives this pattern builds on.

### Simple Thread Pool Implementation
```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <future>

class ThreadPool {
public:
    ThreadPool(size_t threads) : stop_(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers_.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex_);
                        condition_.wait(lock, [this]() {
                            return stop_ || !tasks_.empty();
                        });
                        
                        if (stop_ && tasks_.empty()) {
                            return;
                        }
                        
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    
                    task();
                }
            });
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::invoke_result<F, Args...>::type> {
        
        using return_type = typename std::invoke_result<F, Args...>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> result = task->get_future();
        
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            
            if (stop_) {
                throw std::runtime_error("enqueue on stopped ThreadPool");
            }
            
            tasks_.emplace([task]() { (*task)(); });
        }
        
        condition_.notify_one();
        return result;
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex_);
            stop_ = true;
        }
        
        condition_.notify_all();
        
        for (std::thread& worker : workers_) {
            worker.join();
        }
    }
    
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    
    std::mutex queue_mutex_;
    std::condition_variable condition_;
    bool stop_;
};

// Usage
int main() {
    ThreadPool pool(4);  // 4 worker threads
    
    std::vector<std::future<int>> results;
    
    for (int i = 0; i < 8; ++i) {
        results.emplace_back(
            pool.enqueue([i]() {
                std::this_thread::sleep_for(std::chrono::seconds(1));
                return i * i;
            })
        );
    }
    
    for (auto& result : results) {
        std::cout << result.get() << ' ';
    }
    std::cout << '\n';
    
    return 0;
}
```

### Visual: Thread Pool
```
Task Queue:          Worker Threads:
┌──────────┐        ┌─────────────┐
│ Task 1   │───────►│  Thread 1   │
│ Task 2   │───────►│  Thread 2   │
│ Task 3   │───────►│  Thread 3   │
│ Task 4   │───────►│  Thread 4   │
│ Task 5   │        └─────────────┘
│ Task 6   │              ↓
│  ...     │        ┌─────────────┐
└──────────┘        │   Results   │
                    └─────────────┘

Benefits:
• Reuse threads (avoid creation overhead)
• Control concurrency level
• Queue tasks efficiently
```

---

## Coroutines (C++20)

Coroutines (`co_await`, `co_yield`, `co_return`) offer structured async control flow with very low overhead — ideal for generators and I/O-bound work. This is an introductory taste; see [Coroutines](22_coroutines.md) for promise types, awaitables, and production patterns.

### Basic Coroutine
```cpp
#include <coroutine>
#include <iostream>

struct Generator {
    struct promise_type {
        int current_value;
        
        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void unhandled_exception() {}
        
        std::suspend_always yield_value(int value) {
            current_value = value;
            return {};
        }
        
        void return_void() {}
    };
    
    std::coroutine_handle<promise_type> h_;
    
    ~Generator() { if (h_) h_.destroy(); }
    
    bool move_next() {
        h_.resume();
        return !h_.done();
    }
    
    int current_value() {
        return h_.promise().current_value;
    }
};

Generator counter(int start, int end) {
    for (int i = start; i < end; ++i) {
        co_yield i;  // Suspend and return value
    }
}

int main() {
    auto gen = counter(0, 5);
    
    while (gen.move_next()) {
        std::cout << gen.current_value() << ' ';
    }
    // Output: 0 1 2 3 4
    
    return 0;
}
```

### co_await, co_yield, co_return
```
┌────────────────────────────────────────────────────────┐
│              Coroutine Keywords                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│ co_await expr:                                         │
│   • Suspends coroutine                                 │
│   • Waits for async operation                          │
│   • Resumes when ready                                 │
│                                                        │
│ co_yield value:                                        │
│   • Suspends and returns value                         │
│   • Generator pattern                                  │
│   • Can resume later                                   │
│                                                        │
│ co_return value:                                       │
│   • Returns final value                                │
│   • Completes coroutine                                │
│   • Cannot resume                                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Best Practices

### 1. Use std::async for Simple Cases
```cpp
// GOOD: Simple and clean
auto future = std::async([]() { return expensive_calculation(); });
auto result = future.get();
```

### 2. Use Thread Pools for Many Tasks
```cpp
// BAD: Creating thread for each task
for (int i = 0; i < 1000; ++i) {
    std::thread t([i]() { process(i); });
    t.detach();  // Also bad: no way to get results
}

// GOOD: Use thread pool
ThreadPool pool(std::thread::hardware_concurrency());
std::vector<std::future<int>> results;
for (int i = 0; i < 1000; ++i) {
    results.push_back(pool.enqueue(process, i));
}
```

### 3. Check future Status Before Blocking
```cpp
auto future = std::async(long_calculation);

// Do other work...

// Check if ready before blocking
if (future.wait_for(std::chrono::seconds(0)) == std::future_status::ready) {
    auto result = future.get();
} else {
    // Not ready, do something else
}
```

### 4. Use Parallel Algorithms When Appropriate
```cpp
std::vector<int> data(1000000);

// If operation is CPU-intensive and thread-safe:
std::sort(std::execution::par, data.begin(), data.end());
```

---

## Common Pitfalls

### 1. Blocking in Destructor (std::async)
```cpp
// BAD: future destructor blocks until async task completes
{
    auto future = std::async(std::launch::async, long_task);
    // Leaving scope calls ~future(), which waits for the task!
}

// GOOD: Explicitly get or wait before destruction
{
    auto future = std::async(std::launch::async, long_task);
    future.get();  // Or wait() if you don't need the result
}
```

### 2. Default Launch Policy
```cpp
// BAD: May not actually run concurrently
auto future = std::async(task);  // Might be deferred!

// GOOD: Explicit policy
auto future = std::async(std::launch::async, task);
```

### 3. Calling get() Multiple Times
```cpp
auto future = std::async(calculate);

int result1 = future.get();  // OK
// int result2 = future.get();  // ERROR: Can only call once!

// Use shared_future for multiple gets
auto shared = std::async(calculate).share();
int result1 = shared.get();  // OK
int result2 = shared.get();  // OK
```

### 4. Exception Handling
```cpp
// BAD: Exception stored in shared state; if never retrieved → std::terminate on ~future
auto future = std::async(may_throw);

// GOOD: Always get() and handle exceptions — see [Exception Handling](17_exceptions.md)
try {
    auto result = future.get();
} catch (const std::exception& e) {
    // Handle error
}
```

---

## Performance Comparison

```
┌───────────────────────────────────────────────────────┐
│         Async Approach Performance                    │
├───────────────────────────────────────────────────────┤
│ Approach       │ Overhead │ Control │ Use case        │
├────────────────┼──────────┼─────────┼─────────────────┤
│ std::async     │ Medium   │ Low     │ Simple tasks    │
│ Thread pool    │ Low      │ High    │ Many tasks      │
│ Parallel algo  │ Low      │ None    │ STL operations  │
│ Manual threads │ Low      │ High    │ Custom needs    │
│ Coroutines     │ Very low │ High    │ Async I/O       │
└───────────────────────────────────────────────────────┘
```

---

## Real-World Examples

### Parallel File Processing
```cpp
#include <future>
#include <vector>
#include <filesystem>

std::vector<std::string> process_files(const std::vector<std::string>& files) {
    std::vector<std::future<std::string>> futures;
    
    for (const auto& file : files) {
        futures.push_back(std::async(std::launch::async, [file]() {
            // Process file
            return file + " processed";
        }));
    }
    
    std::vector<std::string> results;
    for (auto& future : futures) {
        results.push_back(future.get());
    }
    
    return results;
}
```

### Timeout Pattern
```cpp
#include <future>
#include <chrono>

template<typename F>
auto run_with_timeout(F func, std::chrono::milliseconds timeout) {
    auto future = std::async(std::launch::async, func);
    
    if (future.wait_for(timeout) == std::future_status::timeout) {
        throw std::runtime_error("Operation timed out");
    }
    
    return future.get();
}
```

---

## Summary

```
Choose based on needs:
• Simple parallel task → std::async
• Many small tasks → Thread pool
• STL operations → Parallel algorithms
• Complex coordination → promise/future
• Generator pattern → Coroutines
```

---

## Complete Practical Example: Parallel Image Processing Pipeline

Here's a comprehensive example integrating async, futures, promises, and parallel algorithms:

```cpp
#include <iostream>
#include <future>
#include <vector>
#include <string>
#include <chrono>
#include <algorithm>
#include <execution>
#include <numeric>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>

// Simulated image data
struct Image {
    std::string name;
    int width, height;
    std::vector<uint8_t> pixels;
    
    Image(std::string n, int w, int h)
        : name(std::move(n)), width(w), height(h)
        , pixels(w * h * 3, 128) {}  // RGB
};

// Image processing operations
class ImageProcessor {
public:
    // Simulate expensive operation
    static Image resize(const Image& img, int new_width, int new_height) {
        std::cout << "  [Resizing " << img.name << "]\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        
        return Image{img.name + "_resized", new_width, new_height};
    }
    
    static Image apply_filter(const Image& img, const std::string& filter) {
        std::cout << "  [Applying " << filter << " to " << img.name << "]\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(150));
        
        return Image{img.name + "_" + filter, img.width, img.height};
    }
    
    static Image compress(const Image& img, int quality) {
        std::cout << "  [Compressing " << img.name << " (Q=" << quality << ")]\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(80));
        
        return Image{img.name + "_compressed", img.width, img.height};
    }
    
    static bool save(const Image& img, const std::string& path) {
        std::cout << "  [Saving " << img.name << " to " << path << "]\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        return true;
    }
};

// 1. Async task launching
void demo_async_basics() {
    std::cout << "\n=== std::async Basics ===\n";
    
    Image img("photo.jpg", 1920, 1080);
    
    // Launch async task
    auto future_resized = std::async(std::launch::async,
        ImageProcessor::resize, img, 800, 600);
    
    std::cout << "Resize task launched, doing other work...\n";
    
    // Do other work while resizing happens
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    
    // Get result (blocks if not ready)
    Image resized = future_resized.get();
    std::cout << "Got result: " << resized.name << "\n";
}

// 2. Multiple concurrent tasks
void demo_concurrent_processing() {
    std::cout << "\n=== Concurrent Processing ===\n";
    
    std::vector<Image> images = {
        Image("img1.jpg", 1920, 1080),
        Image("img2.jpg", 1920, 1080),
        Image("img3.jpg", 1920, 1080),
        Image("img4.jpg", 1920, 1080)
    };
    
    auto start = std::chrono::steady_clock::now();
    
    // Launch multiple async tasks
    std::vector<std::future<Image>> futures;
    
    for (const auto& img : images) {
        futures.push_back(std::async(std::launch::async,
            ImageProcessor::resize, img, 800, 600));
    }
    
    // Wait for all results
    std::vector<Image> resized_images;
    for (auto& future : futures) {
        resized_images.push_back(future.get());
    }
    
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Processed " << images.size() << " images in " 
              << duration.count() << "ms (parallel)\n";
}

// 3. Promise and Future for custom async operations
void demo_promise_future() {
    std::cout << "\n=== Promise and Future ===\n";
    
    std::promise<Image> promise;
    std::future<Image> future = promise.get_future();
    
    // Producer thread
    std::thread producer([&promise]() {
        std::cout << "  [Producer: Starting work...]\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        
        Image img("async_image.jpg", 1024, 768);
        std::cout << "  [Producer: Setting value]\n";
        promise.set_value(img);
    });
    
    // Consumer
    std::cout << "Consumer: Waiting for result...\n";
    Image result = future.get();  // Blocks until promise.set_value()
    std::cout << "Consumer: Got " << result.name << "\n";
    
    producer.join();
}

// 4. Processing pipeline with futures
void demo_processing_pipeline() {
    std::cout << "\n=== Processing Pipeline ===\n";
    
    Image img("original.jpg", 1920, 1080);
    
    // Chain async operations
    auto future1 = std::async(std::launch::async,
        ImageProcessor::resize, img, 800, 600);
    
    auto future2 = std::async(std::launch::async, [](std::future<Image>& f) {
        Image resized = f.get();
        return ImageProcessor::apply_filter(resized, "blur");
    }, std::ref(future1));
    
    auto future3 = std::async(std::launch::async, [](std::future<Image>& f) {
        Image filtered = f.get();
        return ImageProcessor::compress(filtered, 85);
    }, std::ref(future2));
    
    Image final_image = future3.get();
    std::cout << "Pipeline complete: " << final_image.name << "\n";
}

// 5. Parallel algorithms
void demo_parallel_algorithms() {
    std::cout << "\n=== Parallel Algorithms ===\n";
    
    // Generate large dataset
    std::vector<int> data(1000000);
    std::iota(data.begin(), data.end(), 1);
    
    auto start = std::chrono::steady_clock::now();
    
    // Parallel transform
    std::transform(std::execution::par,
        data.begin(), data.end(), data.begin(),
        [](int x) { return x * x; });
    
    // Parallel reduce
    long long sum = std::reduce(std::execution::par,
        data.begin(), data.end(), 0LL);
    
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    
    std::cout << "Parallel processing: sum = " << sum 
              << " (took " << duration.count() << "ms)\n";
}

// 6. Thread pool implementation
class ThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool stop_ = false;
    
public:
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock<std::mutex> lock(mutex_);
                        cv_.wait(lock, [this]() {
                            return stop_ || !tasks_.empty();
                        });
                        
                        if (stop_ && tasks_.empty()) return;
                        
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    
                    task();
                }
            });
        }
    }
    
    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        cv_.notify_all();
        
        for (auto& worker : workers_) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) 
        -> std::future<typename std::invoke_result<F, Args...>::type> {
        // std::result_of was deprecated in C++17 and REMOVED in C++20;
        // use std::invoke_result instead.
        using return_type = typename std::invoke_result<F, Args...>::type;
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> future = task->get_future();
        
        {
            std::unique_lock<std::mutex> lock(mutex_);
            if (stop_) {
                throw std::runtime_error("ThreadPool is stopped");
            }
            tasks_.emplace([task]() { (*task)(); });
        }
        
        cv_.notify_one();
        return future;
    }
};

void demo_thread_pool() {
    std::cout << "\n=== Thread Pool ===\n";
    
    ThreadPool pool(4);
    
    std::vector<std::future<Image>> results;
    
    // Submit tasks to pool
    for (int i = 0; i < 6; ++i) {
        Image img("img" + std::to_string(i) + ".jpg", 1920, 1080);
        
        results.push_back(pool.enqueue(
            ImageProcessor::resize, img, 800, 600
        ));
    }
    
    std::cout << "Submitted " << results.size() << " tasks to pool\n";
    
    // Get results
    for (size_t i = 0; i < results.size(); ++i) {
        Image result = results[i].get();
        std::cout << "Task " << i << " complete: " << result.name << "\n";
    }
}

// 7. Batch processing with futures
void demo_batch_processing() {
    std::cout << "\n=== Batch Processing ===\n";
    
    std::vector<Image> images;
    for (int i = 0; i < 10; ++i) {
        images.emplace_back("batch_" + std::to_string(i) + ".jpg", 1920, 1080);
    }
    
    // Process in batches
    const size_t batch_size = 3;
    std::vector<std::future<std::vector<Image>>> batch_futures;
    
    for (size_t i = 0; i < images.size(); i += batch_size) {
        size_t end = std::min(i + batch_size, images.size());
        
        std::vector<Image> batch(images.begin() + i, images.begin() + end);
        
        batch_futures.push_back(std::async(std::launch::async,
            [](std::vector<Image> imgs) {
                std::vector<Image> processed;
                for (const auto& img : imgs) {
                    processed.push_back(ImageProcessor::resize(img, 800, 600));
                }
                return processed;
            }, batch));
    }
    
    // Collect results
    int total_processed = 0;
    for (auto& future : batch_futures) {
        auto batch_result = future.get();
        total_processed += batch_result.size();
    }
    
    std::cout << "Processed " << total_processed << " images in batches\n";
}

// 8. Error handling with futures
void demo_error_handling() {
    std::cout << "\n=== Error Handling with Futures ===\n";
    
    auto future = std::async(std::launch::async, []() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        throw std::runtime_error("Simulated processing error");
        return Image("never_reached.jpg", 800, 600);
    });
    
    try {
        Image result = future.get();  // Rethrows exception
        std::cout << "Got result: " << result.name << "\n";
    }
    catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << "\n";
    }
}

int main() {
    std::cout << "=== Async Programming Demo ===\n";
    
    // 1. Async basics
    demo_async_basics();
    
    // 2. Concurrent processing
    demo_concurrent_processing();
    
    // 3. Promise and future
    demo_promise_future();
    
    // 4. Processing pipeline
    demo_processing_pipeline();
    
    // 5. Parallel algorithms
    demo_parallel_algorithms();
    
    // 6. Thread pool
    demo_thread_pool();
    
    // 7. Batch processing
    demo_batch_processing();
    
    // 8. Error handling
    demo_error_handling();
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **std::async**: Launching async tasks
- **std::future**: Getting async results
- **std::promise**: Setting values from another thread
- **std::packaged_task**: Wrapping functions for async execution
- **Launch policies**: async vs deferred
- **Parallel algorithms**: `std::execution::par`
- **Thread pool**: Custom implementation with futures
- **Pipeline processing**: Chaining async operations
- **Batch processing**: Processing groups concurrently
- **Error handling**: Exception propagation through futures
- **wait_for/wait_until**: Timeout operations
- **std::shared_future**: Multiple waiters
- **Concurrent task coordination**: Multiple futures

This example shows real-world async programming patterns!

---

## Related Topics

- [Multithreading](14_multithreading.md) — `std::thread`, mutexes, and condition variables underlying pools and `std::async`
- [Algorithms](06_algorithms.md) — STL algorithms that parallel policies accelerate (`sort`, `transform`, `reduce`)
- [Coroutines](22_coroutines.md) — deep dive into `co_await`, promise types, and async I/O patterns
- [Exception Handling](17_exceptions.md) — exception propagation through futures; `std::future_error`
- [Lambdas](10_lambdas.md) — lambdas are the usual way to pass work to `std::async` and thread pools
- [I/O and Filesystem](16_io_filesystem.md) — parallel file processing and I/O-bound async workloads
- [Quick Reference](99_quick_reference.md) — async primitives at a glance

## Next Steps
- **Next**: [I/O and Filesystem →](16_io_filesystem.md)
- **Previous**: [← Multithreading](14_multithreading.md)

---
*Chapter 15 — Asynchronous Programming and Futures*

