# Memory Management

## Stack vs Heap

```rust
// Stack: fixed size, fast, automatic cleanup
fn stack_allocation() {
    let x: i32 = 42;           // 4 bytes on stack
    let arr: [u8; 1024] = [0; 1024]; // 1KB on stack
    let point = (3.0_f64, 4.0_f64);  // 16 bytes on stack
}   // All automatically cleaned up

// Heap: dynamic size, slower, manual management (via ownership)
fn heap_allocation() {
    let s = String::from("hello");  // Heap-allocated string
    let v = vec![1, 2, 3];          // Heap-allocated vector
    let b = Box::new(42);           // Explicit heap allocation
}   // Drop is called, heap memory freed
```

## Box, Rc, Arc

```rust
use std::rc::Rc;
use std::sync::Arc;

// Box<T> — single ownership heap allocation
// Use for: recursive types, large data, trait objects
enum List {
    Cons(i32, Box<List>),
    Nil,
}

let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));

// Rc<T> — reference counting (single-threaded)
// Use for: shared ownership within one thread
struct TreeNode {
    value: String,
    children: Vec<Rc<TreeNode>>,
}

let shared = Rc::new(TreeNode {
    value: "root".to_string(),
    children: vec![],
});
let ref1 = Rc::clone(&shared); // Increment ref count (cheap)
let ref2 = Rc::clone(&shared);
println!("References: {}", Rc::strong_count(&shared)); // 3

// Arc<T> — atomic reference counting (thread-safe)
// Use for: shared ownership across threads
let data = Arc::new(vec![1, 2, 3]);
let handles: Vec<_> = (0..4).map(|_| {
    let data = Arc::clone(&data);
    std::thread::spawn(move || {
        println!("Sum: {}", data.iter().sum::<i32>());
    })
}).collect();

for h in handles { h.join().unwrap(); }

// Weak<T> — non-owning reference (prevents cycles)
use std::rc::Weak;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

## Drop Trait

```rust
// Custom cleanup logic
struct TempFile {
    path: std::path::PathBuf,
}

impl Drop for TempFile {
    fn drop(&mut self) {
        if let Err(e) = std::fs::remove_file(&self.path) {
            eprintln!("Failed to cleanup temp file: {e}");
        }
    }
}

// RAII guard pattern
struct MutexGuard<'a, T> {
    data: &'a mut T,
    mutex: &'a Mutex,
}

impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        self.mutex.unlock(); // Automatically release lock
    }
}

// Explicit drop
let file = TempFile { path: "temp.txt".into() };
// Use file...
drop(file); // Cleanup NOW, don't wait for scope end
```

## Memory Layout

```rust
use std::mem;

// Size and alignment
println!("i32: {} bytes", mem::size_of::<i32>());      // 4
println!("bool: {} bytes", mem::size_of::<bool>());     // 1
println!("String: {} bytes", mem::size_of::<String>()); // 24 (ptr + len + cap)
println!("Vec<u8>: {} bytes", mem::size_of::<Vec<u8>>()); // 24
println!("Option<&u8>: {} bytes", mem::size_of::<Option<&u8>>()); // 8 (niche optimization)
println!("Option<bool>: {} bytes", mem::size_of::<Option<bool>>()); // 1

// Enum size = largest variant + tag
enum Payload {
    Small(u8),           // 1 byte
    Medium([u8; 100]),   // 100 bytes
    Large(Box<[u8]>),    // 16 bytes (ptr + len)
}
// Size = 100 + tag alignment ≈ 104 bytes

// Optimize with Box for large variants
enum OptimizedPayload {
    Small(u8),
    Medium(Box<[u8; 100]>), // Now just 8 bytes (pointer)
    Large(Box<[u8]>),
}
// Size = 16 + tag ≈ 24 bytes

// repr(C) for FFI-compatible layout
#[repr(C)]
struct FfiPoint {
    x: f64,
    y: f64,
}
```

## Zero-Copy Parsing

```rust
// Parse without allocating — borrow from input
#[derive(Debug)]
struct LogEntry<'a> {
    timestamp: &'a str,
    level: &'a str,
    message: &'a str,
}

fn parse_log_line(line: &str) -> Option<LogEntry<'_>> {
    let mut parts = line.splitn(3, ' ');
    Some(LogEntry {
        timestamp: parts.next()?,
        level: parts.next()?,
        message: parts.next()?,
    })
}

// Zero-copy with bytes crate
use bytes::Bytes;

fn split_message(data: Bytes) -> (Bytes, Bytes) {
    let split_point = data.iter().position(|&b| b == b'\n').unwrap_or(data.len());
    let header = data.slice(..split_point);
    let body = data.slice(split_point + 1..);
    (header, body)  // Both share the same underlying allocation
}
```

## Arena Allocation

```rust
// Bump allocator for batch processing — allocate many, free all at once
use bumpalo::Bump;

fn process_batch(items: &[RawItem]) -> Vec<String> {
    let arena = Bump::new();

    // All allocations in the arena
    let processed: Vec<&str> = items.iter()
        .map(|item| {
            let s = arena.alloc_str(&item.transform());
            &*s
        })
        .collect();

    // Convert to owned before arena is dropped
    processed.iter().map(|s| s.to_string()).collect()
}   // Arena drops all allocations at once — very fast

// typed-arena for homogeneous allocations
use typed_arena::Arena;

fn build_ast(tokens: &[Token]) -> &AstNode {
    let arena = Arena::new();
    let root = arena.alloc(AstNode::new("root"));
    for token in tokens {
        let child = arena.alloc(AstNode::from(token));
        root.add_child(child);
    }
    root
}
```
