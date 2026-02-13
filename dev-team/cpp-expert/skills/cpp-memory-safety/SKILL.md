---
name: C++ Memory Safety
description: This skill should be used when the user asks about "smart pointers", "unique_ptr", "shared_ptr", "RAII", "C++ memory management", "rule of zero", "rule of five", "dangling reference", "use after free", "memory leak", "C++ ownership", "C++ lifetime", "double free", or "undefined behavior". It covers safe memory management patterns, smart pointer usage, ownership semantics, and strategies for avoiding undefined behavior in C++.
version: 0.1.0
---

# C++ Memory Safety

Safe memory management is the foundation of correct C++ programs. This skill covers RAII, smart pointers, ownership semantics, and strategies for avoiding undefined behavior.

## RAII (Resource Acquisition Is Initialization)

Tie resource lifetime to object scope. Acquire resources in constructors, release in destructors. The compiler guarantees destructor calls when objects leave scope, including during stack unwinding from exceptions.

Apply RAII to all resources: heap memory, file handles, mutex locks, network connections, GPU buffers.

```cpp
// RAII wrapper for a POSIX file descriptor
class FileDescriptor {
    int fd_ = -1;
public:
    explicit FileDescriptor(const char* path, int flags)
        : fd_(::open(path, flags)) {
        if (fd_ < 0) throw std::system_error(errno, std::generic_category());
    }

    ~FileDescriptor() { if (fd_ >= 0) ::close(fd_); }

    // Non-copyable, movable
    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;
    FileDescriptor(FileDescriptor&& other) noexcept : fd_(std::exchange(other.fd_, -1)) {}
    FileDescriptor& operator=(FileDescriptor&& other) noexcept {
        std::swap(fd_, other.fd_);
        return *this;
    }

    int get() const noexcept { return fd_; }
};
// fd automatically closed when scope exits, even on exception
```

## The Rule of Zero, Five, and Three

**Rule of Zero**: Prefer classes that need no custom special members. Compose classes from RAII types (unique_ptr, string, vector) so the compiler-generated destructor, copy, and move operations are correct.

**Rule of Five**: When managing a raw resource directly, implement all five special members: destructor, copy constructor, copy assignment, move constructor, move assignment.

**Rule of Three** (legacy): Pre-C++11 version covering destructor, copy constructor, copy assignment.

```cpp
// Rule of Zero — no custom special members needed
struct User {
    std::string name;
    std::unique_ptr<Session> session;  // Handles cleanup automatically
    std::vector<Permission> perms;
};

// Rule of Five — managing a raw resource
class RawBuffer {
    void* ptr_ = nullptr;
    size_t size_ = 0;
public:
    explicit RawBuffer(size_t sz) : ptr_(std::malloc(sz)), size_(sz) {
        if (!ptr_ && sz > 0) throw std::bad_alloc();
    }
    ~RawBuffer() { std::free(ptr_); }

    RawBuffer(const RawBuffer& o) : RawBuffer(o.size_) { std::memcpy(ptr_, o.ptr_, size_); }
    RawBuffer& operator=(const RawBuffer& o) { auto tmp = o; std::swap(*this, tmp); return *this; }
    RawBuffer(RawBuffer&& o) noexcept : ptr_(std::exchange(o.ptr_, nullptr)), size_(std::exchange(o.size_, 0)) {}
    RawBuffer& operator=(RawBuffer&& o) noexcept { std::swap(ptr_, o.ptr_); std::swap(size_, o.size_); return *this; }
};
```

## Smart Pointers

### std::unique_ptr — Sole Ownership

Use `std::unique_ptr` as the default smart pointer. It has zero overhead compared to raw pointers. Transfer ownership explicitly with `std::move`.

```cpp
// Factory returning unique ownership
auto create_widget(int id) -> std::unique_ptr<Widget> {
    return std::make_unique<Widget>(id);
}

// Custom deleter for C APIs
auto file = std::unique_ptr<FILE, decltype(&fclose)>(fopen("data.bin", "rb"), &fclose);

// Polymorphic ownership
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(5.0));
shapes.push_back(std::make_unique<Rectangle>(3.0, 4.0));
```

