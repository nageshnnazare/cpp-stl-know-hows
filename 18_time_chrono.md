# Time and Chrono Library

## Overview

C++11 introduced the chrono library for type-safe time operations, and C++20 added calendar and timezone support. This chapter covers durations, time points, clocks, calendar/timezone handling, formatting, and benchmarking. For threading timeouts and async work, see [Multithreading](14_multithreading.md) and [Async & Futures](15_async_futures.md).

```
┌───────────────────────────────────────────────────────┐
│              CHRONO LIBRARY COMPONENTS                │
├───────────────────────────────────────────────────────┤
│                                                       │
│  DURATIONS      │  TIME POINTS     │  CLOCKS          │
│  ─────────      │  ───────────     │  ──────          │
│  • seconds      │  • time_point    │  • system_clock  │
│  • milliseconds │  • epoch         │  • steady_clock  │
│  • microseconds │  • arithmetic    │  • high_res...   │
│  • Custom       │                  │  • utc_clock     │
│                                                       │
│  CALENDAR (C++20)          │  TIMEZONE (C++20)        │
│  ────────────────          │  ───────────────         │
│  • year_month_day          │  • time_zone             │
│  • weekday                 │  • zoned_time            │
│  • month, year             │  • tzdb                  │
│                                                       │
└───────────────────────────────────────────────────────┘
```

---

## Durations

### Basic Durations

A `std::chrono::duration` is `duration<Rep, Period>` where `Period` is a `std::ratio` of seconds (e.g. `std::ratio<1>` = seconds, `std::ratio<1, 1000>` = milliseconds). Use [user-defined literals](07_modern_features.md) via `using namespace std::chrono_literals` for readable constants (`1s`, `100ms`, `2024y`).

```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;
using namespace std::chrono_literals;

int main() {
    // Predefined durations
    auto h = hours(1);
    auto m = minutes(60);
    auto s = seconds(3600);
    auto ms = milliseconds(1000);
    auto us = microseconds(1000000);
    auto ns = nanoseconds(1000000000);
    
    // Literals (requires std::chrono_literals)
    auto one_hour = 1h;
    auto half_sec = 500ms;
    
    // All represent 1 hour
    std::cout << "1 hour in different units:\n";
    std::cout << h.count() << " hours\n";
    std::cout << duration_cast<minutes>(h).count() << " minutes\n";
    std::cout << duration_cast<seconds>(h).count() << " seconds\n";
    
    return 0;
}
```

### Duration Arithmetic
```cpp
#include <chrono>

using namespace std::chrono;

int main() {
    auto d1 = seconds(10);
    auto d2 = seconds(5);
    
    // Addition
    auto sum = d1 + d2;  // 15 seconds
    
    // Subtraction
    auto diff = d1 - d2;  // 5 seconds
    
    // Multiplication
    auto doubled = d1 * 2;  // 20 seconds
    auto tripled = 3 * d1;  // 30 seconds
    
    // Division
    auto half = d1 / 2;     // 5 seconds
    auto ratio = d1 / d2;   // 2 (ratio, not duration)
    
    // Modulo
    auto remainder = d1 % d2;  // 0 seconds
    
    // Comparison
    bool less = d1 < d2;
    bool equal = d1 == d2;
    
    return 0;
}
```

### Duration Conversion
```cpp
#include <chrono>
#include <ratio>

using namespace std::chrono;

int main() {
    // Implicit conversion: only when exact (no truncation)
    seconds s = hours(1);  // OK: 3600 seconds exactly
    
    // duration_cast: required for lossy / truncating conversions
    auto h = duration_cast<hours>(seconds(3661));  // 1 hour (loses 61 seconds)
    
    // floor, ceil, round (C++17)
    auto s1 = floor<seconds>(milliseconds(1500));   // 1 second
    auto s2 = ceil<seconds>(milliseconds(1500));    // 2 seconds
    auto s3 = round<seconds>(milliseconds(1500));   // 2 seconds
    
    // Custom duration (Period is a ratio of seconds)
    using days = duration<int, std::ratio<86400>>;
    days d(7);  // 7 days
    auto hours_in_week = duration_cast<hours>(d);  // 168 hours
    
    return 0;
}
```

