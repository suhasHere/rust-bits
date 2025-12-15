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

NonNull guarantees the pointer is never null (so `Option<NonNull<T>>` is pointer-sized), and exposes safe conversions to/from raw pointers. Dereferencing or converting back to references is unsafe because you must ensure the pointer is valid and properly aligned.

Example: create a `NonNull` from a `Box`, store it in a struct, access the value, and free it in `Drop`.

```rust
rust
use std::ptr::NonNull;

struct MyData {
    value: i32,
}

// A handle that stores a non-null raw pointer to MyData
struct Handle {
    ptr: NonNull<MyData>,
}

impl Handle {
    // Create a Handle from an owned Box<MyData>
    fn new(data: MyData) -> Self {
        let boxed = Box::new(data);
        // Box::into_raw returns a non-null *mut T; wrap it in NonNull
        let ptr = NonNull::new(Box::into_raw(boxed)).expect("Box returned null (impossible)");
        Self { ptr }
    }

    // Read the value (unsafe because we rely on the pointer being valid)
    fn value(&self) -> i32 {
        unsafe { self.ptr.as_ref().value }
    }

    // Mutate the value (unsafe for the same reason)
    fn set_value(&mut self, v: i32) {
        unsafe { self.ptr.as_mut().value = v; }
    }
}

impl Drop for Handle {
    fn drop(&mut self) {
        // Recreate the Box to drop the allocation exactly once
        unsafe {
            let _ = Box::from_raw(self.ptr.as_ptr());
            // Box is dropped here, freeing the allocation
        }
    }
}

fn main() {
    let mut h = Handle::new(MyData { value: 10 });
    println!("value = {}", h.value()); // 10
    h.set_value(42);
    println!("value = {}", h.value()); // 42
    // When `h` goes out of scope, Drop reconstructs Box and frees memory
}
```