### std::shared_ptr — Shared Ownership

Use `std::shared_ptr` only when multiple owners must share a resource. Prefer `std::make_shared` to reduce allocations (single allocation for object + control block).

### std::weak_ptr — Non-Owning Observer

Use `std::weak_ptr` to break ownership cycles and observe shared resources without extending lifetime.

```cpp
// Cache with weak_ptr to avoid keeping objects alive
class TextureCache {
    std::unordered_map<std::string, std::weak_ptr<Texture>> cache_;
public:
    std::shared_ptr<Texture> get(const std::string& path) {
        auto it = cache_.find(path);
        if (it != cache_.end()) {
            if (auto tex = it->second.lock()) return tex;
        }
        auto tex = std::make_shared<Texture>(path);
        cache_[path] = tex;
        return tex;
    }
};
```

### Parameter Passing Guidelines

| Ownership intent | Parameter type |
|-----------------|---------------|
| Transfer ownership | `std::unique_ptr<T>` by value |
| Share ownership | `std::shared_ptr<T>` by value |
| Observe only (nullable) | `T*` raw pointer |
| Observe only (non-null) | `T&` reference |
| Observe a range | `std::span<T>` |
| Observe a string | `std::string_view` |

## Ownership Semantics

Express ownership intent clearly in APIs. Non-owning access uses raw pointers (nullable) or references (non-null). Ownership transfer uses unique_ptr by value. Shared ownership uses shared_ptr by value.

```cpp
// Ownership transfer — caller gives up ownership
void registry_add(std::unique_ptr<Widget> widget);

// Non-owning access — callee borrows, caller retains ownership
void process(const Widget& widget);

// Non-owning range access
void analyze(std::span<const float> data);

// Non-owning string access
void log_message(std::string_view msg);
```

## Common Undefined Behavior

### Use-After-Free / Dangling Reference

```cpp
// BAD: dangling reference to destroyed temporary
const std::string& bad_ref() {
    std::string s = "hello";
    return s;  // UB: s destroyed at end of scope
}

// GOOD: return by value (RVO applies)
std::string good_val() {
    std::string s = "hello";
    return s;  // Moved or elided, no dangling
}
```

### Iterator Invalidation

```cpp
// BAD: modifying container while iterating
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it % 2 == 0) v.erase(it);  // UB: iterator invalidated
}

// GOOD: erase-remove idiom or std::erase_if (C++20)
std::erase_if(v, [](int x) { return x % 2 == 0; });
```

### Signed Integer Overflow

```cpp
// BAD: signed overflow is UB
int x = INT_MAX;
int y = x + 1;  // UB

// GOOD: check before operation or use unsigned/wider type
if (x < INT_MAX) { int y = x + 1; }
```

## Sanitizers and Detection

Enable sanitizers in CMake for development and CI builds:

```cmake
# Add sanitizer options as a CMake option
option(ENABLE_SANITIZERS "Enable ASan and UBSan" OFF)

if(ENABLE_SANITIZERS)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()
```

| Sanitizer | Detects | Flag |
|-----------|---------|------|
| ASan | Use-after-free, buffer overflow, memory leaks | `-fsanitize=address` |
| UBSan | Signed overflow, null deref, misalignment | `-fsanitize=undefined` |
| MSan | Uninitialized memory reads | `-fsanitize=memory` |
| TSan | Data races, deadlocks | `-fsanitize=thread` |

Combine ASan + UBSan freely. Do not combine ASan with TSan or MSan (mutually exclusive).

## Additional Resources

### Reference Files

For in-depth coverage, consult:

- **`references/smart-pointers.md`** — Deep dive into unique_ptr custom deleters, shared_ptr control block internals, weak_ptr patterns, enable_shared_from_this, performance characteristics
- **`references/undefined-behavior.md`** — Comprehensive UB catalog (memory, numeric, type, concurrency), detection strategies, hardening techniques, real-world examples with fixes
