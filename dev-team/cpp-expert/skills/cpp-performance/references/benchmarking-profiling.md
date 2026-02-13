# Benchmarking and Profiling — Deep Dive Reference

## Google Benchmark

### Setup with CMake

```cmake
include(FetchContent)
FetchContent_Declare(benchmark
    GIT_REPOSITORY https://github.com/google/benchmark.git
    GIT_TAG        v1.8.4
)
set(BENCHMARK_ENABLE_TESTING OFF)
FetchContent_MakeAvailable(benchmark)

add_executable(my_benchmarks bench_main.cpp)
target_link_libraries(my_benchmarks PRIVATE benchmark::benchmark)
```

### Basic Benchmarks

```cpp
#include <benchmark/benchmark.h>

static void BM_StringCreation(benchmark::State& state) {
    for (auto _ : state) {
        std::string s("hello world");
        benchmark::DoNotOptimize(s.data());
    }
}
BENCHMARK(BM_StringCreation);

static void BM_StringCopy(benchmark::State& state) {
    std::string src("hello world");
    for (auto _ : state) {
        std::string copy = src;
        benchmark::DoNotOptimize(copy.data());
    }
}
BENCHMARK(BM_StringCopy);

BENCHMARK_MAIN();
```

### Parameterized Benchmarks

```cpp
static void BM_VectorSort(benchmark::State& state) {
    auto n = state.range(0);
    std::vector<int> v(n);
    for (auto _ : state) {
        state.PauseTiming();
        std::iota(v.begin(), v.end(), 0);
        std::ranges::shuffle(v, std::mt19937{42});
        state.ResumeTiming();

        std::ranges::sort(v);
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(n);
}
BENCHMARK(BM_VectorSort)
    ->RangeMultiplier(10)
    ->Range(100, 1'000'000)
    ->Complexity(benchmark::oNLogN);
```

### Benchmark Fixtures

```cpp
class DatabaseFixture : public benchmark::Fixture {
public:
    Database db;

    void SetUp(const benchmark::State& state) override {
        db.connect("localhost:5432");
        db.seed(state.range(0));
    }

    void TearDown(const benchmark::State& state) override {
        db.disconnect();
    }
};

BENCHMARK_DEFINE_F(DatabaseFixture, BM_Query)(benchmark::State& state) {
    for (auto _ : state) {
        auto result = db.query("SELECT * FROM users LIMIT 100");
        benchmark::DoNotOptimize(result);
    }
}
BENCHMARK_REGISTER_F(DatabaseFixture, BM_Query)->Range(1000, 100000);
```

### Custom Counters

```cpp
static void BM_Throughput(benchmark::State& state) {
    auto n = state.range(0);
    std::vector<char> data(n);
    for (auto _ : state) {
        process(data);
        benchmark::DoNotOptimize(data.data());
    }
    state.SetBytesProcessed(int64_t(state.iterations()) * n);
    state.counters["items_per_second"] =
        benchmark::Counter(state.iterations() * n, benchmark::Counter::kIsRate);
}
```

### Avoiding Common Pitfalls

```cpp
// BAD: loop body optimized away
static void BM_Bad(benchmark::State& state) {
    for (auto _ : state) {
        int result = compute(42);
        // Result unused — compiler may eliminate compute()
    }
}

// GOOD: prevent dead code elimination
static void BM_Good(benchmark::State& state) {
    for (auto _ : state) {
        int result = compute(42);
        benchmark::DoNotOptimize(result);
    }
}

// GOOD: prevent memory operations from being reordered out of the loop
static void BM_Memory(benchmark::State& state) {
    std::vector<int> v(1000);
    for (auto _ : state) {
        modify_vector(v);
        benchmark::ClobberMemory();  // Force memory writes to be visible
    }
}
```

## perf (Linux)

### Recording and Analysis

```bash
# Record CPU cycles with call graph
perf record -g --call-graph dwarf ./my_program

# Record specific events
perf record -e cache-misses,cache-references,branch-misses ./my_program

# View the report interactively
perf report

# Annotate source code with per-line cycle counts
perf annotate --source

# Statistical summary (no recording needed)
perf stat -d ./my_program
# -d adds detailed cache and TLB statistics
```

