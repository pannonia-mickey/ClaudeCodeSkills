---
name: C++ Modern Features
description: This skill should be used when the user asks about "C++20 features", "C++23 features", "C++ concepts", "C++ ranges", "C++ modules", "structured bindings", "move semantics", "std::expected", "constexpr", "consteval", "fold expressions", "deducing this", "C++ coroutines", "std::format", or "std::optional". It covers modern C++ language features from C++11 through C++23 with migration guidance and idiomatic usage patterns.
version: 0.1.0
---

# C++ Modern Features

Modern C++ (C++11 through C++23) introduces powerful language features that improve safety, expressiveness, and performance. This skill covers the most impactful features with idiomatic usage patterns.

## Move Semantics and Value Categories

Move semantics enable transferring resources from temporary objects instead of copying them. Understand three value categories: lvalues (named, addressable), prvalues (pure rvalues, temporaries), and xvalues (expiring values via std::move).

Use `std::move` to cast an lvalue to an rvalue reference, enabling move construction or assignment. Use `std::forward` in template code to preserve the value category of forwarded arguments (perfect forwarding).

```cpp
class Buffer {
    std::unique_ptr<char[]> data_;
    size_t size_;
public:
    // Move constructor — transfers ownership, leaves source empty
    Buffer(Buffer&& other) noexcept
        : data_(std::move(other.data_)), size_(std::exchange(other.size_, 0)) {}

    // Move assignment — swap idiom for exception safety
    Buffer& operator=(Buffer&& other) noexcept {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
        return *this;
    }
};

// Perfect forwarding in a factory function
template <typename T, typename... Args>
std::unique_ptr<T> make(Args&&... args) {
    return std::make_unique<T>(std::forward<Args>(args)...);
}
```

## Structured Bindings (C++17)

Decompose aggregates, tuples, and pairs into named variables. Apply structured bindings in range-for loops, if-init statements, and switch-init statements.

```cpp
// Decompose map entries
std::map<std::string, int> scores = {{"Alice", 95}, {"Bob", 87}};
for (const auto& [name, score] : scores) {
    std::println("{}: {}", name, score);
}

// With if-init statement
if (auto [iter, inserted] = scores.emplace("Charlie", 91); !inserted) {
    std::println("Key already exists with value {}", iter->second);
}
```

## Concepts and Constraints (C++20)

Define named sets of requirements on template parameters using concepts. Replace SFINAE with readable, composable constraints that produce clear error messages.

```cpp
// Define a custom concept
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template <typename T>
concept Summable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

// Constrain a function template
template <Arithmetic T>
T clamp_add(T a, T b, T max_val) {
    auto result = a + b;
    return result > max_val ? max_val : result;
}

// Constrain with requires clause
template <typename Container>
    requires std::ranges::range<Container>
          && Summable<std::ranges::range_value_t<Container>>
auto sum(const Container& c) {
    using T = std::ranges::range_value_t<Container>;
    return std::accumulate(std::ranges::begin(c), std::ranges::end(c), T{});
}
```

## Ranges and Views (C++20/23)

Compose lazy data transformation pipelines using range adaptors and the pipe operator. Views are non-owning, lazily evaluated, and composable.

```cpp
#include <ranges>
#include <vector>
#include <string>

std::vector<std::string> names = {"Alice", "Bob", "Charlie", "Diana", "Eve"};

// Compose a pipeline: filter, transform, take
auto result = names
    | std::views::filter([](const auto& n) { return n.size() > 3; })
    | std::views::transform([](const auto& n) { return n.substr(0, 3); })
    | std::views::take(2);

for (const auto& name : result) {
    std::println("{}", name);  // "Ali", "Cha"
}

// C++23: enumerate and zip
for (auto [i, val] : std::views::enumerate(names)) {
    std::println("[{}] {}", i, val);
}
```

## std::optional, std::variant, std::expected

