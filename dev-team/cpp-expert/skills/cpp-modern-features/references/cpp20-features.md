# C++20 Features — Comprehensive Reference

## Concepts

### Full Concept Syntax

```cpp
// Simple concept with a single requirement
template <typename T>
concept Hashable = requires(T a) {
    { std::hash<T>{}(a) } -> std::convertible_to<std::size_t>;
};

// Compound concept with multiple requirements
template <typename T>
concept Serializable = requires(T obj, std::ostream& os, std::istream& is) {
    { obj.serialize(os) } -> std::same_as<void>;
    { T::deserialize(is) } -> std::same_as<T>;
    requires std::is_move_constructible_v<T>;
};

// Concept with nested requirements
template <typename T>
concept Container = requires(T c) {
    typename T::value_type;
    typename T::iterator;
    { c.begin() } -> std::same_as<typename T::iterator>;
    { c.end() } -> std::same_as<typename T::iterator>;
    { c.size() } -> std::convertible_to<std::size_t>;
    requires std::input_iterator<typename T::iterator>;
};
```

### Standard Library Concepts

Key concepts from `<concepts>`:
- `std::same_as<T, U>` — types are identical
- `std::derived_from<Derived, Base>` — inheritance relationship
- `std::convertible_to<From, To>` — implicit conversion exists
- `std::integral`, `std::floating_point` — numeric types
- `std::constructible_from<T, Args...>` — construction from arguments
- `std::movable`, `std::copyable`, `std::semiregular`, `std::regular` — value type categories

Key concepts from `<iterator>`:
- `std::input_iterator`, `std::forward_iterator`, `std::bidirectional_iterator`, `std::random_access_iterator`, `std::contiguous_iterator`

Key concepts from `<ranges>`:
- `std::ranges::range`, `std::ranges::sized_range`, `std::ranges::view`, `std::ranges::input_range`, `std::ranges::contiguous_range`

### Concept Subsumption

When multiple constrained overloads match, the compiler selects the most constrained one:

```cpp
template <typename T>
concept Animal = requires(T a) { a.breathe(); };

template <typename T>
concept Pet = Animal<T> && requires(T p) { p.name(); };

void interact(Animal auto& a) { a.breathe(); }    // Less constrained
void interact(Pet auto& p) { p.name(); }           // More constrained — preferred for pets

// Pet subsumes Animal, so Pet overload is selected for types satisfying both
```

### Constraining Class Templates

```cpp
template <std::integral T>
class BitField {
    T bits_;
public:
    void set(int pos) { bits_ |= (T{1} << pos); }
    bool test(int pos) const { return bits_ & (T{1} << pos); }
};

// Partial specialization with concepts
template <typename T>
class Wrapper { /* general implementation */ };

template <std::floating_point T>
class Wrapper<T> { /* optimized for floating point */ };
```

## Ranges

### Range Adaptors (C++20)

| Adaptor | Description |
|---------|-------------|
| `views::filter(pred)` | Keep elements where predicate is true |
| `views::transform(fn)` | Apply function to each element |
| `views::take(n)` | First n elements |
| `views::drop(n)` | Skip first n elements |
| `views::reverse` | Reverse order |
| `views::join` | Flatten nested ranges |
| `views::split(delim)` | Split range by delimiter |
| `views::elements<N>` | Extract Nth element from tuple-like |
| `views::keys` / `views::values` | Extract keys/values from pair-like |
| `views::common` | Convert to common range (begin/end same type) |
| `views::counted(it, n)` | n elements from iterator |
| `views::iota(start)` | Infinite sequence from start |

### C++23 Range Additions

| Adaptor | Description |
|---------|-------------|
| `views::zip(r1, r2, ...)` | Zip multiple ranges into tuple |
| `views::zip_transform(fn, r1, r2)` | Zip + transform |
| `views::enumerate` | Pairs of (index, element) |
| `views::chunk(n)` | Split into chunks of n |
| `views::slide(n)` | Sliding window of n |
| `views::stride(n)` | Every nth element |
| `views::cartesian_product(r1, r2)` | Cross product |
| `views::adjacent<N>` | Adjacent N-tuples |
| `views::chunk_by(pred)` | Group by predicate |

### Creating Custom Views

```cpp
// Range algorithm using projections
std::vector<Person> people = { ... };

// Sort by age using projection
std::ranges::sort(people, {}, &Person::age);

// Find by name
auto it = std::ranges::find(people, "Alice", &Person::name);

// Partition: adults first
auto pivot = std::ranges::partition(people, [](int age) { return age >= 18; }, &Person::age);
```

### Range Concepts Hierarchy

```
input_range
├── forward_range
│   ├── bidirectional_range
│   │   ├── random_access_range
│   │   │   └── contiguous_range
│   │   └── ...
│   └── ...
└── ...
```

## Coroutines

### The Coroutine Machinery

A coroutine is any function containing `co_await`, `co_yield`, or `co_return`. The compiler transforms it into a state machine with a heap-allocated coroutine frame.

Key types:
- **promise_type** — controls coroutine behavior (initial/final suspend, yield/return values)
- **coroutine_handle<P>** — low-level handle to resume/destroy the coroutine
- **awaitable** — any type with `await_ready()`, `await_suspend()`, `await_resume()`

### Implementing a Generator (Pre-C++23)

