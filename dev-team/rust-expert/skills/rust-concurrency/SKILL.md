---
name: Rust Concurrency
description: This skill should be used when the user asks about "Rust async", "Tokio", "Rust futures", "Rust channels", "Rust Send Sync", "Rust threads", "Rust async/await", "Rust spawning tasks", "Rust concurrent patterns", or "Rust Rayon". It covers async/await with Tokio, channels, thread safety traits, parallel processing, and concurrent patterns.
---

# Rust Concurrency

## Async/Await with Tokio

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Spawn concurrent tasks
    let handle1 = tokio::spawn(async {
        fetch_data("https://api.example.com/users").await
    });

    let handle2 = tokio::spawn(async {
        fetch_data("https://api.example.com/orders").await
    });

    // Await both results
    let (users, orders) = tokio::join!(handle1, handle2);
    let users = users.expect("task panicked")?;
    let orders = orders.expect("task panicked")?;
}

// Async function
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let response = reqwest::get(url).await?;
    response.text().await
}

// Select — race multiple futures
async fn fetch_with_timeout(url: &str) -> Result<String> {
    tokio::select! {
        result = reqwest::get(url) => {
            Ok(result?.text().await?)
        }
        _ = sleep(Duration::from_secs(5)) => {
            Err(anyhow::anyhow!("request timed out"))
        }
    }
}
```

## Channels

```rust
use tokio::sync::{mpsc, oneshot, broadcast, watch};

// mpsc — multiple producers, single consumer
async fn worker_pool() {
    let (tx, mut rx) = mpsc::channel::<Job>(100);

    // Spawn workers
    for _ in 0..4 {
        let tx = tx.clone();
        tokio::spawn(async move {
            // produce work
            tx.send(Job::new()).await.ok();
        });
    }
    drop(tx); // Drop original sender so rx completes

    // Consume results
    while let Some(job) = rx.recv().await {
        process(job).await;
    }
}

// oneshot — single value, single use
async fn request_response() -> Result<Response> {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let result = expensive_computation().await;
        tx.send(result).ok();
    });

    rx.await.map_err(|_| anyhow::anyhow!("channel closed"))
}

// broadcast — multiple consumers, each gets every message
async fn event_bus() {
    let (tx, _) = broadcast::channel::<Event>(100);

    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        while let Ok(event) = rx1.recv().await {
            println!("Logger: {event:?}");
        }
    });

    tokio::spawn(async move {
        while let Ok(event) = rx2.recv().await {
            println!("Metrics: {event:?}");
        }
    });

    tx.send(Event::UserCreated { id: 1 }).ok();
}

// watch — single producer, multiple consumers, only latest value
async fn config_watcher() {
    let (tx, rx) = watch::channel(Config::default());

    // Consumers always get the latest config
    let mut rx2 = rx.clone();
    tokio::spawn(async move {
        while rx2.changed().await.is_ok() {
            let config = rx2.borrow();
            println!("Config updated: {:?}", *config);
        }
    });

    tx.send(Config::load("config.toml")?).ok();
}
```

## Send and Sync Traits

```rust
// Send: safe to transfer between threads
// Sync: safe to share references between threads (&T is Send)

// Most types are Send + Sync automatically
// Notable exceptions:
// - Rc<T>: not Send, not Sync (use Arc<T> instead)
// - RefCell<T>: Send but not Sync (use Mutex<T> instead)
// - *mut T / *const T: neither Send nor Sync

// Require Send + Sync for spawned tasks
fn spawn_task<F, T>(f: F) -> tokio::task::JoinHandle<T>
where
    F: Future<Output = T> + Send + 'static,
    T: Send + 'static,
{
    tokio::spawn(f)
}

// Common error: "future is not Send"
// Usually caused by holding a non-Send type across .await
async fn problematic() {
    let data = Rc::new(vec![1, 2, 3]); // Rc is not Send!
    some_async_fn().await;               // .await here
    println!("{data:?}");                // data held across await = error
}

// Fix: use Arc, or drop before .await
async fn fixed() {
    let data = Arc::new(vec![1, 2, 3]); // Arc is Send + Sync
    some_async_fn().await;
    println!("{data:?}");                // OK
}
```

## Concurrent Collections

```rust
use dashmap::DashMap;
use tokio::sync::RwLock;

// DashMap — concurrent HashMap (no global lock)
let cache: DashMap<String, Vec<u8>> = DashMap::new();

cache.insert("key".to_string(), vec![1, 2, 3]);
if let Some(entry) = cache.get("key") {
    println!("Found: {:?}", entry.value());
}

// Tokio RwLock — async-aware read/write lock
struct AppState {
    users: RwLock<HashMap<String, User>>,
}

async fn get_user(state: &AppState, id: &str) -> Option<User> {
    state.users.read().await.get(id).cloned()
}

async fn update_user(state: &AppState, user: User) {
    state.users.write().await.insert(user.id.clone(), user);
}
```

## References

- [Async Patterns](references/async-patterns.md) — Task spawning strategies, graceful shutdown, backpressure, stream processing, async traits.
- [Parallel Processing](references/parallel-processing.md) — Rayon parallel iterators, scoped threads, crossbeam channels, work stealing, SIMD.
