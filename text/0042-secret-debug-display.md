+ Feature name: `secret-debug-display`
+ Start date: 2020-07-03
+ RFC PR: [iotaledger/bee-rfcs#42](https://github.com/iotaledger/bee-rfcs/pull/42)
+ Bee issue: [iotaledger/bee#100](https://github.com/iotaledger/bee/issues/100)

# Summary

This RFC introduces two [procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html), `SecretDebug`
and `SecretDisplay`, to derive the traits [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) and
[Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) for secret material types in order to avoid leaking
their internals.

# Motivation

Secret materials, private keys for example, are types that are expected to remain private and be shared/leaked under no
circumstances. However, these types may be wrapped in other higher level types, a wallet for example, that may want to
implement `Debug` and/or `Display`. Not implementing these traits on the secret types would then not permit it.
At the same time, secret materials should not be exposed by logging them. The consensus is then to implement `Debug` and
`Display` but to not actually log the inner secret. To make things easier, two procedural macros are proposed to
automatically derive  such implementations.

# Detailed design

## Crate

Procedural macros need to be defined in a different type of crate so a new dedicated crate has to be created with the following in its `Cargo.toml`:
```toml
[lib]
proc-macro = true
```

## Derivation

This feature is expected to be used by deriving `SecretDebug` and/or `SecretDisplay`.

```rust
#[derive(SecretDebug, SecretDisplay)]
struct PrivateKey {
    ...
}
```

## Output

When an attempt to display or debug a secret type is made, a generic message should be displayed instead.
This RFC proposes `<Omitted secret>`.

## Implementation

This feature makes use of the [syn](https://crates.io/crates/syn) and [quote](https://crates.io/crates/quote) crates to
create derivation macros.

Implementation of `SecretDebug`:
```rust
#[proc_macro_derive(SecretDebug)]
pub fn derive_secret_debug(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    // Used in the quasi-quotation below as `#name`.
    let name = input.ident;
    // Get the different implementation elements from the input.
    let (impl_generics, ty_generics, _) = input.generics.split_for_impl();

    let expanded = quote! {
        // The generated implementation.
        impl #impl_generics std::fmt::Debug for #name #ty_generics {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                write!(f, "<Omitted secret>")
            }
        }
    };

    // Hand the output tokens back to the compiler.
    expanded.into()
}
```

Implementation of `SecretDisplay` is similar.

# Drawbacks

User may actually really want to debug/display their secret types. With this feature in place, they would have to rely
on APIs providing getters to the inner secret and/or serialization primitives.

# Rationale and alternatives

By providing macros, we offer a consistent way to approach the problem and potentially reduce the risk of users leaking
their secret materials by accident.

# Unresolved questions

No questions at the moment.
