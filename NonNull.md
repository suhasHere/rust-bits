`std::ptr::NonNull<T>` is a small wrapper around a raw non-null pointer (`*mut T`) that encodes the invariant "this pointer is never null". Key points:

- Guarantees non-null: you cannot create a `NonNull<T>` from a null pointer with `NonNull::new` (it returns `None`) and `NonNull::new_unchecked` is `unsafe`.
- Safe to copy and cheap to pass around (it's `Copy`/`Clone`).
- `as_ptr()` is safe and returns the raw `*mut T`.
- Dereferencing the pointer (getting `&T` or `&mut T`) requires `unsafe` (e.g., `unsafe { nn.as_ref() }` or `unsafe { &*nn.as_ptr() }`).
- Useful for FFI and low-level data structures; `Option<NonNull<T>>` is pointer-size optimized (no extra discriminant).
- Use `NonNull` when you need a non-null raw pointer and want to document/encode that guarantee.

Example (creates a `NonNull` from an FFI-returned pointer and uses it safely):

Brief explanation: create a `NonNull` from a raw pointer returned by FFI, check for null, then call into the raw pointer or dereference it inside an `unsafe` block.

```rust
use std::ptr::NonNull;

fn use_ffi_client(raw: *mut ffi::QuicrClient) {
    // Convert to NonNull safely (returns None if raw was null)
    let inner = NonNull::new(raw).expect("FFI returned null pointer");

    // Get back the raw pointer (safe)
    let raw_again: *mut ffi::QuicrClient = inner.as_ptr();

    // Dereference to a Rust reference (unsafe — you must ensure validity)
    unsafe {
        let client_ref: &ffi::QuicrClient = inner.as_ref();
        // use `client_ref`...
        let _ = raw_again; // or pass to FFI calls
    }
}
```

Short checklist when using `NonNull`:
- Use `NonNull::new` when you can check null safely.
- Use `NonNull::new_unchecked` only when you can guarantee non-null.
- Always perform pointer dereference in `unsafe` blocks and ensure proper aliasing/lifetime rules.
