+ Feature name: `signing-scheme-traits`
+ Start date: 2019-10-28
+ RFC PR: [iotaledger/bee-rfcs#26](https://github.com/iotaledger/bee-rfcs/pull/26)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

<!-- TODO -->

# Motivation

At the time of writing, the following signing schemes are being used in the IOTA network:

+ the ledger uses a Winternitz One-Time Signature (WOTS) scheme;
+ the coordinator uses a Merkle Signature Scheme (MSS) scheme on top of WOTS to enable reusable addresses;
+ the first iteration of MAM uses another WOTS scheme;
+ the first iteration of MAM uses another MSS scheme on top of WOTS to enable reusable addresses;
+ the second iteration of MAM uses another WOTS scheme;
+ the second iteration of MAM uses another MSS scheme on top of WOTS to enable reusable addresses;
+ other schemes are being investigated for the ledger;

Even though these implementations are very different, they all share the same behaviour, justifying the existence of the
proposed traits.

Useful links:

+ [Lamport signature](https://en.wikipedia.org/wiki/Lamport_signature)
+ [On the Security of the Winternitz One-Time Signature Scheme](https://eprint.iacr.org/2011/191.pdf)
+ [Merkle signature scheme](https://en.wikipedia.org/wiki/Merkle_signature_scheme)

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
