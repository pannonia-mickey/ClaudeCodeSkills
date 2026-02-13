# Hardening and Fuzzing — Deep Dive Reference

## Compiler and Linker Hardening Deep Dive

### Stack Protection

#### Stack Canaries (`-fstack-protector-strong`)

Place a random "canary" value between local variables and the return address. If a buffer overflow overwrites the canary, it is detected before the function returns.

| Flag | Coverage |
|------|----------|
| `-fstack-protector` | Functions with buffers > 8 bytes |
| `-fstack-protector-strong` | Functions with local arrays, address-taken locals, or alloca |
| `-fstack-protector-all` | All functions (high overhead, rarely needed) |

Prefer `-fstack-protector-strong` — good coverage with minimal overhead (~1-2%).

#### Stack Clash Protection (`-fstack-clash-protection`)

Prevents "stack clash" attacks where an attacker causes the stack to grow into the heap (or vice versa) by skipping guard pages. The compiler inserts probes at every page boundary during large stack allocations.

### Address Space Layout Randomization (ASLR)

ASLR randomizes the base addresses of the executable, libraries, stack, and heap. Requires the executable to be position-independent:

```cmake
# Enable PIE (Position-Independent Executable)
target_compile_options(myapp PRIVATE -fPIE)
target_link_options(myapp PRIVATE -pie)

# For shared libraries, -fPIC is standard
target_compile_options(mylib PRIVATE -fPIC)
```

ASLR levels:
- Stack and library randomization (enabled by default on modern OS)
- Full ASLR requires PIE for the main executable
- High-entropy ASLR (`/HIGHENTROPYVA` on MSVC) uses 64-bit address space

### RELRO (RELocation Read-Only)

Protects the Global Offset Table (GOT) from overwrite attacks:

| Mode | Flag | Protection |
|------|------|-----------|
| Partial RELRO | `-Wl,-z,relro` | GOT is read-only after relocation, but PLT GOT is writable |
| Full RELRO | `-Wl,-z,relro,-z,now` | All GOT entries resolved at load time, entire GOT read-only |

Full RELRO adds startup latency (all symbols resolved eagerly) but eliminates GOT overwrite as an exploitation technique. Prefer full RELRO for security-critical applications.

### Control-Flow Integrity (CFI)

Prevents control-flow hijacking attacks (ROP, JOP) by validating indirect call targets:

```cmake
# Clang CFI (software-based)
target_compile_options(myapp PRIVATE -fsanitize=cfi -fvisibility=hidden -flto)
target_link_options(myapp PRIVATE -fsanitize=cfi -flto)

# GCC/Clang Intel CET (hardware-based, requires CPU support)
target_compile_options(myapp PRIVATE -fcf-protection=full)

# MSVC Control Flow Guard
target_compile_options(myapp PRIVATE /guard:cf)
target_link_options(myapp PRIVATE /guard:cf)
```

CFI modes:
- **Forward-edge CFI** (indirect call/jump validation): `-fsanitize=cfi-icall`, `/guard:cf`
- **Backward-edge CFI** (return address protection): Shadow stacks, Intel CET shadow stack
- **Hardware CFI** (Intel CET): Near-zero overhead, requires CPU support

### FORTIFY_SOURCE

Replaces vulnerable libc functions (`memcpy`, `strcpy`, `sprintf`, etc.) with bounds-checked versions that detect buffer overflows at compile time or runtime:

```cmake
target_compile_definitions(myapp PRIVATE _FORTIFY_SOURCE=3)
```

| Level | Behavior |
|-------|----------|
| `_FORTIFY_SOURCE=1` | Compile-time checks where buffer sizes are known |
| `_FORTIFY_SOURCE=2` | Additional runtime checks (more coverage, slight overhead) |
| `_FORTIFY_SOURCE=3` | Uses `__builtin_dynamic_object_size` for runtime-determined sizes (GCC 12+, Clang 15+) |

Requires optimization (`-O1` or higher) to be effective.

### Comprehensive Hardening CMake Module

