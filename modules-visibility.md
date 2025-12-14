# pub(create)

`pub(crate)` makes an item visible to the entire current crate but not to 
code in other crates. It's a crate-level visibility modifier — more permissive 
than private, but more restrictive than `pub` (which exports to other crates). 

You can also use `pub(super)`, `pub(in path)`, etc., for other scopes.

Example:

```rust
// Rust
mod inner {
    // Visible everywhere inside this crate, but not to other crates.
    pub(crate) fn crate_visible() {
        println!("visible in crate");
    }

    // Visible only inside `inner` module.
    fn private_fn() {
        println!("private");
    }

    // Public to other crates as well.
    pub fn exported() {
        println!("exported");
    }
}

fn call_from_same_crate() {
    // allowed: same crate
    inner::crate_visible();
    inner::exported();
    // inner::private_fn(); // not allowed: private to module
}
```

# `pub(super)`, `pub(in path)` and more


- `pub` — visible everywhere (any crate).
- `pub(crate)` — visible to the entire current crate only.
- `pub(super)` — visible to the parent module only (one level up).
- `pub(self)` — visible only to the current module (effectively private).
- `pub(in path)` — visible to the specified module `path` and that module's children.
-   `path` can be `crate::...`, `super::...`, or a full module path.

Notes:
- The general syntax is `pub(restricted)` where `restricted` is `crate`, `self`, `super`
  or an explicit module path.
- You can use these on functions, structs, enum variants, fields, impl blocks, etc.
- `pub(in ..)` requires a module path, not an arbitrary type or expression.

Example:

Explanation: the example sets up a `crate` with `top` containing modules `a` and `b`. 
It shows which calls are allowed from sibling/parent scopes for 
`pub`, `pub(crate)`, `pub(super)`, `pub(in ...)`, and private items.

```rust
// rust
mod top {
    pub mod a {
        // Visible anywhere in the crate
        pub(crate) fn crate_fn() { }

        // Visible to parent (`top`) and its children
        pub(super) fn parent_fn() { }

        // Visible only inside `crate::top::a` and its child modules
        pub(in crate::top::a) fn in_a_only() { }

        // Public to other crates
        pub fn exported() { }

        // Private to this module (`a`)
        fn private_fn() { }

        // Same as private
        pub(self) fn self_fn() { }
    }

    pub mod b {
        pub fn try_calls() {
            // Allowed: same crate -> can call pub(crate)
            super::a::crate_fn();

            // Allowed: b is inside `top`, and `parent_fn` is pub(super) from `a` (parent is `top`)
            super::a::parent_fn();

            // Not allowed: `in_a_only` is restricted to `a` and its children
            // super::a::in_a_only(); // ERROR: not visible here

            // Allowed: exported is public
            super::a::exported();

            // Not allowed: private/self functions of `a`
            // super::a::private_fn(); // ERROR
            // super::a::self_fn(); // ERROR
        }
    }

    pub fn top_calls() {
        // top (the parent of a) can call pub(super) from a
        a::parent_fn();

        // top cannot call `in_a_only`
        // a::in_a_only(); // ERROR
    }
}
```