### Visual: Duration Conversion
```
Implicit (safe):
hours ──► minutes ──► seconds ──► milliseconds ──► microseconds
  1       60         3600       3,600,000       3,600,000,000

Explicit (may truncate):
microseconds ──► milliseconds ──► seconds ──► minutes ──► hours
3,600,000,000    3,600,000       3600         60          1
                  (duration_cast required)
```

---

## Time Points

### Basic Time Points
```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;

int main() {
    // Current time
    auto now = system_clock::now();
    
    // Time point arithmetic
    auto future = now + hours(24);      // 24 hours from now
    auto past = now - minutes(30);      // 30 minutes ago
    
    // Difference between time points (gives duration)
    auto elapsed = future - now;        // duration: 24 hours
    
    // Time since epoch
    auto since_epoch = now.time_since_epoch();
    std::cout << "Milliseconds since epoch: " 
              << duration_cast<milliseconds>(since_epoch).count() << '\n';
    
    return 0;
}
```

### Time Point Conversions
```cpp
#include <chrono>
#include <ctime>

using namespace std::chrono;

int main() {
    // C++11 time_point to C time_t
    auto now = system_clock::now();
    std::time_t now_c = system_clock::to_time_t(now);
    
    std::cout << "C time: " << std::ctime(&now_c);
    
    // C time_t to C++11 time_point
    std::time_t some_time = std::time(nullptr);
    auto tp = system_clock::from_time_t(some_time);
    
    // Format time (traditional)
    std::tm* tm = std::localtime(&now_c);
    char buffer[80];
    std::strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", tm);
    std::cout << "Formatted: " << buffer << '\n';
    
    return 0;
}
```

---

## Clocks

### Three Standard Clocks

```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;

int main() {
    // system_clock: wall-clock time (can jump backward/forward with NTP, DST, manual adjustment)
    auto sys_now = system_clock::now();
    
    // steady_clock: monotonic — use for elapsed time and benchmarks
    auto steady_now = steady_clock::now();
    
    // high_resolution_clock: often an alias of system_clock or steady_clock — avoid relying on it
    auto hires_now = high_resolution_clock::now();
    
    std::cout << "system_clock steady: " << system_clock::is_steady << '\n';
    std::cout << "steady_clock steady: " << steady_clock::is_steady << '\n';
    std::cout << "high_resolution_clock steady: " << high_resolution_clock::is_steady << '\n';
    
    return 0;
}
```

⚠️ **Gotchas**: `high_resolution_clock` is implementation-defined and may alias `system_clock` (non-monotonic) or `steady_clock`. Prefer `steady_clock` for timing; use `system_clock` only when you need wall time or interoperability with `time_t` / [filesystem](16_io_filesystem.md) timestamps.

### Clock Comparison
```
┌───────────────────────────────────────────────────────┐
│                Clock Characteristics                  │
├───────────────────────────────────────────────────────┤
│ Clock         │ Monotonic │ Adjustable │ Use Case     │
├───────────────┼───────────┼────────────┼──────────────┤
│ system_clock  │ No        │ Yes        │ Wall time    │
│ steady_clock  │ Yes       │ No         │ Timing/perf  │
│ high_res...   │ Maybe     │ Maybe      │ Precision    │
│ utc_clock*    │ No        │ Yes        │ UTC time     │
│ tai_clock*    │ Yes       │ No         │ TAI time     │
│ gps_clock*    │ Yes       │ No         │ GPS time     │
│ file_clock*   │ No        │ Yes        │ File times   │
│                                                       │
│ * C++20                                               │
│                                                       │
│ Use system_clock for: Displaying time to users        │
│ Use steady_clock for: Measuring elapsed time          │
│ Avoid high_resolution_clock for timing (alias risk)   │
└───────────────────────────────────────────────────────┘
```

---

## Timing and Benchmarking

Use `steady_clock` for all elapsed-time measurements. Cast the resulting `duration` to the unit you need (`milliseconds`, `microseconds`, etc.) — see [Algorithms](06_algorithms.md) for profiling hot loops.

### Measuring Execution Time
```cpp
#include <chrono>
#include <iostream>
#include <thread>

using namespace std::chrono;
using namespace std::chrono_literals;

void expensive_operation() {
    std::this_thread::sleep_for(100ms);
}

int main() {
    auto start = steady_clock::now();
    expensive_operation();
    auto elapsed = steady_clock::now() - start;
    
    std::cout << "Elapsed: "
              << duration_cast<milliseconds>(elapsed).count() << " ms\n";
    std::cout << "Elapsed: "
              << duration_cast<microseconds>(elapsed).count() << " µs\n";
    
    return 0;
}
```

