# Template Patterns — Deep Dive Reference

## SFINAE Deep Dive

SFINAE (Substitution Failure Is Not An Error) enables conditional template overloads. When template argument substitution fails, the overload is silently removed from the candidate set instead of causing a compilation error.

### enable_if

```cpp
// Pre-C++20: enable_if for conditional overloads
template <typename T>
std::enable_if_t<std::is_integral_v<T>, T>
safe_divide(T a, T b) {
    if (b == 0) throw std::domain_error("division by zero");
    return a / b;
}

template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
safe_divide(T a, T b) {
    return a / b;  // Floating point division by zero is well-defined (inf/nan)
}
```

### decltype-Based SFINAE

```cpp
// Check if type has a .serialize() method
template <typename T>
auto serialize_impl(const T& obj, int)
    -> decltype(obj.serialize(), std::string{}) {
    return obj.serialize();  // Has .serialize()
}

template <typename T>
auto serialize_impl(const T& obj, ...) -> std::string {
    return std::to_string(obj);  // Fallback
}

template <typename T>
std::string serialize(const T& obj) {
    return serialize_impl(obj, 0);  // int preferred over ...
}
```

### void_t and Detection Idiom

```cpp
// void_t: maps any type sequence to void
template <typename, typename = void>
struct has_serialize : std::false_type {};

template <typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<T>().serialize())>>
    : std::true_type {};

// is_detected pattern (Library Fundamentals TS)
template <typename T>
using serialize_t = decltype(std::declval<T>().serialize());

template <typename T>
constexpr bool has_serialize_v = std::experimental::is_detected_v<serialize_t, T>;
```

### Why Concepts Are Preferred

```cpp
// SFINAE: verbose, hard to read, poor error messages
template <typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
T add(T a, T b) { return a + b; }

// Concepts: clean, readable, excellent error messages
template <std::integral T>
T add(T a, T b) { return a + b; }

// Concepts with custom requirement
template <typename T>
concept HasSerialize = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

template <HasSerialize T>
std::string serialize(const T& obj) { return obj.serialize(); }
```

## Expression Templates

Expression templates defer computation and eliminate temporaries in arithmetic expressions. Used in math libraries (Eigen, Blaze).

```cpp
// Problem: naive operator overloading creates temporaries
Vector a, b, c, d;
Vector result = a + b + c + d;
// Without expression templates: 3 temporary vectors created

// Expression template approach
template <typename LHS, typename RHS>
struct VecAdd {
    const LHS& lhs;
    const RHS& rhs;

    auto operator[](size_t i) const { return lhs[i] + rhs[i]; }
    size_t size() const { return lhs.size(); }
};

template <typename LHS, typename RHS>
VecAdd<LHS, RHS> operator+(const LHS& a, const RHS& b) {
    return {a, b};
}

class Vector {
    std::vector<double> data_;
public:
    // Evaluate expression template in assignment
    template <typename Expr>
    Vector& operator=(const Expr& expr) {
        data_.resize(expr.size());
        for (size_t i = 0; i < data_.size(); ++i) {
            data_[i] = expr[i];  // Single pass, no temporaries
        }
        return *this;
    }

    double operator[](size_t i) const { return data_[i]; }
    size_t size() const { return data_.size(); }
};

// Now: result = a + b + c + d
// Builds VecAdd<VecAdd<VecAdd<Vector, Vector>, Vector>, Vector>
// Evaluates in a single pass in operator= — zero temporaries
```

## CRTP Advanced

### Base Class Counting

```cpp
// Count instances of derived class
template <typename Derived>
class InstanceCounter {
    static inline std::atomic<int> count_{0};
protected:
    InstanceCounter() { ++count_; }
    ~InstanceCounter() { --count_; }
    InstanceCounter(const InstanceCounter&) { ++count_; }
    InstanceCounter(InstanceCounter&&) noexcept { ++count_; }
public:
    static int instance_count() { return count_.load(); }
};

class Widget : public InstanceCounter<Widget> { /* ... */ };
class Gadget : public InstanceCounter<Gadget> { /* ... */ };

// Widget::instance_count() and Gadget::instance_count() are independent
```

### Static Interface Checking

