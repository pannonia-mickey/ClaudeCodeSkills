# Optimization Techniques — Deep Dive Reference

## SIMD and Vectorization

### Auto-Vectorization

Modern compilers auto-vectorize simple loops. Help the compiler by:
- Using contiguous data (arrays, `std::vector`)
- Avoiding loop-carried dependencies
- Using `__restrict` to guarantee no aliasing
- Aligning data to SIMD width boundaries

```cpp
// Easily auto-vectorizable
void add_arrays(float* __restrict out, const float* __restrict a,
                const float* __restrict b, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];
    }
}

// Check auto-vectorization with compiler reports
// GCC: -fopt-info-vec-missed
// Clang: -Rpass=loop-vectorize -Rpass-missed=loop-vectorize
// MSVC: /Qvec-report:2
```

### Compiler Intrinsics

For explicit SIMD control, use intrinsics:

```cpp
#include <immintrin.h>

void add_arrays_avx(float* out, const float* a, const float* b, size_t n) {
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(&a[i]);
        __m256 vb = _mm256_loadu_ps(&b[i]);
        __m256 vr = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&out[i], vr);
    }
    // Scalar remainder
    for (; i < n; ++i) {
        out[i] = a[i] + b[i];
    }
}
```

### std::experimental::simd

Portable SIMD abstraction (part of Parallelism TS v2):

```cpp
#include <experimental/simd>
namespace stdx = std::experimental;

void add_arrays_portable(float* out, const float* a, const float* b, size_t n) {
    using V = stdx::native_simd<float>;
    size_t i = 0;
    for (; i + V::size() <= n; i += V::size()) {
        V va(a + i, stdx::element_aligned);
        V vb(b + i, stdx::element_aligned);
        (va + vb).copy_to(out + i, stdx::element_aligned);
    }
    for (; i < n; ++i) out[i] = a[i] + b[i];
}
```

## Memory Allocators

### Arena Allocator (Linear/Bump Allocator)

Fast allocation, batch deallocation. Ideal for frame-based processing:

```cpp
class ArenaAllocator {
    std::byte* buffer_;
    size_t capacity_;
    size_t offset_ = 0;
public:
    explicit ArenaAllocator(size_t capacity)
        : buffer_(static_cast<std::byte*>(std::malloc(capacity))), capacity_(capacity) {}
    ~ArenaAllocator() { std::free(buffer_); }

    void* allocate(size_t size, size_t alignment = alignof(std::max_align_t)) {
        size_t aligned_offset = (offset_ + alignment - 1) & ~(alignment - 1);
        if (aligned_offset + size > capacity_) throw std::bad_alloc();
        void* ptr = buffer_ + aligned_offset;
        offset_ = aligned_offset + size;
        return ptr;
    }

    void reset() noexcept { offset_ = 0; }  // "Free" everything at once
};
```

### Pool Allocator

Fixed-size block allocation using a free list. O(1) allocate and deallocate:

```cpp
template <typename T>
class PoolAllocator {
    union Block {
        T object;
        Block* next;
    };
    Block* free_list_ = nullptr;
    std::vector<std::unique_ptr<Block[]>> chunks_;
    static constexpr size_t CHUNK_SIZE = 64;

public:
    T* allocate() {
        if (!free_list_) grow();
        Block* block = free_list_;
        free_list_ = block->next;
        return &block->object;
    }

    void deallocate(T* ptr) noexcept {
        auto* block = reinterpret_cast<Block*>(ptr);
        block->next = free_list_;
        free_list_ = block;
    }

private:
    void grow() {
        auto chunk = std::make_unique<Block[]>(CHUNK_SIZE);
        for (size_t i = 0; i < CHUNK_SIZE - 1; ++i) {
            chunk[i].next = &chunk[i + 1];
        }
        chunk[CHUNK_SIZE - 1].next = free_list_;
        free_list_ = &chunk[0];
        chunks_.push_back(std::move(chunk));
    }
};
```

### Polymorphic Memory Resource (PMR)

C++17's `std::pmr` provides allocator-aware containers with runtime-polymorphic allocators:

```cpp
#include <memory_resource>

// Stack-based buffer for small allocations
std::byte buffer[4096];
std::pmr::monotonic_buffer_resource pool(buffer, sizeof(buffer));

// Use PMR containers with the pool
std::pmr::vector<int> vec(&pool);
std::pmr::string str(&pool);
std::pmr::unordered_map<std::pmr::string, int> map(&pool);

// All allocations come from the stack buffer (no heap allocation)
vec.reserve(100);  // Uses the pool
```

## String Optimization

### std::string_view

Zero-copy string references. No allocation or copying:

```cpp
// BAD: unnecessary string copy for read-only access
void log(const std::string& msg);  // Requires std::string argument

// GOOD: accepts any string-like type without copying
void log(std::string_view msg);  // Works with string, char*, string_view, etc.

// Avoid creating temporary strings
std::string_view substr(std::string_view sv, size_t pos, size_t len) {
    return sv.substr(pos, len);  // No allocation — returns a view
}
```

