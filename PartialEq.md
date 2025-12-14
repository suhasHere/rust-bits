PartialEq is the Rust trait that enables equality comparisons with `==` and `!=`. It provides `fn eq(&self, other: &Rhs) -> bool` (with `Rhs = Self` by default). You usually get it via `#[derive(PartialEq)]`. `PartialEq` allows "partial" equality (e.g., floating-point `NaN` breaks total ordering), while `Eq` is a marker trait for types where equality is a full equivalence relation (no exceptional cases).

Examples (derive, manual impl, and float caveat):

```rust
// rust
// Derive PartialEq (field-wise equality)
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn example1() {
    let a = Point { x: 1, y: 2 };
    let b = Point { x: 1, y: 2 };
    assert!(a == b);
}

// Derive both PartialEq and Eq for types with total equality
#[derive(Debug, PartialEq, Eq)]
struct Id(u64);

// Manual PartialEq implementation (custom logic)
#[derive(Debug)]
struct AlmostEqual(f32);

impl PartialEq for AlmostEqual {
    fn eq(&self, other: &Self) -> bool {
        (self.0 - other.0).abs() < 1e-6
    }
}

// Floats implement PartialEq but not Eq because of NaN:
// You cannot derive `Eq` for a struct that contains `f32`/`f64`.
fn example_usage() {
    example1();
    let id1 = Id(42);
    let id2 = Id(42);
    assert!(id1 == id2);

    let a = AlmostEqual(0.1 + 0.2);
    let b = AlmostEqual(0.3);
    assert!(a == b); // uses our custom tolerance
}
```
