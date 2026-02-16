---
name: Rust Testing
description: This skill should be used when the user asks about "Rust tests", "cargo test", "Rust integration tests", "Rust mocking", "Rust test organization", "Rust doc tests", "Rust property testing", "Rust benchmarks", or "Rust test fixtures". It covers unit tests, integration tests, doc tests, mocking, property-based testing, and benchmarks.
---

# Rust Testing

## Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_parse_valid() {
        let result = parse_amount("$10.50").unwrap();
        assert_eq!(result, 1050);
    }

    #[test]
    fn test_parse_invalid() {
        assert!(parse_amount("invalid").is_err());
    }

    #[test]
    #[should_panic(expected = "division by zero")]
    fn test_divide_by_zero() {
        divide(10, 0);
    }

    #[test]
    fn test_with_result() -> Result<(), Box<dyn std::error::Error>> {
        let config = Config::parse("key=value")?;
        assert_eq!(config.get("key"), Some("value"));
        Ok(())
    }
}
```

## Async Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_fetch_user() {
        let pool = setup_test_db().await;
        let repo = UserRepo::new(pool);

        let user = repo.create(&NewUser {
            name: "Alice".into(),
            email: "alice@test.com".into(),
        }).await.unwrap();

        let found = repo.find_by_id(&user.id).await.unwrap();
        assert_eq!(found.name, "Alice");
    }

    #[tokio::test]
    async fn test_timeout() {
        let result = tokio::time::timeout(
            Duration::from_millis(100),
            slow_operation(),
        ).await;
        assert!(result.is_err()); // Should timeout
    }
}
```

## Doc Tests

```rust
/// Adds two numbers together.
///
/// # Examples
///
/// ```
/// use my_crate::add;
///
/// assert_eq!(add(2, 3), 5);
/// assert_eq!(add(-1, 1), 0);
/// ```
///
/// # Panics
///
/// Panics if the result overflows.
///
/// ```should_panic
/// use my_crate::add;
/// add(i64::MAX, 1);
/// ```
pub fn add(a: i64, b: i64) -> i64 {
    a.checked_add(b).expect("overflow")
}

/// Parses a config string.
///
/// ```
/// # use my_crate::Config;
/// # fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let config = Config::parse("key=value")?;
/// assert_eq!(config.get("key"), Some("value"));
/// # Ok(())
/// # }
/// ```
pub fn parse(input: &str) -> Result<Config, ParseError> {
    // ...
}
```

## Integration Tests

```rust
// tests/api_tests.rs — separate from src/
use my_crate::app;

#[tokio::test]
async fn test_create_and_get_user() {
    let app = TestApp::spawn().await;

    let response = app.post("/api/users")
        .json(&serde_json::json!({
            "name": "Alice",
            "email": "alice@test.com"
        }))
        .send()
        .await;

    assert_eq!(response.status(), 201);

    let user: User = response.json().await;
    assert_eq!(user.name, "Alice");

    let get_response = app.get(&format!("/api/users/{}", user.id))
        .send()
        .await;

    assert_eq!(get_response.status(), 200);
}

// Test helper in tests/common/mod.rs
pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

impl TestApp {
    pub async fn spawn() -> Self {
        let db_pool = setup_test_database().await;
        let app = build_app(db_pool.clone());
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let address = format!("http://{}", listener.local_addr().unwrap());
        tokio::spawn(async move { axum::serve(listener, app).await });
        Self { address, db_pool }
    }
}
```

## Mocking with mockall

```rust
use mockall::automock;

#[automock]
trait UserRepository {
    fn find_by_id(&self, id: &str) -> Result<Option<User>>;
    fn save(&self, user: &User) -> Result<()>;
}

#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[test]
    fn test_get_user_found() {
        let mut mock = MockUserRepository::new();
        mock.expect_find_by_id()
            .with(eq("123"))
            .times(1)
            .returning(|_| Ok(Some(User {
                id: "123".into(),
                name: "Alice".into(),
            })));

        let service = UserService::new(Box::new(mock));
        let user = service.get_user("123").unwrap().unwrap();
        assert_eq!(user.name, "Alice");
    }

    #[test]
    fn test_get_user_not_found() {
        let mut mock = MockUserRepository::new();
        mock.expect_find_by_id()
            .returning(|_| Ok(None));

        let service = UserService::new(Box::new(mock));
        assert!(service.get_user("999").unwrap().is_none());
    }
}
```

## References

- [Property Testing](references/property-testing.md) — proptest strategies, shrinking, regression files, stateful testing, fuzzing with cargo-fuzz.
- [Test Infrastructure](references/test-infrastructure.md) — Test fixtures, test database setup, wiremock HTTP mocking, snapshot testing, coverage, CI configuration.
