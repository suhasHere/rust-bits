Explanation: Vec<T> implements Deref<Target = [T]> so a borrowed 
Vec (i.e. &Vec<T>) coerces to a slice (&[T]) automatically. The same 
applies to bytes::Bytes which implements Deref<Target = [u8]>; 
thus &Bytes coerces to &[u8]. This enables passing &Vec<u8> or &Bytes to 
functions expecting &[u8] and lets you call slice methods 
(like iter(), as_ptr(), len()).

Example:

```rust
use bytes::Bytes;

/// accepts any byte slice
fn accept_bytes(slice: &[u8]) {
    println!("len = {}", slice.len());
    // use slice.iter(), slice.as_ptr(), etc.
}

/// shows coercion from &Vec<u8> -> &[u8]
fn from_vec() {
    let v = vec![1u8, 2, 3];
    // &v is coerced to &[u8] automatically
    accept_bytes(&v);

    // you can also call slice methods directly on &v
    println!("first = {}", v.first().unwrap());
}

/// shows coercion from &Bytes -> &[u8]
fn from_bytes() {
    let b = Bytes::from("hello");
    // &Bytes coerces to &[u8]
    accept_bytes(&b);

    // you can get raw pointer/len for FFI
    let ptr = b.as_ptr();
    let len = b.len();
    println!("ptr={:?} len={}", ptr, len);
}

/// Vec<Bytes> -> &[Bytes] coercion and iter usage
fn vec_of_bytes_examples() {
    let entries: Vec<Bytes> = vec![Bytes::from("chat"), Bytes::from("room1")];

    // &entries coerces to &[Bytes], so a function expecting &[Bytes] accepts &entries
    fn take_entries_slice(slice: &[Bytes]) {
        for e in slice.iter() {
            // each `e` is a &Bytes which also derefs to &[u8]
            println!("{}", String::from_utf8_lossy(e));
        }
    }

    take_entries_slice(&entries);
}

fn main() {
    from_vec();
    from_bytes();
    vec_of_bytes_examples();
}
```

Notes:
- You do not need to call `.as_ref()` or `.into()` when passing `&Vec<u8>` or `&Bytes` to a function taking `&[u8]`; Rust will do the coercion.
- Coercion happens for references (e.g. `&Vec<T>` -> `&[T]`), not by value (you still move/clone the Vec if you pass it by value).
