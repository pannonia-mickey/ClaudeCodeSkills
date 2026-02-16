# Project Structure

## Workspace Layout

```
my-project/
├── Cargo.toml              # Workspace root
├── crates/
│   ├── api/                # HTTP API binary
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── routes/
│   │       │   ├── mod.rs
│   │       │   ├── users.rs
│   │       │   └── orders.rs
│   │       ├── middleware/
│   │       │   ├── mod.rs
│   │       │   └── auth.rs
│   │       └── extractors.rs
│   ├── core/               # Domain logic library
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── models/
│   │       ├── services/
│   │       └── errors.rs
│   ├── db/                 # Database layer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── repos/
│   │       └── migrations/
│   └── cli/                # CLI binary
│       ├── Cargo.toml
│       └── src/main.rs
├── tests/                  # Integration tests
│   └── api_tests.rs
└── .cargo/
    └── config.toml
```

```toml
# Root Cargo.toml
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.package]
version = "0.1.0"
edition = "2021"
rust-version = "1.75"

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
thiserror = "2"
tracing = "0.1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
```

```toml
# crates/api/Cargo.toml
[package]
name = "my-project-api"
version.workspace = true
edition.workspace = true

[dependencies]
my-project-core = { path = "../core" }
my-project-db = { path = "../db" }
tokio.workspace = true
serde.workspace = true
axum = "0.7"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "compression"] }
tracing.workspace = true
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

## Module Organization

```rust
// src/lib.rs — public API surface
pub mod models;
pub mod services;
pub mod errors;

// Re-exports for convenience
pub use errors::{AppError, Result};
pub use models::{User, Order};

// src/models/mod.rs
mod user;
mod order;

pub use user::User;
pub use order::Order;

// src/models/user.rs
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct User {
    pub id: uuid::Uuid,
    pub name: String,
    pub email: String,
    pub(crate) password_hash: String, // Visible within crate only
}
```

## Feature Flags

```toml
# Cargo.toml
[features]
default = ["postgres"]
postgres = ["sqlx/postgres"]
sqlite = ["sqlx/sqlite"]
cache = ["redis"]
tracing-otel = ["opentelemetry", "tracing-opentelemetry"]

[dependencies]
redis = { version = "0.25", optional = true }
opentelemetry = { version = "0.22", optional = true }
tracing-opentelemetry = { version = "0.23", optional = true }
```

```rust
// Conditional compilation with features
#[cfg(feature = "cache")]
mod cache;

#[cfg(feature = "cache")]
pub use cache::CacheLayer;

// In code
fn get_user(&self, id: &str) -> Result<User> {
    #[cfg(feature = "cache")]
    if let Some(cached) = self.cache.get(id)? {
        return Ok(cached);
    }

    let user = self.db.find_user(id)?;

    #[cfg(feature = "cache")]
    self.cache.set(id, &user)?;

    Ok(user)
}
```

## Dependency Management

```toml
# Cargo.toml — common dependency patterns

[dependencies]
# Exact version (avoid in libraries)
serde = "=1.0.197"

# Compatible range (default, recommended for libraries)
serde = "1.0"

# Minimum version
serde = ">=1.0.180"

[dev-dependencies]
tokio = { version = "1", features = ["test-util", "macros"] }
assert_cmd = "2"
predicates = "3"
tempfile = "3"
wiremock = "0.6"

[build-dependencies]
prost-build = "0.12"
```

```bash
# Useful cargo commands
cargo update                  # Update Cargo.lock
cargo audit                   # Check for security advisories
cargo outdated                # Find outdated dependencies
cargo tree                    # Dependency tree
cargo tree -d                 # Find duplicate dependencies
cargo deny check              # Policy enforcement (licenses, bans)
```

## Cargo Configuration

```toml
# .cargo/config.toml
[build]
# Use mold linker for faster builds (Linux)
# [target.x86_64-unknown-linux-gnu]
# linker = "clang"
# rustflags = ["-C", "link-arg=-fuse-ld=mold"]

[alias]
xtask = "run --package xtask --"
ci = "clippy --all-targets --all-features -- -D warnings"

[env]
RUST_BACKTRACE = "1"

# Profile optimizations
[profile.dev]
opt-level = 0

[profile.dev.package."*"]
opt-level = 2  # Optimize dependencies in dev mode

[profile.release]
lto = "thin"
codegen-units = 1
strip = true
```