```cmake
# cmake/Hardening.cmake
function(apply_hardening target)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        target_compile_options(${target} PRIVATE
            -fstack-protector-strong
            -fstack-clash-protection
            -fPIE
            -D_FORTIFY_SOURCE=3
            -Wformat=2
            -Wformat-security
            -Werror=format-security
            -Wimplicit-fallthrough
            -fno-delete-null-pointer-checks
            -fno-strict-overflow
            -ftrivial-auto-var-init=zero      # Zero-init stack variables
        )
        target_link_options(${target} PRIVATE
            -pie
            -Wl,-z,relro,-z,now
            -Wl,-z,noexecstack
            -Wl,-z,separate-code
        )

        # Hardware CFI if available
        include(CheckCXXCompilerFlag)
        check_cxx_compiler_flag(-fcf-protection=full HAS_CF_PROTECTION)
        if(HAS_CF_PROTECTION)
            target_compile_options(${target} PRIVATE -fcf-protection=full)
        endif()
    elseif(MSVC)
        target_compile_options(${target} PRIVATE
            /GS           # Stack buffer overrun detection
            /guard:cf     # Control flow guard
            /sdl          # Security Development Lifecycle checks
            /Qspectre     # Spectre variant 1 mitigation
        )
        target_link_options(${target} PRIVATE
            /DYNAMICBASE  # ASLR
            /HIGHENTROPYVA
            /NXCOMPAT     # DEP
            /CETCOMPAT    # Intel CET
            /guard:cf
        )
    endif()
endfunction()

# Usage
apply_hardening(myapp)
```

### Verifying Hardening

```bash
# Check binary hardening on Linux
checksec --file=./myapp
# Expected output:
# RELRO:     Full RELRO
# Stack:     Canary found
# NX:        NX enabled
# PIE:       PIE enabled
# FORTIFY:   Enabled

# Check with readelf
readelf -d ./myapp | grep BIND_NOW    # Full RELRO
readelf -l ./myapp | grep GNU_STACK   # Non-executable stack
readelf -h ./myapp | grep Type        # DYN = PIE, EXEC = no PIE
```

## Fuzzing Methodology

### Fuzzing Strategy

1. **Identify attack surface**: Find all entry points that process untrusted input (parsers, protocol handlers, file loaders, API endpoints)
2. **Write fuzz targets**: One target per entry point, each exercising a specific code path
3. **Seed corpus**: Collect valid input samples as seeds for the fuzzer
4. **Run with sanitizers**: Always enable ASan + UBSan during fuzzing
5. **Triage and fix**: Analyze crashes, minimize reproducers, fix root causes
6. **Regression corpus**: Save crash-triggering inputs as test cases
7. **Continuous fuzzing**: Integrate into CI for ongoing coverage

### libFuzzer Integration

#### Fuzz Target Structure

```cpp
// fuzz/fuzz_json_parser.cpp
#include <cstdint>
#include <cstddef>
#include <span>
#include "mylib/json_parser.h"

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Optional: reject very large inputs for efficiency
    if (size > 1'000'000) return 0;

    auto input = std::span<const std::byte>(
        reinterpret_cast<const std::byte*>(data), size);

    // Exercise the code under test — no need to check return values
    // libFuzzer detects crashes, sanitizer violations, and leaks
    auto result = mylib::parse_json(input);

    // Optional: exercise more of the parsed result
    if (result) {
        result->serialize();  // Exercise serialization path too
    }

    return 0;
}
```

#### CMake Integration

```cmake
option(ENABLE_FUZZING "Build fuzz targets" OFF)

if(ENABLE_FUZZING)
    if(NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        message(FATAL_ERROR "Fuzzing requires Clang (libFuzzer)")
    endif()

    set(FUZZ_FLAGS -fsanitize=fuzzer,address,undefined -fno-omit-frame-pointer)

    # Create fuzz targets
    file(GLOB FUZZ_SOURCES fuzz/fuzz_*.cpp)
    foreach(fuzz_src ${FUZZ_SOURCES})
        get_filename_component(fuzz_name ${fuzz_src} NAME_WE)
        add_executable(${fuzz_name} ${fuzz_src})
        target_link_libraries(${fuzz_name} PRIVATE mylib)
        target_compile_options(${fuzz_name} PRIVATE ${FUZZ_FLAGS})
        target_link_options(${fuzz_name} PRIVATE ${FUZZ_FLAGS})
    endforeach()
endif()
```

#### Running and Managing Fuzzing

```bash
# Basic run with corpus directory
mkdir -p corpus/json_parser
./fuzz_json_parser corpus/json_parser/ -max_total_time=3600

# With additional options
./fuzz_json_parser corpus/ \
    -max_len=65536 \           # Maximum input size
    -timeout=10 \              # Per-input timeout (seconds)
    -rss_limit_mb=4096 \       # Memory limit
    -jobs=8 \                  # Parallel fuzzing jobs
    -workers=8 \               # Worker processes
    -print_final_stats=1 \     # Print statistics on exit
    -dict=dictionaries/json.dict  # Token dictionary

# Minimize a crash reproducer
./fuzz_json_parser -minimize_crash=1 -exact_artifact_path=minimized.bin crash-abc123

# Merge corpus (deduplicate, keep only coverage-increasing inputs)
./fuzz_json_parser -merge=1 corpus_merged/ corpus/ new_corpus/
```

#### Corpus Management