Use `std::optional<T>` for values that may not exist. Use `std::variant<Ts...>` for type-safe unions. Use `std::expected<T, E>` (C++23) for operations that may fail with a typed error.

Chain operations using monadic methods on optional and expected:

```cpp
// std::expected with monadic chaining (C++23)
std::expected<int, std::string> parse_int(std::string_view sv);
std::expected<double, std::string> to_celsius(int fahrenheit);

auto result = parse_int("72")
    .and_then(to_celsius)
    .transform([](double c) { return std::format("{:.1f}C", c); })
    .or_else([](const std::string& err) -> std::expected<std::string, std::string> {
        return std::unexpected(std::format("Failed: {}", err));
    });

// std::variant with overloaded visitor
using Shape = std::variant<Circle, Rectangle, Triangle>;

auto area = std::visit(overloaded{
    [](const Circle& c)    { return std::numbers::pi * c.radius * c.radius; },
    [](const Rectangle& r) { return r.width * r.height; },
    [](const Triangle& t)  { return 0.5 * t.base * t.height; }
}, shape);
```

## constexpr and consteval

Mark functions and variables `constexpr` to allow compile-time evaluation. Use `consteval` (C++20) to guarantee compile-time-only execution. Use `constinit` to ensure constant initialization of static/thread-local variables.

```cpp
// constexpr function — usable at both compile-time and runtime
constexpr auto fibonacci(int n) -> int {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}

// consteval — compile-time only
consteval auto compile_time_hash(std::string_view sv) -> uint64_t {
    uint64_t hash = 0xcbf29ce484222325ULL;
    for (char c : sv) {
        hash ^= static_cast<uint64_t>(c);
        hash *= 0x100000001b3ULL;
    }
    return hash;
}

// Usage: guaranteed compile-time evaluation
constexpr auto fib10 = fibonacci(10);          // OK: compile-time
static_assert(compile_time_hash("hello") != 0); // OK: consteval
```

## Modules (C++20)

Replace header files with modules for faster compilation and better encapsulation. Export only the intended public interface.

```cpp
// math_utils.cppm — module interface unit
export module math_utils;

export template <typename T>
constexpr T square(T x) { return x * x; }

export constexpr double pi = 3.14159265358979323846;

// Internal (not exported) — invisible to importers
double internal_helper(double x) { return x * x * x; }
```

```cpp
// main.cpp — consumer
import math_utils;

int main() {
    auto result = square(5);    // OK
    // internal_helper(3.0);    // Error: not exported
}
```

## std::format (C++20)

Use `std::format` for type-safe, extensible string formatting. Replaces printf (unsafe) and iostream (verbose).

```cpp
auto msg = std::format("Name: {}, Score: {:.2f}, Rank: {:>5}", name, 95.123, 3);
// Custom formatter for user types via std::formatter specialization
```

## Deducing this (C++23)

Use explicit object parameters to deduplicate const/non-const overloads and replace CRTP for some use cases.

```cpp
class Widget {
    std::vector<int> data_;
public:
    // Single function replaces const and non-const overloads
    template <typename Self>
    auto&& get_data(this Self&& self) {
        return std::forward<Self>(self).data_;
    }

    // Recursive lambda enabled by deducing this
    void traverse(auto visitor) {
        auto recurse = [&](this auto& self, int depth) -> void {
            visitor(depth);
            if (depth > 0) self(depth - 1);
        };
        recurse(10);
    }
};
```

## Additional Resources

### Reference Files

For detailed coverage of specific standard versions, consult:

- **`references/cpp20-features.md`** — Comprehensive C++20 guide: concepts full syntax, ranges adaptors, coroutines machinery, three-way comparison, modules, std::format customization, constexpr improvements
- **`references/cpp23-features.md`** — Comprehensive C++23 guide: std::expected monadic operations, deducing this patterns, std::print, flat_map/flat_set, std::generator, std::stacktrace, C++26 preview
