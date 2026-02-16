# Hardening Guide

## Compiler Flags

```toml
# Cargo.toml — release profile hardening
[profile.release]
opt-level = 3
lto = "fat"           # Link-time optimization (smaller, slower build)
codegen-units = 1     # Better optimization (slower build)
panic = "abort"       # Smaller binary, no unwinding
strip = true          # Remove debug symbols
overflow-checks = true # Keep overflow checks in release

# For security-critical builds
[profile.release]
debug = 1             # Keep debug info for crash analysis
```

```rust
// Crate-level lints for security
#![deny(unsafe_code)]          // Forbid all unsafe (or use #![forbid(unsafe_code)])
#![deny(clippy::all)]
#![deny(clippy::pedantic)]
#![warn(clippy::nursery)]

// Allow unsafe only where needed
#[allow(unsafe_code)]
mod ffi_bindings;

// Security-relevant clippy lints
#![deny(clippy::integer_arithmetic)]  // Catch overflow
#![deny(clippy::unwrap_used)]         // Force proper error handling
#![deny(clippy::expect_used)]
#![deny(clippy::indexing_slicing)]    // Prefer .get() to avoid panics
```

## Sanitizers

```bash
# AddressSanitizer — detect memory errors (buffer overflow, use-after-free)
RUSTFLAGS="-Z sanitizer=address" cargo +nightly test

# MemorySanitizer — detect uninitialized memory reads
RUSTFLAGS="-Z sanitizer=memory" cargo +nightly test

# ThreadSanitizer — detect data races
RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test

# LeakSanitizer — detect memory leaks
RUSTFLAGS="-Z sanitizer=leak" cargo +nightly test

# Run with all sanitizers in CI
# Note: sanitizers require nightly and may not work on all platforms
```

## Secure Defaults

```rust
// Always validate and sanitize input
fn process_input(input: &str) -> Result<Output> {
    // Length check
    if input.len() > MAX_INPUT_SIZE {
        return Err(AppError::InputTooLarge);
    }

    // UTF-8 is guaranteed by &str, but validate content
    if input.chars().any(|c| c.is_control() && c != '\n') {
        return Err(AppError::InvalidInput("control characters not allowed".into()));
    }

    // Parse strictly
    let parsed: Request = serde_json::from_str(input)?;

    // Validate business rules
    parsed.validate()?;

    Ok(process(parsed))
}

// Secure random number generation
use rand::Rng;

fn generate_token() -> String {
    let mut rng = rand::thread_rng(); // Uses OS entropy
    let bytes: [u8; 32] = rng.gen();
    hex::encode(bytes)
}

// Zeroize sensitive data on drop
use zeroize::Zeroize;

struct Secret {
    key: Vec<u8>,
}

impl Drop for Secret {
    fn drop(&mut self) {
        self.key.zeroize(); // Overwrite memory with zeros
    }
}

// Or use the derive macro
#[derive(Zeroize)]
#[zeroize(drop)]
struct ApiKey {
    value: String,
}
```

## Supply Chain Security

```toml
# deny.toml — comprehensive dependency policy
[advisories]
db-path = "~/.cargo/advisory-db"
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
notice = "warn"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-DFS-2016",
]
copyleft = "deny"

[bans]
multiple-versions = "warn"
wildcards = "deny"
# Prefer specific implementations
deny = [
    { name = "openssl-sys", wrappers = ["openssl"] },
]
# Allow only specific sources
[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

```bash
# Full security pipeline
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo audit
cargo deny check
cargo test
```

## WASM Sandboxing

```rust
// Compile to WASM for sandboxed execution
// Cargo.toml
// [lib]
// crate-type = ["cdylib"]

// WASM-specific security
#[cfg(target_arch = "wasm32")]
mod wasm {
    use wasm_bindgen::prelude::*;

    // No filesystem access in WASM by default
    // No network access without explicit host binding
    // Memory is sandboxed to WASM linear memory

    #[wasm_bindgen]
    pub fn process(input: &str) -> String {
        // Safe: runs in WASM sandbox
        transform(input)
    }
}

// Using wasmtime for server-side WASM sandboxing
use wasmtime::*;

fn run_sandboxed(wasm_bytes: &[u8], input: &[u8]) -> Result<Vec<u8>> {
    let engine = Engine::default();
    let module = Module::new(&engine, wasm_bytes)?;

    let mut store = Store::new(&engine, ());
    let instance = Instance::new(&mut store, &module, &[])?;

    // WASM module has no access to host filesystem, network, etc.
    // Only explicitly exported functions are callable
    let process = instance.get_typed_func::<(i32, i32), i32>(&mut store, "process")?;

    // Write input to WASM memory, call function, read output
    // ...
    Ok(vec![])
}
```

## Constant-Time Operations

```rust
use subtle::{Choice, ConditionallySelectable, ConstantTimeEq};

// Constant-time comparison prevents timing attacks
fn verify_hmac(computed: &[u8; 32], received: &[u8; 32]) -> bool {
    computed.ct_eq(received).into()
}

// Constant-time conditional select
fn constant_time_select(condition: bool, a: u64, b: u64) -> u64 {
    let choice = Choice::from(condition as u8);
    u64::conditional_select(&b, &a, choice)
}

// Never branch on secrets
// BAD:
// if secret_byte == 0 { ... }
// GOOD:
// use ct_eq and conditional operations
```
