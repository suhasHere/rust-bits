# matches!

```rust
pub fn is_ready(&self) -> bool {
        matches!(self, Status::Ready | Status::Ok)
    }
```

The expression uses the `matches!` macro to test whether the 
current `Status` value is the `Connecting` variant and returns a `bool`. 
It expands to a `match` that returns `true` for that variant and 
`false` otherwise; pattern matching ergonomics let you omit manual 
dereferencing in methods that take `&self`.

Example (equivalent forms):

```rust
rust
impl Status {
    // concise: dereference explicitly
    pub fn is_connecting(&self) -> bool {
        matches!(*self, Status::Connecting)
    }

    // explicit expansion of what `matches!` does
    pub fn is_connecting_explicit(&self) -> bool {
        match *self {
            Status::Connecting => true,
            _ => false,
        }
    }
}
```
