# C++ Idioms and Techniques — Deep Dive Reference

## Type Erasure Full Implementation

Based on Sean Parent's "Inheritance Is The Base Class of Evil" approach. Type erasure provides value-semantic polymorphism without inheritance.

### Runtime Concept Idiom

```cpp
class Drawable {
    // The external (concept) interface
    struct concept_t {
        virtual ~concept_t() = default;
        virtual void draw_(Canvas& c, int x, int y) const = 0;
        virtual std::unique_ptr<concept_t> clone_() const = 0;
        virtual std::string name_() const = 0;
    };

    // The model wraps any type that satisfies the concept
    template <typename T>
    struct model_t final : concept_t {
        T data_;

        explicit model_t(T obj) : data_(std::move(obj)) {}

        void draw_(Canvas& c, int x, int y) const override {
            data_.draw(c, x, y);  // Calls T::draw — duck typing
        }

        std::unique_ptr<concept_t> clone_() const override {
            return std::make_unique<model_t>(*this);
        }

        std::string name_() const override {
            return data_.name();
        }
    };

    std::unique_ptr<concept_t> self_;

public:
    // Accept any type with draw() and name() methods
    template <typename T>
    Drawable(T obj) : self_(std::make_unique<model_t<T>>(std::move(obj))) {}

    // Value semantics: copyable
    Drawable(const Drawable& other) : self_(other.self_->clone_()) {}
    Drawable(Drawable&&) noexcept = default;

    Drawable& operator=(Drawable other) {
        self_ = std::move(other.self_);
        return *this;
    }

    // Public interface delegates to concept
    void draw(Canvas& c, int x, int y) const { self_->draw_(c, x, y); }
    std::string name() const { return self_->name_(); }
};

// Usage — no inheritance needed in concrete types
struct Circle {
    double radius;
    void draw(Canvas& c, int x, int y) const { /* draw circle */ }
    std::string name() const { return "Circle"; }
};

struct Square {
    double side;
    void draw(Canvas& c, int x, int y) const { /* draw square */ }
    std::string name() const { return "Square"; }
};

// Polymorphic container with value semantics
std::vector<Drawable> shapes;
shapes.push_back(Circle{5.0});
shapes.push_back(Square{3.0});

for (const auto& shape : shapes) {
    shape.draw(canvas, 0, 0);  // Polymorphic dispatch
}

// Copies work correctly (deep copy)
auto shapes_copy = shapes;
```

### Custom vtable (Manual Type Erasure)

For performance-critical cases, avoid virtual dispatch overhead with a manual vtable:

```cpp
class Function {
    using invoke_fn = void(*)(void*);
    using destroy_fn = void(*)(void*);
    using clone_fn = void*(*)(const void*);

    struct vtable {
        invoke_fn invoke;
        destroy_fn destroy;
        clone_fn clone;
    };

    template <typename F>
    static constexpr vtable vtable_for = {
        [](void* p) { (*static_cast<F*>(p))(); },
        [](void* p) { delete static_cast<F*>(p); },
        [](const void* p) -> void* { return new F(*static_cast<const F*>(p)); }
    };

    const vtable* vptr_ = nullptr;
    void* data_ = nullptr;

public:
    template <typename F>
    Function(F f)
        : vptr_(&vtable_for<F>)
        , data_(new F(std::move(f))) {}

    ~Function() { if (vptr_) vptr_->destroy(data_); }

    void operator()() { vptr_->invoke(data_); }
};
```

## PIMPL Advanced

### Fast PIMPL (Stack Allocation)

Avoid heap allocation by reserving stack space for the implementation:

```cpp
class Widget {
    static constexpr size_t ImplSize = 128;
    static constexpr size_t ImplAlign = 8;

    alignas(ImplAlign) std::byte storage_[ImplSize];
    struct Impl;
    Impl* impl() { return reinterpret_cast<Impl*>(storage_); }
    const Impl* impl() const { return reinterpret_cast<const Impl*>(storage_); }

public:
    Widget();
    ~Widget();
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;

    void do_work();
};

// In .cpp — verify size at compile time
struct Widget::Impl {
    HeavyDependency dep;
    int state = 0;
};

Widget::Widget() {
    static_assert(sizeof(Impl) <= ImplSize, "Increase ImplSize");
    static_assert(alignof(Impl) <= ImplAlign, "Increase ImplAlign");
    new (storage_) Impl();
}

Widget::~Widget() { impl()->~Impl(); }
```

### PIMPL with Move Semantics

```cpp
class Connection {
    struct Impl;
    std::unique_ptr<Impl> pimpl_;
public:
    Connection(std::string host, int port);
    ~Connection();

    // Move operations — must be declared here, defined in .cpp
    Connection(Connection&& other) noexcept;
    Connection& operator=(Connection&& other) noexcept;

    // Delete copy operations explicitly
    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;

    void send(std::span<const std::byte> data);
    std::vector<std::byte> receive();
};
```

### PIMPL Testing Strategy

To test implementation details hidden by PIMPL:
- Test through the public interface (preferred)
- Use a test-only friend class
- Expose internal state through debug/test-only methods
- Use PIMPL only for binary-boundary types; internal types can use direct members

## Policy Composition

### Combining Multiple Policies

