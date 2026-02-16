---
name: Rust Ownership
description: This skill should be used when the user asks about "Rust ownership", "Rust borrowing", "Rust lifetimes", "Rust borrow checker", "Rust move semantics", "Rust references", "Rust lifetime annotations", "Rust lifetime elision", or "Rust memory safety". It covers ownership rules, borrowing, lifetimes, and common borrow checker patterns.
---

# Rust Ownership

## Ownership Rules

```rust
// Rule 1: Each value has exactly one owner
// Rule 2: When the owner goes out of scope, the value is dropped
// Rule 3: There can be either one mutable reference OR any number of immutable references

fn ownership_basics() {
    let s1 = String::from("hello");
    let s2 = s1;                    // s1 is MOVED to s2
    // println!("{s1}");            // Compile error: s1 is no longer valid

    let s3 = s2.clone();            // Explicit deep copy
    println!("{s2} {s3}");          // Both valid

    // Copy types (integers, floats, bool, char) are copied, not moved
    let x = 5;
    let y = x;
    println!("{x} {y}");            // Both valid — i32 implements Copy
}

// Move semantics in functions
fn take_ownership(s: String) {
    println!("{s}");
}   // s is dropped here

fn borrow(s: &String) {
    println!("{s}");
}   // s goes out of scope but doesn't drop the value

fn main() {
    let name = String::from("Alice");
    borrow(&name);                   // Borrow — name still valid
    take_ownership(name);            // Move — name no longer valid
    // borrow(&name);               // Compile error: name was moved
}
```

## Borrowing Patterns

```rust
// Immutable borrowing — multiple readers
fn calculate_stats(data: &[f64]) -> (f64, f64) {
    let sum: f64 = data.iter().sum();
    let mean = sum / data.len() as f64;
    let variance = data.iter()
        .map(|x| (x - mean).powi(2))
        .sum::<f64>() / data.len() as f64;
    (mean, variance)
}

// Mutable borrowing — exclusive writer
fn sort_and_dedup(data: &mut Vec<String>) {
    data.sort();
    data.dedup();
}

// Return references tied to input lifetime
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    s
}

// Reborrowing — common pattern
fn process(data: &mut Vec<i32>) {
    // Passing &mut to another function reborrows, doesn't move
    add_defaults(&mut *data);  // Explicit reborrow
    validate(data);             // Implicit reborrow
}
```

## Lifetime Annotations

```rust
// Explicit lifetimes when compiler can't infer
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Struct with references — must have lifetime annotations
struct Excerpt<'a> {
    text: &'a str,
    page: usize,
}

impl<'a> Excerpt<'a> {
    fn text(&self) -> &str {
        self.text // Lifetime elision: output tied to &self
    }

    fn combine(&self, other: &Excerpt<'a>) -> &'a str {
        if self.text.len() > other.text.len() {
            self.text
        } else {
            other.text
        }
    }
}

// Multiple lifetimes
struct Parser<'input, 'config> {
    input: &'input str,
    config: &'config Config,
}

// 'static lifetime — lives for entire program
fn make_static() -> &'static str {
    "I live forever"  // String literals are 'static
}

// Owned data satisfies any lifetime
fn to_owned_avoids_lifetime(s: &str) -> String {
    s.to_string() // No lifetime needed — returns owned data
}
```

## Common Borrow Checker Solutions

```rust
// Problem: Cannot borrow as mutable while also borrowed as immutable
// Solution: Restructure to separate the borrows
fn update_map(map: &mut HashMap<String, Vec<i32>>, key: &str) {
    // BAD: map.get(key) borrows map, then map.insert() borrows mutably
    // let existing = map.get(key);
    // map.insert(key.to_string(), vec![]);

    // GOOD: Use entry API
    map.entry(key.to_string())
        .or_insert_with(Vec::new)
        .push(42);
}

// Problem: Self-referential struct
// Solution: Use indices instead of references
struct Arena {
    nodes: Vec<Node>,
}

struct Node {
    value: String,
    parent: Option<usize>,    // Index, not reference
    children: Vec<usize>,
}

impl Arena {
    fn add_child(&mut self, parent_idx: usize, value: String) -> usize {
        let child_idx = self.nodes.len();
        self.nodes.push(Node {
            value,
            parent: Some(parent_idx),
            children: vec![],
        });
        self.nodes[parent_idx].children.push(child_idx);
        child_idx
    }
}

// Problem: Returning reference to local variable
// Solution: Return owned data
fn create_greeting(name: &str) -> String { // Return String, not &str
    format!("Hello, {name}!")
}
```

## References

- [Lifetime Patterns](references/lifetime-patterns.md) — Lifetime elision rules, HRTB, lifetime bounds on traits, GATs, common lifetime errors and fixes.
- [Memory Management](references/memory-management.md) — Stack vs heap, Box/Rc/Arc, Drop trait, memory layout, zero-copy parsing, arena allocation.
