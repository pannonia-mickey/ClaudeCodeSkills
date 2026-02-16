# Property Testing

## proptest Strategies

```rust
use proptest::prelude::*;

proptest! {
    // Basic property test
    #[test]
    fn test_sort_preserves_length(ref v in prop::collection::vec(any::<i32>(), 0..100)) {
        let mut sorted = v.clone();
        sorted.sort();
        prop_assert_eq!(v.len(), sorted.len());
    }

    // Custom strategy
    #[test]
    fn test_parse_format_roundtrip(amount in 0i64..=999_999_999i64) {
        let formatted = format_cents(amount);
        let parsed = parse_cents(&formatted).unwrap();
        prop_assert_eq!(amount, parsed);
    }

    // String strategies
    #[test]
    fn test_email_validation(
        local in "[a-z]{1,20}",
        domain in "[a-z]{1,10}\\.[a-z]{2,4}",
    ) {
        let email = format!("{local}@{domain}");
        prop_assert!(is_valid_email(&email));
    }
}

// Custom strategy composition
fn valid_user_strategy() -> impl Strategy<Value = User> {
    (
        "[a-zA-Z]{2,50}",                    // name
        "[a-z]{5,15}@[a-z]{5,10}\\.com",     // email
        18u32..=120,                           // age
        prop::sample::select(vec!["user", "admin", "mod"]), // role
    ).prop_map(|(name, email, age, role)| User {
        id: uuid::Uuid::new_v4().to_string(),
        name,
        email,
        age,
        role: role.to_string(),
    })
}

proptest! {
    #[test]
    fn test_user_serialization(user in valid_user_strategy()) {
        let json = serde_json::to_string(&user).unwrap();
        let deserialized: User = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(user.name, deserialized.name);
        prop_assert_eq!(user.email, deserialized.email);
    }
}
```

## Shrinking

```rust
// proptest automatically shrinks failing inputs to minimal examples
// The shrunk example is saved to proptest-regressions/ for replay

proptest! {
    #[test]
    fn test_no_overflow(a in 0i32..1000, b in 0i32..1000) {
        // If this fails for (500, 600), proptest will shrink to find
        // the smallest (a, b) that still fails
        let result = multiply(a, b);
        prop_assert!(result >= 0);
    }
}

// Custom shrinking with prop_filter
proptest! {
    #[test]
    fn test_with_constraints(
        v in prop::collection::vec(1i32..100, 1..50)
            .prop_filter("sum must be > 10", |v| v.iter().sum::<i32>() > 10)
    ) {
        let result = process(&v);
        prop_assert!(result.is_ok());
    }
}
```

## Stateful Testing

```rust
use proptest::prelude::*;
use proptest_state_machine::{prop_state_machine, ReferenceStateMachine, StateMachineTest};

// Test a data structure against a reference implementation
#[derive(Debug, Clone)]
enum Op {
    Insert(String, i32),
    Remove(String),
    Get(String),
}

fn op_strategy() -> impl Strategy<Value = Op> {
    prop_oneof![
        ("[a-z]{1,5}", any::<i32>()).prop_map(|(k, v)| Op::Insert(k, v)),
        "[a-z]{1,5}".prop_map(Op::Remove),
        "[a-z]{1,5}".prop_map(Op::Get),
    ]
}

proptest! {
    #[test]
    fn test_map_matches_reference(ops in prop::collection::vec(op_strategy(), 0..100)) {
        let mut actual = MyCustomMap::new();
        let mut reference = std::collections::HashMap::new();

        for op in ops {
            match op {
                Op::Insert(k, v) => {
                    actual.insert(k.clone(), v);
                    reference.insert(k, v);
                }
                Op::Remove(k) => {
                    actual.remove(&k);
                    reference.remove(&k);
                }
                Op::Get(k) => {
                    prop_assert_eq!(actual.get(&k), reference.get(&k));
                }
            }
        }
    }
}
```

## Fuzzing with cargo-fuzz

```rust
// fuzz/fuzz_targets/parse_input.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(input) = std::str::from_utf8(data) {
        // Should never panic regardless of input
        let _ = parse_message(input);
    }
});

// With arbitrary for structured fuzzing
use arbitrary::Arbitrary;

#[derive(Arbitrary, Debug)]
struct FuzzInput {
    header: String,
    body: Vec<u8>,
    flags: u8,
}

fuzz_target!(|input: FuzzInput| {
    let _ = process_message(&input.header, &input.body, input.flags);
});
```

```bash
# Setup and run fuzzer
cargo install cargo-fuzz
cargo fuzz init
cargo fuzz add parse_input

# Run fuzzer
cargo +nightly fuzz run parse_input

# Run with timeout and max length
cargo +nightly fuzz run parse_input -- -max_total_time=300 -max_len=4096

# Minimize corpus
cargo +nightly fuzz cmin parse_input
```
