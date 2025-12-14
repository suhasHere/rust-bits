In the example 
```rust
impl TransportConfigBuilder {
    /// Set the TLS certificate file path
    pub fn tls_cert(mut self, path: impl Into<String>) -> Self {
        self.config.tls_cert_filename = Some(path.into());
        self
    }
}
```

`Some(..)` is the **“present value”** variant of Rust’s `Option<T>` type.

## What `Some(..)` means here
If `tls_cert_filename` has type `Option<String>`, then it can be either:

- `None` → “no TLS cert filename provided”
- `Some(value)` → “a TLS cert filename is provided, and it is `value`”

So this assignment:

```rust
self.config.tls_cert_filename = Some(path.into());
```

means:

1. Convert `path` into a `String` (`path.into()`), producing something like `"cert.pem".to_owned()`.
2. Wrap that `String` in `Some(...)`.
3. Store it into the `Option<String>` field.

Effectively: “set the optional cert filename to *be present* with this string”.

## Why wrap it in `Some`?
Because the field is optional. The `Option` wrapper forces you to be 
explicit about whether the value exists:

- default / not set yet: `tls_cert_filename = None`
- set by the builder: `tls_cert_filename = Some("...".to_string())`

Some(...) is the enum variant of Option<T> that holds a value. 
Use Some(value) to represent “there is a value”, and None to represent “no value”. 

Common patterns:
- Store an optional string: tls_cert_filename: Option<String>
- Set it: tls_cert_filename = Some(path.into())

Check/use it:
-
```rust
if let Some(path) = &cfg.tls_cert_filename { ... }
cfg.tls_cert_filename.as_ref().map(|s| ...) to operate on a borrowed &String
cfg.tls_cert_filename.unwrap_or(default) to get a value or fallback
```

Example showing the builder setter and safe FFI conversion that handles 
Option<String> and potential NUL bytes:

```rust
// Rust
impl TransportConfigBuilder {
    pub fn tls_cert(mut self, path: impl Into<String>) -> Self {
        // Store a value inside Option using Some(...)
        self.config.tls_cert_filename = Some(path.into());
        self
    }
}

// Later, when converting to C strings for FFI:
let tls_cert_cstring: Option<CString> = config
    .transport
    .tls_cert_filename
    .as_ref()                       // Option<&String>
    .map(|s| CString::new(s.as_str())) // Option<Result<CString, _>>
    .transpose()                    // Result<Option<CString>, _>
    .map_err(|_| /* handle NUL bytes error */ )?;

// Use pointer or null for the FFI struct:
let tls_cert_ptr = tls_cert_cstring
    .as_ref()
    .map(|c| c.as_ptr())
    .unwrap_or(std::ptr::null());
```
This demonstrates setting an optional value with Some(...), 
safely converting to CString while propagating errors, and passing 
either a pointer or NULL to the C side


Briefly: the expression converts an Option<String> into a 
Result<Option<CString>, _>  that fails if any present string contains a NUL byte, 
then ? unwraps the Result so you end up with Option<CString> or return an error.

Key steps:
- as_ref() turns Option<String> into Option<&String> so you don't move out of the original config.
- map(|s| CString::new(s.as_str())) turns Option<&String> into Option<Result<CString, NulError>>.
- transpose() flips that to Result<Option<CString>, NulError>.
- map_err(...) converts the NulError into your crate Error.
- ? propagates the error or yields Option<CString> on success.

Example equivalents (expanded and concise):

```rust
use std::ffi::CString;
use crate::error::Error;

/// Expanded, explicit version
fn convert_opt_string_explicit(opt: &Option<String>) -> Result<Option<CString>, Error> {
    match opt.as_ref() { // Option<&String>
        Some(s) => {
            // CString::new returns Result<CString, std::ffi::NulError>
            match CString::new(s.as_str()) {
                Ok(cstr) => Ok(Some(cstr)),
                Err(_) => Err(Error::InvalidArgument("contains null bytes".into())),
            }
        }
        None => Ok(None),
    }
}

/// Concise version doing the same thing (same logic as the fluent chain)
fn convert_opt_string_concise(opt: &Option<String>) -> Result<Option<CString>, Error> {
    opt.as_ref()                                   // Option<&String>
        .map(|s| CString::new(s.as_str()))        // Option<Result<CString, NulError>>
        .transpose()                              // Result<Option<CString>, NulError>
        .map_err(|_| Error::InvalidArgument("contains null bytes".into()))
}

```



