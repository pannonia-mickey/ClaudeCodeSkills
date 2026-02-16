# Rust Patterns

## Builder Pattern

```rust
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    max_connections: usize,
    tls: Option<TlsConfig>,
}

#[derive(Debug, Default)]
struct ServerConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    max_connections: Option<usize>,
    tls: Option<TlsConfig>,
}

impl ServerConfigBuilder {
    fn new() -> Self {
        Self::default()
    }

    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    fn max_connections(mut self, max: usize) -> Self {
        self.max_connections = Some(max);
        self
    }

    fn tls(mut self, config: TlsConfig) -> Self {
        self.tls = Some(config);
        self
    }

    fn build(self) -> Result<ServerConfig, &'static str> {
        Ok(ServerConfig {
            host: self.host.unwrap_or_else(|| "0.0.0.0".to_string()),
            port: self.port.ok_or("port is required")?,
            max_connections: self.max_connections.unwrap_or(1024),
            tls: self.tls,
        })
    }
}

// Usage
let config = ServerConfigBuilder::new()
    .host("localhost")
    .port(8080)
    .max_connections(500)
    .build()?;

// Or use the `derive_builder` crate for automatic generation
```

## Type State Pattern

```rust
// Compile-time state machine — invalid transitions don't compile
struct Draft;
struct Published;
struct Archived;

struct Article<State> {
    title: String,
    content: String,
    _state: std::marker::PhantomData<State>,
}

impl Article<Draft> {
    fn new(title: String, content: String) -> Self {
        Article { title, content, _state: std::marker::PhantomData }
    }

    fn publish(self) -> Article<Published> {
        Article { title: self.title, content: self.content, _state: std::marker::PhantomData }
    }
}

impl Article<Published> {
    fn archive(self) -> Article<Archived> {
        Article { title: self.title, content: self.content, _state: std::marker::PhantomData }
    }
}

// Only published articles can be read
impl Article<Published> {
    fn read(&self) -> &str {
        &self.content
    }
}

// article.read() — compile error! Article<Draft> has no read method
let article = Article::<Draft>::new("title".into(), "content".into());
let published = article.publish();
println!("{}", published.read()); // OK
```

## Iterator Chains

```rust
// Functional iterator processing
let result: Vec<String> = users
    .iter()
    .filter(|u| u.is_active)
    .filter_map(|u| u.email.as_ref())
    .map(|email| email.to_lowercase())
    .collect();

// Chaining with early termination
let first_admin = users
    .iter()
    .find(|u| u.role == Role::Admin);

// Parallel iteration with rayon
use rayon::prelude::*;

let results: Vec<_> = data
    .par_iter()
    .filter(|item| item.needs_processing())
    .map(|item| expensive_transform(item))
    .collect();

// Custom iterator
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let result = self.a;
        let next = self.a.checked_add(self.b)?;
        self.a = self.b;
        self.b = next;
        Some(result)
    }
}

fn fibonacci() -> Fibonacci {
    Fibonacci { a: 0, b: 1 }
}

// Usage: fibonacci().take(10).collect::<Vec<_>>()
```

## Smart Pointers

```rust
use std::sync::{Arc, Mutex, RwLock};

// Arc<Mutex<T>> for shared mutable state across threads
struct SharedState {
    data: Arc<Mutex<HashMap<String, String>>>,
}

impl SharedState {
    fn new() -> Self {
        Self { data: Arc::new(Mutex::new(HashMap::new())) }
    }

    fn get(&self, key: &str) -> Option<String> {
        self.data.lock().unwrap().get(key).cloned()
    }

    fn set(&self, key: String, value: String) {
        self.data.lock().unwrap().insert(key, value);
    }
}

// Arc<RwLock<T>> for read-heavy workloads
struct Cache {
    data: Arc<RwLock<HashMap<String, CacheEntry>>>,
}

impl Cache {
    fn get(&self, key: &str) -> Option<CacheEntry> {
        self.data.read().unwrap().get(key).cloned()
    }

    fn set(&self, key: String, entry: CacheEntry) {
        self.data.write().unwrap().insert(key, entry);
    }
}

// Box<dyn Trait> for trait objects
fn get_handler(route: &str) -> Box<dyn Handler + Send> {
    match route {
        "/users" => Box::new(UserHandler),
        "/orders" => Box::new(OrderHandler),
        _ => Box::new(NotFoundHandler),
    }
}
```

## Cow — Clone on Write

```rust
use std::borrow::Cow;

// Avoid unnecessary cloning — borrow when possible, clone when needed
fn normalize_name(name: &str) -> Cow<'_, str> {
    if name.contains(char::is_uppercase) {
        Cow::Owned(name.to_lowercase())
    } else {
        Cow::Borrowed(name) // No allocation
    }
}

// Useful for function parameters that might or might not need ownership
fn process_data(data: Cow<'_, [u8]>) {
    // Can work with both borrowed and owned data
    println!("Processing {} bytes", data.len());
}

// Call with borrowed data
process_data(Cow::Borrowed(&[1, 2, 3]));
// Call with owned data
process_data(Cow::Owned(vec![1, 2, 3]));
```

## Interior Mutability

```rust
use std::cell::{Cell, RefCell};

// Cell<T> — for Copy types, zero-cost interior mutability
struct Counter {
    count: Cell<u32>,
}

impl Counter {
    fn increment(&self) { // Note: &self, not &mut self
        self.count.set(self.count.get() + 1);
    }
}

// RefCell<T> — runtime borrow checking
struct Logger {
    messages: RefCell<Vec<String>>,
}

impl Logger {
    fn log(&self, msg: &str) {
        self.messages.borrow_mut().push(msg.to_string());
    }

    fn dump(&self) -> Vec<String> {
        self.messages.borrow().clone()
    }
}

// OnceCell / OnceLock — lazy initialization
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        Config::load("config.toml").expect("failed to load config")
    })
}
```
