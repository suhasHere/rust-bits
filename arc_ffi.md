Short explanation:

- Show basic `Arc` usage (share read-only and shared mutable via `Mutex`).
- Show safe patterns when crossing the FFI boundary: passing ownership to C with
  `Box::into_raw`, and sharing `Arc` pointers with `Arc::into_raw` / `Arc::from_raw` in
  callbacks while avoiding accidental drops.
- Show how to clean up/restore ownership when unregistering callbacks.

```rust
// rust
use std::ffi::c_void;
use std::sync::{Arc, Mutex, atomic::{AtomicUsize, Ordering}};
use std::thread;
use std::time::Duration;

/// -------------------------
/// Simple Arc examples
/// -------------------------
fn arc_read_only_example() {
    let data = Arc::new("hello Arc".to_string());
    let mut handles = Vec::new();
    for i in 0..4 {
        let d = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            println!("thread {} sees: {}", i, d);
        }));
    }
    for h in handles { h.join().unwrap(); }
}

fn arc_mutex_example() {
    let counter = Arc::new(Mutex::new(0usize));
    let mut handles = Vec::new();
    for _ in 0..4 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..100 {
                let mut g = c.lock().unwrap();
                *g += 1;
            }
        }));
    }
    for h in handles { h.join().unwrap(); }
    println!("counter = {}", *counter.lock().unwrap());
}

/// -------------------------
/// FFI callback simulation
/// -------------------------
/// We simulate a simple C library that accepts a callback and a `void* user_data`.
/// In real code this would be provided by a C header and linked library.
/// The important patterns shown:
/// - `Box<T>` -> `Box::into_raw` to give C ownership of a heap allocation and recoverwith `Box::from_raw`
/// - `Arc<T>` -> `Arc::into_raw` / `Arc::from_raw` in callbacks. If you call `Arc::from_raw` you take ownership;
///   to avoid decreasing the refcount accidentally you can `clone` it for local use and then `Arc::into_raw` the original again.

type CCallback = extern "C" fn(user_data: *mut c_void, value: i32);

struct FakeCLib {
    cb: Option<CCallback>,
    user_data: *mut c_void,
}

impl FakeCLib {
    fn new() -> Self { Self { cb: None, user_data: std::ptr::null_mut() } }

    // "register" a callback with a *mut c_void user_data (C style)
    fn register_callback(&mut self, cb: CCallback, user_data: *mut c_void) {
        self.cb = Some(cb);
        self.user_data = user_data;
    }

    // "invoke" the callback as if C called it
    fn invoke(&self, v: i32) {
        if let Some(cb) = self.cb {
            cb(self.user_data, v);
        }
    }

    // "unregister" returns the stored user_data so the caller can free it
    fn unregister(&mut self) -> *mut c_void {
        let ud = self.user_data;
        self.cb = None;
        self.user_data = std::ptr::null_mut();
        ud
    }
}

/// Callback data that we want shared across Rust and callbacks.
struct CallbackData {
    name: String,
    counter: AtomicUsize,
}

impl CallbackData {
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into(), counter: AtomicUsize::new(0) }
    }
}

/// C-style callback function. This demonstrates safely using an `Arc<T>`
/// that was passed as a raw pointer. The pattern:
/// 1. Recreate an `Arc<T>` with `Arc::from_raw(ptr as *const T)`
///    This takes one strong reference (ownership). If we returned from the function
///    without re-storing the pointer we'd decrement the refcount.
/// 2. Clone the `Arc` for local use.
/// 3. Convert the original `Arc` back to raw with `Arc::into_raw` to restore ownership
///    on the heap and avoid dropping it.
///
/// Note: Do not call `Arc::from_raw` twice on the same raw pointer without balancing
/// with `Arc::into_raw` or else you'll cause UB by double-free / refcount issues.
extern "C" fn c_callback_with_arc(user_data: *mut c_void, value: i32) {
    if user_data.is_null() { return; }

    // SAFETY: pointer must have come from `Arc::into_raw`
    let arc_ptr = user_data as *const CallbackData;
    // Recreate Arc from raw (this temporarily owns one strong ref)
    let arc = unsafe { Arc::from_raw(arc_ptr) };
    // Clone for local use (this increments the strong count)
    let local = Arc::clone(&arc);
    // Restore the original Arc back to a raw pointer so we don't decrement the count when `arc` drops
    let _ = Arc::into_raw(arc);

    // Use `local` safely. When `local` goes out of scope the strong count decreases by one.
    let prev = local.counter.fetch_add(1, Ordering::SeqCst);
    println!("callback: {} got value {} (count was {})", local.name, value, prev + 1);
}

/// Example using Box (single owner) with FFI:
extern "C" fn c_callback_with_box(user_data: *mut c_void, value: i32) {
    if user_data.is_null() { return; }
    // SAFETY: pointer originates from Box::into_raw
    let boxed: Box<CallbackData> = unsafe { Box::from_raw(user_data as *mut CallbackData) };
    // Use it; because we reconstructed with Box::from_raw, when this function returns
    // the Box will be dropped and memory freed. If C expects to keep the pointer alive,
    // do not `from_raw` here — instead clone/re-box before returning.
    println!("boxed callback: {} got value {} (counter was {})",
        boxed.name, value, boxed.counter.load(Ordering::SeqCst));
    // boxed dropped here -> memory freed
}

/// Demo wiring it together
fn demo() {
    // --- Arc + FFI demo ---
    let mut clib = FakeCLib::new();

    // Create shared callback data and keep an Arc in Rust
    let data = Arc::new(CallbackData::new("arc-shared"));

    // Give a clone of the Arc to the C library by converting to a raw pointer.
    // Important: we still keep an Arc in Rust so the data won't be freed until we decide.
    let user_data_raw = Arc::into_raw(Arc::clone(&data)) as *mut c_void;
    clib.register_callback(c_callback_with_arc, user_data_raw);

    // Simulate C invoking the callback from a different thread
    let clib_for_thread = Arc::new(Mutex::new(clib));
    let t = {
        let clib_for_thread = Arc::clone(&clib_for_thread);
        thread::spawn(move || {
            // pretend C calls twice
            thread::sleep(Duration::from_millis(50));
            clib_for_thread.lock().unwrap().invoke(10);
            thread::sleep(Duration::from_millis(50));
            clib_for_thread.lock().unwrap().invoke(20);
        })
    };

    // Wait and then unregister from Rust side. When unregistering we must recover the raw pointer
    // that the fake C lib stored so we can drop our Arc reference that we handed to C.
    thread::sleep(Duration::from_millis(200));
    let user_data_returned = clib_for_thread.lock().unwrap().unregister();

    // Convert raw pointer back to Arc and drop it (decrement the reference count)
    if !user_data_returned.is_null() {
        // SAFETY: pointer previously came from `Arc::into_raw`
        let arc_from_c = unsafe { Arc::from_raw(user_data_returned as *const CallbackData) };
        // dropping `arc_from_c` lowers the strong count
        drop(arc_from_c);
    }

    // We still hold `data` in this scope (Rust-side Arc), show final counter:
    println!("final counter (rust-side): {}", data.counter.load(Ordering::SeqCst));

    t.join().unwrap();

    // --- Box + FFI demo ---
    let mut clib2 = FakeCLib::new();
    // Box::into_raw gives C ownership. This example uses Box so the callback will free the data.
    let boxed = Box::new(CallbackData::new("boxed-one"));
    let raw_box = Box::into_raw(boxed) as *mut c_void;
    clib2.register_callback(c_callback_with_box, raw_box);
    // invoke once; the callback reconstructs the Box and frees it
    clib2.invoke(42);
    // After invocation the stored pointer is dangling; unregister to clear pointer (but in real C you must ensure no use-after-free).
    let _ = clib2.unregister();
}

fn main() {
    arc_read_only_example();
    arc_mutex_example();
    demo();
}
```
