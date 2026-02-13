---
name: C++ Design Patterns
description: This skill should be used when the user asks about "CRTP", "type erasure", "PIMPL", "policy-based design", "C++ strong types", "tag dispatch", "SFINAE", "C++ visitor pattern", "expression templates", "compile-time polymorphism", "C++ factory pattern", "C++ singleton", or "C++ pattern". It covers C++-specific design patterns and idioms including template-based patterns, type erasure, compile-time polymorphism, and modern alternatives to classic GoF patterns.
version: 0.1.0
---

# C++ Design Patterns

C++ provides unique mechanisms (templates, RAII, value semantics) that enable patterns impossible in other languages. This skill covers C++-specific idioms and modern alternatives to classic GoF patterns.

## Pattern Selection Guide

Choose between compile-time and runtime polymorphism based on whether the set of types is known at compile time.

| GoF Pattern | C++ Idiomatic Alternative |
|-------------|--------------------------|
| Strategy | `template` parameter, concepts, `std::function` |
| Observer | Signals/slots, `std::function` callbacks |
| Singleton | Meyer's singleton (`static` local) |
| Factory | `std::variant` + `std::visit`, `std::unique_ptr` factory |
| Adapter | Template wrapper, concepts |
| Decorator | CRTP mixin chain |
| Visitor | `std::variant` + `std::visit` + overloaded lambda |
| Command | `std::function`, `std::packaged_task` |
| Iterator | Ranges, sentinel-based iteration |

Prefer compile-time polymorphism (templates/concepts) when the type set is closed and known. Use runtime polymorphism (virtual functions, type erasure) when types are open or loaded dynamically.

## CRTP (Curiously Recurring Template Pattern)

CRTP enables static polymorphism by having a derived class pass itself as a template argument to the base.

```cpp
// CRTP for static interface enforcement
template <typename Derived>
class Comparable {
public:
    bool operator>(const Derived& other) const {
        return other < static_cast<const Derived&>(*this);
    }
    bool operator<=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) > other);
    }
    bool operator>=(const Derived& other) const {
        return !(static_cast<const Derived&>(*this) < other);
    }
};

struct Temperature : Comparable<Temperature> {
    double value;
    bool operator<(const Temperature& other) const { return value < other.value; }
};
// Temperature automatically gets >, <=, >= operators via CRTP

// CRTP mixin chaining
template <typename Derived>
struct Printable {
    void print() const { std::println("{}", static_cast<const Derived&>(*this).to_string()); }
};

template <typename Derived>
struct Serializable {
    std::string serialize() const { return static_cast<const Derived&>(*this).to_json(); }
};

struct Config : Printable<Config>, Serializable<Config> {
    std::string to_string() const;
    std::string to_json() const;
};
```

**C++23 alternative**: Deducing this can replace simple CRTP use cases without the template boilerplate.

## Type Erasure

Type erasure hides concrete types behind a uniform interface without requiring inheritance. It combines three components: a concept (interface), a model (type-specific wrapper), and an outer wrapper.

`std::function`, `std::any`, and `std::move_only_function` are standard type-erased types.

```cpp
// Custom type-erased Drawable
class Drawable {
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw(Canvas&) const = 0;
        virtual std::unique_ptr<Concept> clone() const = 0;
    };

    template <typename T>
    struct Model final : Concept {
        T obj_;
        explicit Model(T obj) : obj_(std::move(obj)) {}
        void draw(Canvas& c) const override { obj_.draw(c); }
        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(*this);
        }
    };

    std::unique_ptr<Concept> pimpl_;
public:
    template <typename T>
    Drawable(T obj) : pimpl_(std::make_unique<Model<T>>(std::move(obj))) {}

    Drawable(const Drawable& o) : pimpl_(o.pimpl_->clone()) {}
    Drawable(Drawable&&) noexcept = default;
    Drawable& operator=(Drawable o) { std::swap(pimpl_, o.pimpl_); return *this; }

    void draw(Canvas& c) const { pimpl_->draw(c); }
};

// Any type with a draw(Canvas&) method works — no inheritance required
Circle c{5.0};
Rectangle r{3.0, 4.0};
std::vector<Drawable> shapes = {c, r};  // Type-erased collection
```

## PIMPL (Pointer to Implementation)

PIMPL hides implementation details behind a forward-declared pointer, providing ABI stability and faster compilation (compilation firewall).

```cpp
// widget.h — public header
#include <memory>

class Widget {
public:
    Widget();
    ~Widget();                    // Must be declared, defined in .cpp
    Widget(Widget&&) noexcept;    // Must be declared, defined in .cpp
    Widget& operator=(Widget&&) noexcept;

    void do_work();
    int get_result() const;

private:
    struct Impl;                  // Forward declaration only
    std::unique_ptr<Impl> pimpl_;
};

// widget.cpp — implementation hidden from consumers
#include "widget.h"
#include <heavy_dependency.h>     // Not exposed in header

struct Widget::Impl {
    HeavyDependency dep;
    int cached_result = 0;
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;     // Defined where Impl is complete
Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;

void Widget::do_work() { pimpl_->cached_result = pimpl_->dep.compute(); }
int Widget::get_result() const { return pimpl_->cached_result; }
```