```bash
# Create seed corpus from real inputs
cp test/fixtures/*.json corpus/json_parser/

# Create a fuzzer dictionary (helps discover structured input faster)
cat > dictionaries/json.dict << 'EOF'
"{"
"}"
"["
"]"
"\""
":"
","
"null"
"true"
"false"
"\\n"
"\\t"
"\\u0000"
EOF
```

### AFL++ Integration

```bash
# Build with AFL++ instrumentation
CC=afl-clang-fast CXX=afl-clang-fast++ \
    cmake -B build-afl -DCMAKE_BUILD_TYPE=Release

cmake --build build-afl

# Create a harness that reads from stdin (AFL++ convention)
# fuzz/afl_json_parser.cpp
int main() {
    std::vector<char> input(std::istreambuf_iterator<char>(std::cin), {});
    auto result = mylib::parse_json(
        {reinterpret_cast<const std::byte*>(input.data()), input.size()});
    return 0;
}

# Run AFL++
afl-fuzz -i seeds/ -o findings/ -m none -- ./build-afl/afl_json_parser

# AFL++ persistent mode (much faster)
# In the harness:
__AFL_FUZZ_INIT();
int main() {
    unsigned char *buf = __AFL_FUZZ_TESTCASE_BUF;
    while (__AFL_LOOP(10000)) {
        int len = __AFL_FUZZ_TESTCASE_LEN;
        mylib::parse_json({reinterpret_cast<const std::byte*>(buf), static_cast<size_t>(len)});
    }
}
```

### Structure-Aware Fuzzing

For complex input formats, use protobuf-based structure-aware fuzzing:

```protobuf
// json_input.proto — describes valid JSON structure
syntax = "proto3";
message JsonValue {
    oneof value {
        double number = 1;
        string str = 2;
        bool boolean = 3;
        JsonArray array = 4;
        JsonObject object = 5;
    }
}
message JsonArray { repeated JsonValue items = 1; }
message JsonObject { map<string, JsonValue> entries = 1; }
```

Use libprotobuf-mutator with libFuzzer:
```cpp
#include <libprotobuf-mutator/src/libfuzzer/libfuzzer_macro.h>
#include "json_input.pb.h"

DEFINE_PROTO_FUZZER(const JsonValue& input) {
    std::string json_str = proto_to_json(input);
    mylib::parse_json(json_str);
}
```

## Supply Chain Security

### Dependency Auditing

```bash
# Conan: list all dependencies and their versions
conan graph info . --format=json | jq '.nodes[].ref'

# vcpkg: list installed packages
vcpkg list

# Check for known vulnerabilities
# Use OSV (Open Source Vulnerabilities) database
# pip install osv-scanner
osv-scanner --lockfile conan.lock
```

### Reproducible Builds

```cmake
# Ensure deterministic builds
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -frandom-seed=0")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdebug-prefix-map=${CMAKE_SOURCE_DIR}=.")

# Pin all dependency versions
# Conan: use lockfiles
conan lock create .
conan install . --lockfile=conan.lock

# vcpkg: use builtin-baseline in vcpkg.json
```

### SBOM (Software Bill of Materials)

Generate SBOM for vulnerability tracking:

```bash
# Conan: export dependency graph
conan graph info . --format=json > sbom.json

# Convert to SPDX or CycloneDX format using tools like:
# - cyclonedx-conan
# - conan-sbom-plugin
```

## Privilege Separation

### Principle of Least Privilege

Drop privileges as soon as they are no longer needed:

```cpp
#include <unistd.h>
#include <grp.h>

void drop_privileges(uid_t target_uid, gid_t target_gid) {
    // Drop supplementary groups
    if (setgroups(0, nullptr) != 0) {
        throw std::system_error(errno, std::generic_category(), "setgroups");
    }
    // Drop group privileges first (before dropping user)
    if (setgid(target_gid) != 0) {
        throw std::system_error(errno, std::generic_category(), "setgid");
    }
    // Drop user privileges
    if (setuid(target_uid) != 0) {
        throw std::system_error(errno, std::generic_category(), "setuid");
    }
    // Verify privileges were actually dropped
    if (getuid() != target_uid || geteuid() != target_uid) {
        throw std::runtime_error("Failed to drop user privileges");
    }
}

// Usage: bind port 443, then drop root
int main() {
    auto socket = bind_port(443);  // Requires root
    drop_privileges(nobody_uid, nobody_gid);  // Drop to unprivileged user
    serve(socket);  // Run as unprivileged
}
```

### Sandboxing with seccomp

Restrict system calls to the minimum needed:

```cpp
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/prctl.h>

void apply_seccomp_filter() {
    // Allow only: read, write, exit, exit_group, brk, mmap (for allocation)
    struct sock_filter filter[] = {
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        // ... more allowed syscalls
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),  // Kill on anything else
    };

    struct sock_fprog prog = {
        .len = sizeof(filter) / sizeof(filter[0]),
        .filter = filter,
    };

    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
}
```