### RAII Timer Class
```cpp
#include <chrono>
#include <iostream>
#include <string>
#include <thread>

class Timer {
    std::string name_;
    std::chrono::steady_clock::time_point start_;
    
public:
    explicit Timer(std::string name) 
        : name_(std::move(name))
        , start_(std::chrono::steady_clock::now()) {}
    
    ~Timer() {
        auto end = std::chrono::steady_clock::now();
        auto elapsed = end - start_;
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(elapsed);
        
        std::cout << name_ << " took " << ms.count() << " ms\n";
    }
};

void function_to_time() {
    Timer t("function_to_time");
    // Function code here
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
}  // Timer destroyed, prints elapsed time

int main() {
    function_to_time();
    return 0;
}
```

---

## Calendar Types (C++20)

C++20 adds type-safe calendar types (`year`, `month`, `day`, `year_month_day`) instead of manual `tm` arithmetic. Check `__cpp_lib_chrono >= 201907L` — Apple Clang/libc++ may lag; fall back to `strftime` or a third-party library when unavailable.

### Year, Month, Day
```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;
using namespace std::chrono_literals;

int main() {
    // Construct date
    auto date1 = year_month_day(year(2024), month(3), day(15));
    auto date2 = 2024y / March / 15d;  // Using literals
    auto date3 = 2024y / 3 / 15;
    
    // Extract components
    auto y = date1.year();
    auto m = date1.month();
    auto d = date1.day();
    
    std::cout << "Year: " << static_cast<int>(y) << '\n';
    std::cout << "Month: " << static_cast<unsigned>(m) << '\n';
    std::cout << "Day: " << static_cast<unsigned>(d) << '\n';
    
    // Check validity
    if (date1.ok()) {
        std::cout << "Valid date\n";
    }
    
    // Invalid date
    auto invalid = 2024y / February / 30d;
    if (!invalid.ok()) {
        std::cout << "Invalid date\n";
    }
    
    return 0;
}
```

### Weekday Operations
```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;

int main() {
    // Get weekday for a date
    auto date = 2024y / December / 4d;
    auto wd = weekday(sys_days(date));
    
    std::cout << "Weekday: " << wd << '\n';  // Wednesday
    
    // Weekday arithmetic
    auto tomorrow = wd + days(1);
    auto yesterday = wd - days(1);
    
    // Check specific weekday
    if (wd == Wednesday) {
        std::cout << "It's Wednesday!\n";
    }
    
    // Last weekday of month
    auto last_sunday = 2024y / December / Sunday[last];
    
    // Nth weekday of month
    auto second_monday = 2024y / December / Monday[2];
    
    // Distance between weekdays
    auto days_diff = Wednesday - Monday;  // 2 days
    
    return 0;
}
```

### Date Arithmetic
```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;
using namespace std::chrono_literals;

int main() {
    auto date = 2024y / March / 15d;
    
    // Add days
    auto next_week = sys_days(date) + days(7);
    auto next_month = sys_days(date) + months(1);
    auto next_year = sys_days(date) + years(1);
    
    // Subtract dates
    auto date1 = 2024y / December / 31d;
    auto date2 = 2024y / January / 1d;
    auto diff = sys_days(date1) - sys_days(date2);  // duration in days
    
    std::cout << "Days in 2024: " << diff.count() << '\n';  // 365
    
    // Last day of month
    auto last = year_month_day_last(2024y / February / last);
    auto last_day = last.day();  // 29 (leap year)
    
    return 0;
}
```

---

## Time Zones (C++20)

Timezone support uses the IANA tz database via `get_tzdb()` and `std::chrono::zoned_time`. Requires OS tzdata and full libc++ implementation — verify `__cpp_lib_chrono >= 201907L` on your platform.

