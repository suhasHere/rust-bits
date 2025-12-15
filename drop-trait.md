`Drop` is the Rust trait for destructors (defined as `std::ops::Drop`). 
`impl Drop for Client` provides code that runs automatically when a `Client` value 
is dropped (goes out of scope or is otherwise freed).

A short Rust example showing two uses of the `Drop` trait: 
a simple logger and an FFI-style handle that reconstructs a `Box` 
from a `NonNull` pointer to free memory.

```rust
rust
use std::ptr::NonNull;

/// Simple type that logs when dropped
struct LoggingDrop {
    name: String,
}

impl LoggingDrop {
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into() }
    }
}

impl Drop for LoggingDrop {
    fn drop(&mut self) {
        println!("Dropping LoggingDrop `{}`", self.name);
    }
}

/// FFI-style handle that owns a heap allocation via a raw pointer
struct MyData {
    value: i32,
}

struct RawHandle {
    ptr: NonNull<MyData>,
}

impl RawHandle {
    fn new(data: MyData) -> Self {
        // Move `data` into a Box and take its raw pointer
        let boxed = Box::new(data);
        let ptr = NonNull::new(Box::into_raw(boxed)).expect("Box::into_raw returned null");
        Self { ptr }
    }

    fn value(&self) -> i32 {
        // Access through raw pointer (unsafe because we must ensure validity)
        unsafe { self.ptr.as_ref().value }
    }

    fn set_value(&mut self, v: i32) {
        unsafe { self.ptr.as_mut().value = v; }
    }
}

impl Drop for RawHandle {
    fn drop(&mut self) {
        // Recreate Box to free the allocation exactly once
        unsafe {
            let _ = Box::from_raw(self.ptr.as_ptr());
            // Box is dropped here
        }
        println!("RawHandle freed the heap allocation");
    }
}

fn main() {
    {
        let _log = LoggingDrop::new("example");
        // `_log` will print when it goes out of scope
    } // "Dropping LoggingDrop `example`" printed here

    {
        let mut h = RawHandle::new(MyData { value: 10 });
        println!("value = {}", h.value()); // 10
        h.set_value(42);
        println!("value = {}", h.value()); // 42
    } // RawHandle's Drop reconstructs the Box and frees memory, printing a message
}
```


## Real world example
In this file the `drop` implementation:

- runs when a `Client` is being destroyed,
- calls `unsafe { ffi::quicr_client_destroy(self.inner.as_ptr()) }` to free the underlying C\++ resource.

Important notes:
- The `drop` method runs synchronously and cannot be `async`.
- Field drop order: the `drop` body runs while fields are still present;
  after `drop` returns, Rust will drop the fields (e.g., the boxed `ClientCallbackData`) automatically.
- Safety: ensure the C\++ side will not invoke callbacks that access `callback_data`
  after `quicr_client_destroy` returns (or clear/unregister callbacks before destroying) to
  avoid use\-after\-free. If you need async cleanup, provide an explicit async `close`/`disconnect`
  method and call it before the `Client` is dropped.
