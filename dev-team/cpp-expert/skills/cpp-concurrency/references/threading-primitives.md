# Threading Primitives — Deep Dive Reference

## std::thread Deep Dive

### Hardware Concurrency

```cpp
// Query number of hardware threads
unsigned int n = std::thread::hardware_concurrency();
// Returns 0 if unknown, otherwise the number of concurrent threads supported

// Common pattern: create thread pool sized to hardware
auto num_workers = std::max(1u, std::thread::hardware_concurrency());
```

### Passing Arguments

```cpp
// By value — copies arguments
std::thread t(process, data);  // data is copied into the thread

// By reference — use std::ref
std::thread t(process, std::ref(data));  // data is passed by reference
// WARNING: caller must ensure data outlives the thread

// By move — transfers ownership
std::thread t(process, std::move(data));  // data is moved into the thread
```

### Thread IDs

```cpp
std::thread t(work);
auto id = t.get_id();  // std::thread::id — unique per thread
auto this_id = std::this_thread::get_id();  // Current thread's ID

// Thread IDs are hashable and comparable
std::unordered_map<std::thread::id, std::string> thread_names;
thread_names[id] = "worker-1";
```

### Thread-Local Storage

```cpp
// Thread-local variable — each thread gets its own copy
thread_local int counter = 0;

void increment() {
    ++counter;  // Modifies this thread's copy only
}

// Thread-local with complex initialization
thread_local std::mt19937 rng(std::random_device{}());

// Thread-local in class scope
class Logger {
    static thread_local std::string thread_name;
public:
    static void set_name(std::string name) { thread_name = std::move(name); }
    void log(std::string_view msg) {
        std::println("[{}] {}", thread_name, msg);
    }
};
thread_local std::string Logger::thread_name = "unnamed";
```

## Mutex Hierarchy

### std::mutex

Basic mutex. Non-recursive — locking twice from the same thread is UB.

```cpp
std::mutex mtx;
{
    std::lock_guard lock(mtx);  // RAII lock
    // Critical section
}  // Automatically unlocked
```

### std::timed_mutex

Supports try_lock_for and try_lock_until:

```cpp
std::timed_mutex mtx;
if (mtx.try_lock_for(std::chrono::milliseconds(100))) {
    // Got the lock within 100ms
    std::lock_guard lock(mtx, std::adopt_lock);
    // Work...
} else {
    // Timed out — handle contention
}
```

### std::recursive_mutex

Allows the same thread to lock multiple times. Generally indicates a design problem — prefer restructuring code to avoid recursive locking.

```cpp
// AVOID: recursive_mutex is a code smell
std::recursive_mutex mtx;
void outer() {
    std::lock_guard lock(mtx);
    inner();  // Also locks mtx — works with recursive_mutex
}
void inner() {
    std::lock_guard lock(mtx);
    // Work...
}

// BETTER: refactor to avoid recursive locking
void outer() {
    std::lock_guard lock(mtx);
    inner_unlocked();  // Does not lock — assumes lock is held
}
void inner_unlocked() { /* Work... */ }
void inner() {
    std::lock_guard lock(mtx);
    inner_unlocked();
}
```

### std::shared_mutex

Reader-writer lock. Multiple shared (read) locks or one exclusive (write) lock:

```cpp
std::shared_mutex rw_mutex;

// Reader — shared access
void read_data() {
    std::shared_lock lock(rw_mutex);
    // Multiple readers can proceed concurrently
}

// Writer — exclusive access
void write_data() {
    std::unique_lock lock(rw_mutex);
    // Only one writer, blocks all readers
}
```

### std::once_flag and std::call_once

Thread-safe one-time initialization:

```cpp
std::once_flag init_flag;
Database* db = nullptr;

Database& get_db() {
    std::call_once(init_flag, []() {
        db = new Database("connection_string");
    });
    return *db;
}
```

## Condition Variable Patterns

### Bounded Buffer (Producer-Consumer)

