# Secure Coding Practices — Deep Dive Reference

## Input Validation Patterns

### Boundary Validation Architecture

Structure input validation as a series of layers:

1. **Size validation** — reject oversized input before any parsing
2. **Structure validation** — verify format, field counts, delimiter placement
3. **Type validation** — ensure fields parse to expected types
4. **Range validation** — check values fall within acceptable bounds
5. **Semantic validation** — verify relationships between fields make sense

```cpp
// Layered validation pipeline
class InputValidator {
public:
    std::expected<Request, Error> validate(std::span<const std::byte> raw) {
        // Layer 1: Size check
        if (raw.size() < MIN_REQUEST_SIZE || raw.size() > MAX_REQUEST_SIZE) {
            return std::unexpected(Error::InvalidSize);
        }

        // Layer 2: Structure — parse header
        auto header = parse_header(raw.subspan(0, HEADER_SIZE));
        if (!header) return std::unexpected(header.error());

        // Layer 3: Type — validate magic number and version
        if (header->magic != EXPECTED_MAGIC) {
            return std::unexpected(Error::InvalidMagic);
        }
        if (header->version < MIN_VERSION || header->version > MAX_VERSION) {
            return std::unexpected(Error::UnsupportedVersion);
        }

        // Layer 4: Range — validate field sizes against remaining data
        if (header->payload_size > raw.size() - HEADER_SIZE) {
            return std::unexpected(Error::PayloadSizeMismatch);
        }

        // Layer 5: Semantic — validate internal consistency
        auto payload = parse_payload(raw.subspan(HEADER_SIZE, header->payload_size));
        if (!payload) return std::unexpected(payload.error());

        return Request{*header, *payload};
    }
};
```

### Path Traversal Prevention

```cpp
#include <filesystem>

std::expected<std::filesystem::path, Error> safe_resolve(
    const std::filesystem::path& base_dir,
    std::string_view user_path)
{
    // Normalize and resolve
    auto resolved = std::filesystem::weakly_canonical(base_dir / user_path);

    // Verify the resolved path is within the base directory
    auto base_canonical = std::filesystem::weakly_canonical(base_dir);
    auto [base_end, resolved_begin] = std::mismatch(
        base_canonical.native().begin(), base_canonical.native().end(),
        resolved.native().begin(), resolved.native().end()
    );

    if (base_end != base_canonical.native().end()) {
        return std::unexpected(Error::PathTraversal);
    }

    return resolved;
}
```

### Command Injection Prevention

Never construct shell commands from user input:

```cpp
// BAD: command injection via string concatenation
void bad_ping(const std::string& host) {
    std::string cmd = "ping -c 1 " + host;
    std::system(cmd.c_str());  // host = "8.8.8.8; rm -rf /" → disaster
}

// GOOD: use exec-family directly, bypassing shell
void safe_ping(const std::string& host) {
    // Validate host is an IP address or hostname
    if (!is_valid_hostname(host)) {
        throw std::invalid_argument("Invalid hostname");
    }

    pid_t pid = fork();
    if (pid == 0) {
        // Child: exec directly, no shell interpretation
        execlp("ping", "ping", "-c", "1", host.c_str(), nullptr);
        _exit(127);
    }
    int status;
    waitpid(pid, &status, 0);
}

// GOOD: use posix_spawn for modern systems
#include <spawn.h>

void safe_ping_spawn(const std::string& host) {
    if (!is_valid_hostname(host)) throw std::invalid_argument("Invalid hostname");

    char* argv[] = {
        const_cast<char*>("ping"),
        const_cast<char*>("-c"), const_cast<char*>("1"),
        const_cast<char*>(host.c_str()),
        nullptr
    };
    pid_t pid;
    posix_spawn(&pid, "/usr/bin/ping", nullptr, nullptr, argv, environ);
    waitpid(pid, nullptr, 0);
}
```

### SQL Injection (Embedded Databases)

For C++ applications using SQLite or other embedded databases:

```cpp
// BAD: string concatenation → SQL injection
void bad_query(sqlite3* db, const std::string& name) {
    std::string sql = "SELECT * FROM users WHERE name = '" + name + "'";
    sqlite3_exec(db, sql.c_str(), callback, nullptr, nullptr);
}

// GOOD: parameterized query
void safe_query(sqlite3* db, const std::string& name) {
    sqlite3_stmt* stmt;
    sqlite3_prepare_v2(db, "SELECT * FROM users WHERE name = ?", -1, &stmt, nullptr);
    sqlite3_bind_text(stmt, 1, name.c_str(), -1, SQLITE_TRANSIENT);
    while (sqlite3_step(stmt) == SQLITE_ROW) {
        // Process row
    }
    sqlite3_finalize(stmt);
}
```

## Safe Integer Arithmetic

### CWE-190: Integer Overflow or Wraparound

Integer overflow in size calculations is a top C/C++ vulnerability class. It leads to undersized buffer allocations, which are then overflowed.

