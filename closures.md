# Lazy Closure

A lazy closure is a closure that is not executed until its value is needed. 
In Rust this is commonly used with methods like `ok_or_else` where the 
closure is called only when the `Option` is `None`, avoiding expensive 
work or side effects when the `Option` is `Some`. Zero-argument closures 
use `||` (e.g., `|| compute_error()`).

Example: the closure that builds the error runs only for 
`ok_or_else` when the option is `None`; `ok_or` evaluates its argument eagerly.

Brief explanation of the code: it prints when the error constructor 
runs so you can see that `ok_or_else` avoids calling it for `Some`, 
while `ok_or` calls it immediately.

```rust
// rust
fn make_err() -> String {
    println!("make_err called");
    "expensive error".to_string()
}

fn main() {
    let some: Option<i32> = Some(42);
    let none: Option<i32> = None;

    // ok_or_else: make_err is called only when option is None
    let _ = some.ok_or_else(|| make_err()); // no print
    let _ = none.ok_or_else(|| make_err()); // prints "make_err called"

    // ok_or: make_err is evaluated immediately (eager)
    let _ = some.ok_or(make_err()); // prints "make_err called" even though Some
}
```
