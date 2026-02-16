# Lifetime Patterns

## Lifetime Elision Rules

```rust
// The compiler applies three rules to infer lifetimes:
// Rule 1: Each reference parameter gets its own lifetime
// Rule 2: If there's exactly one input lifetime, it's assigned to all output lifetimes
// Rule 3: If there's a &self or &mut self, its lifetime is assigned to all outputs

// Rule 2 applies — one input, output gets same lifetime
fn first_char(s: &str) -> &str {
    &s[..1]
}
// Equivalent to: fn first_char<'a>(s: &'a str) -> &'a str

// Rule 3 applies — method on &self
impl MyStruct {
    fn name(&self) -> &str {
        &self.name
    }
    // Equivalent to: fn name<'a>(&'a self) -> &'a str
}

// Rules DON'T apply — must annotate manually
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

## Higher-Ranked Trait Bounds (HRTB)

```rust
// "for any lifetime 'a" — the function works with any lifetime
fn apply_to_str<F>(f: F) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let owned = String::from("hello world");
    f(&owned).to_string()
}

// Common with closures that borrow
fn find_matching<'a, F>(items: &'a [String], predicate: F) -> Vec<&'a str>
where
    F: Fn(&str) -> bool,  // Implicitly: for<'b> Fn(&'b str) -> bool
{
    items.iter()
        .filter(|s| predicate(s))
        .map(|s| s.as_str())
        .collect()
}
```

## Lifetime Bounds on Traits

```rust
// Trait object with lifetime bound
trait Processor {
    fn process(&self, data: &str) -> String;
}

// 'static bound — processor doesn't borrow anything
fn spawn_processor(p: Box<dyn Processor + Send + 'static>) {
    std::thread::spawn(move || {
        let result = p.process("input");
        println!("{result}");
    });
}

// Borrowed trait object — limited to lifetime 'a
fn use_processor<'a>(p: &'a dyn Processor, data: &str) -> String {
    p.process(data)
}

// Trait with lifetime in associated type
trait Parser<'input> {
    type Output;
    fn parse(&self, input: &'input str) -> Self::Output;
}

struct JsonParser;

impl<'input> Parser<'input> for JsonParser {
    type Output = &'input str; // Output borrows from input
    fn parse(&self, input: &'input str) -> Self::Output {
        &input[1..input.len()-1] // Strip quotes
    }
}
```

## Generic Associated Types (GATs)

```rust
// GATs enable lifetime-parameterized associated types
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// Implementation that returns references to its own data
struct WindowIter<'data> {
    data: &'data [u8],
    pos: usize,
    window_size: usize,
}

impl<'data> LendingIterator for WindowIter<'data> {
    type Item<'a> = &'a [u8] where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.pos + self.window_size > self.data.len() {
            return None;
        }
        let window = &self.data[self.pos..self.pos + self.window_size];
        self.pos += 1;
        Some(window)
    }
}
```

## Common Lifetime Errors and Fixes

```rust
// ERROR: Returns reference to temporary
// fn bad() -> &str { &String::from("hello") }
// FIX: Return owned data
fn good() -> String { String::from("hello") }

// ERROR: Conflicting lifetimes in struct
// struct Bad<'a, 'b> { x: &'a str, y: &'b str }
// fn use_bad<'a, 'b>(b: &Bad<'a, 'b>) -> &'a str { b.y }  // Wrong: y is 'b
// FIX: Use correct lifetime or unify
struct Fixed<'a> { x: &'a str, y: &'a str }

// ERROR: Borrowed value does not live long enough
fn dangling_ref() {
    let r;
    {
        let x = 5;
        r = &x;  // Error: x dropped at end of block
    }
    // FIX: Ensure referenced data outlives the reference
    let x = 5;
    let r = &x;  // OK: x lives as long as r
    println!("{r}");
}

// ERROR: Cannot return value referencing local variable
// FIX: Use owned types or restructure
struct Config {
    name: String,  // Own the data instead of borrowing
}

impl Config {
    fn name(&self) -> &str {
        &self.name  // Return reference tied to &self
    }
}
```

## Subtyping and Variance

```rust
// Longer lifetimes are subtypes of shorter lifetimes
// 'static is a subtype of all lifetimes

fn covariant_example<'long, 'short>(long: &'long str) -> &'short str
where
    'long: 'short, // 'long outlives 'short
{
    long // A &'long str can be used where &'short str is expected
}

// &T is covariant in T and in its lifetime
// &mut T is invariant in T (cannot substitute subtypes)
// This prevents:
fn invariant_reason() {
    let mut s = String::from("hello");
    let r: &mut String = &mut s;
    // If &mut T were covariant, you could:
    // let r2: &mut dyn Display = r;  // Upcast
    // *r2 = some_other_display_type;  // Violate type safety
}
```
