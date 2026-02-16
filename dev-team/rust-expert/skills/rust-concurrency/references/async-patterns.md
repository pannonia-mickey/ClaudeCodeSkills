# Async Patterns

## Task Spawning Strategies

```rust
use tokio::task;

// Spawn on the Tokio runtime — requires Send + 'static
let handle = tokio::spawn(async {
    expensive_async_work().await
});
let result = handle.await?;

// spawn_blocking — run CPU-intensive work on blocking thread pool
let hash = task::spawn_blocking(move || {
    compute_hash(&large_data) // Blocking operation
}).await?;

// spawn_local — for !Send futures (single-threaded runtime)
let local = task::LocalSet::new();
local.run_until(async {
    task::spawn_local(async {
        // Can use Rc, RefCell, etc.
        let data = Rc::new(vec![1, 2, 3]);
        process(data).await;
    }).await.unwrap();
}).await;

// JoinSet — manage a dynamic set of tasks
use tokio::task::JoinSet;

let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move {
        fetch(url).await
    });
}

let mut results = Vec::new();
while let Some(result) = set.join_next().await {
    results.push(result??);
}
```

## Graceful Shutdown

```rust
use tokio::signal;
use tokio_util::sync::CancellationToken;

async fn run_server() -> Result<()> {
    let token = CancellationToken::new();

    // Spawn background workers
    let worker_token = token.clone();
    let worker = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = worker_token.cancelled() => {
                    tracing::info!("Worker shutting down");
                    break;
                }
                _ = do_periodic_work() => {}
            }
        }
    });

    // Start HTTP server
    let server_token = token.clone();
    let server = tokio::spawn(async move {
        let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
        loop {
            tokio::select! {
                _ = server_token.cancelled() => break,
                result = listener.accept() => {
                    let (stream, _) = result?;
                    tokio::spawn(handle_connection(stream));
                }
            }
        }
        Ok::<_, anyhow::Error>(())
    });

    // Wait for shutdown signal
    signal::ctrl_c().await?;
    tracing::info!("Shutdown signal received");
    token.cancel();

    // Wait for tasks to finish (with timeout)
    let _ = tokio::time::timeout(
        Duration::from_secs(30),
        futures::future::join_all([worker, server]),
    ).await;

    tracing::info!("Shutdown complete");
    Ok(())
}
```

## Backpressure

```rust
use tokio::sync::{mpsc, Semaphore};

// Bounded channels provide natural backpressure
async fn pipeline() {
    let (tx, mut rx) = mpsc::channel::<Job>(100); // Buffer of 100

    // Producer blocks when channel is full
    tokio::spawn(async move {
        for job in generate_jobs() {
            tx.send(job).await.expect("receiver dropped");
            // Blocks here if channel has 100 items
        }
    });

    // Consumer
    while let Some(job) = rx.recv().await {
        process(job).await;
    }
}

// Semaphore for fine-grained concurrency control
async fn bounded_processing(items: Vec<Item>) -> Vec<Result<Output>> {
    let semaphore = Arc::new(Semaphore::new(10)); // Max 10 concurrent
    let mut handles = Vec::new();

    for item in items {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        handles.push(tokio::spawn(async move {
            let result = process_item(item).await;
            drop(permit); // Release when done
            result
        }));
    }

    futures::future::join_all(handles).await
        .into_iter()
        .map(|r| r.unwrap())
        .collect()
}
```

## Stream Processing

```rust
use tokio_stream::{StreamExt, wrappers::ReceiverStream};
use futures::stream;

// Process items as a stream
async fn stream_processing(items: Vec<String>) -> Vec<Result<Output>> {
    stream::iter(items)
        .map(|item| async move {
            process_item(item).await
        })
        .buffer_unordered(10) // Process up to 10 concurrently
        .collect()
        .await
}

// Convert channel to stream
async fn channel_stream() {
    let (tx, rx) = mpsc::channel(100);
    let mut stream = ReceiverStream::new(rx);

    tokio::spawn(async move {
        for i in 0..100 {
            tx.send(i).await.ok();
        }
    });

    while let Some(value) = stream.next().await {
        println!("Got: {value}");
    }
}

// Chunked processing
async fn batch_process(mut rx: mpsc::Receiver<Event>) {
    let mut batch = Vec::with_capacity(100);

    loop {
        tokio::select! {
            Some(event) = rx.recv() => {
                batch.push(event);
                if batch.len() >= 100 {
                    flush_batch(&mut batch).await;
                }
            }
            _ = tokio::time::sleep(Duration::from_secs(5)) => {
                if !batch.is_empty() {
                    flush_batch(&mut batch).await;
                }
            }
        }
    }
}
```

## Async Traits

```rust
// Use async-trait crate (or native async fn in traits on nightly / Rust 1.75+)
use async_trait::async_trait;

#[async_trait]
trait Repository: Send + Sync {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>>;
    async fn save(&self, user: &User) -> Result<()>;
}

#[async_trait]
impl Repository for PostgresRepo {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>> {
        sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
            .fetch_optional(&self.pool)
            .await
            .map_err(Into::into)
    }

    async fn save(&self, user: &User) -> Result<()> {
        sqlx::query!(
            "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)
             ON CONFLICT (id) DO UPDATE SET name = $2, email = $3",
            user.id, user.name, user.email
        )
        .execute(&self.pool)
        .await?;
        Ok(())
    }
}

// Native async fn in traits (Rust 1.75+)
trait SimpleService {
    async fn process(&self, input: &str) -> String;
}
// Note: native async trait methods are not dyn-compatible yet
```

## Retry with Backoff

```rust
use tokio::time::{sleep, Duration};

async fn retry_with_backoff<F, Fut, T, E>(
    max_retries: u32,
    base_delay: Duration,
    mut f: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut attempt = 0;
    loop {
        match f().await {
            Ok(value) => return Ok(value),
            Err(e) if attempt < max_retries => {
                attempt += 1;
                let delay = base_delay * 2u32.pow(attempt - 1);
                let jitter = Duration::from_millis(rand::random::<u64>() % 1000);
                tracing::warn!("Attempt {attempt} failed: {e}, retrying in {:?}", delay + jitter);
                sleep(delay + jitter).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```