## Policy-Based Design

Use template parameters to inject behavior (policies) at compile time. Each policy is a small class providing a specific capability.

```cpp
// Policy-based logger
struct ConsoleOutput {
    static void write(std::string_view msg) { std::println("{}", msg); }
};

struct FileOutput {
    static void write(std::string_view msg) {
        static std::ofstream file("log.txt", std::ios::app);
        file << msg << '\n';
    }
};

struct TimestampFormat {
    static std::string format(std::string_view msg) {
        auto now = std::chrono::system_clock::now();
        return std::format("[{}] {}", now, msg);
    }
};

struct PlainFormat {
    static std::string format(std::string_view msg) { return std::string(msg); }
};

template <typename OutputPolicy, typename FormatPolicy>
class Logger {
public:
    void log(std::string_view msg) {
        OutputPolicy::write(FormatPolicy::format(msg));
    }
};

// Compose policies at compile time
using AppLogger = Logger<ConsoleOutput, TimestampFormat>;
using TestLogger = Logger<FileOutput, PlainFormat>;
```

## Strong Types

Wrap primitive types to prevent accidental mixing of semantically different values.

```cpp
template <typename Tag, typename T = double>
class StrongType {
    T value_;
public:
    constexpr explicit StrongType(T val) : value_(val) {}
    constexpr T value() const { return value_; }

    constexpr auto operator<=>(const StrongType&) const = default;
};

// Define distinct types for different units
using Meters  = StrongType<struct MetersTag>;
using Seconds = StrongType<struct SecondsTag>;
using Mps     = StrongType<struct MpsTag>;  // Meters per second

// Type-safe computation — mixing Meters and Seconds won't compile
constexpr Mps speed(Meters distance, Seconds time) {
    return Mps{distance.value() / time.value()};
}

auto v = speed(Meters{100.0}, Seconds{9.58});  // OK
// auto bad = speed(Seconds{1.0}, Meters{2.0}); // Compile error!
```

## Tag Dispatch and Overload Resolution

Select algorithm implementations based on type properties using tag types or `if constexpr`.

```cpp
// Tag dispatch by iterator category
template <typename Iter>
void advance_impl(Iter& it, int n, std::random_access_iterator_tag) {
    it += n;  // O(1)
}

template <typename Iter>
void advance_impl(Iter& it, int n, std::input_iterator_tag) {
    while (n-- > 0) ++it;  // O(n)
}

template <typename Iter>
void my_advance(Iter& it, int n) {
    advance_impl(it, n, typename std::iterator_traits<Iter>::iterator_category{});
}

// Modern alternative with concepts (C++20)
template <std::random_access_iterator Iter>
void my_advance(Iter& it, int n) { it += n; }

template <std::input_iterator Iter>
void my_advance(Iter& it, int n) { while (n-- > 0) ++it; }
```

## Visitor Pattern with std::variant

Replace the classic visitor pattern with `std::variant` + `std::visit` for closed type hierarchies.

```cpp
// Overloaded helper for lambda visitors
template <typename... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// AST node types
struct Literal { double value; };
struct BinOp { char op; std::unique_ptr<struct Expr> lhs, rhs; };
struct UnaryOp { char op; std::unique_ptr<struct Expr> operand; };

using Expr = std::variant<Literal, BinOp, UnaryOp>;

// Evaluate using std::visit with overloaded lambdas
double evaluate(const Expr& expr) {
    return std::visit(overloaded{
        [](const Literal& lit) -> double {
            return lit.value;
        },
        [](const BinOp& bin) -> double {
            double l = evaluate(*bin.lhs);
            double r = evaluate(*bin.rhs);
            switch (bin.op) {
                case '+': return l + r;
                case '*': return l * r;
                default:  throw std::invalid_argument("Unknown operator");
            }
        },
        [](const UnaryOp& un) -> double {
            double val = evaluate(*un.operand);
            return un.op == '-' ? -val : val;
        }
    }, expr);
}
```

Advantages over classic visitor: no double dispatch, no base class needed, exhaustive matching enforced by the compiler.

## Additional Resources

### Reference Files

For advanced pattern implementations, consult:

- **`references/template-patterns.md`** — SFINAE deep dive, expression templates, CRTP advanced patterns, concepts-based dispatch, template specialization, compile-time computation patterns
- **`references/idioms-and-techniques.md`** — Full type erasure implementation (Sean Parent approach), PIMPL advanced patterns, policy composition, niebloid pattern, hidden friend idiom, copy-and-swap, scope guard, Meyer's singleton