### Basic Timezone Operations
```cpp
#include <chrono>
#include <iostream>

using namespace std::chrono;
using namespace std::chrono_literals;

int main() {
    // Get timezone database
    const auto& db = get_tzdb();
    
    // Get specific timezone
    const auto* tz_ny = db.locate_zone("America/New_York");
    const auto* tz_tokyo = db.locate_zone("Asia/Tokyo");
    const auto* tz_utc = db.locate_zone("UTC");
    
    // Current time in different zones
    auto now = system_clock::now();
    
    auto time_ny = zoned_time(tz_ny, now);
    auto time_tokyo = zoned_time(tz_tokyo, now);
    auto time_utc = zoned_time(tz_utc, now);
    
    std::cout << "New York: " << time_ny << '\n';
    std::cout << "Tokyo: " << time_tokyo << '\n';
    std::cout << "UTC: " << time_utc << '\n';
    
    return 0;
}
```

### Timezone Conversions
```cpp
#include <chrono>

using namespace std::chrono;

int main() {
    const auto& db = get_tzdb();
    
    // Create time in one zone
    auto tz_la = db.locate_zone("America/Los_Angeles");
    auto time_la = zoned_time(tz_la, local_days(2024y/March/15d) + 10h);
    
    // Convert to another zone
    auto tz_paris = db.locate_zone("Europe/Paris");
    auto time_paris = zoned_time(tz_paris, time_la);
    
    std::cout << "LA time: " << time_la << '\n';
    std::cout << "Paris time: " << time_paris << '\n';
    
    return 0;
}
```

---

## Formatting Time (C++20)

C++20 chrono types integrate with `std::format` (and C++23 `std::print` where available). This replaces brittle `strftime`/`ostringstream` chains for many use cases.

### std::format with Chrono
```cpp
#include <chrono>
#include <format>
#include <iostream>

using namespace std::chrono;
using namespace std::chrono_literals;

int main() {
    auto now = system_clock::now();
    auto today = floor<days>(now);
    
    // Format time point
    std::cout << std::format("Current time: {}\n", now);
    std::cout << std::format("Date: {:%Y-%m-%d}\n", today);
    std::cout << std::format("Time: {:%H:%M:%S}\n", now);
    std::cout << std::format("Full: {:%Y-%m-%d %H:%M:%S}\n", now);
    
    // Format duration
    auto duration = 1h + 23min + 45s;
    std::cout << std::format("Duration: {}\n", duration);
    std::cout << std::format("Duration: {}h {}m {}s\n",
                            duration_cast<hours>(duration).count(),
                            duration_cast<minutes>(duration % 1h).count(),
                            duration_cast<seconds>(duration % 1min).count());
    
    // Format calendar types
    auto date = 2024y / March / 15d;
    std::cout << std::format("Date: {}\n", date);
    std::cout << std::format("Date: {:%A, %B %d, %Y}\n", sys_days(date));
    
    return 0;
}
```

### Custom Date Formatting
```cpp
#include <chrono>
#include <format>
#include <iostream>

using namespace std::chrono;

std::string format_date_custom(const year_month_day& date) {
    return std::format("{:%Y-%m-%d}", sys_days(date));
}

std::string format_time_custom(const system_clock::time_point& tp) {
    return std::format("{:%H:%M:%S}", floor<seconds>(tp));
}

std::string format_datetime_custom(const system_clock::time_point& tp) {
    return std::format("{:%Y-%m-%d %H:%M:%S}", floor<seconds>(tp));
}

int main() {
    auto now = system_clock::now();
    auto today = year_month_day(floor<days>(now));
    
    std::cout << format_date_custom(today) << '\n';
    std::cout << format_time_custom(now) << '\n';
    std::cout << format_datetime_custom(now) << '\n';
    
    return 0;
}
```

---

## Common Patterns

### Timeout Implementation

See [Async & Futures](15_async_futures.md) for `std::async` patterns and cancellation trade-offs.

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <thread>

using namespace std::chrono;
using namespace std::chrono_literals;

template<typename Func>
bool run_with_timeout(Func func, milliseconds timeout) {
    auto future = std::async(std::launch::async, func);
    return future.wait_for(timeout) == std::future_status::ready;
}

void example() {
    bool completed = run_with_timeout([]() {
        std::this_thread::sleep_for(seconds(1));
    }, seconds(2));
    
    if (completed) {
        std::cout << "Completed within timeout\n";
    } else {
        std::cout << "Timed out\n";
    }
}
```

### Rate Limiting
```cpp
#include <chrono>
#include <thread>