```cpp
template <typename T, size_t Capacity>
class BoundedBuffer {
    std::array<T, Capacity> buffer_;
    size_t head_ = 0, tail_ = 0, count_ = 0;
    std::mutex mutex_;
    std::condition_variable not_full_, not_empty_;

public:
    void push(T item) {
        std::unique_lock lock(mutex_);
        not_full_.wait(lock, [this] { return count_ < Capacity; });
        buffer_[tail_] = std::move(item);
        tail_ = (tail_ + 1) % Capacity;
        ++count_;
        not_empty_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mutex_);
        not_empty_.wait(lock, [this] { return count_ > 0; });
        T item = std::move(buffer_[head_]);
        head_ = (head_ + 1) % Capacity;
        --count_;
        not_full_.notify_one();
        return item;
    }
};
```

### Timed Waits

```cpp
std::unique_lock lock(mutex_);
bool got_data = cv_.wait_for(lock, std::chrono::seconds(5),
    [this] { return !queue_.empty(); });

if (got_data) {
    // Process queue
} else {
    // Timed out after 5 seconds
}
```

### Notification Patterns

```cpp
// Notify one: wake one waiting thread (for work distribution)
cv.notify_one();

// Notify all: wake all waiting threads (for state changes)
cv.notify_all();

// Rule of thumb:
// - notify_one: when any one waiter can handle the event
// - notify_all: when all waiters need to re-evaluate their condition
```

## std::async Pitfalls

### Blocking Destructor

```cpp
// DANGER: temporary future blocks immediately
void fire_and_forget() {
    std::async(std::launch::async, heavy_computation);
    // BAD: temporary future from std::async blocks in destructor
    // This runs SYNCHRONOUSLY, not asynchronously!
}

// GOOD: store the future to get actual async behavior
void actually_async() {
    auto future = std::async(std::launch::async, heavy_computation);
    // Do other work...
    auto result = future.get();  // Block when result is needed
}
```

### Launch Policy

```cpp
// std::launch::async — always runs on a new thread
auto f1 = std::async(std::launch::async, work);

// std::launch::deferred — runs lazily on .get()/.wait()
auto f2 = std::async(std::launch::deferred, work);

// Default (async | deferred) — implementation chooses
auto f3 = std::async(work);  // May be deferred — dangerous!

// RECOMMENDATION: always specify std::launch::async explicitly
```

## Thread Pool Implementation

```cpp
class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this](std::stop_token stoken) {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mutex_);
                        cv_.wait(lock, [this, &stoken] {
                            return stoken.stop_requested() || !tasks_.empty();
                        });
                        if (stoken.stop_requested() && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
        }
    }

    template <typename F>
    auto submit(F&& f) -> std::future<std::invoke_result_t<F>> {
        using R = std::invoke_result_t<F>;
        auto task = std::make_shared<std::packaged_task<R()>>(std::forward<F>(f));
        auto future = task->get_future();
        {
            std::lock_guard lock(mutex_);
            tasks_.emplace([task]() { (*task)(); });
        }
        cv_.notify_one();
        return future;
    }

    ~ThreadPool() {
        for (auto& w : workers_) w.request_stop();
        cv_.notify_all();
        // jthreads auto-join
    }
};
```

## std::jthread and stop_token

### Cooperative Cancellation

```cpp
std::jthread worker([](std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        auto item = get_next_item();
        if (!item) {
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            continue;
        }
        process(*item);
    }
});

// Later: request cancellation
worker.request_stop();
// jthread destructor waits for the thread to finish
```

### stop_callback

Register a callback to run when stop is requested:

```cpp
std::jthread worker([](std::stop_token stoken) {
    // Register cleanup callback
    std::stop_callback cleanup(stoken, []() {
        // Called when stop is requested (from any thread)
        cancel_pending_io();
    });

    while (!stoken.stop_requested()) {
        blocking_io_operation();
    }
});
```

### Integration with condition_variable_any

```cpp
std::jthread worker([&](std::stop_token stoken) {
    std::unique_lock lock(mutex);
    // Automatically wakes up when stop is requested
    cv.wait(lock, stoken, [&] { return !queue.empty(); });
    // Returns false if stopped, true if predicate is satisfied
});
```

### stop_source

Manually create stop sources for custom cancellation:

```cpp
std::stop_source source;
auto token = source.get_token();

// Pass token to multiple workers
std::jthread w1([token] { while (!token.stop_requested()) { /* work */ } });
std::jthread w2([token] { while (!token.stop_requested()) { /* work */ } });

// Cancel all workers at once
source.request_stop();
```