```cpp
// The classic vulnerability pattern:
// 1. Attacker controls count and size
// 2. count * size overflows to a small number
// 3. Small buffer allocated
// 4. Large amount of data written → heap buffer overflow
void vulnerable(uint32_t count, uint32_t element_size) {
    size_t total = count * element_size;  // Can overflow!
    char* buf = new char[total];          // Undersized allocation
    for (uint32_t i = 0; i < count; ++i) {
        read_element(buf + i * element_size, element_size);  // Buffer overflow!
    }
}
```

### Safe Arithmetic Utilities

```cpp
#include <limits>
#include <optional>

template <std::unsigned_integral T>
constexpr std::optional<T> checked_add(T a, T b) {
    if (a > std::numeric_limits<T>::max() - b) return std::nullopt;
    return a + b;
}

template <std::unsigned_integral T>
constexpr std::optional<T> checked_mul(T a, T b) {
    if (a == 0 || b == 0) return T{0};
    if (a > std::numeric_limits<T>::max() / b) return std::nullopt;
    return a * b;
}

template <std::unsigned_integral T>
constexpr std::optional<T> checked_sub(T a, T b) {
    if (a < b) return std::nullopt;
    return a - b;
}

// Usage in security-critical allocation
std::expected<Buffer, Error> safe_allocate(size_t count, size_t elem_size) {
    auto total = checked_mul(count, elem_size);
    if (!total || *total > MAX_ALLOC_SIZE) {
        return std::unexpected(Error::IntegerOverflow);
    }
    return Buffer(*total);
}
```

### Compiler Builtins

```cpp
// GCC/Clang builtins — single instruction on most architectures
size_t total;
if (__builtin_mul_overflow(count, elem_size, &total)) {
    return Error::Overflow;
}

// MSVC: use SafeInt library or manual checks
#include <safeint.h>
try {
    msl::utilities::SafeInt<size_t> safe_total = SafeInt<size_t>(count) * elem_size;
} catch (const msl::utilities::SafeIntException&) {
    return Error::Overflow;
}
```

### Signed vs Unsigned in Security Contexts

```cpp
// BAD: signed/unsigned mismatch in size comparison
void process(const char* data, int length) {
    if (length > MAX_LENGTH) return;  // Negative length bypasses this check!
    // length implicitly converted to size_t (unsigned) → huge value
    memcpy(buf, data, length);  // Buffer overflow with negative length
}

// GOOD: use unsigned types for sizes, validate at boundaries
void process(const char* data, size_t length) {
    if (length > MAX_LENGTH) return;
    std::memcpy(buf, data, length);
}

// GOOD: if you receive signed from external API, validate non-negative first
void process_from_api(const char* data, int32_t length) {
    if (length < 0 || static_cast<size_t>(length) > MAX_LENGTH) return;
    std::memcpy(buf, data, static_cast<size_t>(length));
}
```

## Format String Prevention

### Vulnerability Classes

Format string vulnerabilities allow:
- **Information disclosure**: Reading stack memory via `%x`, `%p`
- **Denial of service**: Crashing via `%n` or invalid format specifiers
- **Code execution**: Writing to arbitrary memory via `%n`

### Prevention Rules

1. **Never pass user data as the format string** — always use a literal format string
2. Enable `-Wformat=2 -Wformat-security` compiler warnings
3. Use `std::format` / `std::println` instead of `printf` family when possible
4. For logging libraries, ensure user data goes in arguments, not the format string

```cpp
// Dangerous patterns to audit for:
printf(user_string);           // Direct user format string
syslog(LOG_INFO, user_string); // Same issue with syslog
fprintf(fp, user_string);     // Same issue with fprintf

// Safe patterns:
printf("%s", user_string);
syslog(LOG_INFO, "%s", user_string);
std::println("{}", user_string);
spdlog::info("{}", user_string);
```

## Deserialization Safety

### Parsing Untrusted Binary Data

```cpp
// Hardened binary parser with resource limits
class SecureParser {
    std::span<const std::byte> data_;
    size_t pos_ = 0;
    int depth_ = 0;
    static constexpr int MAX_DEPTH = 32;
    static constexpr size_t MAX_STRING_LEN = 1'000'000;
    static constexpr size_t MAX_ARRAY_LEN = 100'000;

public:
    explicit SecureParser(std::span<const std::byte> data) : data_(data) {}

    std::expected<Value, Error> parse_value() {
        if (++depth_ > MAX_DEPTH) {
            return std::unexpected(Error::NestingTooDeep);
        }
        auto guard = ScopeGuard([this] { --depth_; });

        auto type_byte = read_byte();
        if (!type_byte) return std::unexpected(type_byte.error());

        switch (*type_byte) {
            case 0x01: return parse_integer();
            case 0x02: return parse_string();
            case 0x03: return parse_array();
            case 0x04: return parse_object();
            default: return std::unexpected(Error::UnknownType);
        }
    }

private:
    std::expected<std::string, Error> parse_string() {
        auto len = read_u32();
        if (!len) return std::unexpected(len.error());
        if (*len > MAX_STRING_LEN) return std::unexpected(Error::StringTooLong);
        if (pos_ + *len > data_.size()) return std::unexpected(Error::UnexpectedEnd);

        std::string result(reinterpret_cast<const char*>(data_.data() + pos_), *len);
        pos_ += *len;

        if (!is_valid_utf8(result)) return std::unexpected(Error::InvalidUtf8);
        return result;
    }

    std::expected<std::vector<Value>, Error> parse_array() {
        auto count = read_u32();
        if (!count) return std::unexpected(count.error());
        if (*count > MAX_ARRAY_LEN) return std::unexpected(Error::ArrayTooLong);

        std::vector<Value> items;
        items.reserve(std::min(*count, size_t{1024}));  // Cap initial reservation
        for (uint32_t i = 0; i < *count; ++i) {
            auto item = parse_value();
            if (!item) return std::unexpected(item.error());
            items.push_back(std::move(*item));
        }
        return items;
    }
};
```

