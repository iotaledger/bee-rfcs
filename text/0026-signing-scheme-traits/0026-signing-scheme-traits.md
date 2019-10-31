+ Feature name: `signing-scheme-traits`
+ Start date: 2019-10-28
+ RFC PR: [iotaledger/bee-rfcs#26](https://github.com/iotaledger/bee-rfcs/pull/26)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

Asymmetric cryptography is mainly used for two purposes:

+ **Confidentiality**. Messages are encrypted with a public key and only the owner of the matching private key is able
  to decrypt it;
+ **Authenticity**. Messages are signed with a private key and can be verified by anyone with the matching public key;

In the IOTA network, asymmetric cryptography is used to authenticate the transfer of tokens or data. For a payment
settlement to be accepted by the network, a proof that the initiator of the transfer is the actual tokens owner has to
be provided, this proof is the digital signature of the transfer. In the IOTA core protocol, data transfer signatures
are not enforced but some second layer protocol like MAM may rely on data digital signatures.

In this RFC we will only focus on the authenticity aspect of asymmetric cryptography.

Useful links:

+ [Asymmetric cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)
+ [Digital signature](https://en.wikipedia.org/wiki/Digital_signature)
+ [Addresses and signatures](https://docs.iota.org/docs/dev-essentials/0.1/concepts/addresses-and-signatures)

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

The proposed traits are at the very core of asymmetric cryptography in which a pair of private and public keys are used
to create and verify digital signatures. Public keys and signatures are public while private keys remain secret.

We thus provide 3 traits `PrivateKey`, `PublicKey` and `Signature`.

Associated types are being used to bind implementations of these 3 traits together within a signing scheme. For example,
a `PrivateKey` implementation of a signing scheme shouldn't be used with a `PublicKey` implementation of another
signing scheme. 

## `PrivateKey` trait

<!-- TODO -->

```rust
pub trait PrivateKey {
    type PublicKey;
    type Signature;

    fn generate_public_key(&self) -> Self::PublicKey;
    fn sign(&self, message: &[i8]) -> Self::Signature;
}
```

## `PublicKey` trait

<!-- TODO -->

```rust
pub trait PublicKey {
    type Signature;
    type Error;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> Result<(), Self::Error>;
}
```

## `Signature` trait

<!-- TODO -->

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

- Should we prefix `Signing` to the `PrivateKey` and `PublicKey` traits since we expect encryption keys to be developed
at some point (e.g. NTRU for MAM) ?
