---
name: C++ Performance
description: This skill should be used when the user asks about "C++ optimization", "C++ profiling", "cache performance", "move vs copy", "compile-time computation", "C++ benchmarking", "Google Benchmark", "perf", "VTune", "SIMD", "small buffer optimization", "SBO", "data-oriented design", or "C++ inlining". It covers performance analysis, optimization techniques, cache-friendly data layout, and benchmarking practices for C++ applications.
version: 0.1.0
---

# C++ Performance

Performance optimization begins with measurement. This skill covers profiling, cache-friendly design, move semantics for performance, compile-time computation, and benchmarking methodology.

## Profiling First

Always measure before optimizing. Profile in release mode with debug info (`-O2 -g`) to get representative performance data with readable stack traces.

```bash
# Linux perf — record and generate flame graph
perf record -g ./my_program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg

# perf stat — hardware counters summary
perf stat -e cache-misses,cache-references,branch-misses,instructions ./my_program

# Intel VTune — hotspot analysis
vtune -collect hotspots -result-dir r001 ./my_program
vtune -report hotspots -result-dir r001
```

Key metrics to examine:
- **CPU time distribution** — identify hot functions via flame graphs
- **Cache miss rate** — L1/L2/L3 miss ratios indicate data layout problems
- **Branch mispredictions** — suggest branchless alternatives or data reordering
- **Instructions per cycle (IPC)** — low IPC suggests memory-bound or branch-bound code

## Cache-Friendly Data Layout

CPU caches operate on 64-byte cache lines. Optimize for spatial locality (access sequential memory) and temporal locality (reuse recently accessed data).

**Array of Structs (AoS) vs Struct of Arrays (SoA)**: When processing only a subset of fields, SoA avoids loading unused fields into cache lines.

```cpp
// AoS — poor cache utilization when only accessing positions
struct ParticleAoS {
    float x, y, z;       // Position (used in physics update)
    float r, g, b, a;    // Color (unused in physics update)
    float life, size;     // Metadata (unused in physics update)
};
std::vector<ParticleAoS> particles_aos;  // 36 bytes per particle, only 12 used

// SoA — excellent cache utilization for position-only access
struct ParticlesSoA {
    std::vector<float> x, y, z;       // Positions packed together
    std::vector<float> r, g, b, a;    // Colors packed separately
    std::vector<float> life, size;     // Metadata packed separately
};
ParticlesSoA particles_soa;  // Physics loop touches only x, y, z arrays
```

Use `alignas` for cache-line alignment to prevent false sharing in multithreaded code:

```cpp
struct alignas(64) ThreadLocalCounter {
    std::atomic<uint64_t> count{0};
    // Padding to 64 bytes prevents false sharing with adjacent counters
};
```

## Move Semantics for Performance

Avoid unnecessary copies by leveraging move semantics, return value optimization (RVO/NRVO), and emplace operations.

```cpp
// RVO — compiler elides copy/move entirely
std::vector<int> make_data(int n) {
    std::vector<int> v(n);
    std::iota(v.begin(), v.end(), 0);
    return v;  // Guaranteed copy elision (C++17) or NRVO
}

// Emplace instead of push_back to construct in-place
std::vector<std::pair<std::string, int>> entries;
entries.emplace_back("key", 42);  // Constructs in-place, no temporary

// Reserve to avoid reallocations
std::vector<Widget> widgets;
widgets.reserve(1000);  // One allocation instead of ~10 reallocations
for (int i = 0; i < 1000; ++i) {
    widgets.emplace_back(i);
}
```

Move costs by type:
- `std::unique_ptr`: O(1) — pointer swap
- `std::vector`, `std::string` (heap): O(1) — pointer + size swap
- `std::string` (SSO): O(n) — small strings copied, not moved
- `std::array<T, N>`: O(n) — must move each element

## Compile-Time Computation

Shift computation from runtime to compile-time using `constexpr` and `consteval`.

