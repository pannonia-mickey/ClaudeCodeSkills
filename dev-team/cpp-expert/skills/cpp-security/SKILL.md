---
name: C++ Security
description: This skill should be used when the user asks about "C++ security", "input validation", "buffer overflow prevention", "secure coding", "C++ hardening", "stack protector", "ASLR", "PIE", "RELRO", "CFI", "FORTIFY_SOURCE", "secure memory", "wiping secrets", "constant-time comparison", "C++ cryptography", "OpenSSL", "libsodium", "Botan", "secure random", "C++ fuzzing", "libFuzzer", "AFL", "format string vulnerability", "safe integer arithmetic", "deserialize untrusted", "supply chain security", "privilege separation", "sandbox", or "CWE". It covers secure coding practices, compiler hardening, cryptographic library usage, fuzzing, and defense-in-depth strategies for C++ applications.
version: 0.1.0
---

# C++ Security

C++ applications face unique security challenges due to manual memory management and low-level system access. This skill covers secure coding practices, hardening, cryptography, fuzzing, and defense-in-depth strategies that go beyond memory safety into adversarial threat defense.

## Input Validation

Treat all external input as untrusted: network data, file contents, environment variables, command-line arguments, IPC messages, and user input. Validate at system boundaries before any processing.

```cpp
// Validate and constrain input at the boundary
std::expected<Config, Error> parse_config(std::span<const std::byte> raw_input) {
    if (raw_input.size() > MAX_CONFIG_SIZE) {
        return std::unexpected(Error::InputTooLarge);
    }

    // Parse with explicit limits — never trust size fields in untrusted data
    BoundedReader reader(raw_input);
    auto name_len = reader.read_u32();
    if (name_len > MAX_NAME_LENGTH) {
        return std::unexpected(Error::FieldTooLarge);
    }
    auto name = reader.read_string(name_len);
    if (!is_valid_utf8(name)) {
        return std::unexpected(Error::InvalidEncoding);
    }
    // ...
}
```

Key principles:
- **Allowlist over denylist** — validate that input matches expected patterns, do not try to filter out bad patterns
- **Validate lengths before allocation** — prevent denial-of-service via excessive allocation
- **Check arithmetic before using results** — prevent integer overflow from creating undersized buffers
- **Reject invalid data early** — fail fast at the boundary, not deep inside business logic

## Secure Memory Handling

### Wiping Secrets from Memory

Compilers optimize away `memset` on memory about to be freed. Use platform-specific or standardized secure wiping:

```cpp
#include <cstring>

// C11/C23: memset_explicit (guaranteed not optimized away)
void wipe(void* ptr, size_t len) {
#if __cplusplus >= 202302L || defined(__STDC_LIB_EXT1__)
    memset_explicit(ptr, 0, len);
#elif defined(_WIN32)
    SecureZeroMemory(ptr, len);
#else
    // Volatile pointer prevents optimization
    volatile unsigned char* p = static_cast<volatile unsigned char*>(ptr);
    while (len--) *p++ = 0;
#endif
}

// RAII wrapper for secret data
template <typename T>
class Secret {
    T value_;
public:
    explicit Secret(T val) : value_(std::move(val)) {}
    ~Secret() { wipe(&value_, sizeof(value_)); }

    const T& get() const { return value_; }

    // Non-copyable to prevent accidental secret duplication
    Secret(const Secret&) = delete;
    Secret& operator=(const Secret&) = delete;
    Secret(Secret&& o) noexcept : value_(std::move(o.value_)) { wipe(&o.value_, sizeof(o.value_)); }
};
```

### Constant-Time Comparison

Prevent timing side-channel attacks when comparing secrets (passwords, tokens, MACs):

```cpp
// Constant-time comparison — no early exit on mismatch
bool constant_time_equal(std::span<const std::byte> a, std::span<const std::byte> b) {
    if (a.size() != b.size()) return false;
    volatile std::byte diff{0};
    for (size_t i = 0; i < a.size(); ++i) {
        diff |= a[i] ^ b[i];
    }
    return std::to_integer<int>(diff) == 0;
}
```

### Locking Memory Pages

Prevent secrets from being swapped to disk:

```cpp
#include <sys/mman.h>

// Lock memory to prevent paging to swap
void* alloc_locked(size_t size) {
    void* ptr = std::malloc(size);
    if (ptr && mlock(ptr, size) != 0) {
        std::free(ptr);
        throw std::runtime_error("Failed to lock memory");
    }
    return ptr;
}

void free_locked(void* ptr, size_t size) {
    wipe(ptr, size);
    munlock(ptr, size);
    std::free(ptr);
}
```

## Compiler and Linker Hardening

Enable defense-in-depth through compiler and linker flags. These make exploitation significantly harder even when vulnerabilities exist.

```cmake
# Comprehensive hardening flags
target_compile_options(myapp PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:
        -fstack-protector-strong     # Stack canaries on vulnerable functions
        -fstack-clash-protection     # Prevent stack clash attacks
        -fcf-protection=full         # Control-flow integrity (Intel CET)
        -D_FORTIFY_SOURCE=3          # Checked libc functions (buffer overflow detection)
        -fPIE                        # Position-independent executable (for ASLR)
        -Wformat=2 -Wformat-security # Format string warnings
        -Wimplicit-fallthrough       # Prevent switch fallthrough bugs
    >
    $<$<CXX_COMPILER_ID:MSVC>:
        /GS      # Buffer security check (stack canaries)
        /guard:cf # Control flow guard
        /DYNAMICBASE /HIGHENTROPYVA  # ASLR
        /sdl     # Security development lifecycle checks
    >
)

target_link_options(myapp PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:
        -pie                         # Position-independent executable
        -Wl,-z,relro,-z,now          # Full RELRO (GOT protection)
        -Wl,-z,noexecstack           # Non-executable stack
    >
)
```

| Flag | Protection |
|------|-----------|
| `-fstack-protector-strong` | Detects stack buffer overflows via canary |
| `-D_FORTIFY_SOURCE=3` | Runtime bounds checking for libc functions |
| `-fPIE -pie` | Enables ASLR for the executable |
| `-Wl,-z,relro,-z,now` | Makes GOT read-only after relocation (prevents GOT overwrite) |
| `-fcf-protection=full` | Hardware-enforced control-flow integrity (CET) |
| `-fstack-clash-protection` | Prevents stack-clash attacks via guard pages |

## Safe Integer Arithmetic

Integer overflow in security-critical code (size calculations, buffer allocation) leads to exploitable vulnerabilities:

```cpp
// Safe multiplication for allocation sizes
std::optional<size_t> safe_multiply(size_t a, size_t b) {
    if (a == 0 || b == 0) return 0;
    if (a > SIZE_MAX / b) return std::nullopt;  // Would overflow
    return a * b;
}

// Safe buffer allocation — prevent integer overflow → undersized buffer
std::expected<std::vector<std::byte>, Error> allocate_records(size_t count, size_t record_size) {
    auto total = safe_multiply(count, record_size);
    if (!total) return std::unexpected(Error::IntegerOverflow);
    if (*total > MAX_ALLOCATION) return std::unexpected(Error::AllocationTooLarge);
    return std::vector<std::byte>(*total);
}

// Compiler builtins for overflow-checked arithmetic
size_t result;
if (__builtin_mul_overflow(count, record_size, &result)) {
    return Error::IntegerOverflow;
}
```

## Format String Safety

Never pass user-controlled strings as format specifiers:

```cpp
// BAD: user-controlled format string
std::string user_input = get_user_input();
std::printf(user_input.c_str());            // Format string vulnerability!
spdlog::info(user_input);                   // Same issue with some loggers

// GOOD: user input as data argument, never as format
std::printf("%s", user_input.c_str());      // Safe: %s format, user data as arg
spdlog::info("{}", user_input);             // Safe: {} placeholder, user data as arg
std::println("{}", user_input);             // Safe: std::format-based
```

Enable `-Wformat=2 -Wformat-security` to catch format string issues at compile time.

## Parsing Untrusted Data

When deserializing network protocols, file formats, or IPC messages:

```cpp
// Bounded reader — prevents buffer overreads
class BoundedReader {
    std::span<const std::byte> data_;
    size_t pos_ = 0;
public:
    explicit BoundedReader(std::span<const std::byte> data) : data_(data) {}

    std::expected<uint32_t, Error> read_u32() {
        if (pos_ + 4 > data_.size()) return std::unexpected(Error::UnexpectedEnd);
        uint32_t val;
        std::memcpy(&val, data_.data() + pos_, 4);
        pos_ += 4;
        return val;  // Note: handle endianness as needed
    }

    std::expected<std::string_view, Error> read_string(size_t len) {
        if (len > MAX_STRING_LENGTH) return std::unexpected(Error::FieldTooLarge);
        if (pos_ + len > data_.size()) return std::unexpected(Error::UnexpectedEnd);
        auto sv = std::string_view(reinterpret_cast<const char*>(data_.data() + pos_), len);
        pos_ += len;
        return sv;
    }

    size_t remaining() const { return data_.size() - pos_; }
};
```

Key rules for parsing untrusted data:
- Never trust length fields — validate against remaining buffer and maximum limits
- Use `std::memcpy` for type punning, never `reinterpret_cast` dereference (alignment, aliasing)
- Reject invalid or inconsistent data early
- Set recursion depth limits for nested structures
- Set total processing time limits to prevent algorithmic DoS

## Fuzzing

Use fuzzing to discover vulnerabilities in parsers, protocol handlers, and any code processing untrusted input.

### libFuzzer

```cpp
// fuzz_target.cpp — libFuzzer entry point
#include <cstdint>
#include <cstddef>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    auto result = parse_message({reinterpret_cast<const std::byte*>(data), size});
    // No assertions needed — libFuzzer detects crashes, leaks, UB via sanitizers
    return 0;
}
```

```cmake
# CMake integration for fuzzing
if(ENABLE_FUZZING)
    add_executable(fuzz_parser fuzz/fuzz_parser.cpp)
    target_link_libraries(fuzz_parser PRIVATE mylib)
    target_compile_options(fuzz_parser PRIVATE -fsanitize=fuzzer,address,undefined)
    target_link_options(fuzz_parser PRIVATE -fsanitize=fuzzer,address,undefined)
endif()
```

```bash
# Run fuzzer with corpus directory
./fuzz_parser corpus/ -max_total_time=3600 -max_len=4096
```

### AFL++

```bash
# Compile with AFL++ instrumentation
afl-clang-fast++ -fsanitize=address -o fuzz_target fuzz_target.cpp

# Run AFL++
afl-fuzz -i seeds/ -o findings/ ./fuzz_target @@
```

Always fuzz with ASan + UBSan enabled to catch memory errors and undefined behavior during fuzzing.

## Cryptography

Never implement cryptographic algorithms from scratch. Use established libraries.

| Library | Use Case | License |
|---------|----------|---------|
| OpenSSL/BoringSSL | TLS, general crypto | Apache 2.0 |
| libsodium | Simple, safe crypto API | ISC |
| Botan | Modern C++ crypto library | BSD-2 |

```cpp
// libsodium: secure random generation
#include <sodium.h>

void generate_token(std::span<std::byte> out) {
    randombytes_buf(out.data(), out.size());
}

// libsodium: password hashing (Argon2id)
std::array<char, crypto_pwhash_STRBYTES> hash_password(std::string_view password) {
    std::array<char, crypto_pwhash_STRBYTES> hash;
    if (crypto_pwhash_str(hash.data(), password.data(), password.size(),
            crypto_pwhash_OPSLIMIT_MODERATE,
            crypto_pwhash_MEMLIMIT_MODERATE) != 0) {
        throw std::runtime_error("Password hashing failed");
    }
    return hash;
}
```

Key cryptographic rules:
- Use `std::random_device` or `randombytes_buf` (libsodium) for security-critical randomness, never `std::mt19937`
- Use Argon2id for password hashing, never SHA-256 or bcrypt for new designs
- Use authenticated encryption (AES-GCM, ChaCha20-Poly1305), never unauthenticated
- Validate TLS certificates — do not disable verification in production

## Additional Resources

### Reference Files

For in-depth security guidance, consult:

- **`references/secure-coding-practices.md`** — Input validation patterns, safe integer arithmetic, format string prevention, deserialization safety, CWE mapping, secure API design
- **`references/hardening-and-fuzzing.md`** — Compiler/linker hardening deep dive, fuzzing methodology (libFuzzer, AFL++, corpus management), supply chain security, privilege separation, sandboxing, cryptographic library integration