```cpp
template <typename Derived>
class Renderer {
public:
    void render(const Scene& scene) {
        // Compile-time checked: Derived must implement these
        static_cast<Derived*>(this)->setup_pipeline();
        static_cast<Derived*>(this)->draw(scene);
        static_cast<Derived*>(this)->present();
    }
};

class OpenGLRenderer : public Renderer<OpenGLRenderer> {
    friend class Renderer<OpenGLRenderer>;
    void setup_pipeline() { /* OpenGL setup */ }
    void draw(const Scene& s) { /* OpenGL draw calls */ }
    void present() { /* Swap buffers */ }
};
// Missing any of setup_pipeline/draw/present causes a compile error
```

### CRTP with Variadic Mixins

```cpp
template <typename Derived, template <typename> class... Mixins>
class Base : public Mixins<Derived>... {
public:
    void init_all() { (Mixins<Derived>::init(), ...); }  // Fold expression
};

template <typename D> struct Logging {
    void init() { /* init logging */ }
    void log(std::string_view msg) { /* ... */ }
};

template <typename D> struct Metrics {
    void init() { /* init metrics */ }
    void record(std::string_view name, double value) { /* ... */ }
};

class App : public Base<App, Logging, Metrics> {
    // Inherits log() from Logging and record() from Metrics
};
```

## Concepts-Based Dispatch

### Constraining Template Overloads

Concepts provide clean overload dispatch through subsumption:

```cpp
template <typename T>
concept Printable = requires(std::ostream& os, const T& t) {
    { os << t } -> std::same_as<std::ostream&>;
};

template <typename T>
concept FormattablePrintable = Printable<T> && requires(const T& t) {
    { std::format("{}", t) } -> std::convertible_to<std::string>;
};

// Less constrained — fallback
void display(Printable auto const& obj) {
    std::cout << obj << '\n';
}

// More constrained — preferred when both match
void display(FormattablePrintable auto const& obj) {
    std::println("{}", obj);  // Uses std::format
}
```

### Concept Overload Sets

```cpp
template <std::integral T>
auto process(T val) { return val * 2; }           // For integers

template <std::floating_point T>
auto process(T val) { return std::round(val); }   // For floats

template <std::ranges::range R>
auto process(const R& r) {                        // For ranges
    return std::ranges::distance(r);
}
```

## Template Specialization Patterns

### Full Specialization

```cpp
template <typename T>
struct Serializer {
    static std::string serialize(const T& obj) {
        // Generic implementation
        return std::to_string(obj);
    }
};

// Full specialization for std::string
template <>
struct Serializer<std::string> {
    static std::string serialize(const std::string& obj) {
        return "\"" + obj + "\"";
    }
};
```

### Partial Specialization

```cpp
// Primary template
template <typename T>
struct Container { using type = std::vector<T>; };

// Partial specialization for pointer types
template <typename T>
struct Container<T*> { using type = std::vector<std::unique_ptr<T>>; };

// Partial specialization for pairs
template <typename K, typename V>
struct Container<std::pair<K, V>> { using type = std::map<K, V>; };
```

## Compile-Time Computation Patterns

### Pack Expansion

```cpp
// Sum all arguments
template <typename... Ts>
auto sum(Ts... args) {
    return (args + ...);  // Fold expression (C++17)
}

// Apply function to each argument
template <typename F, typename... Ts>
void for_each_arg(F&& f, Ts&&... args) {
    (f(std::forward<Ts>(args)), ...);
}

// Type list operations
template <typename... Ts>
struct TypeList {
    static constexpr size_t size = sizeof...(Ts);
};

// Check if type is in list
template <typename T, typename... Ts>
constexpr bool contains_v = (std::is_same_v<T, Ts> || ...);
```

### constexpr-if Chains

```cpp
template <typename T>
std::string to_debug_string(const T& val) {
    if constexpr (std::is_integral_v<T>) {
        return std::format("int({})", val);
    } else if constexpr (std::is_floating_point_v<T>) {
        return std::format("float({:.6f})", val);
    } else if constexpr (requires { val.to_string(); }) {
        return val.to_string();
    } else if constexpr (requires { std::to_string(val); }) {
        return std::to_string(val);
    } else {
        return "<unknown>";
    }
}
```

### Recursive Templates (Pre-constexpr)

```cpp
// Compile-time factorial (pre-C++14, now prefer constexpr)
template <int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template <>
struct Factorial<0> {
    static constexpr int value = 1;
};

// Modern equivalent
consteval int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}
```
