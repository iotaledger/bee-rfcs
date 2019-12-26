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

The proposed design is at the very core of asymmetric cryptography in which a pair of private and public keys are used
to create and verify digital signatures. Public keys and signatures are public while private keys remain secret.

## Traits

Following a common asymmetric cryptography pattern, we provide 5 traits `PrivateKeyGenerator`, `PrivateKey`, `PublicKey`, `Signature` and `RecoverableSignature`.

Associated types are being used to bind implementations of these 3 traits together within a signing scheme. For example,
a `PrivateKey` implementation of a signing scheme shouldn't be used with a `PublicKey` implementation of another
signing scheme.

### `PrivateKeyGenerator` trait

```rust
pub trait PrivateKeyGenerator {
    type PrivateKey;

    fn generate(&self, seed: &[i8], index: usize) -> Self::PrivateKey;
}
```

### `PrivateKey` trait

The creation of a private key differs a lot from a signing scheme to another so we do not enforce a specific way to
to build one with a trait method. We expect each private key implementation to provide a `new` method with appropriate
parameters. Once a private key is created, it is responsible for the creation of public keys and signatures which are
enforced by trait methods.

<!-- TODO -->

```rust
pub trait PrivateKey {
    type PublicKey;
    type Signature;

    fn generate_public_key(&self) -> Self::PublicKey;
    fn sign(&mut self, message: &[i8]) -> Self::Signature;
}
```

### `PublicKey` trait

<!-- TODO -->

```rust
pub trait PublicKey {
    type Signature;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool;
    fn to_bytes(&self) -> &[i8];
}
```

### `Signature` trait

<!-- TODO -->

```rust
pub trait Signature {
    fn size(&self) -> usize;
    fn to_bytes(&self) -> &[i8];
}
```

### `RecoverableSignature` trait

```rust
pub trait RecoverableSignature {
    type PublicKey;

    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey;
}

```

## Implementation & workflow

### WOTS example

#### Structures and implementations

```rust
pub struct WotsPrivateKeyGeneratorBuilder<S> {
    security_level: Option<usize>,
    _sponge: PhantomData<S>
}

pub struct WotsPrivateKeyGenerator<S> {
    security_level: usize,
    _sponge: PhantomData<S>
}

pub struct WotsPrivateKey<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>
}

pub struct WotsPublicKey<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>
}

pub struct WotsSignature<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>
}

impl<S: Sponge> WotsPrivateKeyGeneratorBuilder<S> {
    pub fn security_level(&mut self, security_level: usize) -> &mut Self {
      unimplemented!();
    }

    pub fn build(&mut self) -> WotsPrivateKeyGenerator<S> {
      unimplemented!();
    }
}

impl<S: Sponge> crate::PrivateKeyGenerator for WotsPrivateKeyGenerator<S> {
    type PrivateKey = WotsPrivateKey<S>;

    fn generate(&self, seed: &[i8], index: usize) -> Self::PrivateKey {
      unimplemented!();
    }
}

impl<S: Sponge> crate::PrivateKey for WotsPrivateKey<S> {
    type PublicKey = WotsPublicKey<S>;
    type Signature = WotsSignature<S>;

    fn generate_public_key(&self) -> Self::PublicKey {
      unimplemented!();
    }

    fn sign(&mut self, message: &[i8]) -> Self::Signature {
      unimplemented!();
    }
}

impl<S: Sponge> crate::PublicKey for WotsPublicKey<S> {
    type Signature = WotsSignature<S>;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool {
      unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
      unimplemented!();
    }
}

impl<S: Sponge> crate::Signature for WotsSignature<S> {
    fn size(&self) -> usize {
      unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
      unimplemented!();
    }
}

impl<S: Sponge> crate::RecoverableSignature for WotsSignature<S> {
    type PublicKey = WotsPublicKey<S>;

    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey {
      unimplemented!();
    }
}
```

#### Workflow

```rust
let private_key_generator = WotsPrivateKeyGeneratorBuilder::<Kerl>::default().security_level(2).build();
let mut private_key = private_key_generator.generate(seed, index);
let public_key = private_key.generate_public_key();
let signature = private_key.sign(message);
let valid = public_key.verify(message, &signature);
```

### MSS example

# Drawbacks

<!-- TODO -->

# Rationale and alternatives

<!-- TODO -->

# Unresolved questions

- Should we prefix `Signing` to the `PrivateKey` and `PublicKey` traits since we expect encryption keys to be developed
at some point (e.g. NTRU for MAM) ?
- Should `PublicKey` provide an `address` function that return an `Address` type ?
- `verify` currently returns `bool`, we may consider returning an `Error` type if need be ?
