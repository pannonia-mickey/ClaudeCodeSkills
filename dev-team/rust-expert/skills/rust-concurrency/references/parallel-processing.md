# Parallel Processing

## Rayon Parallel Iterators

```rust
use rayon::prelude::*;

// Drop-in parallel iteration
let sum: i64 = numbers.par_iter().sum();

let results: Vec<Output> = items
    .par_iter()
    .filter(|item| item.is_valid())
    .map(|item| expensive_transform(item))
    .collect();

// Parallel sort
let mut data = vec![5, 3, 1, 4, 2];
data.par_sort();
data.par_sort_by(|a, b| b.cmp(a)); // Reverse

// Parallel fold + reduce
let word_count: HashMap<String, usize> = documents
    .par_iter()
    .fold(HashMap::new, |mut map, doc| {
        for word in doc.split_whitespace() {
            *map.entry(word.to_string()).or_default() += 1;
        }
        map
    })
    .reduce(HashMap::new, |mut a, b| {
        for (k, v) in b {
            *a.entry(k).or_default() += v;
        }
        a
    });

// Configure thread pool
rayon::ThreadPoolBuilder::new()
    .num_threads(8)
    .build_global()
    .unwrap();

// Scoped thread pool (bounded lifetime)
let pool = rayon::ThreadPoolBuilder::new()
    .num_threads(4)
    .build()
    .unwrap();

pool.install(|| {
    data.par_iter().for_each(|item| process(item));
});
```

## Scoped Threads (std)

```rust
use std::thread;

// Scoped threads can borrow from the enclosing scope
fn process_chunks(data: &mut [u32]) {
    let chunk_size = data.len() / 4;

    thread::scope(|s| {
        for chunk in data.chunks_mut(chunk_size) {
            s.spawn(|| {
                for item in chunk.iter_mut() {
                    *item = expensive_computation(*item);
                }
            });
        }
    }); // All spawned threads are joined here automatically

    // data is now processed in parallel
}
```

## Crossbeam Channels

```rust
use crossbeam_channel::{bounded, select, unbounded, Receiver, Sender};

// Bounded channel with backpressure
fn pipeline() {
    let (tx, rx) = bounded::<Job>(100);

    // Multiple producers
    for i in 0..4 {
        let tx = tx.clone();
        thread::spawn(move || {
            for job in generate_jobs(i) {
                tx.send(job).unwrap(); // Blocks when full
            }
        });
    }
    drop(tx);

    // Single consumer
    for job in rx {
        process(job);
    }
}

// Select across multiple channels
fn multiplexer(rx1: Receiver<Event>, rx2: Receiver<Event>, quit: Receiver<()>) {
    loop {
        select! {
            recv(rx1) -> msg => {
                if let Ok(event) = msg {
                    handle_event(event);
                }
            }
            recv(rx2) -> msg => {
                if let Ok(event) = msg {
                    handle_event(event);
                }
            }
            recv(quit) -> _ => {
                println!("Shutting down");
                return;
            }
        }
    }
}

// Tick channel for periodic work
use crossbeam_channel::tick;

let ticker = tick(Duration::from_secs(1));
let timeout = crossbeam_channel::after(Duration::from_secs(30));

loop {
    select! {
        recv(ticker) -> _ => {
            do_periodic_work();
        }
        recv(timeout) -> _ => {
            println!("Timeout reached");
            break;
        }
    }
}
```

## Work Stealing

```rust
// Rayon's work-stealing is automatic, but you can customize
use rayon::prelude::*;

// Recursive divide-and-conquer with automatic work stealing
fn parallel_quicksort<T: Ord + Send>(data: &mut [T]) {
    if data.len() <= 32 {
        data.sort(); // Sequential for small inputs
        return;
    }

    let pivot = partition(data);
    let (left, right) = data.split_at_mut(pivot);
    rayon::join(
        || parallel_quicksort(left),
        || parallel_quicksort(right),
    );
}

// Custom join threshold
fn parallel_merge_sort<T: Ord + Send + Clone>(data: &mut [T]) {
    const THRESHOLD: usize = 1024;

    if data.len() <= THRESHOLD {
        data.sort();
        return;
    }

    let mid = data.len() / 2;
    let (left, right) = data.split_at_mut(mid);

    rayon::join(
        || parallel_merge_sort(left),
        || parallel_merge_sort(right),
    );

    merge(data, mid);
}
```

## Thread-Safe Data Structures

```rust
use crossbeam_queue::ArrayQueue;
use crossbeam_skiplist::SkipMap;

// Lock-free bounded queue
let queue = ArrayQueue::new(1000);
queue.push(42).ok();
let val = queue.pop();

// Lock-free concurrent sorted map
let map = SkipMap::new();
map.insert("key", "value");
if let Some(entry) = map.get("key") {
    println!("{}: {}", entry.key(), entry.value());
}

// Atomic types for simple shared state
use std::sync::atomic::{AtomicU64, Ordering};

struct Metrics {
    requests: AtomicU64,
    errors: AtomicU64,
}

impl Metrics {
    fn record_request(&self) {
        self.requests.fetch_add(1, Ordering::Relaxed);
    }

    fn record_error(&self) {
        self.errors.fetch_add(1, Ordering::Relaxed);
    }

    fn snapshot(&self) -> (u64, u64) {
        (
            self.requests.load(Ordering::Relaxed),
            self.errors.load(Ordering::Relaxed),
        )
    }
}
```

## CPU-Bound vs IO-Bound Strategy

```rust
// CPU-bound: Use Rayon for data parallelism
fn process_images(images: Vec<Image>) -> Vec<Thumbnail> {
    images.par_iter()
        .map(|img| generate_thumbnail(img))
        .collect()
}

// IO-bound: Use Tokio for async concurrency
async fn fetch_all(urls: Vec<String>) -> Vec<Result<Response>> {
    let futures: Vec<_> = urls.into_iter()
        .map(|url| reqwest::get(url))
        .collect();
    futures::future::join_all(futures).await
}

// Mixed: Use Tokio + spawn_blocking for CPU work in async context
async fn process_upload(data: Vec<u8>) -> Result<String> {
    // IO: receive file
    let file_data = receive_file(data).await?;

    // CPU: process on blocking thread
    let result = tokio::task::spawn_blocking(move || {
        compress_and_encrypt(file_data)
    }).await?;

    // IO: store result
    store_result(result).await
}
```
