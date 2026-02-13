# Lock-Free Programming and Coroutines — Deep Dive Reference

## C++ Memory Model

### Fundamental Relationships

The C++ memory model defines ordering guarantees between operations across threads:

- **Sequenced-before**: Within a single thread, operations are ordered by program order
- **Synchronizes-with**: A release store synchronizes-with an acquire load of the same atomic variable
- **Happens-before**: Transitive closure of sequenced-before and synchronizes-with. If A happens-before B, then A's effects are visible to B

### Memory Orderings Explained

#### memory_order_relaxed

No ordering guarantees between operations. Only atomicity is guaranteed.

```cpp
// Use case: simple counters, statistics
std::atomic<int> counter{0};

// Thread 1
counter.fetch_add(1, std::memory_order_relaxed);

// Thread 2
counter.fetch_add(1, std::memory_order_relaxed);

// Thread 3
auto val = counter.load(std::memory_order_relaxed);
// val is guaranteed to be a valid value, but may not reflect all increments yet
```

#### memory_order_acquire / memory_order_release

Release-acquire pairs create happens-before relationships:

```cpp
std::atomic<bool> flag{false};
int data = 0;

// Producer (release)
void producer() {
    data = 42;                                          // A
    flag.store(true, std::memory_order_release);        // B
    // A is sequenced-before B, and B is a release store
}

// Consumer (acquire)
void consumer() {
    while (!flag.load(std::memory_order_acquire)) {}    // C
    assert(data == 42);                                 // D
    // C synchronizes-with B, so A happens-before D
    // Therefore data = 42 is guaranteed visible
}
```

#### memory_order_acq_rel

For read-modify-write operations that need both acquire and release semantics:

```cpp
std::atomic<Node*> head{nullptr};

void push(Node* node) {
    node->next = head.load(std::memory_order_relaxed);
    while (!head.compare_exchange_weak(node->next, node,
            std::memory_order_acq_rel, std::memory_order_relaxed)) {}
}
```

#### memory_order_seq_cst

Total order across all seq_cst operations. Safest but slowest. Default when no ordering is specified.

```cpp
// All threads agree on the order of seq_cst operations
std::atomic<bool> x{false}, y{false};
std::atomic<int> z{0};

// Thread 1
x.store(true, std::memory_order_seq_cst);

// Thread 2
y.store(true, std::memory_order_seq_cst);

// Thread 3
while (!x.load(std::memory_order_seq_cst)) {}
if (y.load(std::memory_order_seq_cst)) ++z;

// Thread 4
while (!y.load(std::memory_order_seq_cst)) {}
if (x.load(std::memory_order_seq_cst)) ++z;

// z is guaranteed to be at least 1 (with seq_cst, not guaranteed with relaxed)
```

## Lock-Free Data Structures

### Treiber Stack (Lock-Free Stack)

```cpp
template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T d) : data(std::move(d)), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(T value) {
        auto* node = new Node(std::move(value));
        node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(node->next, node,
                std::memory_order_release, std::memory_order_relaxed)) {
            // CAS failed — node->next updated to current head, retry
        }
    }

    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_acquire);
        while (old_head && !head_.compare_exchange_weak(old_head, old_head->next,
                std::memory_order_acq_rel, std::memory_order_acquire)) {
            // CAS failed — old_head updated to current head, retry
        }
        if (!old_head) return std::nullopt;
        T value = std::move(old_head->data);
        delete old_head;  // DANGER: ABA problem (see below)
        return value;
    }
};
```

### Michael-Scott Queue (Lock-Free Queue)

```cpp
template <typename T>
class LockFreeQueue {
    struct Node {
        std::optional<T> data;
        std::atomic<Node*> next{nullptr};
    };

    std::atomic<Node*> head_;
    std::atomic<Node*> tail_;

public:
    LockFreeQueue() {
        auto* sentinel = new Node{};
        head_.store(sentinel);
        tail_.store(sentinel);
    }

    void enqueue(T value) {
        auto* node = new Node{std::move(value)};
        while (true) {
            Node* tail = tail_.load(std::memory_order_acquire);
            Node* next = tail->next.load(std::memory_order_acquire);
            if (tail == tail_.load(std::memory_order_acquire)) {
                if (next == nullptr) {
                    if (tail->next.compare_exchange_weak(next, node,
                            std::memory_order_release)) {
                        tail_.compare_exchange_strong(tail, node,
                            std::memory_order_release);
                        return;
                    }
                } else {
                    tail_.compare_exchange_weak(tail, next,
                        std::memory_order_release);
                }
            }
        }
    }

    std::optional<T> dequeue() {
        while (true) {
            Node* head = head_.load(std::memory_order_acquire);
            Node* tail = tail_.load(std::memory_order_acquire);
            Node* next = head->next.load(std::memory_order_acquire);

            if (head == head_.load(std::memory_order_acquire)) {
                if (head == tail) {
                    if (next == nullptr) return std::nullopt;
                    tail_.compare_exchange_weak(tail, next, std::memory_order_release);
                } else {
                    T value = std::move(*next->data);
                    if (head_.compare_exchange_weak(head, next,
                            std::memory_order_acq_rel)) {
                        delete head;  // Reclaim old sentinel
                        return value;
                    }
                }
            }
        }
    }
};
```

### The ABA Problem

The ABA problem occurs when:
1. Thread reads value A from an atomic
2. Thread is preempted
3. Other threads change A → B → A
4. Original thread's CAS succeeds (sees A), but the underlying data has changed

#### Solutions

