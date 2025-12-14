Lifetime parameters in Rust let the compiler check that references do not 
outlive the data they point to. They do not change runtime behavior — they are a 
**static (compile-time) contract** that prevents dangling pointers.

Key ideas:

- A lifetime like `'a` names how long a reference must be valid.
- You attach lifetimes to references in function signatures and structs when the compiler cannot infer them automatically.
- Lifetime elision gives short, common patterns default lifetimes (no annotation needed).
- `'static` is the longest lifetime — data that lives for the entire program.
- Lifetimes express relationships (which reference must outlive which), not ownership or borrowing rules themselves.

## Short example: function signatures with lifetimes.

This function returns the longer of two string slices; the returned reference must live 
at least as long as both inputs, so the lifetime parameter links them.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() >= y.len() { x } else { y }
}

fn example() {
    let s1 = String::from("short");
    let s2 = String::from("a longer string");
    let result = longest(s1.as_str(), s2.as_str());
    println!("{}", result);
}
```

## Structs holding references require a lifetime parameter.

`Holder<'a>` contains a reference; the `'a` ensures the referenced `String` outlives the `Holder`. This pattern is the same used in FFI helpers like `FfiClientConfig<'a>` (see below for the real-world example).

```rust
struct Holder<'a> {
    name: &'a str,
}

impl<'a> Holder<'a> {
    fn name(&self) -> &'a str {
        self.name
    }
}

fn holder_example() {
    let owned = String::from("owned data");
    let h = Holder { name: &owned };
    println!("{}", h.name());
    // `owned` must not be dropped while `h` is in use.
}
```

Quick tips:

- Prefer owning types (String) unless you must borrow; they avoid lifetime plumbing.
- Use elision: many function signatures don't need explicit lifetimes.
- When returning a reference, annotate lifetimes so the compiler knows how inputs and outputs relate.
- For FFI, keep CString owned inside a struct and return raw pointers only while that owning struct remains alive. That is exactly why `FfiClientConfig<'a>` carries CString fields plus `&'a TransportConfig`.

Common compiler messages:
- "borrowed value does not live long enough" — fix by extending the source's scope or changing to an owned value.
- "missing lifetime specifier" — add a lifetime parameter and use it consistently.

This is the minimal practical view; apply lifetimes where the compiler requires them and prefer ownership to reduce annotations.

# Real World Example

'a is a lifetime parameter that marks how long borrowed data inside `FfiClientConfig<'a>` must remain valid. 
In `FfiClientConfig<'a>`, the field `transport: &'a TransportConfig` 
borrows a `TransportConfig`, so any `FfiClientConfig<'a>` value cannot 
outlive the referenced `TransportConfig`. The compiler uses `'a` to prevent dangling 
references and ensure safety when you create C-strings and also return pointers 
into them or borrow other objects.

Brief illustration: the first example is valid because the borrowed `Transport` lives as long as the `FfiClientConfig`. The second example shows the compile error you get if you try to return a struct that borrows from a local value that will be dropped.

```rust
// Valid: transport lives as long as ffi
struct Transport { value: i32 }

struct FfiClientConfig<'a> {
    transport: &'a TransportConfig,
}

fn make<'a>(t: &'a Transport) -> FfiClientConfig<'a> {
    FfiClientConfig { transport: t }
}

fn valid_example() {
    let transport = Transport { value: 42 };
    let ffi = make(&transport); // 'transport' outlives 'ffi' -> OK
    println!("{}", ffi.transport.value);
}

// Invalid: borrows a local value that is dropped at function end
fn invalid_example() -> FfiClientConfig<'static> {
    let local = Transport { value: 1 };
    // ERROR: cannot return value that references local `local`
    // FfiClientConfig { transport: &local }
    unimplemented!()
}
```