### Parsing Untrusted Text (JSON, XML, etc.)

- Use well-tested libraries (nlohmann/json, RapidJSON, pugixml) instead of hand-rolled parsers
- Set parsing limits: maximum depth, maximum string length, maximum total size
- Validate the parsed structure against an expected schema
- Fuzz the parser with malformed input

```cpp
// nlohmann/json with limits
#include <nlohmann/json.hpp>

std::expected<Config, Error> parse_json_config(std::string_view input) {
    if (input.size() > MAX_JSON_SIZE) {
        return std::unexpected(Error::InputTooLarge);
    }

    try {
        auto j = nlohmann::json::parse(input,
            nullptr,    // callback
            true,       // allow exceptions
            true);      // ignore comments

        // Validate structure
        if (!j.is_object()) return std::unexpected(Error::InvalidStructure);
        if (!j.contains("version") || !j["version"].is_number_integer()) {
            return std::unexpected(Error::MissingField);
        }
        // ... validate all expected fields
        return Config::from_json(j);
    } catch (const nlohmann::json::parse_error& e) {
        return std::unexpected(Error::ParseError);
    }
}
```

## CWE Mapping for C++

Common Weakness Enumeration entries most relevant to C++:

| CWE | Name | C++ Mitigation |
|-----|------|----------------|
| CWE-119 | Buffer overflow | `std::span`, bounds checking, ASan |
| CWE-120 | Classic buffer overflow | `std::string`, `std::vector`, no raw arrays |
| CWE-125 | Out-of-bounds read | Range-checked access, `std::span` |
| CWE-134 | Format string vulnerability | `std::format`, `-Wformat-security` |
| CWE-190 | Integer overflow | Checked arithmetic, compiler builtins |
| CWE-191 | Integer underflow | Unsigned types for sizes, checked subtraction |
| CWE-416 | Use-after-free | Smart pointers, RAII, ASan |
| CWE-415 | Double free | `std::unique_ptr`, rule of zero |
| CWE-476 | Null pointer dereference | References, `std::optional`, `std::expected` |
| CWE-787 | Out-of-bounds write | `std::vector`, bounds checking, ASan |
| CWE-362 | Race condition | `std::mutex`, `std::atomic`, TSan |
| CWE-78 | OS command injection | `exec` family, never `system()` |
| CWE-89 | SQL injection | Parameterized queries |
| CWE-22 | Path traversal | `std::filesystem::canonical`, path prefix validation |
| CWE-327 | Broken crypto | Use established libraries (libsodium, Botan) |
| CWE-338 | Weak PRNG | `std::random_device` / libsodium, not `rand()` |

## Secure API Design

### Principle of Least Surprise

Design APIs where the obvious usage is the safe usage:

```cpp
// BAD: easy to misuse — caller must remember to check
char* get_buffer();          // Who owns this? When to free? What size?
int parse(const char* data); // Return code for error? What if null?

// GOOD: safe by default
std::unique_ptr<Buffer> get_buffer();              // Ownership is clear
std::expected<Result, Error> parse(std::span<const std::byte> data);  // Clear error handling
```

### Secure Defaults

```cpp
// TLS configuration with secure defaults
struct TlsConfig {
    std::string min_version = "TLSv1.3";  // Secure default
    bool verify_certificates = true;       // Secure default
    std::string cipher_suites = "";        // Empty = library defaults (secure)

    // Deliberately no "disable_verification" option
    // Force callers to use a separate, auditable code path for testing
};
```

### Make Dangerous Operations Explicit

```cpp
// Require explicit opt-in for dangerous operations
class DatabaseConnection {
public:
    // Safe: parameterized query
    Result query(std::string_view sql, std::span<const Parameter> params);

    // Dangerous: raw SQL — requires explicit tag type to call
    struct UnsafeRawSqlTag {};
    Result raw_query(UnsafeRawSqlTag, std::string_view raw_sql);
};

// Caller must explicitly acknowledge the risk
db.raw_query(DatabaseConnection::UnsafeRawSqlTag{}, sql);
```
