`ok_or_else` converts an `Option<T>` to a `Result<T, E>` by returning `Ok(value)` for `Some(value)` or calling a closure to produce the `Err` value for `None`. The closure is lazy — it runs only when needed (i.e., on `None`). This is useful when constructing the error is expensive or depends on runtime info.

Brief examples showing behavior and equivalence to a `match`:

```rust
// rust
use std::time::Instant;

fn main() {
    // Basic usage
    let maybe: Option<i32> = Some(10);
    let r: Result<i32, &'static str> = maybe.ok_or_else(|| "missing value");
    assert_eq!(r, Ok(10));

    let none: Option<i32> = None;
    let r2: Result<i32, String> = none.ok_or_else(|| format!("computed error at {:?}", Instant::now()));
    assert!(r2.is_err());

    // Demonstrate laziness vs ok_or (ok_or constructs the error eagerly)
    fn make_err() -> String {
        println!("make_err called");
        "error".to_string()
    }

    let some_val: Option<i32> = Some(1);
    // ok_or_else: make_err is not called
    let _ = some_val.ok_or_else(|| make_err());
    // ok_or: make_err is called even though value exists
    let _ = some_val.ok_or(make_err());

    // Equivalent match
    let opt: Option<&str> = None;
    let res: Result<&str, String> = match opt {
        Some(v) => Ok(v),
        None => Err("build error".to_string()),
    };
    assert!(res.is_err());
}
```

Use `ok_or_else` when you want to avoid creating the error value unless the `Option` is `None`.