```cpp
// Compile-time CRC32 lookup table
consteval auto make_crc32_table() {
    std::array<uint32_t, 256> table{};
    for (uint32_t i = 0; i < 256; ++i) {
        uint32_t crc = i;
        for (int j = 0; j < 8; ++j) {
            crc = (crc >> 1) ^ (0xEDB88320 & (-(crc & 1)));
        }
        table[i] = crc;
    }
    return table;
}

constexpr auto crc32_table = make_crc32_table();  // Computed at compile time

constexpr uint32_t crc32(std::span<const uint8_t> data) {
    uint32_t crc = 0xFFFFFFFF;
    for (auto byte : data) {
        crc = crc32_table[(crc ^ byte) & 0xFF] ^ (crc >> 8);
    }
    return crc ^ 0xFFFFFFFF;
}
```

## Small Buffer Optimization (SBO)

SBO stores small objects inline to avoid heap allocation. `std::string` typically uses SBO for strings up to 15-22 characters (implementation-dependent). `std::function` uses SBO for small callables.

```cpp
// Custom SBO container — stores small objects inline, large on heap
template <typename T, size_t BufSize = 64>
class SmallObject {
    alignas(T) std::byte buffer_[BufSize];
    T* ptr_ = nullptr;
    bool on_heap_ = false;

public:
    template <typename... Args>
    void emplace(Args&&... args) {
        if constexpr (sizeof(T) <= BufSize) {
            ptr_ = new (buffer_) T(std::forward<Args>(args)...);
            on_heap_ = false;
        } else {
            ptr_ = new T(std::forward<Args>(args)...);
            on_heap_ = true;
        }
    }
    // ... destructor handles both cases
};
```

When SBO helps: many short-lived small objects (small strings, small functors). When SBO hurts: moves become copies for the inline case, and the fixed buffer wastes space for empty objects.

## Inlining and Devirtualization

Enable Link-Time Optimization (LTO) for cross-translation-unit inlining:

```cmake
# Enable LTO in CMake
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)

# Or per-target
set_target_properties(my_app PROPERTIES INTERPROCEDURAL_OPTIMIZATION TRUE)
```

Use `final` on classes and virtual methods to enable devirtualization:

```cpp
class Renderer final : public IRenderer {
    void draw(const Mesh& m) override final { /* ... */ }
};
// Compiler can devirtualize calls through Renderer* because class is final
```

Use CRTP for compile-time polymorphism when the set of types is known at compile time, eliminating virtual dispatch entirely.

## Benchmarking

Use Google Benchmark for reliable microbenchmarks. Prevent dead code elimination with `benchmark::DoNotOptimize` and `benchmark::ClobberMemory`.

```cpp
#include <benchmark/benchmark.h>

static void BM_VectorPushBack(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        for (int i = 0; i < state.range(0); ++i) {
            v.push_back(i);
        }
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(state.range(0));
}

static void BM_VectorReserved(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(state.range(0));
        for (int i = 0; i < state.range(0); ++i) {
            v.push_back(i);
        }
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(state.range(0));
}

BENCHMARK(BM_VectorPushBack)->RangeMultiplier(10)->Range(10, 100000)->Complexity();
BENCHMARK(BM_VectorReserved)->RangeMultiplier(10)->Range(10, 100000)->Complexity();
BENCHMARK_MAIN();
```

Common pitfalls: measuring debug builds, not warming up caches, benchmark loop getting optimized away, comparing results across different machines or power states.

## Additional Resources

### Reference Files

For in-depth optimization and profiling techniques, consult:

- **`references/optimization-techniques.md`** — SIMD/vectorization, memory allocators (arena, pool, PMR), string optimization, container performance, algorithm selection, branch prediction, LTO, PGO
- **`references/benchmarking-profiling.md`** — Google Benchmark full setup, perf/VTune/Valgrind usage, Compiler Explorer tips, benchmarking methodology and statistical analysis