**Hazard Pointers**: Each thread publishes pointers it's currently using. Other threads check hazard lists before reclaiming memory.

**Epoch-Based Reclamation**: Threads track a global epoch. Memory is only reclaimed when all threads have advanced past the epoch when the memory was freed.

**Tagged Pointers**: Pack a version counter alongside the pointer. Each modification increments the counter.

```cpp
// Tagged pointer approach (simplified)
struct TaggedPtr {
    Node* ptr;
    uint64_t tag;  // Monotonically increasing version

    bool operator==(const TaggedPtr& o) const {
        return ptr == o.ptr && tag == o.tag;
    }
};

std::atomic<TaggedPtr> head_{{nullptr, 0}};

void push(Node* node) {
    auto old = head_.load(std::memory_order_relaxed);
    do {
        node->next = old.ptr;
    } while (!head_.compare_exchange_weak(old, {node, old.tag + 1},
            std::memory_order_release));
}
```

## Coroutines Deep Dive

### Promise Type Implementation

A complete promise_type for a Task<T> coroutine:

```cpp
template <typename T>
class Task {
public:
    struct promise_type {
        std::variant<std::monostate, T, std::exception_ptr> result_;
        std::coroutine_handle<> continuation_;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }

        auto final_suspend() noexcept {
            struct Awaiter {
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    if (h.promise().continuation_)
                        return h.promise().continuation_;  // Symmetric transfer
                    return std::noop_coroutine();
                }
                void await_resume() noexcept {}
            };
            return Awaiter{};
        }

        void return_value(T value) {
            result_.template emplace<1>(std::move(value));
        }

        void unhandled_exception() {
            result_.template emplace<2>(std::current_exception());
        }
    };

    using handle_type = std::coroutine_handle<promise_type>;

    explicit Task(handle_type h) : handle_(h) {}
    ~Task() { if (handle_) handle_.destroy(); }

    Task(Task&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

    // Awaitable: co_await a Task<T>
    bool await_ready() const noexcept { return handle_.done(); }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) {
        handle_.promise().continuation_ = caller;
        return handle_;  // Symmetric transfer to the task
    }

    T await_resume() {
        auto& result = handle_.promise().result_;
        if (auto* ex = std::get_if<2>(&result)) std::rethrow_exception(*ex);
        return std::move(std::get<1>(result));
    }

private:
    handle_type handle_;
};
```

### Symmetric Transfer

Symmetric transfer avoids stack overflow in deep coroutine chains. Instead of resuming a coroutine from within another (growing the stack), `await_suspend` returns a handle, and the runtime resumes it directly:

```cpp
// Without symmetric transfer — stack overflow risk
void await_suspend(std::coroutine_handle<> h) {
    continuation_.resume();  // Resumes on THIS stack frame
}

// With symmetric transfer — tail-call optimization
std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) {
    return continuation_;  // Runtime resumes without growing stack
}
```

### Custom Awaiters

```cpp
// Awaiter that schedules resumption on a thread pool
struct ScheduleOnPool {
    ThreadPool& pool;

    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) const {
        pool.submit([h]() { h.resume(); });
    }

    void await_resume() const noexcept {}
};

// Usage
Task<int> compute_on_pool(ThreadPool& pool) {
    co_await ScheduleOnPool{pool};
    // Now running on the thread pool
    return heavy_computation();
}
```

### Async Generator

```cpp
template <typename T>
class AsyncGenerator {
public:
    struct promise_type {
        std::optional<T> current_value;
        std::coroutine_handle<> consumer;
        std::exception_ptr exception;

        AsyncGenerator get_return_object() {
            return AsyncGenerator{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        auto yield_value(T value) {
            current_value = std::move(value);
            struct Transfer {
                std::coroutine_handle<> consumer;
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(auto) noexcept {
                    return consumer;  // Transfer to consumer
                }
                void await_resume() noexcept {}
            };
            return Transfer{consumer};
        }

        void return_void() {}
        void unhandled_exception() { exception = std::current_exception(); }
    };

    // Iterator interface for co_await-based consumption...
};
```

## std::generator (C++23)

Standard synchronous generator:

```cpp
#include <generator>

// Basic generator
std::generator<int> range(int start, int end) {
    for (int i = start; i < end; ++i) {
        co_yield i;
    }
}

// Recursive generator with elements_of
std::generator<int> flatten(std::vector<std::vector<int>>& vv) {
    for (auto& v : vv) {
        co_yield std::ranges::elements_of(v);  // Yield all elements of v
    }
}

// Generator as range — composable with views
auto evens = range(0, 100)
    | std::views::filter([](int i) { return i % 2 == 0; })
    | std::views::take(10);
```

## Executors and std::execution (P2300)

The std::execution proposal (expected in C++26) introduces a sender/receiver model for structured concurrency:

```cpp
// Conceptual (P2300 proposal, not yet standardized)
auto work = std::execution::just(42)
    | std::execution::then([](int x) { return x * 2; })
    | std::execution::let_value([](int x) {
        return std::execution::on(thread_pool, std::execution::just(x))
            | std::execution::then([](int x) { return expensive_compute(x); });
    });

// Start and wait for completion
auto result = std::this_thread::sync_wait(std::move(work));
```

Key concepts:
- **Sender**: A lazy description of work (like a future factory)
- **Receiver**: Handles completion, error, or cancellation
- **Scheduler**: Determines where work runs (thread pool, io_uring, GPU)
- **Structured concurrency**: Child operations complete before parent, preventing resource leaks

This is the future direction for C++ async programming, replacing ad-hoc coroutine frameworks with a standardized model.