using namespace std::chrono;

class RateLimiter {
    steady_clock::time_point last_call_;
    milliseconds min_interval_;
    
public:
    explicit RateLimiter(milliseconds interval)
        : last_call_(steady_clock::now())
        , min_interval_(interval) {}
    
    void wait_if_needed() {
        auto now = steady_clock::now();
        auto elapsed = now - last_call_;
        
        if (elapsed < min_interval_) {
            std::this_thread::sleep_for(min_interval_ - elapsed);
        }
        
        last_call_ = steady_clock::now();
    }
};

void example() {
    RateLimiter limiter(milliseconds(100));  // Max 10 calls per second
    
    for (int i = 0; i < 5; ++i) {
        limiter.wait_if_needed();
        std::cout << "API call " << i << '\n';
    }
}
```

### Performance Profiler

For production profiling, pair this pattern with [associative containers](02_associative_containers.md) or external tools; the RAII scope timer is the reusable core.

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <string>
#include <thread>

class Profiler {
    struct Stat {
        size_t calls = 0;
        std::chrono::nanoseconds total{0};
    };
    
    std::map<std::string, Stat> stats_;
    
public:
    class ScopedTimer {
        Profiler& profiler_;
        std::string name_;
        std::chrono::steady_clock::time_point start_;
        
    public:
        ScopedTimer(Profiler& p, std::string name)
            : profiler_(p)
            , name_(std::move(name))
            , start_(std::chrono::steady_clock::now()) {}
        
        ~ScopedTimer() {
            auto elapsed = std::chrono::steady_clock::now() - start_;
            profiler_.record(name_, elapsed);
        }
    };
    
    void record(const std::string& name, std::chrono::nanoseconds duration) {
        auto& stat = stats_[name];
        stat.calls++;
        stat.total += duration;
    }
    
    ScopedTimer time(std::string name) {
        return ScopedTimer(*this, std::move(name));
    }
    
    void report() const {
        using namespace std::chrono;
        
        for (const auto& [name, stat] : stats_) {
            auto avg = stat.total / stat.calls;
            std::cout << name << ": "
                      << stat.calls << " calls, "
                      << "total: " << duration_cast<milliseconds>(stat.total).count() << "ms, "
                      << "avg: " << duration_cast<microseconds>(avg).count() << "µs\n";
        }
    }
};

void example() {
    Profiler profiler;
    
    {
        auto t = profiler.time("operation1");
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    
    {
        auto t = profiler.time("operation2");
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
    }
    
    profiler.report();
}
```

---

## Best Practices

### 1. Use steady_clock for Timing
```cpp
// GOOD: Monotonic, not affected by system clock changes
auto start = std::chrono::steady_clock::now();
expensive_operation();
auto elapsed = std::chrono::steady_clock::now() - start;

// BAD: Can go backward if system time is adjusted
auto start = std::chrono::system_clock::now();
```

### 2. Prefer Chrono Over C Time Functions
```cpp
// GOOD: Type-safe, portable
auto now = std::chrono::system_clock::now();
auto duration = std::chrono::hours(24);

// BAD: Not type-safe, platform-dependent
time_t now = time(nullptr);
struct timespec ts;
clock_gettime(CLOCK_MONOTONIC, &ts);
```

### 3. Use Appropriate Precision
```cpp
// Don't use more precision than needed
auto start = steady_clock::now();
// ...
auto ms = duration_cast<milliseconds>(steady_clock::now() - start);

// Not: duration_cast<nanoseconds>(...)  if milliseconds sufficient
```

### 4. Be Aware of Leap Seconds and Platform Gaps
```cpp
// system_clock may include leap seconds (implementation-defined)
// For precise time intervals, use steady_clock
// For UTC time, use utc_clock (C++20)

// Apple Clang/libc++ may lack full C++20 chrono tz/format — check macros:
// __cpp_lib_chrono, __cpp_lib_format
```

---

## Common Pitfalls

### 1. Using system_clock for Intervals
```cpp
// BAD: Can be affected by time adjustments
auto start = system_clock::now();
// ... if system time changes here ...
auto end = system_clock::now();
auto elapsed = end - start;  // May be wrong!

// GOOD: Use steady_clock
auto start = steady_clock::now();
auto end = steady_clock::now();
auto elapsed = end - start;  // Always correct
```

