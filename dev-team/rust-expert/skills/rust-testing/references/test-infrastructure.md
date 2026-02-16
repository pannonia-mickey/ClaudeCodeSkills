# Test Infrastructure

## Test Fixtures

```rust
use once_cell::sync::Lazy;
use std::sync::Arc;

// Shared test state with lazy initialization
static TEST_DB: Lazy<Arc<PgPool>> = Lazy::new(|| {
    let rt = tokio::runtime::Runtime::new().unwrap();
    Arc::new(rt.block_on(setup_database()))
});

// Test fixture trait
trait TestFixture {
    fn setup() -> Self;
    fn teardown(self);
}

struct UserFixture {
    pool: PgPool,
    user_ids: Vec<String>,
}

impl UserFixture {
    async fn with_users(pool: PgPool, count: usize) -> Self {
        let mut user_ids = Vec::new();
        for i in 0..count {
            let id = format!("test-user-{i}");
            sqlx::query!("INSERT INTO users (id, name) VALUES ($1, $2)", id, format!("User {i}"))
                .execute(&pool)
                .await
                .unwrap();
            user_ids.push(id);
        }
        Self { pool, user_ids }
    }

    async fn cleanup(self) {
        for id in &self.user_ids {
            sqlx::query!("DELETE FROM users WHERE id = $1", id)
                .execute(&self.pool)
                .await
                .ok();
        }
    }
}
```

## Test Database Setup

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};

async fn setup_test_database() -> PgPool {
    let database_url = std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://test:test@localhost:5432/test_db".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test database");

    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");

    pool
}

// Isolated test with transaction rollback
async fn with_test_transaction<F, Fut>(pool: &PgPool, test: F)
where
    F: FnOnce(sqlx::Transaction<'_, sqlx::Postgres>) -> Fut,
    Fut: Future<Output = ()>,
{
    let tx = pool.begin().await.unwrap();
    test(tx).await;
    // Transaction is automatically rolled back when dropped
}

// Each test gets its own database (using template databases)
async fn create_test_db(base_pool: &PgPool) -> PgPool {
    let db_name = format!("test_{}", uuid::Uuid::new_v4().to_string().replace('-', ""));

    sqlx::query(&format!("CREATE DATABASE {db_name} TEMPLATE test_template"))
        .execute(base_pool)
        .await
        .unwrap();

    let url = format!("postgres://test:test@localhost:5432/{db_name}");
    PgPool::connect(&url).await.unwrap()
}
```

## Wiremock HTTP Mocking

```rust
use wiremock::{Mock, MockServer, ResponseTemplate};
use wiremock::matchers::{method, path, header, body_json};

#[tokio::test]
async fn test_external_api_call() {
    let mock_server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/api/users/123"))
        .and(header("Authorization", "Bearer test-token"))
        .respond_with(ResponseTemplate::new(200)
            .set_body_json(serde_json::json!({
                "id": "123",
                "name": "Alice"
            })))
        .expect(1)
        .mount(&mock_server)
        .await;

    let client = ApiClient::new(&mock_server.uri(), "test-token");
    let user = client.get_user("123").await.unwrap();
    assert_eq!(user.name, "Alice");

    // Expectations are verified on drop
}

// Mock with dynamic responses
Mock::given(method("POST"))
    .and(path("/api/users"))
    .respond_with(|req: &wiremock::Request| {
        let body: serde_json::Value = serde_json::from_slice(&req.body).unwrap();
        ResponseTemplate::new(201)
            .set_body_json(serde_json::json!({
                "id": "new-id",
                "name": body["name"],
            }))
    })
    .mount(&mock_server)
    .await;

// Verify no unexpected requests
Mock::given(method("DELETE"))
    .respond_with(ResponseTemplate::new(500))
    .expect(0) // Must NOT be called
    .mount(&mock_server)
    .await;
```

## Snapshot Testing

```rust
use insta::{assert_snapshot, assert_json_snapshot, assert_debug_snapshot};

#[test]
fn test_render_template() {
    let output = render_email_template(&user);
    assert_snapshot!(output);
}

#[test]
fn test_api_response() {
    let response = build_user_response(&user);
    assert_json_snapshot!(response, {
        ".created_at" => "[timestamp]",  // Redact dynamic fields
        ".id" => "[uuid]",
    });
}

#[test]
fn test_ast_parsing() {
    let ast = parse("fn main() {}");
    assert_debug_snapshot!(ast);
}

// Review snapshots: cargo insta review
// Update snapshots: cargo insta accept
```

## Coverage

```bash
# Using cargo-tarpaulin
cargo install cargo-tarpaulin
cargo tarpaulin --out Html --output-dir coverage/
cargo tarpaulin --out Lcov --output-dir coverage/

# Using cargo-llvm-cov (more accurate)
cargo install cargo-llvm-cov
cargo llvm-cov --html --output-dir coverage/
cargo llvm-cov --lcov --output-path coverage/lcov.info

# Coverage with threshold
cargo llvm-cov --fail-under-lines 80

# Exclude test code from coverage
cargo llvm-cov --ignore-filename-regex "tests?/"
```

## CI Configuration

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

      - name: Run tests
        env:
          TEST_DATABASE_URL: postgres://test:test@localhost:5432/test_db
        run: cargo test --all-features

      - name: Run tests with coverage
        run: |
          cargo install cargo-llvm-cov
          cargo llvm-cov --all-features --lcov --output-path lcov.info

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo install cargo-audit
      - run: cargo audit
```
