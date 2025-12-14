A compact tutorial on Rust's `Box<T>`: what it is, when to use it, and practical examples.

- `Box<T>` allocates `T` on the heap and gives single ownership with a fixed-size pointer on the stack.
- Use `Box` for large values to avoid stack growth, recursive types, trait objects (`Box<dyn Trait>`), stable pointers for FFI, and when you need to control drop location.
- Create with `Box::new(value)`. Access via `*` or automatic deref coercions. Convert to/from raw pointer with `Box::into_raw` / `Box::from_raw`. Use `Box::leak` to intentionally leak and get `'static` reference. Use `Pin<Box<T>>` for self-referential types.

Example usages:

```rust
// rust
// 1) Basic usage
let b = Box::new(42);          // allocate on heap
assert_eq!(*b, 42);           // deref to read

let mut bm = Box::new(String::from("hi"));
bm.push_str(" there");        // deref coercion lets you call methods
assert_eq!(&*bm, "hi there");

// 2) Recursive type (common use)
enum List {
    Cons(i32, Box<List>),
    Nil,
}
use List::*;
let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));

// 3) Trait object via Box<dyn Trait>
trait Draw { fn draw(&self) -> String; }
struct Circle(f32);
impl Draw for Circle { fn draw(&self) -> String { format!("Circle r={}", self.0) } }
let shape: Box<dyn Draw> = Box::new(Circle(3.0));
assert_eq!(shape.draw(), "Circle r=3");

// 4) Moving out of a Box (take ownership)
let bstr = Box::new(String::from("owned"));
let s: String = *bstr; // moves String out of box; box is consumed

// 5) FFI / stable pointer example
#[repr(C)]
struct CData { x: u32, y: u32 }
let boxed = Box::new(CData { x: 1, y: 2 });
// pass pointer to C: stable heap address
let raw = Box::into_raw(boxed); // ownership moved to raw
// ... call C code with `raw` ...
// reclaim ownership when done
let boxed = unsafe { Box::from_raw(raw) };

// 6) Leak a Box to get a &'static reference
let leaked: &'static mut i32 = Box::leak(Box::new(100));
*leaked += 1; // intentionally leaked; won't be dropped

// 7) Pin<Box<T>> for self-referential types
use std::pin::Pin;
struct SelfRef { data: String, ptr: *const str }
let mut s = Box::pin(SelfRef { data: String::from("hello"), ptr: std::ptr::null() });
// now you can safely create internal pointers without moving the box
let sr: Pin<&mut SelfRef> = s.as_mut();
let data_ptr = sr.get_ref().data.as_str() as *const str;
// ... store pointer inside if needed (careful with safety)

// Notes:
// - `Box` adds a heap allocation cost; don't use it solely to avoid stack variables.
// - For shared ownership use `Rc` or `Arc`. For mutation across threads use `Arc<Mutex<T>>`.
// - For FFI stability, ensure lifetime and safety when using raw pointers.
```

Because `*bm` moves the owned `String` out of the `Box` (consuming the box), the `&` borrows instead: `&*bm` produces a `&String` which coerces to `&str` and lets you compare to the string literal without taking ownership. Alternatives: borrow via `as_str()` or slicing, or intentionally move with `*` if you want to consume the box.

Example alternatives:
```rust
// rust
let b = Box::new(String::from("hi there"));

// borrow as &str via method (no move)
assert_eq!(b.as_str(), "hi there");

// borrow as &str via slice (no move)
assert_eq!(&b[..], "hi there");

// move the String out of the Box (consumes `b`)
let s: String = *b;
assert_eq!(s, String::from("hi there"));
```