```cpp
template <typename T>
class Generator {
public:
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) {
            current_value = std::move(value);
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    using handle_type = std::coroutine_handle<promise_type>;

    explicit Generator(handle_type h) : handle_(h) {}
    ~Generator() { if (handle_) handle_.destroy(); }

    Generator(Generator&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}
    Generator& operator=(Generator&& o) noexcept {
        if (handle_) handle_.destroy();
        handle_ = std::exchange(o.handle_, nullptr);
        return *this;
    }

    struct iterator {
        handle_type handle;
        bool operator!=(std::default_sentinel_t) const { return !handle.done(); }
        iterator& operator++() { handle.resume(); return *this; }
        T operator*() const { return handle.promise().current_value; }
    };

    iterator begin() { handle_.resume(); return {handle_}; }
    std::default_sentinel_t end() { return {}; }

private:
    handle_type handle_;
};
```

### Custom Awaitable Types

```cpp
struct SleepAwaitable {
    std::chrono::milliseconds duration;

    bool await_ready() const noexcept { return duration.count() <= 0; }

    void await_suspend(std::coroutine_handle<> h) const {
        // Schedule resumption after duration
        scheduler::post_delayed(h, duration);
    }

    void await_resume() const noexcept {}
};

// Usage in a coroutine
Task<void> delayed_operation() {
    co_await SleepAwaitable{std::chrono::milliseconds{100}};
    // ... continues after 100ms
}
```

## Three-Way Comparison (operator<=>)

### Ordering Categories

- `std::strong_ordering` — total order, substitutability (equal objects are indistinguishable)
- `std::weak_ordering` — total order, no substitutability (e.g., case-insensitive string)
- `std::partial_ordering` — not all pairs comparable (e.g., floating point with NaN)

### Defaulted Comparison

```cpp
struct Point {
    double x, y, z;
    auto operator<=>(const Point&) const = default;  // Memberwise comparison
    // Also generates ==, !=, <, >, <=, >=
};

// Custom three-way comparison
struct CaseInsensitiveString {
    std::string value;

    std::weak_ordering operator<=>(const CaseInsensitiveString& other) const {
        auto lower = [](std::string_view sv) {
            std::string s(sv);
            std::ranges::transform(s, s.begin(), ::tolower);
            return s;
        };
        return lower(value) <=> lower(other.value);
    }

    bool operator==(const CaseInsensitiveString& other) const {
        return (*this <=> other) == std::weak_ordering::equivalent;
    }
};
```

## Modules

### Module Interface Unit

```cpp
// math.cppm
export module math;

export namespace math {
    constexpr double pi = 3.14159265358979323846;

    constexpr double degrees_to_radians(double deg) {
        return deg * pi / 180.0;
    }

    class Vector3 {
        double x_, y_, z_;
    public:
        constexpr Vector3(double x, double y, double z) : x_(x), y_(y), z_(z) {}
        constexpr double length() const;
        constexpr Vector3 normalized() const;
    };
}

// Not exported — internal to the module
double internal_helper(double x) { return x * x; }
```

### Module Partitions

```cpp
// math-vector.cppm — module partition
export module math:vector;

export class Vector3 { /* ... */ };

// math-matrix.cppm — module partition
export module math:matrix;

export class Matrix4x4 { /* ... */ };

// math.cppm — primary module interface
export module math;
export import :vector;
export import :matrix;
```

### Implementation Units

```cpp
// math_impl.cpp — module implementation unit
module math;

// Implement exported functions, can use internal_helper
constexpr double math::Vector3::length() const {
    return std::sqrt(x_ * x_ + y_ * y_ + z_ * z_);
}
```

## std::format Customization

### Custom Formatter

```cpp
struct Color { uint8_t r, g, b; };

template <>
struct std::formatter<Color> {
    enum class Format { hex, rgb, named };
    Format fmt = Format::hex;

    constexpr auto parse(std::format_parse_context& ctx) {
        auto it = ctx.begin();
        if (it != ctx.end() && *it == 'r') { fmt = Format::rgb; ++it; }
        else if (it != ctx.end() && *it == 'h') { fmt = Format::hex; ++it; }
        return it;
    }

    auto format(const Color& c, std::format_context& ctx) const {
        switch (fmt) {
            case Format::hex: return std::format_to(ctx.out(), "#{:02x}{:02x}{:02x}", c.r, c.g, c.b);
            case Format::rgb: return std::format_to(ctx.out(), "rgb({}, {}, {})", c.r, c.g, c.b);
            default: return std::format_to(ctx.out(), "({},{},{})", c.r, c.g, c.b);
        }
    }
};

// Usage
Color red{255, 0, 0};
auto s1 = std::format("{:h}", red);  // "#ff0000"
auto s2 = std::format("{:r}", red);  // "rgb(255, 0, 0)"
```

## constexpr Improvements in C++20

C++20 expands what can be done at compile time:
- `constexpr virtual` functions
- `constexpr` dynamic allocation (`new`/`delete`) — memory must be freed within constant evaluation
- `constexpr` `std::vector` and `std::string`
- `constexpr try-catch` (catch blocks are unreachable at compile time)
- `constexpr` union active member changes

```cpp
constexpr auto make_squares(int n) {
    std::vector<int> v;
    v.reserve(n);
    for (int i = 0; i < n; ++i) v.push_back(i * i);
    return v;  // Vector must not escape compile-time context
}

// Use in a constant expression by converting to fixed-size result
constexpr auto squares = []() {
    auto v = make_squares(10);
    std::array<int, 10> arr{};
    std::ranges::copy(v, arr.begin());
    return arr;
}();

static_assert(squares[3] == 9);
```