### 2. Integer Truncation
```cpp
auto ms = milliseconds(1500);
auto s = duration_cast<seconds>(ms);  // 1 (truncates)

// Use floor, ceil, or round for clarity
auto s_floor = floor<seconds>(ms);    // 1
auto s_ceil = ceil<seconds>(ms);      // 2
auto s_round = round<seconds>(ms);    // 2
```

### 3. Forgetting to Check Date Validity
```cpp
auto date = 2024y / February / 30d;  // Invalid!

if (!date.ok()) {
    // Handle invalid date
}
```

---

## Complete Practical Example: Performance Monitor and Scheduler

Here's a comprehensive example integrating durations, time points, clocks, and timing:

```cpp
#include <functional>
#include <iomanip>
#include <iostream>
#include <map>
#include <queue>
#include <sstream>
#include <string>
#include <thread>
#include <vector>
#include <chrono>

using namespace std::chrono;

// 1. Performance timer with RAII
class ScopedTimer {
private:
    std::string name_;
    steady_clock::time_point start_;
    
public:
    explicit ScopedTimer(std::string name)
        : name_(std::move(name))
        , start_(steady_clock::now()) {}
    
    ~ScopedTimer() {
        auto end = steady_clock::now();
        auto duration_ms = duration_cast<milliseconds>(end - start_);
        auto duration_us = duration_cast<microseconds>(end - start_);
        
        std::cout << "[TIMER] " << name_ << " took "
                  << duration_ms.count() << "ms ("
                  << duration_us.count() << "µs)\n";
    }
};

// 2. Rate limiter using steady_clock
class RateLimiter {
private:
    milliseconds min_interval_;
    steady_clock::time_point last_call_;
    
public:
    explicit RateLimiter(milliseconds interval)
        : min_interval_(interval)
        , last_call_(steady_clock::now() - interval) {}
    
    bool try_acquire() {
        auto now = steady_clock::now();
        auto elapsed = duration_cast<milliseconds>(now - last_call_);
        
        if (elapsed >= min_interval_) {
            last_call_ = now;
            return true;
        }
        return false;
    }
    
    void wait_and_acquire() {
        auto now = steady_clock::now();
        auto elapsed = duration_cast<milliseconds>(now - last_call_);
        
        if (elapsed < min_interval_) {
            std::this_thread::sleep_for(min_interval_ - elapsed);
        }
        
        last_call_ = steady_clock::now();
    }
};

// 3. Scheduled task system
struct ScheduledTask {
    std::string name;
    std::function<void()> func;
    system_clock::time_point scheduled_time;
    
    bool operator<(const ScheduledTask& other) const {
        return scheduled_time > other.scheduled_time;  // Min heap
    }
};

class TaskScheduler {
private:
    std::priority_queue<ScheduledTask> tasks_;
    
public:
    void schedule(std::string name, std::function<void()> func,
                 milliseconds delay_from_now) {
        auto scheduled_time = system_clock::now() + delay_from_now;
        tasks_.push({std::move(name), std::move(func), scheduled_time});
        
        std::cout << "Scheduled: " << name << " (in " 
                  << delay_from_now.count() << "ms)\n";
    }
    
    void run_pending() {
        auto now = system_clock::now();
        
        while (!tasks_.empty() && tasks_.top().scheduled_time <= now) {
            auto task = tasks_.top();
            tasks_.pop();
            
            std::cout << "Executing: " << task.name << "\n";
            task.func();
        }
    }
    
    bool has_pending() const {
        return !tasks_.empty();
    }
    
    milliseconds time_until_next() const {
        if (tasks_.empty()) return milliseconds::max();
        
        auto next_time = tasks_.top().scheduled_time;
        auto now = system_clock::now();
        
        if (next_time <= now) return milliseconds(0);
        
        return duration_cast<milliseconds>(next_time - now);
    }
};

// 4. Performance profiler
class Profiler {
private:
    struct FunctionStats {
        size_t call_count = 0;
        nanoseconds total_time{0};
        nanoseconds min_time{nanoseconds::max()};
        nanoseconds max_time{0};
    };
    
    std::map<std::string, FunctionStats> stats_;
    
public:
    class ProfileScope {
        Profiler& profiler_;
        std::string name_;
        steady_clock::time_point start_;
        
    public:
        ProfileScope(Profiler& profiler, std::string name)
            : profiler_(profiler)
            , name_(std::move(name))
            , start_(steady_clock::now()) {}
        
        ~ProfileScope() {
            auto elapsed = steady_clock::now() - start_;
            profiler_.record(name_, elapsed);
        }
    };
    
    void record(const std::string& name, nanoseconds duration) {
        auto& stats = stats_[name];
        stats.call_count++;
        stats.total_time += duration;
        stats.min_time = std::min(stats.min_time, duration);
        stats.max_time = std::max(stats.max_time, duration);
    }
    
    ProfileScope profile(std::string name) {
        return ProfileScope(*this, std::move(name));
    }
    
    void print_report() const {
        std::cout << "\n=== Performance Report ===\n";
        std::cout << std::left;
        
        for (const auto& [name, stats] : stats_) {
            auto avg = stats.total_time / stats.call_count;
            
            std::cout << std::setw(25) << name << ": "
                      << stats.call_count << " calls, "
                      << "avg: " << duration_cast<microseconds>(avg).count() << "µs, "
                      << "min: " << duration_cast<microseconds>(stats.min_time).count() << "µs, "
                      << "max: " << duration_cast<microseconds>(stats.max_time).count() << "µs\n";
        }
    }
};

// 5. Timeout handler
class TimeoutHandler {
public:
    template<typename Func, typename Duration>
    static bool execute_with_timeout(Func func, Duration timeout) {
        auto start = steady_clock::now();
        
        while (true) {
            if (func()) {
                return true;  // Success
            }
            
            auto elapsed = steady_clock::now() - start;
            if (elapsed >= timeout) {
                return false;  // Timeout
            }
            
            std::this_thread::sleep_for(milliseconds(10));
        }
    }
};

// 6. Duration formatting
std::string format_duration(nanoseconds ns) {
    auto hours = duration_cast<std::chrono::hours>(ns);
    ns -= hours;
    auto minutes = duration_cast<std::chrono::minutes>(ns);
    ns -= minutes;
    auto seconds = duration_cast<std::chrono::seconds>(ns);
    ns -= seconds;
    auto ms = duration_cast<milliseconds>(ns);
    
    std::ostringstream oss;
    if (hours.count() > 0) {
        oss << hours.count() << "h ";
    }
    if (minutes.count() > 0) {
        oss << minutes.count() << "m ";
    }
    if (seconds.count() > 0) {
        oss << seconds.count() << "s ";
    }
    if (ms.count() > 0) {
        oss << ms.count() << "ms";
    }
    
    return oss.str();
}

// Demo functions to profile
void fast_function() {
    std::this_thread::sleep_for(milliseconds(10));
}

void medium_function() {
    std::this_thread::sleep_for(milliseconds(50));
}

void slow_function() {
    std::this_thread::sleep_for(milliseconds(100));
}

int main() {
    std::cout << "=== Time and Chrono Demo ===\n\n";
    
    // 1. Scoped timing
    std::cout << "--- Scoped Timer ---\n";
    {
        ScopedTimer timer("Task execution");
        std::this_thread::sleep_for(milliseconds(100));
    }
    std::cout << "\n";
    
    // 2. Rate limiting
    std::cout << "--- Rate Limiter ---\n";
    RateLimiter limiter(milliseconds(200));
    
    for (int i = 0; i < 5; ++i) {
        if (limiter.try_acquire()) {
            std::cout << "Action " << i << " executed\n";
        } else {
            std::cout << "Action " << i << " rate-limited\n";
        }
        std::this_thread::sleep_for(milliseconds(100));
    }
    std::cout << "\n";
    
    // 3. Task scheduler
    std::cout << "--- Task Scheduler ---\n";
    TaskScheduler scheduler;
    
    scheduler.schedule("Task 1", []() {
        std::cout << "  Task 1 running\n";
    }, milliseconds(100));
    
    scheduler.schedule("Task 2", []() {
        std::cout << "  Task 2 running\n";
    }, milliseconds(200));
    
    scheduler.schedule("Task 3", []() {
        std::cout << "  Task 3 running\n";
    }, milliseconds(50));
    
    while (scheduler.has_pending()) {
        auto wait_time = scheduler.time_until_next();
        std::cout << "Waiting " << wait_time.count() << "ms...\n";
        std::this_thread::sleep_for(wait_time);
        scheduler.run_pending();
    }
    std::cout << "\n";
    
    // 4. Performance profiler
    std::cout << "--- Performance Profiler ---\n";
    Profiler profiler;
    
    for (int i = 0; i < 10; ++i) {
        {
            auto scope = profiler.profile("fast_function");
            fast_function();
        }
        
        if (i % 2 == 0) {
            auto scope = profiler.profile("medium_function");
            medium_function();
        }
        
        if (i % 5 == 0) {
            auto scope = profiler.profile("slow_function");
            slow_function();
        }
    }
    
    profiler.print_report();
    std::cout << "\n";
    
    // 5. Timeout handler
    std::cout << "--- Timeout Handler ---\n";
    
    int counter = 0;
    bool success = TimeoutHandler::execute_with_timeout(
        [&counter]() {
            counter++;
            std::cout << "  Attempt " << counter << "\n";
            return counter >= 3;  // Success after 3 attempts
        },
        seconds(2)
    );
    
    std::cout << (success ? "Succeeded" : "Timed out") << "\n\n";
    
    // 6. Duration arithmetic
    std::cout << "--- Duration Arithmetic ---\n";
    
    auto d1 = hours(2) + minutes(30) + seconds(45);
    auto d2 = milliseconds(5432);
    
    std::cout << "Duration 1: " << format_duration(d1) << "\n";
    std::cout << "Duration 2: " << format_duration(d2) << "\n";
    std::cout << "Sum: " << format_duration(d1 + d2) << "\n";
    std::cout << "Difference: " << format_duration(d1 - d2) << "\n\n";
    
    // 7. Time point comparisons
    std::cout << "--- Time Point Comparisons ---\n";
    
    auto start = steady_clock::now();
    std::this_thread::sleep_for(milliseconds(100));
    auto end = steady_clock::now();
    
    auto elapsed = end - start;
    std::cout << "Elapsed: " 
              << duration_cast<milliseconds>(elapsed).count() << "ms\n";
    
    if (elapsed > milliseconds(50)) {
        std::cout << "Operation took longer than 50ms\n";
    }
    std::cout << "\n";
    
    // 8. Clock accuracy comparison
    std::cout << "--- Clock Periods ---\n";
    using sys_period = system_clock::period;
    using steady_period = steady_clock::period;
    using hires_period = high_resolution_clock::period;
    
    std::cout << "system_clock period: " 
              << sys_period::num << "/" << sys_period::den << " seconds\n";
    std::cout << "steady_clock period: " 
              << steady_period::num << "/" << steady_period::den << " seconds\n";
    std::cout << "high_resolution_clock period: "
              << hires_period::num << "/" << hires_period::den << " seconds\n";
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **steady_clock**: Monotonic timing for performance measurement (preferred)
- **system_clock**: Scheduling tasks with wall-clock time
- **Duration arithmetic / casting**: Type-safe unit conversions
- **chrono_literals**: `1h`, `100ms`, `2024y`
- **RAII timer**: Automatic timing with scope
- **Rate limiting**: Enforcing minimum intervals
- **Task scheduling**: [Container adaptors](04_container_adaptors.md) priority queue with time_points
- **Performance profiling**: Min/max/average statistics
- **Timeout handling**: Time-limited operations

This example integrates patterns from earlier sections into one runnable program.

---

## Related Topics

- [Multithreading](14_multithreading.md) — `sleep_for`, rate limiting, and thread timing
- [Async & Futures](15_async_futures.md) — `wait_for` timeouts and async execution
- [I/O & Filesystem](16_io_filesystem.md) — `file_clock` and file timestamp handling
- [Modern C++ Features](07_modern_features.md) — user-defined literals and `std::format`
- [Algorithms](06_algorithms.md) — measuring algorithm performance
- [Best Practices](13_best_practices.md) — RAII timers and profiling discipline
- [Quick Reference](99_quick_reference.md) — chrono type cheat sheet

## Next Steps
- **Next**: [Memory Management →](19_memory_allocators.md)
- **Previous**: [← Exception Handling](17_exceptions.md)

---
*Chapter 18 — Time and Chrono Library*