### Flame Graphs

```bash
# Generate flame graph from perf data
perf record -g --call-graph dwarf ./my_program
perf script > perf.data.txt
stackcollapse-perf.pl perf.data.txt > perf.folded
flamegraph.pl perf.folded > flamegraph.svg

# Differential flame graph (compare two runs)
difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg
```

Reading flame graphs:
- X-axis: alphabetical (NOT time). Width = percentage of samples
- Y-axis: stack depth. Bottom = entry point, top = leaf function
- Wide boxes at the top are where CPU time is spent
- Look for "plateaus" — wide, flat areas indicate hot code

### Useful perf Events

| Event | What It Measures |
|-------|-----------------|
| `cycles` | CPU cycles (default) |
| `instructions` | Instructions retired |
| `cache-misses` | Last-level cache misses |
| `cache-references` | Last-level cache accesses |
| `branch-misses` | Branch mispredictions |
| `L1-dcache-load-misses` | L1 data cache misses |
| `dTLB-load-misses` | Data TLB misses |
| `page-faults` | Page faults |

## Intel VTune

### Hotspot Analysis

```bash
# Collect hotspot data
vtune -collect hotspots -- ./my_program

# View results
vtune -report hotspots -r r000hs

# Top-down microarchitecture analysis
vtune -collect uarch-exploration -- ./my_program
```

### Memory Access Analysis

```bash
vtune -collect memory-access -- ./my_program
# Shows: cache miss rates, memory bandwidth, NUMA locality
```

### Threading Analysis

```bash
vtune -collect threading -- ./my_program
# Shows: thread imbalance, lock contention, synchronization overhead
```

## Valgrind / Callgrind

### Cache Simulation

```bash
# Run with Callgrind for instruction-level profiling
valgrind --tool=callgrind ./my_program

# Analyze results
callgrind_annotate callgrind.out.<pid>

# Interactive visualization
kcachegrind callgrind.out.<pid>
```

### Cachegrind

```bash
# Simulate cache behavior
valgrind --tool=cachegrind ./my_program

# Output shows:
# I1 cache: instruction cache hits/misses
# D1 cache: L1 data cache hits/misses
# LL cache: last-level cache hits/misses
```

## Compiler Explorer (Godbolt)

Use Compiler Explorer (godbolt.org) to inspect generated assembly:

### What to Look For

1. **Inlining**: Small functions should disappear, their body merged into callers
2. **Vectorization**: Look for SIMD instructions (vaddps, vmulps for AVX; addps, mulps for SSE)
3. **Branch elimination**: `cmov` instructions instead of `jmp`/`je` indicate branchless code
4. **Register allocation**: Values staying in registers vs spilling to stack
5. **Loop unrolling**: Multiple iterations in a single loop body

### Useful Flags for Inspection

```
-O2 -g                    # Optimized with debug info
-O2 -march=native         # Enable all CPU-specific optimizations
-O2 -fopt-info-vec-all    # GCC: vectorization report
-O2 -Rpass=loop-vectorize # Clang: vectorization report
```

## Benchmarking Methodology

### Scientific Benchmarking Checklist

1. **Warm up**: Run several iterations before measuring to warm caches and trigger JIT
2. **Multiple iterations**: Run enough iterations for statistical significance (Google Benchmark handles this)
3. **Control variables**: Fix CPU frequency (disable turbo boost), close other applications, pin to specific CPU core
4. **Statistical analysis**: Report median and standard deviation, not just mean
5. **Reproducibility**: Use fixed seeds for random data, version-control benchmark code

### System Configuration for Reliable Benchmarks

```bash
# Linux: disable CPU frequency scaling
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Linux: pin process to CPU core
taskset -c 0 ./my_benchmark

# Linux: disable turbo boost
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# Verify with
cat /proc/cpuinfo | grep MHz
```

### What NOT to Benchmark

- Debug builds (`-O0`) — results are meaningless for production performance
- Tiny operations in isolation — measurement overhead dominates
- Code with I/O or network calls — high variance overwhelms computation differences
- First run only — must warm caches and trigger page faults first
- Different compiler settings — compare apples to apples