For real-world use, prefer libseccomp for a higher-level API, or use Landlock (Linux 5.13+) for filesystem sandboxing.

### Process Isolation Architecture

For security-critical applications, split into multiple processes with different privilege levels:

```
┌─────────────────────────┐
│    Frontend Process      │  Unprivileged, sandboxed
│  (Input parsing, UI)     │  Handles untrusted data
│    seccomp-filtered      │
└──────────┬──────────────┘
           │ IPC (Unix socket, pipe)
┌──────────▼──────────────┐
│    Backend Process       │  Limited privileges
│  (Business logic)        │  No network access
└──────────┬──────────────┘
           │ IPC
┌──────────▼──────────────┐
│    Privileged Process    │  Root or CAP_NET_BIND
│  (Network, crypto keys)  │  Minimal attack surface
└─────────────────────────┘
```

## Cryptographic Library Integration

### libsodium Quick Reference

```cpp
#include <sodium.h>

// Initialize (must be called once)
if (sodium_init() < 0) abort();

// Secure random
std::array<std::byte, 32> key;
randombytes_buf(key.data(), key.size());

// Authenticated encryption (XChaCha20-Poly1305)
std::vector<unsigned char> encrypt(
    std::span<const unsigned char> plaintext,
    std::span<const unsigned char, crypto_aead_xchacha20poly1305_ietf_KEYBYTES> key)
{
    std::vector<unsigned char> ciphertext(
        plaintext.size() + crypto_aead_xchacha20poly1305_ietf_ABYTES +
        crypto_aead_xchacha20poly1305_ietf_NPUBBYTES);

    // Generate random nonce
    unsigned char* nonce = ciphertext.data();
    randombytes_buf(nonce, crypto_aead_xchacha20poly1305_ietf_NPUBBYTES);

    unsigned long long ciphertext_len;
    crypto_aead_xchacha20poly1305_ietf_encrypt(
        ciphertext.data() + crypto_aead_xchacha20poly1305_ietf_NPUBBYTES,
        &ciphertext_len,
        plaintext.data(), plaintext.size(),
        nullptr, 0,  // No additional data
        nullptr,      // No secret nonce
        nonce, key.data());

    ciphertext.resize(crypto_aead_xchacha20poly1305_ietf_NPUBBYTES + ciphertext_len);
    return ciphertext;
}

// Secure memory allocation
void* secure_ptr = sodium_malloc(secret_size);
sodium_mprotect_noaccess(secure_ptr);   // Make inaccessible
sodium_mprotect_readonly(secure_ptr);    // Make read-only
sodium_mprotect_readwrite(secure_ptr);   // Make read-write
sodium_free(secure_ptr);                 // Wipes and frees
```

### Botan Quick Reference

```cpp
#include <botan/auto_rng.h>
#include <botan/aead.h>
#include <botan/hash.h>
#include <botan/pwdhash.h>

// Secure random
Botan::AutoSeeded_RNG rng;
auto key = rng.random_vec(32);

// AES-256-GCM authenticated encryption
auto enc = Botan::AEAD_Mode::create("AES-256/GCM", Botan::Cipher_Dir::Encryption);
enc->set_key(key);
auto nonce = rng.random_vec(enc->default_nonce_length());
enc->start(nonce);
Botan::secure_vector<uint8_t> ciphertext(plaintext.begin(), plaintext.end());
enc->finish(ciphertext);

// Password hashing (Argon2id)
auto pwdhash = Botan::PasswordHashFamily::create("Argon2id");
auto hasher = pwdhash->from_params(65536, 3, 1);  // 64MB memory, 3 iterations, 1 thread
std::vector<uint8_t> salt = rng.random_vec(16);
std::vector<uint8_t> hash(32);
hasher->derive_key(hash.data(), hash.size(),
    password.data(), password.size(),
    salt.data(), salt.size());
```

### Cryptography Checklist

| Requirement | Recommended Algorithm |
|-------------|----------------------|
| Symmetric encryption | AES-256-GCM or XChaCha20-Poly1305 |
| Password hashing | Argon2id |
| General hashing | BLAKE2b or SHA-256 |
| Digital signatures | Ed25519 |
| Key exchange | X25519 |
| Random generation | OS CSPRNG (libsodium, Botan RNG) |
| TLS | TLS 1.3 (via OpenSSL/BoringSSL) |

**Never**: Implement crypto from scratch, use ECB mode, use MD5/SHA-1 for security, use `rand()`/`mt19937` for security-critical randomness, disable TLS certificate verification in production.