### Small String Optimization (SSO) Details

Most implementations store short strings inline (typically 15-22 bytes including null terminator). Know the SSO threshold for the target platform:

| Implementation | SSO Capacity |
|---------------|-------------|
| libstdc++ (GCC) | 15 bytes |
| libc++ (Clang) | 22 bytes |
| MSVC STL | 15 bytes |

```cpp
// This string fits in SSO — no heap allocation
std::string short_str = "hello";  // 5 chars + null = 6 bytes

// This string exceeds SSO — heap allocated
std::string long_str = "this is a longer string that exceeds SSO";
```

### Avoiding Temporary Strings

```cpp
// BAD: creates temporary std::string from char*
void process(const std::string& s);
process("hello");  // Implicit construction of temporary std::string

// GOOD: use string_view to accept without copying
void process(std::string_view s);
process("hello");  // No temporary — string_view wraps the literal directly
```

## Container Performance

### Container Selection Guide

| Need | Container | Time Complexity |
|------|-----------|----------------|
| Dynamic array, fast random access | `std::vector` | O(1) access, O(1) amortized push_back |
| Sorted unique keys, fast lookup | `std::flat_map` (C++23) or sorted vector | O(log n) lookup |
| Unsorted unique keys, fast lookup | `std::unordered_map` | O(1) average lookup |
| Fast insertion/deletion anywhere | `std::list` (rarely needed) | O(1) insert given iterator |
| Fixed-size array | `std::array` | O(1) access, zero overhead |
| Double-ended queue | `std::deque` | O(1) push_front and push_back |

### Vector Optimization

```cpp
// Reserve to avoid reallocations
std::vector<Widget> widgets;
widgets.reserve(expected_count);

// Emplace instead of push_back to avoid temporaries
widgets.emplace_back(arg1, arg2);

// Shrink to fit after bulk removals
widgets.shrink_to_fit();

// Clear without deallocating (reuse memory)
widgets.clear();  // size = 0, capacity unchanged
```

### Hash Map Tuning

```cpp
std::unordered_map<std::string, int> map;

// Pre-allocate buckets
map.reserve(expected_count);

// Control load factor (default: 1.0)
map.max_load_factor(0.7f);  // Lower = fewer collisions, more memory

// Use transparent comparators for heterogeneous lookup (avoid string copies)
struct StringHash {
    using is_transparent = void;
    size_t operator()(std::string_view sv) const { return std::hash<std::string_view>{}(sv); }
};
std::unordered_map<std::string, int, StringHash, std::equal_to<>> map;
map.find("key");  // No temporary string created
```

## Algorithm Selection

| Task | Best Algorithm | Notes |
|------|---------------|-------|
| Full sort | `std::sort` | O(n log n), introsort |
| Partial sort (top K) | `std::partial_sort` | O(n log K) |
| Kth element | `std::nth_element` | O(n) average |
| Check if sorted | `std::is_sorted` | O(n) |
| Binary search | `std::lower_bound` | O(log n) on sorted range |
| Remove elements | `std::erase_if` (C++20) | O(n), stable |
| Transform | `std::transform` or ranges | O(n) |

## Branch Prediction

### likely/unlikely Attributes (C++20)

```cpp
if (error_code != 0) [[unlikely]] {
    handle_error(error_code);
} else [[likely]] {
    process_normally();
}
```

### Branchless Programming

```cpp
// Branching version
int abs_branch(int x) {
    if (x < 0) return -x;
    return x;
}

// Branchless version
int abs_branchless(int x) {
    int mask = x >> 31;        // All 1s if negative, all 0s if positive
    return (x + mask) ^ mask;  // Conditional negate without branching
}

// Branchless min/max
int min_branchless(int a, int b) {
    return b + ((a - b) & ((a - b) >> 31));
}
```

## Link-Time Optimization (LTO)

LTO allows cross-translation-unit optimization: inlining, dead code elimination, devirtualization.

```cmake
# Enable LTO
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)

# Check if LTO is supported
include(CheckIPOSupported)
check_ipo_supported(RESULT lto_supported)
if(lto_supported)
    set_target_properties(myapp PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()
```

Trade-offs:
- Significantly longer link times (2-10x)
- Better runtime performance (5-20% improvement is typical)
- Higher memory usage during linking
- Can interfere with debugging

## Profile-Guided Optimization (PGO)

Two-phase optimization using runtime profiling data:

```bash
# Phase 1: Instrument
g++ -fprofile-generate -O2 source.cpp -o program_instrumented

# Phase 2: Run representative workload
./program_instrumented < training_data.txt

# Phase 3: Optimize using profile data
g++ -fprofile-use -O2 source.cpp -o program_optimized
```

PGO benefits:
- Better branch prediction hints
- Optimal function layout (hot/cold splitting)
- Better inlining decisions
- Improved register allocation
- Typical improvement: 10-30% for branch-heavy code
