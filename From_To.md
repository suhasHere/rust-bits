Brief description:
- `From<T> for U` defines a fallible-free conversion from `T` into `U`. Implementing `From` automatically provides `Into<U> for T` (so you can call `.into()`).
- There is no standard `To` trait in Rust; prefer `From`/`Into` or `ToString`/`FromStr` for string conversions.
- Below: show the existing `From<ffi::QuicrStatus> for Status` (direction from FFI into your safe enum) and add the reverse `From<Status> for ffi::QuicrStatus`. Then show small usage examples taken from your `src/client.rs` usage sites.

```rust
// rust
// Conversion implementations and small usage examples for your `Status` <-> `ffi::QuicrStatus` conversions.

// Convert FFI status into your safe Status enum (you already have this pattern).
impl From<ffi::QuicrStatus> for Status {
    fn from(status: ffi::QuicrStatus) -> Self {
        match status {
            ffi::QuicrStatus_QUICR_STATUS_OK => Status::Ok,
            ffi::QuicrStatus_QUICR_STATUS_CONNECTING => Status::Connecting,
            ffi::QuicrStatus_QUICR_STATUS_READY => Status::Ready,
            ffi::QuicrStatus_QUICR_STATUS_DISCONNECTING => Status::Disconnecting,
            ffi::QuicrStatus_QUICR_STATUS_DISCONNECTED => Status::Disconnected,
            ffi::QuicrStatus_QUICR_STATUS_ERROR => Status::Error,
            ffi::QuicrStatus_QUICR_STATUS_IDLE_TIMEOUT => Status::IdleTimeout,
            ffi::QuicrStatus_QUICR_STATUS_SHUTDOWN => Status::Shutdown,
            _ => Status::Error,
        }
    }
}

// Reverse conversion: implement From<Status> for the FFI enum.
// This lets you call `.into()` on your `Status` to get `ffi::QuicrStatus`.
impl From<Status> for ffi::QuicrStatus {
    fn from(s: Status) -> ffi::QuicrStatus {
        match s {
            Status::Ok => ffi::QuicrStatus_QUICR_STATUS_OK,
            Status::Connecting => ffi::QuicrStatus_QUICR_STATUS_CONNECTING,
            Status::Ready => ffi::QuicrStatus_QUICR_STATUS_READY,
            Status::Disconnecting => ffi::QuicrStatus_QUICR_STATUS_DISCONNECTING,
            Status::Disconnected => ffi::QuicrStatus_QUICR_STATUS_DISCONNECTED,
            Status::Error => ffi::QuicrStatus_QUICR_STATUS_ERROR,
            Status::IdleTimeout => ffi::QuicrStatus_QUICR_STATUS_IDLE_TIMEOUT,
            Status::Shutdown => ffi::QuicrStatus_QUICR_STATUS_SHUTDOWN,
        }
    }
}

// Usage examples (match places in your `src/client.rs`):
fn example_from_ffi(ptr: *mut ffi::QuicrClient) -> Status {
    // the FFI call returns an ffi::QuicrStatus; use .into() or `Status::from(...)`
    let raw: ffi::QuicrStatus = unsafe { ffi::quicr_client_get_status(ptr) };
    let status: Status = raw.into(); // uses From<ffi::QuicrStatus> for Status
    status
}

fn example_into_ffi(status: Status) {
    // convert your safe Status back into the ffi enum (Into is auto-implemented)
    let ffi_status: ffi::QuicrStatus = status.into(); // uses From<Status> for ffi::QuicrStatus
    // pass ffi_status to an ffi function if needed
    let _ = ffi_status;
}
```

Notes:
- Implement `From` in the direction you consider the canonical conversion; the opposite `Into` becomes available automatically.
- Use `Status::from(x)` for explicit conversion or `let s: Status = x.into()` for type-inferred conversion.
- For string conversions use `ToString` (auto from `Display`) and `FromStr` (parsing) rather than a non-existent `To` trait.
