# Slices and Borrowing
`[T]` is the unsized slice type; `&[T]` is the borrowed slice (pointer + length). Common sources are arrays and `Vec<T>`. `&Vec<T>` can coerce to `&[T]`, but an API should usually accept `&[T]` for flexibility.

```rust
// language: rust
use std::fmt::Debug;

fn print_slice<T: Debug>(s: &[T]) {
    // takes a borrowed slice (does not take ownership)
    println!("slice: {:?}", s);
}

fn print_vec_ref<T: Debug>(v: &Vec<T>) {
    // takes a borrowed Vec specifically
    println!("vec ref: {:?}", v);
}

fn main() {
    // array -> slice
    let arr = [10, 20, 30];
    let arr_slice: &[i32] = &arr;        // array coerces to `&[i32]`
    print_slice(arr_slice);
    print_slice(&arr);                  // also works (coercion)

    // vec -> slice
    let v = vec![1, 2, 3];
    print_slice(&v);                    // &Vec<T> coerces to &[T]
    print_vec_ref(&v);                  // you can also pass &Vec<T>

    // slice from a temporary array literal
    print_slice(&[4, 5, 6]);            // array literal coerces to slice

    // note: you *cannot* pass a `&[T]` to a function expecting `&Vec<T>`
    // let s: &[i32] = &arr;
    // print_vec_ref(s); // ERROR: expected `&Vec<i32>`, found `&[i32]`
}
```

Summary:
- Use `&[T]` for APIs that accept any contiguous sequence (array, `Vec`, literal).
- `&Vec<T>` is more specific and less flexible; prefer `&[T]` unless you need `Vec` methods.

## Byte Buffer

`&[&[u8]]` reads as "a borrowed slice of borrowed byte slices."

Brief explanation
- `&[T]` is a borrowed slice: a pointer + length referring to a contiguous sequence of `T`.
- `&[u8]` is a borrowed slice of bytes (non-owning view of a byte buffer).
- Combining them, `&[&[u8]]` is a reference to a contiguous array whose elements are references to byte slices. Memory layout: one pointer+len for the outer slice, and each element is a pointer+len to the underlying bytes.
- Ownership/lifetimes: nothing is owned; all referenced data must live at least as long as the `&[&[u8]]` borrow.
- Common use: API that accepts multiple byte buffers without taking ownership (e.g., copying them into an owned type like `Bytes`).

When to prefer alternatives
- If you want to accept many kinds of inputs (Vec, arrays, Strings, Bytes), use a generic `IntoIterator<Item = impl AsRef<[u8]>>` to be more flexible.
- If you need to own the data, accept `Vec<Vec<u8>>` or `Vec<Bytes>`.

Example
```rust
fn copy_all(buffers: &[&[u8]]) -> Vec<Vec<u8>> {
    buffers.iter().map(|b| b.to_vec()).collect()
}


## String Buffers

`entries: &[&str]` means "a borrowed slice of borrowed string slices".

Brief explanation:
- `&str` is a string slice: a non-owning view into UTF‑8 bytes (pointer + length). It borrows the data.
- `&[T]` is a borrowed slice: a pointer + length referring to a contiguous sequence of `T`.
- So `&[&str]` is a reference to a slice whose elements are `&str`. The function borrows the collection and the individual strings; it does not take ownership.
- Lifetime: the input must live at least as long as the call. Using string literals (which are `'static`) or temporary arrays coerced to slices (e.g. `&["a","b"]`) is common.
- In your code it’s fine because the function copies each `&str` into an owned `Bytes`, so it doesn’t retain the borrow after returning.

Examples (calling and a more flexible signature):

```rust
// calling: array coerces to a slice automatically
let ns = TrackNamespace::from_strings(&["chat", "room1"]);

// more flexible alternative signature that accepts many string-like inputs
pub fn from_strings<I, S>(entries: I) -> Self
where
    I: IntoIterator<Item = S>,
    S: AsRef<str>,
{
    Self {
        entries: entries.into_iter()
            .map(|s| Bytes::copy_from_slice(s.as_ref().as_bytes()))
            .collect(),
    }
}
```

fn main() {
    let a: &[u8] = b"hello";
    let b: &[u8] = b"world";
    let slice_array: &[&[u8]] = &[a, b]; // coerces from array to slice
    let copied = copy_all(slice_array);
    assert_eq!(copied[0].as_slice(), b"hello");
}
```
