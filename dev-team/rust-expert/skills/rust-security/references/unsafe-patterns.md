# Unsafe Patterns

## Safe Abstractions Over Unsafe

```rust
// Pattern: Public safe API wrapping private unsafe implementation

/// A stack-allocated fixed-size ring buffer.
pub struct RingBuffer<T, const N: usize> {
    data: [std::mem::MaybeUninit<T>; N],
    head: usize,
    len: usize,
}

impl<T, const N: usize> RingBuffer<T, N> {
    pub fn new() -> Self {
        Self {
            // SAFETY: MaybeUninit doesn't require initialization
            data: unsafe { std::mem::MaybeUninit::uninit().assume_init() },
            head: 0,
            len: 0,
        }
    }

    pub fn push(&mut self, value: T) -> Option<T> {
        let old = if self.len == N {
            // SAFETY: Buffer is full, so index is initialized
            let old = unsafe { self.data[self.head].assume_init_read() };
            Some(old)
        } else {
            self.len += 1;
            None
        };

        let idx = (self.head + self.len - 1) % N;
        self.data[idx] = std::mem::MaybeUninit::new(value);

        if old.is_some() {
            self.head = (self.head + 1) % N;
        }
        old
    }

    pub fn len(&self) -> usize {
        self.len
    }
}

impl<T, const N: usize> Drop for RingBuffer<T, N> {
    fn drop(&mut self) {
        for i in 0..self.len {
            let idx = (self.head + i) % N;
            // SAFETY: We only drop initialized elements
            unsafe { self.data[idx].assume_init_drop() };
        }
    }
}
```

## Pin and Unpin

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Self-referential struct requires Pin
struct SelfReferential {
    data: String,
    ptr: *const String,
    _pin: PhantomPinned,
}

impl SelfReferential {
    fn new(data: String) -> Pin<Box<Self>> {
        let mut boxed = Box::new(Self {
            data,
            ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });

        let ptr = &boxed.data as *const String;
        // SAFETY: We're initializing the self-reference before anyone can observe it
        unsafe {
            let mut_ref = Pin::as_mut(&mut Pin::new_unchecked(&mut *boxed));
            Pin::get_unchecked_mut(mut_ref).ptr = ptr;
        }

        // SAFETY: We never move the data after this point
        unsafe { Pin::new_unchecked(boxed) }
    }

    fn data(self: Pin<&Self>) -> &str {
        &self.data
    }
}

// Most async futures are !Unpin (self-referential)
// Pin ensures they stay at their memory location
async fn example() {
    let future = async {
        let data = vec![1, 2, 3];
        some_async_fn().await;
        println!("{data:?}"); // data must not move across await
    };
    // future is automatically pinned by the runtime
}
```

## Raw Pointers

```rust
// Creating raw pointers is safe; dereferencing is unsafe
fn raw_pointer_basics() {
    let mut value = 42;

    let r1 = &value as *const i32;     // Immutable raw pointer
    let r2 = &mut value as *mut i32;   // Mutable raw pointer

    // SAFETY: Pointers are valid, aligned, and point to initialized data
    unsafe {
        println!("r1: {}", *r1);
        *r2 = 100;
    }
}

// Implementing a simple linked list with raw pointers
struct LinkedList<T> {
    head: *mut Node<T>,
    len: usize,
}

struct Node<T> {
    data: T,
    next: *mut Node<T>,
}

impl<T> LinkedList<T> {
    fn new() -> Self {
        Self { head: std::ptr::null_mut(), len: 0 }
    }

    fn push_front(&mut self, data: T) {
        let node = Box::into_raw(Box::new(Node {
            data,
            next: self.head,
        }));
        self.head = node;
        self.len += 1;
    }

    fn pop_front(&mut self) -> Option<T> {
        if self.head.is_null() {
            return None;
        }
        // SAFETY: head is non-null and was allocated by Box
        unsafe {
            let node = Box::from_raw(self.head);
            self.head = node.next;
            self.len -= 1;
            Some(node.data)
        }
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

## Transmute — Last Resort

```rust
// transmute reinterprets bits — extremely dangerous
// Always prefer safe alternatives

// AVOID: std::mem::transmute
// PREFER: TryFrom, From, as, to_ne_bytes/from_ne_bytes

// Safe alternative: bytemuck for plain-old-data types
use bytemuck::{Pod, Zeroable};

#[derive(Copy, Clone, Pod, Zeroable)]
#[repr(C)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
}

fn bytes_to_colors(bytes: &[u8]) -> &[Color] {
    bytemuck::cast_slice(bytes) // Safe, checked at compile time
}

// When transmute is truly needed (rare)
fn bool_to_u8(b: bool) -> u8 {
    // SAFETY: bool is guaranteed to be 0 or 1
    unsafe { std::mem::transmute(b) }
}
// Better: just use `b as u8`
```

## Global Mutable State

```rust
use std::sync::{OnceLock, Mutex, RwLock};

// OnceLock for lazy initialization (safe, no unsafe needed)
static CONFIG: OnceLock<AppConfig> = OnceLock::new();

fn init_config(config: AppConfig) {
    CONFIG.set(config).expect("config already initialized");
}

fn get_config() -> &'static AppConfig {
    CONFIG.get().expect("config not initialized")
}

// Mutex for mutable global state
static REGISTRY: OnceLock<Mutex<Vec<String>>> = OnceLock::new();

fn register(name: String) {
    REGISTRY
        .get_or_init(|| Mutex::new(Vec::new()))
        .lock()
        .unwrap()
        .push(name);
}

// RwLock for read-heavy global state
static CACHE: OnceLock<RwLock<HashMap<String, String>>> = OnceLock::new();

fn cache_get(key: &str) -> Option<String> {
    CACHE
        .get_or_init(|| RwLock::new(HashMap::new()))
        .read()
        .unwrap()
        .get(key)
        .cloned()
}
```