```cpp
template <typename StoragePolicy, typename ValidationPolicy, typename LogPolicy>
class ConfigManager : private StoragePolicy, private ValidationPolicy, private LogPolicy {
public:
    using StoragePolicy::load;
    using StoragePolicy::save;

    void set(const std::string& key, const std::string& value) {
        if (!ValidationPolicy::validate(key, value)) {
            LogPolicy::log("Validation failed for key: " + key);
            throw std::invalid_argument("Invalid config value");
        }
        StoragePolicy::set(key, value);
        LogPolicy::log("Set " + key + " = " + value);
    }
};

// Concrete policies
struct FileStorage {
    void load(const std::string& path);
    void save(const std::string& path);
    void set(const std::string& key, const std::string& value);
};

struct StrictValidation {
    static bool validate(const std::string& key, const std::string& value);
};

struct ConsoleLog {
    static void log(const std::string& msg) { std::println("{}", msg); }
};

// Compose
using AppConfig = ConfigManager<FileStorage, StrictValidation, ConsoleLog>;
```

### Policy Validation with Concepts

```cpp
template <typename T>
concept StoragePolicy = requires(T s, std::string key, std::string val) {
    { s.load(key) } -> std::same_as<void>;
    { s.save(key) } -> std::same_as<void>;
    { s.set(key, val) } -> std::same_as<void>;
};

template <StoragePolicy S, typename V, typename L>
class ConfigManager : private S, private V, private L { /* ... */ };
```

## Niebloid Pattern

Standard library range algorithms use niebloids — function objects that prevent ADL (Argument-Dependent Lookup) from finding unintended overloads:

```cpp
// How the standard library defines niebloids
namespace std::ranges {
    struct __sort_fn {
        template <std::random_access_iterator I, std::sentinel_for<I> S>
        I operator()(I first, S last) const {
            // Implementation
        }

        template <std::ranges::random_access_range R>
        auto operator()(R&& r) const {
            return (*this)(std::ranges::begin(r), std::ranges::end(r));
        }
    };

    inline constexpr __sort_fn sort{};  // Niebloid: function object, not function
}

// ADL cannot find unrelated sort() overloads — only std::ranges::sort is called
std::ranges::sort(my_container);
```

## Hidden Friend Idiom

Define operators and swap as friend functions inside the class body. This makes them findable only via ADL, reducing namespace pollution:

```cpp
class Quantity {
    double value_;
    std::string unit_;
public:
    Quantity(double v, std::string u) : value_(v), unit_(std::move(u)) {}

    // Hidden friends — found only via ADL
    friend bool operator==(const Quantity& a, const Quantity& b) {
        return a.value_ == b.value_ && a.unit_ == b.unit_;
    }

    friend auto operator<=>(const Quantity& a, const Quantity& b) {
        if (a.unit_ != b.unit_) throw std::logic_error("Incompatible units");
        return a.value_ <=> b.value_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Quantity& q) {
        return os << q.value_ << ' ' << q.unit_;
    }

    friend void swap(Quantity& a, Quantity& b) noexcept {
        using std::swap;
        swap(a.value_, b.value_);
        swap(a.unit_, b.unit_);
    }
};
```

## Copy-and-Swap Idiom

Provides exception-safe copy assignment using the copy constructor and swap:

```cpp
class Buffer {
    char* data_;
    size_t size_;
public:
    Buffer(size_t sz) : data_(new char[sz]), size_(sz) {}

    ~Buffer() { delete[] data_; }

    // Copy constructor
    Buffer(const Buffer& other) : data_(new char[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }

    // Move constructor
    Buffer(Buffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}

    // Unified assignment (copy-and-swap)
    // Takes by value — invokes copy or move constructor
    Buffer& operator=(Buffer other) noexcept {
        swap(*this, other);
        return *this;
        // other (the old data) is destroyed here
    }

    friend void swap(Buffer& a, Buffer& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }
};
```

Benefits: strong exception safety (if copy constructor throws, the original object is untouched), handles self-assignment correctly, single implementation for both copy and move assignment.

## Scope Guard

RAII wrapper for arbitrary cleanup actions:

```cpp
template <typename F>
class ScopeGuard {
    F cleanup_;
    bool active_ = true;
public:
    explicit ScopeGuard(F f) : cleanup_(std::move(f)) {}
    ~ScopeGuard() { if (active_) cleanup_(); }

    ScopeGuard(ScopeGuard&& o) noexcept
        : cleanup_(std::move(o.cleanup_)), active_(std::exchange(o.active_, false)) {}

    void dismiss() noexcept { active_ = false; }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
};

// Deduction guide
template <typename F>
ScopeGuard(F) -> ScopeGuard<F>;

// Usage
void transfer_files() {
    create_temp_directory();
    auto guard = ScopeGuard{[] { remove_temp_directory(); }};

    copy_files_to_temp();
    validate_files();

    // If everything succeeds, dismiss the guard
    move_temp_to_final();
    guard.dismiss();  // Don't clean up — operation succeeded
}
// If any exception is thrown before dismiss(), temp directory is cleaned up
```

## Meyer's Singleton

Thread-safe lazy initialization using a static local variable (guaranteed by C++11):

```cpp
class Database {
    Database() { /* expensive initialization */ }
public:
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;

    static Database& instance() {
        static Database db;  // Thread-safe initialization (C++11 guarantee)
        return db;
    }

    void query(std::string_view sql);
};

// Usage
Database::instance().query("SELECT * FROM users");
```

**Destruction order**: Static locals are destroyed in reverse order of construction. Be cautious of dependencies between singletons. Prefer dependency injection over singletons when feasible.

**Testing**: Singletons make testing difficult. Consider:
- Using an interface + dependency injection instead
- Providing a `reset()` method for tests
- Using a template parameter to inject a mock
