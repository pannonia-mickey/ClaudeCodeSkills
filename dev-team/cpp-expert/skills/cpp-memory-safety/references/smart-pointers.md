# Smart Pointers — Deep Dive Reference

## std::unique_ptr

### Core Semantics

`std::unique_ptr<T>` models sole ownership. It is move-only: no copy constructor or copy assignment. When the unique_ptr is destroyed, the managed object is deleted. Zero runtime overhead compared to a raw pointer.

```cpp
// Prefer make_unique for exception safety and clarity
auto widget = std::make_unique<Widget>(42, "hello");

// Explicit construction (needed for custom deleters or arrays)
std::unique_ptr<Widget> w(new Widget(42, "hello"));
```

### Custom Deleters

Use custom deleters for C APIs or non-standard cleanup:

```cpp
// Function pointer deleter
auto file = std::unique_ptr<FILE, decltype(&fclose)>(fopen("data.bin", "rb"), &fclose);

// Lambda deleter (zero overhead if stateless)
auto handle = std::unique_ptr<HANDLE__, decltype([](HANDLE__ *h) { CloseHandle(h); })>(
    OpenResource(), [](HANDLE__* h) { CloseHandle(h); }
);

// Custom deleter struct
struct MallocDeleter {
    void operator()(void* p) const noexcept { std::free(p); }
};
auto buffer = std::unique_ptr<char[], MallocDeleter>(
    static_cast<char*>(std::malloc(1024))
);
```

Custom deleter is part of the unique_ptr type. A `unique_ptr<T, D1>` and `unique_ptr<T, D2>` are different types. Stateless lambdas and empty structs add zero size overhead (empty base optimization).

### Array Specialization

```cpp
// Array form — calls delete[], operator[] available
auto arr = std::make_unique<int[]>(100);
arr[42] = 7;

// For non-default-initialized arrays (C++20)
auto arr2 = std::make_unique_for_overwrite<int[]>(100);  // Uninitialized
```

### Polymorphic Ownership

```cpp
// unique_ptr to base class — virtual destructor required
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

class Circle : public Shape {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return std::numbers::pi * radius_ * radius_; }
};

// Implicit upcast — derived to base
std::unique_ptr<Shape> shape = std::make_unique<Circle>(5.0);

// Containers of polymorphic unique_ptrs
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(3.0));
shapes.push_back(std::make_unique<Rectangle>(4.0, 5.0));
```

### unique_ptr as Class Members

```cpp
class Engine {
    std::unique_ptr<Renderer> renderer_;       // Sole ownership
    std::unique_ptr<AudioSystem> audio_;

public:
    // Rule of zero: compiler-generated move operations are correct
    // Compiler-generated destructor calls unique_ptr destructors

    // If the destructor is defined out-of-line (e.g., in .cpp), the forward
    // declaration of Renderer and AudioSystem is sufficient in the header.
    Engine();
    ~Engine();  // Defined in .cpp where Renderer/AudioSystem are complete
};
```

## std::shared_ptr

### Control Block Internals

Every `shared_ptr` points to both the managed object and a control block containing:
- Strong reference count (atomic)
- Weak reference count (atomic)
- Deleter (if custom)
- Allocator (if custom)
- Pointer to the managed object

```
shared_ptr<T> ──> [T object]
     │
     └──> [Control Block]
           ├── strong_refs: 2
           ├── weak_refs: 1
           ├── deleter: default_delete<T>
           └── ptr: → [T object]
```

### make_shared Optimization

`std::make_shared<T>(args...)` allocates the object and control block in a single allocation, improving cache locality and reducing allocation overhead:

```cpp
// Two allocations: control block + object
auto p1 = std::shared_ptr<Widget>(new Widget(42));

// One allocation: control block and object together
auto p2 = std::make_shared<Widget>(42);  // Preferred
```

Caveat: With `make_shared`, the object memory is not freed until all `weak_ptr`s are also destroyed (because the control block and object share the same allocation).

### enable_shared_from_this

Safely obtain a `shared_ptr` from within a member function of a shared-managed object:

```cpp
class Session : public std::enable_shared_from_this<Session> {
public:
    void start() {
        // Capture shared_ptr to self, preventing premature destruction
        auto self = shared_from_this();
        async_read([self](auto data) {
            self->process(data);
        });
    }
};

// Must be created as shared_ptr initially
auto session = std::make_shared<Session>();
session->start();
```

### Aliasing Constructor

Create a `shared_ptr` that shares ownership with another `shared_ptr` but points to a different object (typically a sub-object):

```cpp
struct Pair {
    int first;
    int second;
};

auto pair = std::make_shared<Pair>(1, 2);

// Aliasing: shares ownership of pair but points to pair->first
std::shared_ptr<int> first_ptr(pair, &pair->first);
// pair's ref count is now 2, first_ptr keeps pair alive
```

### Thread Safety

`shared_ptr` reference counting is thread-safe (atomic operations). However, the pointed-to object is NOT thread-safe. Multiple threads can safely copy/destroy `shared_ptr` instances pointing to the same object, but concurrent access to the managed object requires external synchronization.

```cpp
// Safe: copying shared_ptr from different threads
std::shared_ptr<Widget> global_widget = std::make_shared<Widget>();
// Thread 1: auto local = global_widget;  // OK — atomic ref count increment
// Thread 2: auto local = global_widget;  // OK — atomic ref count increment

// NOT safe: modifying shared_ptr itself from different threads
// Thread 1: global_widget = other_widget;  // Race condition on shared_ptr object
// Thread 2: global_widget.reset();         // Race condition on shared_ptr object

// Use std::atomic<std::shared_ptr<T>> (C++20) for concurrent shared_ptr modification
std::atomic<std::shared_ptr<Widget>> atomic_widget;
```

## std::weak_ptr

### Breaking Ownership Cycles

```cpp
// Problem: parent owns children, children reference parent → cycle
struct Node {
    std::shared_ptr<Node> parent;  // BAD: creates cycle
    std::vector<std::shared_ptr<Node>> children;
};

// Solution: weak_ptr for back-references
struct Node {
    std::weak_ptr<Node> parent;  // GOOD: no cycle
    std::vector<std::shared_ptr<Node>> children;

    std::shared_ptr<Node> get_parent() const {
        return parent.lock();  // Returns nullptr if parent destroyed
    }
};
```

### Cache Pattern

```cpp
class ResourceCache {
    std::unordered_map<std::string, std::weak_ptr<Resource>> cache_;
    std::mutex mutex_;
public:
    std::shared_ptr<Resource> get(const std::string& key) {
        std::lock_guard lock(mutex_);
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            if (auto sp = it->second.lock()) {
                return sp;  // Cache hit — resource still alive
            }
            cache_.erase(it);  // Expired — clean up
        }
        auto resource = std::make_shared<Resource>(key);
        cache_[key] = resource;
        return resource;
    }
};
```

### expired() vs lock()

- `expired()` — checks if the managed object has been destroyed (may be stale by the time it returns)
- `lock()` — atomically checks and creates a shared_ptr if object still alive (prefer this)

Always use `lock()` and check the result rather than calling `expired()` then `lock()` (TOCTOU race).

## Performance Characteristics

| Operation | unique_ptr | shared_ptr | raw pointer |
|-----------|-----------|-----------|-------------|
| Size | Same as pointer (+ deleter) | 2 pointers (object + control block) | 1 pointer |
| Dereference | Zero overhead | Zero overhead | Baseline |
| Copy | N/A (move-only) | Atomic ref count increment | Trivial copy |
| Destroy | Calls deleter | Atomic ref count decrement, conditional delete | Nothing |
| make_unique/make_shared | 1 allocation | 1 allocation (fused) | N/A |
| new + shared_ptr | N/A | 2 allocations (object + control block) | N/A |

### When to Use Each

| Scenario | Choice |
|----------|--------|
| Single owner, no sharing | `std::unique_ptr` |
| Shared ownership (rare) | `std::shared_ptr` |
| Observing shared object | `std::weak_ptr` |
| Non-owning function parameter | `T&` or `T*` |
| Non-owning range view | `std::span<T>` |
| Non-owning string view | `std::string_view` |

Default to `unique_ptr`. Use `shared_ptr` only when genuine shared ownership is needed. Never use `shared_ptr` just because "pointers need to be smart."
