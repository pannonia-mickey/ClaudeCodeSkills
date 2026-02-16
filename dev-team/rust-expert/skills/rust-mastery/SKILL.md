---
name: Rust Mastery
description: This skill should be used when the user asks about "Rust traits", "Rust generics", "Rust error handling", "thiserror", "anyhow", "Rust project structure", "Rust modules", "serde", "Rust iterators", "Rust enums", or "Rust design patterns". It covers traits, generics, error handling, project structure, serialization, and idiomatic patterns.
---

# Rust Mastery

## Traits and Generics

```rust
use std::fmt;

// Define traits with default implementations
trait Summary: fmt::Display {
    fn summarize(&self) -> String;
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..50.min(self.summarize().len())])
    }
}

// Generic function with trait bounds
fn notify(item: &impl Summary) {
    println!("Breaking: {}", item.summarize());
}

// Multiple bounds with where clause
fn process<T, U>(t: &T, u: &U) -> String
where
    T: Summary + Clone,
    U: fmt::Debug + Send,
{
    format!("{}: {:?}", t.summarize(), u)
}

// Return impl Trait (opaque return type)
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}

// Newtype pattern for type safety
struct UserId(u64);
struct OrderId(u64);

impl UserId {
    fn new(id: u64) -> Self {
        Self(id)
    }
    fn as_u64(&self) -> u64 {
        self.0
    }
}
```

## Error Handling

```rust
use thiserror::Error;

// Library errors with thiserror
#[derive(Debug, Error)]
enum AppError {
    #[error("user not found: {id}")]
    UserNotFound { id: String },

    #[error("validation failed: {0}")]
    Validation(String),

    #[error("unauthorized")]
    Unauthorized,

    #[error(transparent)]
    Database(#[from] sqlx::Error),

    #[error(transparent)]
    Io(#[from] std::io::Error),
}

// Application errors with anyhow (for binaries)
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;

    let config: Config = toml::from_str(&contents)
        .context("failed to parse config")?;

    Ok(config)
}

// Convert between error types
impl From<AppError> for StatusCode {
    fn from(err: AppError) -> Self {
        match err {
            AppError::UserNotFound { .. } => StatusCode::NOT_FOUND,
            AppError::Validation(_) => StatusCode::BAD_REQUEST,
            AppError::Unauthorized => StatusCode::UNAUTHORIZED,
            _ => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

## Enums and Pattern Matching

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Event {
    UserCreated { user_id: String, email: String },
    OrderPlaced { order_id: String, total: f64 },
    PaymentProcessed { payment_id: String, status: PaymentStatus },
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
enum PaymentStatus {
    Pending,
    Completed,
    Failed,
}

fn handle_event(event: &Event) {
    match event {
        Event::UserCreated { user_id, email } => {
            println!("Welcome {email} (id: {user_id})");
        }
        Event::OrderPlaced { total, .. } if *total > 1000.0 => {
            println!("High-value order: ${total}");
        }
        Event::OrderPlaced { order_id, .. } => {
            println!("Order {order_id} placed");
        }
        Event::PaymentProcessed { status: PaymentStatus::Failed, .. } => {
            println!("Payment failed — trigger retry");
        }
        Event::PaymentProcessed { .. } => {}
    }
}
```

## Serde Serialization

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: String,
    #[serde(rename = "userName")]
    name: String,
    email: String,
    #[serde(skip_serializing)]
    password_hash: String,
    #[serde(default)]
    roles: Vec<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,
    #[serde(with = "chrono::serde::ts_seconds")]
    created_at: chrono::DateTime<chrono::Utc>,
}

// Custom deserialization
fn deserialize_nonempty_string<'de, D>(deserializer: D) -> Result<String, D::Error>
where
    D: serde::Deserializer<'de>,
{
    let s = String::deserialize(deserializer)?;
    if s.is_empty() {
        return Err(serde::de::Error::custom("string must not be empty"));
    }
    Ok(s)
}
```

## References

- [Rust Patterns](references/rust-patterns.md) — Builder pattern, type state, iterator chains, smart pointers, interior mutability, Cow usage.
- [Project Structure](references/project-structure.md) — Workspace layout, module organization, feature flags, dependency management, publishing.
