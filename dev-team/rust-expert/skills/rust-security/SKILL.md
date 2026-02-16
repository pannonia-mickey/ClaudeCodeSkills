---
name: Rust Security
description: This skill should be used when the user asks about "Rust security", "unsafe Rust", "Rust FFI", "Rust crypto", "Rust input validation", "Rust memory safety", "cargo audit", "Rust unsafe code", "Rust C interop", or "Rust security best practices". It covers unsafe code guidelines, FFI, crypto, dependency auditing, and secure coding patterns.
---

# Rust Security

## Unsafe Rust Guidelines

```rust
// Rule: Minimize unsafe blocks, maximize safe abstractions around them

// GOOD: Unsafe implementation, safe public API
pub struct FixedBuffer {
    data: Box<[u8]>,
    len: usize,
}

impl FixedBuffer {
    pub fn new(capacity: usize) -> Self {
        Self {
            data: vec![0u8; capacity].into_boxed_slice(),
            len: 0,
        }
    }

    pub fn push(&mut self, byte: u8) -> bool {
        if self.len >= self.data.len() {
            return false;
        }
        // SAFETY: We just checked that len < data.len()
        unsafe {
            *self.data.get_unchecked_mut(self.len) = byte;
        }
        self.len += 1;
        true
    }

    pub fn as_slice(&self) -> &[u8] {
        &self.data[..self.len]
    }
}

// Document SAFETY invariants for every unsafe block
unsafe fn dangerous_operation(ptr: *mut u8, len: usize) {
    // SAFETY: Caller must ensure:
    // 1. ptr is valid for writes of len bytes
    // 2. ptr is properly aligned for u8
    // 3. No other references to this memory exist
    std::ptr::write_bytes(ptr, 0, len);
}
```

## FFI — Foreign Function Interface

```rust
// Calling C from Rust
extern "C" {
    fn strlen(s: *const std::ffi::c_char) -> usize;
    fn malloc(size: usize) -> *mut std::ffi::c_void;
    fn free(ptr: *mut std::ffi::c_void);
}

// Safe wrapper around unsafe FFI
use std::ffi::{CStr, CString};

pub fn safe_strlen(s: &str) -> Result<usize, std::ffi::NulError> {
    let c_string = CString::new(s)?;
    // SAFETY: CString guarantees null-terminated, valid pointer
    let len = unsafe { strlen(c_string.as_ptr()) };
    Ok(len)
}

// Exposing Rust to C
#[no_mangle]
pub extern "C" fn rust_process(data: *const u8, len: usize) -> i32 {
    // SAFETY: Caller must provide valid pointer and length
    let slice = unsafe {
        if data.is_null() {
            return -1;
        }
        std::slice::from_raw_parts(data, len)
    };

    match process_data(slice) {
        Ok(_) => 0,
        Err(_) => -1,
    }
}

// Using bindgen for automatic bindings
// build.rs
fn main() {
    println!("cargo:rerun-if-changed=wrapper.h");
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");
    bindings.write_to_file(
        std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap()).join("bindings.rs")
    ).expect("Couldn't write bindings");
}
```

## Cryptography

```rust
// Use well-audited crates: ring, rustls, RustCrypto

// Password hashing with argon2
use argon2::{Argon2, PasswordHasher, PasswordVerifier, password_hash::SaltString};

fn hash_password(password: &str) -> Result<String> {
    let salt = SaltString::generate(&mut rand::thread_rng());
    let argon2 = Argon2::default();
    let hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| anyhow::anyhow!("hashing failed: {e}"))?;
    Ok(hash.to_string())
}

fn verify_password(password: &str, hash: &str) -> Result<bool> {
    let parsed = argon2::PasswordHash::new(hash)
        .map_err(|e| anyhow::anyhow!("invalid hash: {e}"))?;
    Ok(Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok())
}

// AES-GCM encryption with RustCrypto
use aes_gcm::{Aes256Gcm, Key, Nonce, aead::{Aead, KeyInit, OsRng}};

fn encrypt(plaintext: &[u8], key: &[u8; 32]) -> Result<Vec<u8>> {
    let cipher = Aes256Gcm::new(Key::<Aes256Gcm>::from_slice(key));
    let nonce_bytes: [u8; 12] = rand::random();
    let nonce = Nonce::from_slice(&nonce_bytes);

    let ciphertext = cipher.encrypt(nonce, plaintext)
        .map_err(|e| anyhow::anyhow!("encryption failed: {e}"))?;

    // Prepend nonce to ciphertext
    let mut output = nonce_bytes.to_vec();
    output.extend(ciphertext);
    Ok(output)
}

// TLS with rustls
use rustls::ServerConfig;

fn tls_config() -> Result<ServerConfig> {
    let certs = load_certs("cert.pem")?;
    let key = load_private_key("key.pem")?;

    let config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)?;

    Ok(config)
}
```

## Input Validation

```rust
use validator::Validate;

#[derive(Debug, Validate, Deserialize)]
struct CreateUserRequest {
    #[validate(length(min = 2, max = 100))]
    name: String,

    #[validate(email)]
    email: String,

    #[validate(range(min = 8, max = 128))]
    #[validate(custom(function = "validate_password_strength"))]
    password: String,
}

fn validate_password_strength(password: &str) -> Result<(), validator::ValidationError> {
    let has_upper = password.chars().any(|c| c.is_uppercase());
    let has_lower = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_ascii_digit());

    if !has_upper || !has_lower || !has_digit {
        return Err(validator::ValidationError::new("weak_password"));
    }
    Ok(())
}

// Constant-time comparison for secrets
use subtle::ConstantTimeEq;

fn verify_api_key(provided: &[u8], stored: &[u8]) -> bool {
    provided.ct_eq(stored).into()
}
```

## Dependency Auditing

```bash
# cargo-audit — check for known vulnerabilities
cargo install cargo-audit
cargo audit

# cargo-deny — comprehensive policy enforcement
cargo install cargo-deny
cargo deny init  # Create deny.toml
cargo deny check

# deny.toml
# [advisories]
# vulnerability = "deny"
# unmaintained = "warn"
#
# [licenses]
# allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
#
# [bans]
# multiple-versions = "warn"
# deny = [{ name = "openssl" }]  # Prefer rustls
```

## References

- [Unsafe Patterns](references/unsafe-patterns.md) — Safe abstractions over unsafe, Pin and Unpin, raw pointers, transmute, global state, inline assembly.
- [Hardening Guide](references/hardening-guide.md) — Compiler flags, sanitizers, fuzzing integration, secure defaults, supply chain security, WASM sandboxing.
