# C++23 Features — Comprehensive Reference

## std::expected

`std::expected<T, E>` represents either a value of type `T` or an error of type `E`. It replaces the pattern of returning error codes or throwing exceptions for expected failure modes.

### Construction and Access

```cpp
#include <expected>

std::expected<int, std::string> parse_port(std::string_view sv) {
    int port = 0;
    auto [ptr, ec] = std::from_chars(sv.data(), sv.data() + sv.size(), port);
    if (ec != std::errc{}) return std::unexpected("Invalid number");
    if (port < 1 || port > 65535) return std::unexpected("Port out of range");
    return port;
}

auto result = parse_port("8080");
if (result) {
    use_port(result.value());  // or *result
} else {
    log_error(result.error());
}
```

### Monadic Operations

Chain operations without manual error checking:

```cpp
// .and_then(f) — if has value, apply f (f returns expected)
// .transform(f) — if has value, apply f (f returns T, wrapped in expected)
// .or_else(f) — if has error, apply f (f returns expected)
// .transform_error(f) — if has error, apply f (f returns E, wrapped in unexpected)

std::expected<Config, Error> load_config(std::string_view path);
std::expected<Database, Error> connect_db(const Config& cfg);
std::expected<void, Error> run_migrations(Database& db);

auto result = load_config("/etc/app.toml")
    .and_then(connect_db)
    .and_then([](Database& db) { return run_migrations(db); })
    .transform_error([](Error e) {
        return Error{std::format("Startup failed: {}", e.message)};
    });
```

### Comparison with std::optional

| Feature | `std::optional<T>` | `std::expected<T, E>` |
|---------|-------------------|----------------------|
| Error info | None (empty = error) | Typed error `E` |
| Monadic ops | `.and_then`, `.transform`, `.or_else` | Same + `.transform_error` |
| Use case | Value may not exist | Operation may fail with reason |

## Deducing this

Explicit object parameter allows member functions to deduce their own object type, eliminating const/non-const duplication and replacing simple CRTP.

### Deduplicating const Overloads

```cpp
// Before C++23: two nearly identical functions
class Widget {
    std::vector<int> data_;
public:
    const std::vector<int>& get_data() const { return data_; }
    std::vector<int>& get_data() { return data_; }
};

// C++23: single function with deducing this
class Widget {
    std::vector<int> data_;
public:
    template <typename Self>
    auto&& get_data(this Self&& self) {
        return std::forward<Self>(self).data_;
    }
};
```

### Replacing CRTP

```cpp
// CRTP version
template <typename Derived>
struct AddEquality {
    bool operator!=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) == other);
    }
};

// Deducing this version — no CRTP needed
struct AddEquality {
    bool operator!=(this auto const& self, auto const& other) {
        return !(self == other);
    }
};
```

### Recursive Lambdas

```cpp
// Before C++23: requires std::function or Y-combinator
auto fib = [](this auto& self, int n) -> int {
    if (n <= 1) return n;
    return self(n - 1) + self(n - 2);
};

auto result = fib(10);  // 55
```

### Mixin Chaining

```cpp
struct Fluent {
    template <typename Self>
    Self&& set_name(this Self&& self, std::string name) {
        self.name_ = std::move(name);
        return std::forward<Self>(self);
    }

    template <typename Self>
    Self&& set_value(this Self&& self, int value) {
        self.value_ = value;
        return std::forward<Self>(self);
    }
};

struct Config : Fluent {
    std::string name_;
    int value_ = 0;
};

auto cfg = Config{}.set_name("app").set_value(42);  // Returns Config, not Fluent
```

## std::print and std::println

Type-safe, efficient formatted output to stdout/stderr:

```cpp
#include <print>

std::println("Hello, {}!", name);           // With newline
std::print("Progress: {:.1f}%", 73.5);     // Without newline
std::println(stderr, "Error: {}", msg);     // To stderr
```

Advantages over `std::cout`: type-safe like `std::format`, no `std::endl` flushing overhead, cleaner syntax, supports all `std::format` specifiers.

## Multidimensional Subscript Operator

