+ Feature name: `signing-scheme`
+ Start date: 2019-10-28
+ RFC PR: [iotaledger/bee-rfcs#26](https://github.com/iotaledger/bee-rfcs/pull/26)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

Asymmetric cryptography is a system that makes use of a pair of keys to allow:

+ **Confidentiality**: messages are encrypted with a public key and only the owner of the matching private key is able
  to decrypt it;
+ **Authenticity**: messages are signed with a private key and can be verified by anyone with the matching public key;

In the IOTA network, asymmetric cryptography is used to authenticate the transfer of tokens or data. For a payment
settlement to be accepted by the network, a proof that the initiator of the transfer is the actual tokens owner has to
be provided, this proof is the digital signature of the transfer. In the IOTA core protocol, data transfer signatures
are not enforced but some second layer protocol like MAM may rely on data digital signatures.

In this RFC we will only focus on the authenticity aspect of asymmetric cryptography.

We define `signing scheme` as being the set of data structures and algorithms allowing authenticity of a message.

Useful links:

+ [Asymmetric cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)
+ [Digital signature](https://en.wikipedia.org/wiki/Digital_signature)
+ [IOTA addresses](https://docs.iota.org/docs/getting-started/0.1/clients/addresses)
+ [IOTA signatures](https://docs.iota.org/docs/getting-started/0.1/clients/signatures)

# Motivation

At the time of writing, the following signing schemes are being used in the IOTA network:

+ the ledger uses a Winternitz One-Time Signature (WOTS) scheme;
+ the coordinator uses a Merkle Signature Scheme (MSS) scheme on top of WOTS to enable reusable addresses;
+ the first iteration of MAM uses another WOTS scheme;
+ the first iteration of MAM uses another MSS scheme on top of WOTS to enable reusable addresses;
+ the second iteration of MAM uses another WOTS scheme;
+ the second iteration of MAM uses another MSS scheme on top of WOTS to enable reusable addresses;
+ other schemes are being investigated for the ledger;

Even though these schemes are very different on an implementation point of view, they all share the same basic
behaviour justifying the existence of the following proposed design composed of traits.

Useful links:

+ [Lamport signature](https://en.wikipedia.org/wiki/Lamport_signature)
+ [On the Security of the Winternitz One-Time Signature Scheme](https://eprint.iacr.org/2011/191.pdf)
+ [Merkle signature scheme](https://en.wikipedia.org/wiki/Merkle_signature_scheme)
+ [EdDSA](https://en.wikipedia.org/wiki/EdDSA)

# Detailed design

The proposed design is at the very core of asymmetric cryptography in which a pair of private and public keys are used
to create and verify digital signatures. Public keys and signatures are public while private keys remain secret.

The [IOTA Signing Scheme](https://github.com/iotaledger/iri/blob/dev/src/main/java/com/iota/iri/crypto/ISS.java) as
implemented in IRI and most libraries can be cumbersome and different from what we could expect from an asymmetric
cryptography API. We then propose to replace the existing methods with a more traditional set of traits enforcing a
shared and expected behaviour.

## Traits

We propose the traits `PrivateKeyGenerator`, `PrivateKey`, `PublicKey`, `Signature` and `RecoverableSignature` together
enforcing the following features: private key generation, public key generation, signing, signature verification and
public key recovery.

Associated types are being used to bind implementations of these traits together within a signing scheme. For example,
a `PrivateKey` implementation of a signing scheme shouldn't be used with a `PublicKey` implementation of another
signing scheme.

### `PrivateKeyGenerator` trait

<!-- TODO seed + index -->
<!-- TODO creation delegation -->

The creation of a private key differs a lot from a signing scheme to another so we do not enforce a specific way to
to build one with a trait method. We expect each private key implementation to provide a `new` method with appropriate
parameters. Once a private key is created, it is responsible for the creation of public keys and signatures which are
enforced by trait methods.

```rust
pub trait PrivateKeyGenerator {
    type PrivateKey;

    ///
    fn generate(&self, seed: &[i8], index: usize) -> Self::PrivateKey;
}
```

### `PrivateKey` trait

<!-- TODO -->

```rust
pub trait PrivateKey {
    type PublicKey;
    type Signature;

    ///
    fn generate_public_key(&self) -> Self::PublicKey;

    ///
    fn sign(&mut self, message: &[i8]) -> Self::Signature;
}
```

### `PublicKey` trait

<!-- TODO -->

```rust
pub trait PublicKey {
    type Signature;

    ///
    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool;

    ///
    fn to_bytes(&self) -> &[i8];
}
```

### `Signature` trait

<!-- TODO -->

```rust
pub trait Signature {
    ///
    fn size(&self) -> usize;

    ///
    fn to_bytes(&self) -> &[i8];
}
```

### `RecoverableSignature` trait

<!-- TODO -->

```rust
pub trait RecoverableSignature {
    type PublicKey;

    ///
    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey;
}

```

## Implementation & workflow

In this section we give example skeleton implementations of these traits and workflows for WOTS and MSS.

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
let mut private_key = private_key_generator.generate(seed, 0);
let public_key = private_key.generate_public_key();
let signature = private_key.sign(message);
let recovered_public_key = signature.recover_public_key(message);
let valid = public_key.verify(message, &signature);
```

### MSS example

#### Structures and implementations

```rust
pub struct MssPrivateKeyGeneratorBuilder<S, G> {
    depth: Option<usize>,
    generator: Option<G>,
    _sponge: PhantomData<S>,
}

pub struct MssPrivateKeyGenerator<S, G> {
    depth: usize,
    generator: G,
    _sponge: PhantomData<S>,
}

pub struct MssPrivateKey<S, K> {
    depth: usize,
    index: usize,
    keys: Vec<K>,
    tree: Vec<i8>,
    _sponge: PhantomData<S>,
}

pub struct MssPublicKey<S, K> {
    state: Vec<i8>,
    depth: usize,
    _sponge: PhantomData<S>,
    _key: PhantomData<K>
}

pub struct MssSignature<S> {
    state: Vec<i8>,
    index: usize,
    _sponge: PhantomData<S>
}

impl<S, G> MssPrivateKeyGeneratorBuilder<S, G>
    where S: Sponge,
          G: PrivateKeyGenerator {

    pub fn depth(&mut self, depth: usize) -> &mut Self {
      unimplemented!();
    }

    pub fn generator(&mut self, generator: G) -> &mut Self {
      unimplemented!();
    }

    pub fn build(&mut self) -> MssPrivateKeyGenerator<S, G> {
      unimplemented!();
    }
}

impl<S, G> crate::PrivateKeyGenerator for MssPrivateKeyGenerator<S, G>
    where S: Sponge,
          G: PrivateKeyGenerator,
          <G as PrivateKeyGenerator>::PrivateKey: PrivateKey,
          <<G as PrivateKeyGenerator>::PrivateKey as PrivateKey>::PublicKey: PublicKey {

    type PrivateKey = MssPrivateKey<S, G::PrivateKey>;

    fn generate(&self, seed: &[i8], index: usize) -> Self::PrivateKey {
      unimplemented!();
    }
}

impl<S, K> crate::PrivateKey for MssPrivateKey<S, K>
    where S: Sponge,
          K: PrivateKey,
          <K as PrivateKey>::PublicKey: PublicKey,
          <K as PrivateKey>::Signature: Signature {

    type PublicKey = MssPublicKey<S, K::PublicKey>;
    type Signature = MssSignature<S>;

    fn generate_public_key(&self) -> Self::PublicKey {
      unimplemented!();
    }

    fn sign(&mut self, message: &[i8]) -> Self::Signature {
      unimplemented!();
    }
}

impl<S, K> crate::PublicKey for MssPublicKey<S, K>
    where S: Sponge,
          K: PublicKey {

    type Signature = MssSignature<S>;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool {
      unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
      unimplemented!();
    }
}

impl<S: Sponge> crate::Signature for MssSignature<S> {
    fn size(&self) -> usize {
      unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
      unimplemented!();
    }
}
```

#### Workflow

```rust
let wots_private_key_generator = WotsPrivateKeyGeneratorBuilder::<Kerl>::default().security_level(2).build();
let mss_private_key_generator = MssPrivateKeyGeneratorBuilder::<Kerl, WotsPrivateKeyGenerator<Kerl>>::default().depth(7).generator(wots_private_key_generator).build();
let mut private_key = mss_private_key_generator.generate(seed, 0);
let public_key = private_key.generate_public_key();
let signature = private_key.sign(message);
let valid = public_key.verify(message, &signature);
```

# Drawbacks

<!-- TODO -->

# Rationale and alternatives

<!-- TODO -->

+ This design fits the standards and expected APIs;
+ The main alternative is the current implementation of ISS as in IRI and most libraries;

# Unresolved questions

<!-- TODO -->

+ Should we prefix `Signing` to the `PrivateKey` and `PublicKey` traits since we expect encryption keys to be developed
at some point (e.g. NTRU for MAM) ?
+ Some schemes like `Ed25519` benefit from faster batch verification, justifying a later introduction of a batch
verifier trait.
+ Should the generator generate a pair of keys ? Can we always generate the public key from the private key ?
+ In the proposed design, we rely on the sponges implementing `Default` which for example doesn't allow different
number of rounds for Curl unless Curl27 and Curl81 are types.
