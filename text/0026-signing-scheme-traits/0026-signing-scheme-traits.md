+ Feature name: `signing-scheme-traits`
+ Start date: 2019-10-28
+ RFC PR: [iotaledger/bee-rfcs#26](https://github.com/iotaledger/bee-rfcs/pull/26)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

<!-- TODO -->

# Motivation

<!-- TODO -->

# Detailed design

```rust
pub trait PrivateKey {
    type PublicKey;
    type Signature;

    fn generate_public_key(&self) -> Self::PublicKey;
    fn sign(&self, message: &[i8]) -> Self::Signature;
}
```

```rust
pub trait PublicKey {
    type Signature;
    type Error;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> Result<(), Self::Error>;
}
```

```rust
pub trait Signature {
    type PublicKey;

    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey;
}
```

# Drawbacks

<!-- TODO -->

# Rationale and alternatives

<!-- TODO -->

# Unresolved questions

<!-- TODO -->