```cpp
template <typename T>
class Matrix {
    std::vector<T> data_;
    size_t rows_, cols_;
public:
    // C++23: multidimensional operator[]
    T& operator[](size_t row, size_t col) {
        return data_[row * cols_ + col];
    }

    const T& operator[](size_t row, size_t col) const {
        return data_[row * cols_ + col];
    }
};

Matrix<double> m(3, 3);
m[1, 2] = 3.14;  // Direct multidimensional access
```

## if consteval

Test at compile-time whether the current context is a constant evaluation:

```cpp
constexpr double power(double base, int exp) {
    if consteval {
        // Compile-time: use simple loop (no std::pow which may not be constexpr)
        double result = 1.0;
        for (int i = 0; i < exp; ++i) result *= base;
        return result;
    } else {
        // Runtime: use optimized std::pow
        return std::pow(base, exp);
    }
}
```

## std::flat_map and std::flat_set

Cache-friendly sorted associative containers backed by sorted vectors instead of trees:

```cpp
#include <flat_map>
#include <flat_set>

std::flat_map<std::string, int> scores;
scores.insert({"Alice", 95});
scores["Bob"] = 87;

// Iteration is faster than std::map due to contiguous memory
for (const auto& [name, score] : scores) {
    std::println("{}: {}", name, score);
}
```

**Performance tradeoffs**:
- Faster iteration and lookup (cache-friendly contiguous storage)
- Slower insertion/deletion (O(n) shifting)
- Lower memory overhead (no per-node allocation)
- Best for read-heavy workloads with infrequent modification

## std::generator

Standard lazy generator coroutine type:

```cpp
#include <generator>

std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Compose with ranges
for (auto val : fibonacci() | std::views::take(10)) {
    std::println("{}", val);
}

// Recursive generator (yields elements from inner generators)
std::generator<int> flatten(std::vector<std::vector<int>>& vv) {
    for (auto& v : vv) {
        co_yield std::ranges::elements_of(v);
    }
}
```

## std::stacktrace

Capture and display stack traces programmatically:

```cpp
#include <stacktrace>

void diagnose_error() {
    auto trace = std::stacktrace::current();
    std::println(stderr, "Stack trace:");
    for (const auto& entry : trace) {
        std::println(stderr, "  {} at {}:{}",
            entry.description(), entry.source_file(), entry.source_line());
    }
}

// Use in custom exception types
class DiagnosticException : public std::exception {
    std::string msg_;
    std::stacktrace trace_;
public:
    DiagnosticException(std::string msg)
        : msg_(std::move(msg)), trace_(std::stacktrace::current()) {}

    const char* what() const noexcept override { return msg_.c_str(); }
    const std::stacktrace& trace() const { return trace_; }
};
```

## Lambda Improvements

### Static Lambdas

```cpp
// static lambda — no captures allowed, can decay to function pointer
auto add = [](int a, int b) static { return a + b; };
int (*fptr)(int, int) = add;  // Guaranteed to work
```

### Attributes on Lambdas

```cpp
auto handler = []() [[nodiscard]] { return Status::ok(); };
auto compute = [](int n) [[gnu::pure]] { return n * n; };
```

## std::unreachable()

Indicate code paths that should never be reached, enabling compiler optimizations:

```cpp
enum class Direction { North, South, East, West };

int dx(Direction d) {
    switch (d) {
        case Direction::East:  return 1;
        case Direction::West:  return -1;
        case Direction::North:
        case Direction::South: return 0;
    }
    std::unreachable();  // Suppress "missing return" warning, enable optimization
}
```

## C++26 Preview

Features expected in C++26:

- **Static Reflection** (`^T`, `[:r:]`) — inspect type properties, members, and names at compile time
- **Contracts** (`pre`, `post`, `assert`) — preconditions, postconditions, and assertions as part of function declarations
- **Pattern Matching** (`inspect` expressions) — structured matching over variants, tuples, and types
- **std::execution** (Senders/Receivers) — standardized async execution framework
- **Trivial Relocatability** — optimize container operations for trivially relocatable types

These features are in various stages of standardization. Check compiler support before use.
