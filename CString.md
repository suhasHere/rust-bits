`as_ptr()` returns a raw C pointer to the NUL-terminated bytes owned by a `CString`. 
The pointer type is `*const std::os::raw::c_char` and is valid only while the `CString` 
is alive and not moved/dropped. Use `as_ptr()` when you need to pass a stable C string to FFI 
but keep the `CString` in Rust to manage its lifetime; if C must take ownership, 
use `CString::into_raw()` and later `CString::from_raw()` to reclaim.

Example showing safe usage and the `Option` pattern:

```rust
use std::ffi::CString;
use std::os::raw::c_char;
use std::ptr;

fn main() {
    // Create a CString (NUL-terminated)
    let s = CString::new("hello").unwrap();

    // Get a C pointer; valid while `s` is alive
    let p: *const c_char = s.as_ptr();
    assert!(!p.is_null());

    // Keep the CString alive if you put the pointer into an FFI struct
    let holder = FfiHolder { s };

    // Example with Option<CString> -> pointer or null
    let maybe = Some(CString::new("optional").unwrap());
    let opt_ptr = maybe
        .as_ref()
        .map(|c| c.as_ptr())
        .unwrap_or(ptr::null());
    assert!(!opt_ptr.is_null());
}

struct FfiHolder {
    s: CString, // keeps the bytes alive for FFI consumers
}

impl FfiHolder {
    fn ptr(&self) -> *const c_char {
        self.s.as_ptr()
    }
}
```


- `unwrap`:
  - On `Result<T, E>`: returns the `T` inside `Ok(T)` or panics with the `E` if `Err`.
  - On `Option<T>`: returns the `T` inside `Some(T)` or panics if `None`.
  - Use only when you are sure the value is present (tests, quick examples). Otherwise prefer error handling.

- `unwrap_or(default)`:
  - On `Result<T, E>`: returns the `T` on `Ok(T)` or the provided `default` if `Err`.
  - On `Option<T>`: returns the `T` on `Some(T)` or the provided `default` if `None`.
  - Does not panic; supplies a fallback value.

Example showing both usages and the FFI-style pointer pattern:

```rust
fn main() {
    // Option examples
    let some: Option<i32> = Some(10);
    let a = some.unwrap(); // returns 10

    let none: Option<i32> = None;
    let b = none.unwrap_or(0); // returns 0 (no panic)

    // Result examples
    let ok: Result<&str, &str> = Ok("ok");
    let r1 = ok.unwrap(); // "ok"

    let err: Result<&str, &str> = Err("bad");
    let r2 = err.unwrap_or("default"); // "default" (no panic)

    // FFI-style pointer example
    use std::ffi::CString;
    use std::ptr;
    let s = CString::new("hello").unwrap(); // panics if string contains NUL
    let p_nonnull = s.as_ptr();

    let maybe_c: Option<CString> = None;
    let p = maybe_c.as_ref().map(|c| c.as_ptr()).unwrap_or(ptr::null());
    // p is NULL when `maybe_c` is None, safe fallback for FFI
}
```


When C needs to take ownership of a Rust `CString`, call `CString::into_raw()` to hand a `*mut c_char` to C. Later, when C is done, call a Rust-provided destructor that does `CString::from_raw(ptr)` exactly once to reclaim and drop the string (restoring Rust ownership). Do not call `from_raw` more than once and do not call the system `free` on the pointer unless allocator compatibility is guaranteed.

```rust
// rust
use std::ffi::CString;
use std::os::raw::c_char;

/// Create a heap-allocated C string and transfer ownership to C.
/// Caller (C) must call `quicr_string_free` when done.
#[no_mangle]
pub extern "C" fn quicr_string_new() -> *mut c_char {
    // Build a CString (checks for interior NULs)
    let s = CString::new("hello from Rust").expect("no null bytes");
    // Transfer ownership to C; Rust will no longer drop `s`.
    s.into_raw()
}

/// Reclaim ownership from C and drop the CString, freeing memory.
/// Must be called exactly once for each pointer returned by `quicr_string_new`.
#[no_mangle]
pub extern "C" fn quicr_string_free(ptr: *mut c_char) {
    if ptr.is_null() {
        return;
    }
    // SAFETY: `from_raw` assumes pointer was produced by `CString::into_raw`
    // and hasn't been reclaimed yet. It will take ownership and drop the CString.
    unsafe {
        let _ = CString::from_raw(ptr);
        // `_` is dropped here, freeing the allocation.
    }
}

/*
C usage example (in a separate C file):

#include <stdio.h>

// signatures provided by Rust library
extern char* quicr_string_new(void);
extern void quicr_string_free(char*);

int main(void) {
    char* s = quicr_string_new();
    if (s) {
        printf("Rust says: %s\n", s);
        // When done, tell Rust to reclaim and free the string:
        quicr_string_free(s);
    }
    return 0;
}

Notes/warnings:
- Only call `quicr_string_free` (which does `CString::from_raw`) once per pointer.
- Do NOT call the C `free()` on the pointer unless you know the allocators are compatible.
- Do not use the pointer after calling `quicr_string_free`.
*/
```
