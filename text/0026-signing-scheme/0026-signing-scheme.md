+ Feature name: `signing-scheme`
+ Start date: 2019-10-28
+ RFC PR: [iotaledger/bee-rfcs#26](https://github.com/iotaledger/bee-rfcs/pull/26)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

Following the principles of asymmetric cryptography and Hierarchical Deterministic Wallets, this RFC establishes a
signing scheme for the IOTA protocol.

## Asymmetric cryptography

Asymmetric cryptography is a system that makes use of a pair of keys to allow:

+ **Confidentiality**: a message is encrypted by anyone with a public key and only the owner of the matching private key
  is able to decrypt it;
+ **Authenticity**: a message is signed with a private key by its owner and the signature can be verified by anyone with
  the matching public key;

This RFC only focuses on the authenticity aspect of asymmetric cryptography and defines `signing scheme` as being the
set of data structures and algorithms allowing authenticity of a message.

In the IOTA protocol, asymmetric cryptography is used to authenticate the transfer of tokens. To move tokens out of an
address - which is a public key - one has to show a proof that he actually is the owner of the tokens. This proof is
the digital signature of the transfer, has been created with the private key associated to that address and the
signature is verifiable by anyone with the address.

Useful links:

+ [Asymmetric cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)
+ [Digital signature](https://en.wikipedia.org/wiki/Digital_signature)
+ [IOTA signatures](https://docs.iota.org/docs/getting-started/0.1/clients/signatures)

## Hierarchical Deterministic Wallets

<!-- TODO -->

Useful links:

+ [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
+ [IOTA seeds](https://docs.iota.org/docs/getting-started/0.1/clients/seeds)
+ [IOTA addresses](https://docs.iota.org/docs/getting-started/0.1/clients/addresses)

# Motivation

At the time of writing, the following signing schemes are being used in the IOTA network:

+ the ledger uses a Winternitz One-Time Signature (WOTS) scheme;
+ the coordinator uses a Merkle Signature Scheme (MSS) scheme on top of WOTS to enable reusable addresses;
+ both iterations of MAM use another WOTS scheme;
+ both iterations of MAM use another MSS scheme on top of WOTS to enable reusable addresses;

Other schemes, like Ed25519, are being investigated for the ledger.

Even though these schemes are very different on an implementation point of view, they all share the same basic
behaviour justifying the existence of the following proposed design.

Useful links:

+ [Lamport signature](https://en.wikipedia.org/wiki/Lamport_signature)
+ [On the Security of the Winternitz One-Time Signature Scheme](https://eprint.iacr.org/2011/191.pdf)
+ [Merkle signature scheme](https://en.wikipedia.org/wiki/Merkle_signature_scheme)
+ [EdDSA](https://en.wikipedia.org/wiki/EdDSA)

# Detailed design

<!-- TODO -->

The proposed design is at the very core of asymmetric cryptography in which a pair of private and public keys are used
to create and verify digital signatures. Public keys and signatures are public while private keys remain secret.

The [IOTA Signing Scheme](https://github.com/iotaledger/iri/blob/dev/src/main/java/com/iota/iri/crypto/ISS.java) as
implemented in IRI and most libraries can be cumbersome and different from what one could expect from an asymmetric
cryptography API. This RFC then propose to replace the existing methods with a more traditional set of traits enforcing
a shared and expected behaviour.

## Seed

As explained earlier, the seed is the root of the hierarchical deterministic wallet and all public/private keys can be
derived from it. **It must then be kept secret**. In the IOTA protocol, a seed is sequence of 81 trytes like `"PUEOTSEITFEVEWCWBTSIZM9NKRGJEIMXTULBACGFRQK9IMGICLBKW9TTEVSDQMGWKBXPVCBMMCXWMNPDX"` (**DO NOT USE THIS SEED**).

The seed is a critical component of the signing scheme. Everywhere a seed is needed as input, the following `Seed`
type must be used only and no other representation like string, bytes or trits. This is why the following design only allows valid and verified instantiation of the `Seed` type.

```rust
// TODO documentation
pub struct Seed([i8; 243]);

// TODO documentation
#[derive(Debug, PartialEq)]
pub enum SeedError {
    InvalidLength(usize),
    InvalidTrit(i8),
}

// TODO documentation
impl Seed {
    // TODO documentation
    pub fn new() -> Self {
      unimplemented!();
    }

    // TODO documentation
    pub fn subseed<S: Sponge + Default>(&self, index: u64) -> Self {
      unimplemented!();
    }

    // TODO documentation
    pub fn from_bytes(bytes: &[i8]) -> Result<Self, SeedError> {
      unimplemented!();
    }

    // TODO documentation
    pub fn to_bytes(&self) -> &[i8] {
      unimplemented!();
    }
}
```

## Sponge

Even though the following design doesn't depend on a sponge construction, the main signing scheme implementations
currently being used by the IOTA protocol - WOTS and MSS - heavily relies on it. For this reason, this RFC doesn't
directly rely on a `Sponge` trait but the example implementations shown below does.

<!-- TODO: provide a link when merged -->
The `Sponge` trait is detailed in the Hash RFC.

## Traits

This RFC proposes the traits `PrivateKeyGenerator`, `PrivateKey`, `PublicKey`, `Signature` and `RecoverableSignature`
together enforcing the following features: private key generation, public key generation, signing, signature
verification and public key recovery.

Associated types are being used to bind implementations of these traits together within a signing scheme as, for
example, a `PrivateKey` implementation of a signing scheme shouldn't be used with a `PublicKey` implementation of
another signing scheme.

Some signing schemes like `Ed25519` benefit from faster batch verification, justifying a later introduction of a batch
verifier trait, probably as a separate RFC.

### `PrivateKeyGenerator` trait

The creation of a private key differs a lot from a signing scheme to another in terms of parameters so there is no
consistent way of building one with a method enforced by the `PrivateKey` trait. Instead, the private key generation is
delegated to a specific `PrivateKeyGenerator` trait which has `PrivateKey` as an associated type.

A `PrivateKeyGenerator` implementation is expected to have all the specific parameters needed for the private key
generation (e.g. security level, depth, ...) embedded as attributes. It would usually itself be built with a builder
pattern. Once it is built, the `generate` method can be used, only requiring consistent parameters seed and index.

```rust
// TODO documentation
pub trait PrivateKeyGenerator {
    // TODO documentation
    type PrivateKey;

    // TODO documentation
    fn generate(&self, seed: &Seed, index: u64) -> Self::PrivateKey;
}
```

<!-- Reviewed -->

### `PrivateKey` trait

A private key is responsible for generating its public counterpart and signing messages; hence, having `PublicKey` and
`Signature` as associated types.

**A private key should remain secret.**

```rust
// TODO documentation
pub trait PrivateKey {
    // TODO documentation
    type PublicKey;
    // TODO documentation
    type Signature;

    // TODO documentation
    fn generate_public_key(&self) -> Self::PublicKey;

    // TODO documentation
    fn sign(&mut self, message: &[i8]) -> Self::Signature;
}
```

<!-- TODO Note on mut self -->

### `PublicKey` trait

A public key is responsible for verifying the authenticity of a message signature; hence, having `Signature` as
associated type. It should also be able to serialise it to bytes.

```rust
// TODO documentation
pub trait PublicKey {
    // TODO documentation
    type Signature;

    /// Verifies the authenticity of a message signature
    ///
    /// # Parameters
    ///
    /// * `message` - A slice that holds the message to verify the signature of
    /// * `signature` - A signature to verify
    ///
    /// # Example
    ///
    /// ```
    /// let valid = public_key.verify(message, &signature);
    /// ```
    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool;

    // TODO documentation
    fn from_bytes(bytes: &[i8]) -> Self;

    /// Serialises a public key to a slice of bytes
    ///
    /// # Example
    ///
    /// ```
    /// let bytes = public_key.to_bytes();
    /// ```
    fn to_bytes(&self) -> &[i8];
}
```

### `Signature` trait

<!-- TODO -->

```rust
// TODO documentation
pub trait Signature {
    // TODO documentation
    fn size(&self) -> usize;

    // TODO documentation
    fn from_bytes(bytes: &[i8]) -> Self;

    // TODO documentation
    fn to_bytes(&self) -> &[i8];
}
```

### `RecoverableSignature` trait

<!-- TODO -->

```rust
// TODO documentation
pub trait RecoverableSignature {
    // TODO documentation
    type PublicKey;

    // TODO documentation
    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey;
}
```

## Implementation & workflow

This section gives example skeleton implementations of these traits and workflows for WOTS and MSS.

### WOTS example

#### Structures and implementations

<!-- TODO WotsError -->

```rust
#[derive(Default)]
pub struct WotsPrivateKeyGeneratorBuilder<S> {
    security_level: Option<u8>,
    _sponge: PhantomData<S>,
}

#[derive(Default, Debug)]
pub struct WotsPrivateKeyGenerator<S> {
    security_level: u8,
    _sponge: PhantomData<S>,
}

pub struct WotsPrivateKey<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>,
}

pub struct WotsPublicKey<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>,
}

pub struct WotsSignature<S> {
    state: Vec<i8>,
    _sponge: PhantomData<S>,
}

impl<S: Sponge + Default> WotsPrivateKeyGeneratorBuilder<S> {
    pub fn security_level(&mut self, security_level: u8) -> &mut Self {
        unimplemented!();
    }

    pub fn build(&mut self) -> Result<WotsPrivateKeyGenerator<S>, String> {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::PrivateKeyGenerator for WotsPrivateKeyGenerator<S> {
    type PrivateKey = WotsPrivateKey<S>;

    fn generate(&self, seed: &Seed, index: u64) -> Self::PrivateKey {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::PrivateKey for WotsPrivateKey<S> {
    type PublicKey = WotsPublicKey<S>;
    type Signature = WotsSignature<S>;

    fn generate_public_key(&self) -> Self::PublicKey {
        unimplemented!();
    }

    fn sign(&mut self, message: &[i8]) -> Self::Signature {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::PublicKey for WotsPublicKey<S> {
    type Signature = WotsSignature<S>;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool {
        unimplemented!();
    }

    fn from_bytes(bytes: &[i8]) -> Self {
        unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::Signature for WotsSignature<S> {
    fn size(&self) -> usize {
        unimplemented!();
    }

    fn from_bytes(bytes: &[i8]) -> Self {
        unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::RecoverableSignature for WotsSignature<S> {
    type PublicKey = WotsPublicKey<S>;

    fn recover_public_key(&self, message: &[i8]) -> Self::PublicKey {
        unimplemented!();
    }
}
```

#### Workflow

<!-- TODO Different kind of workflows -->
<!-- TODO Workflow all  the funcs -->
<!-- TODO new seed -->

```rust
let private_key_generator = WotsPrivateKeyGeneratorBuilder::<Kerl>::default().security_level(2).build().unwrap();
let mut private_key = private_key_generator.generate(seed, 0);
let public_key = private_key.generate_public_key();
let signature = private_key.sign(message);
let recovered_public_key = signature.recover_public_key(message);
let valid = public_key.verify(message, &signature);
```

### MSS example

#### Structures and implementations

```rust
#[derive(Default)]
pub struct MssPrivateKeyGeneratorBuilder<S, G> {
    depth: Option<u8>,
    generator: Option<G>,
    _sponge: PhantomData<S>,
}

pub struct MssPrivateKeyGenerator<S, G> {
    depth: u8,
    generator: G,
    _sponge: PhantomData<S>,
}

pub struct MssPrivateKey<S, K> {
    depth: u8,
    index: u64,
    keys: Vec<K>,
    tree: Vec<i8>,
    _sponge: PhantomData<S>,
}

pub struct MssPublicKey<S, K> {
    state: Vec<i8>,
    depth: u8,
    _sponge: PhantomData<S>,
    _key: PhantomData<K>,
}

pub struct MssSignature<S> {
    state: Vec<i8>,
    index: u64,
    _sponge: PhantomData<S>,
}

impl<S, G> MssPrivateKeyGeneratorBuilder<S, G>
where
    S: Sponge + Default,
    G: PrivateKeyGenerator,
{
    pub fn depth(&mut self, depth: u8) -> &mut Self {
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
where
    S: Sponge + Default,
    G: PrivateKeyGenerator,
    <G as PrivateKeyGenerator>::PrivateKey: PrivateKey,
    <<G as PrivateKeyGenerator>::PrivateKey as PrivateKey>::PublicKey: PublicKey,
{
    type PrivateKey = MssPrivateKey<S, G::PrivateKey>;

    fn generate(&self, seed: &Seed, index: u64) -> Self::PrivateKey {
        unimplemented!();
    }
}

impl<S, K> crate::PrivateKey for MssPrivateKey<S, K>
where
    S: Sponge + Default,
    K: PrivateKey,
    <K as PrivateKey>::PublicKey: PublicKey,
    <K as PrivateKey>::Signature: Signature,
    <<K as PrivateKey>::PublicKey as PublicKey>::Signature: Signature + RecoverableSignature,
    <<<K as PrivateKey>::PublicKey as PublicKey>::Signature as RecoverableSignature>::PublicKey:
        PublicKey,
{
    type PublicKey = MssPublicKey<S, K::PublicKey>;
    type Signature = MssSignature<S>;

    fn generate_public_key(&self) -> Self::PublicKey {
        unimplemented!();
    }

    fn sign(&mut self, message: &[i8]) -> Self::Signature {
        unimplemented!();
    }
}

impl<S, K> MssPublicKey<S, K>
where
    S: Sponge + Default,
    K: PublicKey,
{
    pub fn depth(mut self, depth: u8) -> Self {
        unimplemented!();
    }
}

impl<S, K> crate::PublicKey for MssPublicKey<S, K>
where
    S: Sponge + Default,
    K: PublicKey,
    <K as PublicKey>::Signature: Signature + RecoverableSignature,
    <<K as PublicKey>::Signature as RecoverableSignature>::PublicKey: PublicKey,
{
    type Signature = MssSignature<S>;

    fn verify(&self, message: &[i8], signature: &Self::Signature) -> bool {
        unimplemented!();
    }

    fn from_bytes(bytes: &[i8]) -> Self {
        unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
        unimplemented!();
    }
}

impl<S: Sponge + Default> MssSignature<S> {
    pub fn index(mut self, index: u64) -> Self {
        unimplemented!();
    }
}

impl<S: Sponge + Default> crate::Signature for MssSignature<S> {
    fn size(&self) -> usize {
        unimplemented!();
    }

    fn from_bytes(bytes: &[i8]) -> Self {
        unimplemented!();
    }

    fn to_bytes(&self) -> &[i8] {
        unimplemented!();
    }
}
```

#### Workflow

<!-- TODO Different kind of workflows -->
<!-- TODO Workflow all  the funcs -->

```rust
let wots_private_key_generator = WotsPrivateKeyGeneratorBuilder::<Kerl>::default().security_level(2).build().unwrap();
let mss_private_key_generator = MssPrivateKeyGeneratorBuilder::<Kerl, WotsPrivateKeyGenerator<Kerl>>::default().depth(7).generator(wots_private_key_generator).build().unwrap();
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

+ Should `PrivateKey` and `PublicKey` traits be prefixed with `Signing` since encryption keys are expected to be
developed at some point (e.g. NTRU for MAM) ?
+ Should the generator generate a pair of keys ? Is this always possible to generate the public key from the private
key ?
+ The proposed design relies on the sponges implementing `Default` which for example doesn't allow different number of
rounds for Curl unless Curl27 and Curl81 are types.
+ How should serialisation/deserialisation be handled ? `to_bytes` ? `From`/`Into` ? `Serde` ?
+ Should return values always be wrapped in Result ?
+ Should signature be padded ?
