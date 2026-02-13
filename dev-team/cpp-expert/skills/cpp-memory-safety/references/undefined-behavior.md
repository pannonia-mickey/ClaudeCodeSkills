# Undefined Behavior — Comprehensive Reference

## UB Categories

### Memory UB

#### Use-After-Free

Accessing heap memory after it has been deallocated:

```cpp
// BAD
int* p = new int(42);
delete p;
std::cout << *p;  // UB: use-after-free

// GOOD: use unique_ptr, access only through smart pointer
auto p = std::make_unique<int>(42);
// p automatically manages lifetime
```

#### Buffer Overflow

Reading or writing past the bounds of an array or container:

```cpp
// BAD
int arr[5];
arr[5] = 42;  // UB: out-of-bounds write

// BAD: std::vector operator[] does not bounds-check
std::vector<int> v = {1, 2, 3};
auto val = v[10];  // UB: out-of-bounds read

// GOOD: use .at() for bounds-checked access
try {
    auto val = v.at(10);  // Throws std::out_of_range
} catch (const std::out_of_range& e) { /* handle */ }

// GOOD: use std::span for bounds-safe array access
void process(std::span<const int> data) {
    for (auto val : data) { /* safe iteration */ }
}
```

#### Null Pointer Dereference

```cpp
// BAD
Widget* w = nullptr;
w->process();  // UB: null dereference

// GOOD: check before use, or use references for non-nullable
void process(Widget& w) { w.process(); }  // Cannot be null
```

#### Uninitialized Read

```cpp
// BAD
int x;
if (x > 0) { /* ... */ }  // UB: reading uninitialized variable

// GOOD: always initialize
int x = 0;
int y{};  // Value-initialized to 0
```

#### Double Free

```cpp
// BAD
int* p = new int(42);
delete p;
delete p;  // UB: double free

// GOOD: use smart pointers exclusively
auto p = std::make_unique<int>(42);
// No manual delete needed
```

### Numeric UB

#### Signed Integer Overflow

```cpp
// BAD
int x = INT_MAX;
int y = x + 1;  // UB: signed overflow

// GOOD: check before operation
if (x <= INT_MAX - 1) { int y = x + 1; }

// GOOD: use unsigned for modular arithmetic
unsigned int x = UINT_MAX;
unsigned int y = x + 1;  // Well-defined: wraps to 0

// GOOD: use safe integer library or wider types
auto result = static_cast<int64_t>(x) + 1;
```

#### Division by Zero

```cpp
// BAD
int a = 10, b = 0;
int c = a / b;  // UB: integer division by zero

// GOOD: check divisor
if (b != 0) { int c = a / b; }
```

#### Invalid Shift

```cpp
// BAD
int x = 1;
int y = x << 32;   // UB: shift amount >= bit width (32 for int)
int z = x << -1;   // UB: negative shift amount
int w = -1 << 1;   // UB: shifting negative value (until C++20)

// GOOD: validate shift amount
if (shift >= 0 && shift < 32) { int y = x << shift; }
```

### Type UB

#### Strict Aliasing Violation

Accessing an object through a pointer of incompatible type:

```cpp
// BAD: type punning via pointer cast
float f = 3.14f;
int i = *(int*)&f;  // UB: strict aliasing violation

// GOOD: use std::bit_cast (C++20)
int i = std::bit_cast<int>(f);

// GOOD: use memcpy
int i;
std::memcpy(&i, &f, sizeof(i));
```

#### Invalid Enum Value

```cpp
// BAD
enum Color { Red = 0, Green = 1, Blue = 2 };
Color c = static_cast<Color>(42);  // UB if enum range doesn't include 42
```

#### Object Lifetime Violations

```cpp
// BAD: accessing object before construction or after destruction
struct Widget {
    int value;
    Widget() : value(42) {}
};

alignas(Widget) char buffer[sizeof(Widget)];
auto* w = reinterpret_cast<Widget*>(buffer);
w->value = 10;  // UB: no Widget object exists at this address

// GOOD: use placement new
auto* w = new (buffer) Widget();
w->value = 10;  // OK: Widget object exists
w->~Widget();   // Explicitly destroy
```

### Concurrency UB

#### Data Race

Two threads accessing the same memory location where at least one is a write, with no synchronization:

```cpp
// BAD: data race
int counter = 0;
// Thread 1: ++counter;  // UB
// Thread 2: ++counter;  // UB

// GOOD: use atomic
std::atomic<int> counter{0};
// Thread 1: counter.fetch_add(1);  // OK
// Thread 2: counter.fetch_add(1);  // OK

// GOOD: use mutex
std::mutex mtx;
int counter = 0;
// Thread 1: { std::lock_guard lock(mtx); ++counter; }  // OK
// Thread 2: { std::lock_guard lock(mtx); ++counter; }  // OK
```

## Detection Strategies

### Compiler Warnings

Enable maximum warnings as the first line of defense:

```bash
# GCC/Clang
-Wall -Wextra -Wpedantic -Wshadow -Wconversion -Wsign-conversion
-Wnon-virtual-dtor -Wold-style-cast -Woverloaded-virtual
-Wnull-dereference -Wformat=2

# Treat warnings as errors in CI
-Werror

# MSVC
/W4 /WX
```

### Static Analysis

#### clang-tidy

