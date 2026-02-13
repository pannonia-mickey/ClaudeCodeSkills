---
name: C++ Concurrency
description: This skill should be used when the user asks about "C++ thread", "std::thread", "std::jthread", "C++ mutex", "std::atomic", "C++ async", "C++ future", "lock-free", "C++ coroutine", "co_await", "co_yield", "condition variable", "C++ memory model", "memory ordering", "thread pool", "std::latch", "std::barrier", or "data race". It covers multithreading, synchronization, atomic operations, lock-free programming, and C++20 coroutines.
version: 0.1.0
---

# C++ Concurrency

C++ provides low-level concurrency primitives with precise control over synchronization and memory ordering. This skill covers threading, synchronization, atomics, and coroutines.

## Thread Management

Prefer `std::jthread` (C++20) over `std::thread`. jthread automatically joins on destruction (RAII) and supports cooperative cancellation via `std::stop_token`.

```cpp
#include <thread>
#include <stop_token>

// std::jthread with cooperative cancellation
std::jthread worker([](std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        process_next_item();
    }
    // Cleanup after stop requested
});

// Cancellation is automatic when jthread is destroyed,
// or explicit via:
worker.request_stop();
// worker.~jthread() joins automatically
```

With `std::thread`, always join or detach before destruction — failing to do so calls `std::terminate`. Prefer joining over detaching to avoid dangling references.

## Mutual Exclusion

Use `std::mutex` for exclusive access and `std::shared_mutex` for reader-writer patterns. Always lock through RAII guards.

| Guard | Mutex Type | Behavior |
|-------|-----------|----------|
| `std::lock_guard` | Any mutex | Simple exclusive lock |
| `std::unique_lock` | Any mutex | Deferred/timed locking, condition variable compatible |
| `std::shared_lock` | shared_mutex | Shared (read) access |
| `std::scoped_lock` | Multiple mutexes | Deadlock-free multi-lock |

```cpp
// Reader-writer cache with shared_mutex
class ThreadSafeCache {
    mutable std::shared_mutex mutex_;
    std::unordered_map<std::string, std::string> data_;
public:
    // Multiple readers concurrently
    std::optional<std::string> get(const std::string& key) const {
        std::shared_lock lock(mutex_);
        auto it = data_.find(key);
        return it != data_.end() ? std::optional{it->second} : std::nullopt;
    }

    // Exclusive writer
    void put(const std::string& key, std::string value) {
        std::unique_lock lock(mutex_);
        data_[key] = std::move(value);
    }
};
```

Avoid deadlocks: use `std::scoped_lock` to lock multiple mutexes atomically, or enforce a consistent lock ordering.

## Condition Variables

Use `std::condition_variable` with `std::unique_lock` for thread signaling. Always use the predicate overload to handle spurious wakeups.

```cpp
// Thread-safe bounded queue
template <typename T>
class BoundedQueue {
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable not_empty_;
    std::condition_variable not_full_;
    size_t capacity_;
public:
    explicit BoundedQueue(size_t cap) : capacity_(cap) {}

    void push(T item) {
        std::unique_lock lock(mutex_);
        not_full_.wait(lock, [this] { return queue_.size() < capacity_; });
        queue_.push(std::move(item));
        not_empty_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mutex_);
        not_empty_.wait(lock, [this] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }
};
```

## Async and Futures

Use `std::async` for fire-and-forget parallel tasks. Prefer `std::launch::async` to guarantee a new thread.

```cpp
// Parallel computation with futures
auto future_a = std::async(std::launch::async, compute_part_a, data);
auto future_b = std::async(std::launch::async, compute_part_b, data);

auto result_a = future_a.get();  // Blocks until ready
auto result_b = future_b.get();
auto combined = merge(result_a, result_b);
```

**Pitfall**: `std::future` from `std::async` blocks in its destructor. Assigning to a temporary (`std::async(...)` without capturing the return value) runs synchronously because the temporary future blocks immediately.

Use `std::promise`/`std::future` for manual result passing between threads, and `std::packaged_task` to wrap a callable with a future.

## Atomics and Memory Ordering

Use `std::atomic<T>` for lock-free access to shared variables. Choose the weakest sufficient memory ordering:

| Ordering | Use Case |
|----------|----------|
| `relaxed` | Counters, statistics — no ordering guarantees |
| `acquire` | Load that synchronizes with a release store |
| `release` | Store that makes prior writes visible to acquire loads |
| `acq_rel` | Read-modify-write combining acquire and release |
| `seq_cst` | Default — total order across all threads (safest, slowest) |

```cpp
// Lock-free producer-consumer flag
std::atomic<bool> data_ready{false};
int payload = 0;

// Producer thread
void producer() {
    payload = 42;                                    // Non-atomic write
    data_ready.store(true, std::memory_order_release); // Publish
}

// Consumer thread
void consumer() {
    while (!data_ready.load(std::memory_order_acquire)) {}  // Synchronize
    assert(payload == 42);  // Guaranteed visible due to acquire-release pair
}
```

Use `compare_exchange_weak` in loops (may fail spuriously but is faster on some architectures). Use `compare_exchange_strong` when not in a loop.

## Synchronization Primitives (C++20)

C++20 adds one-shot and reusable barriers:

```cpp
// std::latch — one-shot barrier, countdown to zero
std::latch work_done(num_tasks);

for (int i = 0; i < num_tasks; ++i) {
    pool.submit([&, i] {
        process(tasks[i]);
        work_done.count_down();  // Signal completion
    });
}
work_done.wait();  // Block until all tasks complete

// std::barrier — reusable, with optional completion callback
std::barrier sync_point(num_threads, [&]() noexcept {
    // Called once per phase when all threads arrive
    swap_buffers();
});

// In each worker thread:
while (running) {
    compute_phase();
    sync_point.arrive_and_wait();  // Synchronize between phases
}
```

`std::counting_semaphore` limits concurrent access to a resource pool.

## Coroutines (C++20)

Coroutines are functions that can suspend and resume. Use `co_await` to suspend until a value is ready, `co_yield` to produce a value and suspend, and `co_return` to complete.

```cpp
// Simple generator using co_yield
#include <generator>  // C++23

std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Usage — lazy evaluation
for (auto val : fibonacci() | std::views::take(10)) {
    std::println("{}", val);
}
```

Coroutines require a promise_type that controls suspension, resumption, and value passing. For C++23, use `std::generator` directly. For C++20, implement a custom task or generator type.

## Thread Safety Guidelines

- Identify all shared mutable state and protect it with synchronization
- Minimize the scope of critical sections — hold locks for the shortest duration possible
- Prefer immutable data shared across threads (no synchronization needed)
- Use `const` to express thread-safe read-only access
- Enable ThreadSanitizer (`-fsanitize=thread`) to detect data races at runtime
- Prefer `std::atomic` for simple flags and counters over mutex
- Avoid `recursive_mutex` — it often indicates a design problem

## Additional Resources

### Reference Files

For advanced concurrency topics, consult:

- **`references/threading-primitives.md`** — Thread deep dive, mutex hierarchy, condition variable patterns, std::async pitfalls, thread pool implementation, jthread/stop_token details
- **`references/lock-free-coroutines.md`** — C++ memory model (happens-before, synchronizes-with), lock-free data structures (Treiber stack, Michael-Scott queue), ABA problem, coroutine promise_type implementation, custom awaiters, task<T>, executors preview