```bash
# Run clang-tidy with comprehensive checks
clang-tidy --checks='bugprone-*,cppcoreguidelines-*,clang-analyzer-*' source.cpp

# Key checks for UB:
# bugprone-use-after-move
# bugprone-dangling-handle
# bugprone-integer-division
# cppcoreguidelines-pro-bounds-array-to-pointer-decay
# clang-analyzer-core.NullDereference
# clang-analyzer-core.UndefinedBinaryOperatorResult
```

#### PVS-Studio

Commercial static analyzer with strong UB detection. Finds issues that clang-tidy misses: complex data flow paths, inter-procedural analysis, 64-bit portability.

#### cppcheck

```bash
cppcheck --enable=all --std=c++20 --inconclusive source.cpp
```

### Runtime Sanitizers

#### AddressSanitizer (ASan)

Detects: heap-buffer-overflow, stack-buffer-overflow, heap-use-after-free, stack-use-after-scope, double-free, memory leaks.

```bash
# Compile and link
g++ -fsanitize=address -fno-omit-frame-pointer -g source.cpp -o program

# Run with options
ASAN_OPTIONS=detect_leaks=1:halt_on_error=0 ./program
```

Typical ASan output:
```
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x60200000efd0
READ of size 4 at 0x60200000efd0 thread T0
    #0 0x4c3a5f in main source.cpp:15
    #1 0x7f... in __libc_start_main

0x60200000efd0 is located 0 bytes inside of 4-byte region
freed by thread T0 here:
    #0 0x4c2d3f in operator delete source.cpp:12
```

#### UndefinedBehaviorSanitizer (UBSan)

Detects: signed integer overflow, invalid shifts, null pointer dereference, misaligned access, float-cast overflow, invalid bool, array bounds.

```bash
g++ -fsanitize=undefined -fno-sanitize-recover=all -g source.cpp -o program
```

#### MemorySanitizer (MSan)

Detects: use of uninitialized memory. Requires the entire program and all libraries to be instrumented (typically requires building libc++ with MSan).

```bash
clang++ -fsanitize=memory -fPIE -pie -g source.cpp -o program
```

#### ThreadSanitizer (TSan)

Detects: data races, deadlocks, lock order inversions.

```bash
g++ -fsanitize=thread -g source.cpp -o program
```

### Valgrind

Runtime memory error detector (does not require recompilation):

```bash
# Memcheck — memory errors
valgrind --leak-check=full --show-leak-kinds=all ./program

# Helgrind — data races
valgrind --tool=helgrind ./program

# Cachegrind — cache usage
valgrind --tool=cachegrind ./program
```

Valgrind is slower than sanitizers (10-50x slowdown) but does not require recompilation and can instrument third-party libraries.

## Hardening Techniques

### Bounds-Checked Containers

```cpp
// Use .at() for checked access during development
auto val = vec.at(i);  // Throws on out-of-bounds

// Use hardened libc++ mode (Clang)
// -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST  (asserts on UB)
// -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_EXTENSIVE  (more checks)

// Use std::span for safe array access
void process(std::span<const int> data) {
    // Iteration is bounds-safe
    for (auto val : data) { /* ... */ }
}
```

### Safe Integer Arithmetic

```cpp
// Check for overflow before operation
template <std::integral T>
std::optional<T> safe_add(T a, T b) {
    if (b > 0 && a > std::numeric_limits<T>::max() - b) return std::nullopt;
    if (b < 0 && a < std::numeric_limits<T>::min() - b) return std::nullopt;
    return a + b;
}

// Or use compiler builtins
int result;
if (__builtin_add_overflow(a, b, &result)) {
    // Handle overflow
}
```

### Initialization Discipline

```cpp
// Always initialize variables
int x{};              // Zero-initialized
auto y = int{};       // Zero-initialized
std::vector<int> v{}; // Empty vector

// Use -Wuninitialized and -Wsometimes-uninitialized compiler warnings
// Use -ftrivial-auto-var-init=pattern to poison uninitialized stack variables
```

### Lifetime Safety

```cpp
// Prefer returning by value (RVO eliminates copies)
std::string make_greeting(std::string_view name) {
    return std::format("Hello, {}!", name);  // RVO applies
}

// Never return references to locals
// Never store references/pointers to temporaries
// Use string_view/span only for non-owning parameters, not for storage
```

## Real-World UB Examples

### Example 1: Iterator Invalidation in Loop

```cpp
// BAD: erase invalidates iterators
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it % 2 == 0) {
        v.erase(it);  // Invalidates it and all subsequent iterators
    }
}

// GOOD (C++20)
std::erase_if(v, [](int x) { return x % 2 == 0; });

// GOOD (pre-C++20)
v.erase(std::remove_if(v.begin(), v.end(),
    [](int x) { return x % 2 == 0; }), v.end());
```

### Example 2: Dangling string_view

```cpp
// BAD: string_view pointing to destroyed temporary
std::string_view sv = std::string("hello") + " world";
// Temporary string destroyed, sv is dangling

// GOOD: ensure the underlying string outlives the view
std::string s = std::string("hello") + " world";
std::string_view sv = s;
```

### Example 3: Move-From Access

```cpp
// BAD: using object after move (valid but unspecified state)
std::vector<int> v = {1, 2, 3};
auto v2 = std::move(v);
std::cout << v.size();  // Valid, but value is unspecified
v[0] = 10;              // May be UB if v is now empty

// GOOD: only assign or destroy after move
std::vector<int> v = {1, 2, 3};
auto v2 = std::move(v);
v.clear();  // OK: puts v in known state
v = {4, 5, 6};  // OK: reassignment
```
