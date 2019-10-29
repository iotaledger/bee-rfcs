+ Feature name: `iri_v181_transaction_and_bundle`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3),
  [iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18),
  [iotaledger/bee-rfcs#20](https://github.com/iotaledger/bee-rfcs/pull/20)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

The fundamental communication unit in the IOTA protocol is the transaction. Messages, including payment settlements and
plain data, are propagated through the IOTA network in transactions. Message that are too large are split up into bundles
of several transactions and sent one by one.

This RFC proposes a `Transaction` type and a `Bundle` type to represent the transaction and bundle formats used by the
IOTA Reference Implementation as of release [`iri v1.8.1`]. We also propose the `TransactionBuilder`,
`IncomingBundleBuilder`, and `OutgoingBundleBuilder` types following the builder pattern to construct both message
types, respectively. We distinguish between incoming and outgoing bundles because one is concerned with verifying the
veracity of an existing message being received, while the other one is concerned with constructing a new message
intended to be sent, which includes setting a number of fields like the bundle hash.

By analogy with IP fragmentation, a bundle corresponds to a packet, while transactions correspond to fragments.

Useful links:

+ [Trinary](https://docs.iota.org/docs/dev-essentials/0.1/concepts/trinary)
+ [What is a transaction?](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-transaction)
+ [What is a bundle?](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-bundle)
+ [Bundles and transactions](https://docs.iota.org/docs/dev-essentials/0.1/concepts/bundles-and-transactions)
+ [Structure of a transaction](https://docs.iota.org/docs/dev-essentials/0.1/references/structure-of-a-transaction)
+ [Structure of a bundle](https://docs.iota.org/docs/dev-essentials/0.1/references/structure-of-a-bundle)
+ [Security levels](https://docs.iota.org/docs/dev-essentials/0.1/references/security-levels)

# Motivation

IOTA is a transaction settlement and data transfer layer for IoT (the Internet of Things). Messages on the network are
`Bundle`s of individual `Transaction`s, which in turn are sent one at a time and are stored in a distributed ledger
called the *Tangle*. Each `Transaction` encodes data such as sender and receiver addresses, referenced transactions in
the Tangle, `Bundle` hash, timestamps, and other information required to verify and process each transaction. With
`Transaction` and `Bundle` representing the units of communication within the IOTA network, we need to be able to
construct and represent them in memory.

At the time of this RFC, the transaction format used by the IOTA network is defined by [`iri v1.8.1`]. The transaction
format might change in the future, but for the time being it is not yet understood how a common interface between
different versions transaction formats should be implemented, or if the network will support several transaction types
simultaneously.

We thus do not consider generalizations over or interfaces for transactions, but only propose a basic `Transaction`
type. All mentions of *transactions* in general or the `Transaction` type in particular will be implicitly in reference
to that format used by `iri v1.8.1`.

A transaction is 8019 trits large in total, out of which 6551 trits are available to store a payload. The payload is
defined to either hold a signature fragment or a message fragment. Since it has a limited size, a user often needs more
than one transaction to execute an operation. For example, signatures with [security level 2 or 3](Security levels)
don't fit in a single transaction, and user-provided messages may exceed the maximum payload capacity so they need to be
fragmented across multiple transactions. Moreover, because the total amount of tokens stored in the ledger has to stay
constant, *input transactions* (which are called thus because an address is *put into* a transfer as a source of tokens,
thus removing them from the address) have to always be matched with *output transactions* such that their total value is
zero. For these reasons, transactions have to be processed as a whole in groups called bundles. A bundle is an atomic
operation in the sense that either all or none of its transactions are accepted by the network. Even single transactions
are propagated through the network within a bundle.

The `Transaction` and `Bundle` types are intended to be final and immutable. This contract is enforced by not exposing
any methods that allow manipulating their fields directly. `Transaction`s are intended to be constructed through
the `TransactionBuilder` builder pattern, while `Bundle`s can be created either via the `IncomingBundleBuilder` or
`OutgoingBundleBuilder`. The `IncomingBundleBuilder` takes in existing `Transaction`s, compiles them, and verifies
their veracity before constructing a `Bundle` type. The `OutgoingBundleBuilder` takes in `TransactionBuilder`s,
and is primarily responsible for filling in all information that can only be set in the actual finalization
of the outgoing message into a `Bundle`, such as the bundle hash.

[`iri v1.8.1`]: https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE

# Detailed design

This RFC proposes the implementation of the following types:

+ `Transaction` to represent the transaction format as of `iri v1.8.1`;
+ `TransactionBuilder`, a builder pattern to create a `Transaction`;
+ `Bundle` to represent a set of `Transaction`s;
+ `IncomingBundleBuilder`, a builder pattern to create `Bundle` from a set of already created `Transaction`s, such as
  those coming in over the wire;
+ `OutgoingBundleBuilder` and `SealedBundleBuilder`, two builder patterns working in tandem to create a `Bundle` from
  `TransactionBuilder`s, i.e. transactions that are not yet fully constructed and will be finalized during the
  construction process of the entire `Bundle`.

In this proposal we keep the representation of the various types relatively simple and flat. We assume that the user is
responsible for pushing the appropriate number of transaction drafts into a bundle builder and don't encode this on the
type level. During the construction phase we check that all transactions are consistent and adhere to IOTA convention,
but we don't, for example:

+ attempt to represent different the different types of transactions on the type level;
+ allow the bundle builders to push extra transactions into their collection automatically to encode a signature of
  a certain size.

The present proposal is meant as a building block for client facing code to provide higher level types that provide
more convenient and implicit ways of constructing transactions and bundles.

## Transaction

A transaction is a sequence of 15 fields of constant length. A transaction has a total length of 8019 trits. The fields'
name, description, and size are summarized in the table below, in their order of appearance in a transaction.

| Name                              | Description                                             | Size (in trits) | Set by (functions detailed below)             |
| ---                               | ---                                                     | ---             | ---
| `signature_or_message_fragment`   | contains a signature fragment of the transfer           |                 |                                               |
|                                   | or user-defined message fragment                        | 6561            | `add_message` or `sign`                       |
| `address`                         | receiver (output) if value > 0,
|                                   | or sender (input) if value < 0                          | 243             | `add_message` or `add_input` or `add_output`  |
| `value`                           | the transferred amount in IOTA                          | 81              | `add_message` or `add_input` or `add_output`  |
| `obsolete_tag`                    | currently only used for the M-Bug (see later)           | 81              | `finalize`                                    |
| `timestamp`                       | the time when the transaction was issued                | 27              | `add_message` or `add_input` or `add_output`  |
| `current_index`                   | the index of the transaction in its bundle              | 27              | `finalize`                                    |
| `last_index`                      | the index of the last transaction in the bundle         | 27              | `finalize`                                    |
| `bundle`                          | the hash of the bundle essence                          | 243             | `finalize`                                    |
| `trunk`                           | the hash of the first transaction referenced/approved   | 243             | `calculate_proof_of_work`                     |
| `branch`                          | the hash of the second transaction referenced/approved  | 243             | `calculate_proof_of_work`                     |
| `tag`                             | arbitrary user-defined value                            | 81              | `add_message` or `add_input` or `add_output`  |
| `attachment_timestamp`            | the timestamp for when Proof-of-Work is completed       | 27              | `calculate_proof_of_work`                     |
| `attachment_timestamp_lowerbound` | *not specified*                                         | 27              | `calculate_proof_of_work`                     |
| `attachment_timestamp_upperbound` | *not specified*                                         | 27              | `calculate_proof_of_work`                     |
| `nonce`                           | the Proof-of-Work nonce of the transaction              | 81              | `calculate_proof_of_work`                     |

Each transaction is uniquely identified by its *transaction hash*, which is calculated based on all fields of the
transaction. Note that the transaction hash is not part of the transaction.

Because modifying the fields of a transaction would invalidate its hash, `Transaction` is immutable after creation.

### `Transaction` struct

A transaction is represented by the `Transaction` struct and follows the structure presented in the table above:

```rust
pub struct Transaction {
    signature_or_message_fragment: SignatureOrMessageFragment,
    address: Address,
    value: Value,
    obsolete_tag: Tag,
    timestamp: Timestamp,
    current_index: Index,
    last_index: Index,
    bundle: BundleHash,
    trunk: TransactionHash,
    branch: TransactionHash,
    tag: Tag,
    attachment_timestamp: Timestamp,
    attachment_timestamp_lower_bound: Timestamp,
    attachment_timestamp_upper_bound: Timestamp,
    nonce: Nonce
}
```

The `Transaction` type only contains two constructor methods getter methods for read-only borrows of its fields.

```rust
impl Transaction {
    /// Create a `Transaction` from a reader.
    pub fn from_reader<R: std::io::Read>(reader: R) -> Result<Self, TransactionError> {
        unimplemented!()
    }

    /// Create a `Transaction` from a slice of bytes
    pub fn from_slice(stream: &[u8]) -> Result<Self, TransactionError> {
        unimplemented!()
    }

    pub fn signature_or_message_fragment(&self) -> &SignatureOrMessageFragment {
        &self.signature_fragments
    }

    pub fn address(&self) -> &Address {
        &self.address
    }

    pub fn value(&self) -> &Value {
        &self.value
    }
}
```

We treat the types of the `Transaction` struct's fields as opaque newtypes for now so that we can flesh them out during
the implementation phase or in future RFCs. Because this RFC implements the transaction format as of iri v1.8.1, the
newtypes shall be constructed from byte slices of a certain length matching that of the reference implementation.
Conversion methods shall only verify that the passed byte slices (or fixed-size arrays or vectors of bytes) are of
appropriate length and that each contained byte correctly encodes a balanced trit (this is also refered to as `T1B1`
binary-coded ternary encoding).

```rust
pub struct SignatureOrMessageFragment([u8; 6561]);
pub struct Address([u8; 243]);
pub struct Value([u8; 81]);
pub struct Tag([u8; 81]);
pub struct Timestamp([u8; 27]);
pub struct Index([u8; 27]);
pub struct BundleHash([u8; 243]);
pub struct TransactionHash([u8; 243]);
pub struct Nonce([u8; 81]);
```

Below is an example implementation for the `Tag` type. We only consider conversion from byte slices, `&[u8]`. Checked
conversions ensuring that each byte encodes `-1`, `0`, or `+1` are implemented in terms of the `TryFrom` trait,
while we also provide `Tag::from_unchecked` as an unchecked faster constructor method.

```rust
enum TagError {
    InvalidBinaryEncodedTrit,
    WrongLength { expected: usize, given: usize },
}

impl fmt::Display for TagError {
    fn display(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        use TagError::*;

        let msg: Cow<'_, str> = match self {
            InvalidBinaryEncodedTrit => "Failed to parse u8; only 0b0000_0000, 0b0000_0001, 0b1111_1111 are valid values.".into(),

            WrongLength { expected, given } =>
                format!("Expected {} characters, found {} in String").into(),
        };

        fmt!(f, &msg)
    }
}

impl error::Error for TagError {
    fn source(&self) -> Option<&(dyn error::Error + 'static)> {
        None
    }
}

impl Tag {
    /// Unchecked conversion from a byte slice.
    ///
    /// Panics if the slice has the wrong length, and does not check if the bytes correctly encode
    /// a trit.
    fn from_unchecked(bytes: &[u8]) -> Self {
        let mut array = [0u8; 81];
        // Panics if `bytes` is not of length 81
        Tag(array.copy_from_slice(bytes))
    }
}

impl<'_> TryFrom<&'_ [u8]> for Tag {
    type Error = TagError;

    fn try_from(value: &'_ [u8]) -> Result<Self, Self::Error> {
        if value.len() != 81 {
            Err(TagError::WrongLength { expected: 81, given: value.len()})?;
        }

        for byte in value {
            match byte {
                0b0000_0000 | 0b1111_1111 | 0b0000_0001 => {},
                _ => Err(TagError::InvalidBinaryEncodedTrit)?,
        }

        Ok(Tag::from_unchecked(value))
    }
}
```

### `TransactionBuilder` struct

The `TransactionBuilder` allows setting all the fields of a transaction, and verifies and builds a correct `Transaction`
type. The order of the fields in `TransactionBuilder` follows the table showing the different fields of a transaction,
the same as `Transaction`.

```rust
pub struct TransactionBuilder {
    signature_or_message_fragment: Option<SignatureOrMessageFragment>,
    address: Option<Address>,
    value: Option<Value>,
    obsolete_tag: Option<Tag>,
    timestamp: Option<Timestamp>,
    current_index: Option<Index>,
    last_index: Option<Index>,
    bundle: Option<BundleHash>,
    trunk: Option<TransactionHash>,
    branch: Option<TransactionHash>,
    tag: Option<Tag>,
    attachment_timestamp: Option<Timestamp>,
    attachment_timestamp_lower_bound: Option<Timestamp>,
    attachment_timestamp_upper_bound: Option<Timestamp>,
    nonce: Option<Nonce>,
}
```

`TransactionBuilder`'s setter methods are implemented using generic type parameters `T: TryInto<{FieldType}>` to provide
some convenience when setting fieds from byte slices. The `TryFrom` implementations for each field type ensure that the
byte slices only encode correct trits. We allow to the checks when setting the fields from trusted sources by explicitly
constructing the target object. For example, when setting the `tag` field on a `builder: TransactionBuilder` object with
`trusted_bytes: &[u8]`, calling `builder.tag(Tag::from_unchecked(trusted_bytes))` sets the `tag` field to a value that
was not verified to only contained valid bytes.

```rust
enum TransactionBuilderError {
    Tag(TagError),
}

impl From<TagError> for Error {
    fn from(value: TagError) -> Self {
        Error::Tag(value)
    }
}

impl fmt::Display for TransactionBuilderError {
    fn display(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        use TransactionBuilderError::*;

        let msg = match self {
            Tag(_) => "Failed setting tag from byte input.",
        };

        fmt!(f, &msg)
    }
}

impl error::Error for TransactionBuilderError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Tag(e) => Some(e),
        }
    }
}

impl TransactionBuilder {
    fn tag<T: TryInto<Tag>>(&mut self, tag: T) -> Result<&mut Self, TransactionBuilderError> {
        self.tag.replace(tag.try_into()?);
        self
    }

    /// Signs the `TransactionBuilder` by using the signature scheme `S` with some seed and input.
    ///
    /// TODO: This depends on one trait `SignatureScheme` which is not implemented yet.
    fn sign<I, S>(&mut self, signature_scheme: S, seed: S::Seed, input: I) -> Result<&mut Self, TransactionBuilderError>
    where
        S: SignatureScheme<I>,
    {
        unimplemented!();

        signature_scheme.sign(self, seed, input)?
    }
}
```

### `Error` types

The `TransactionBuilderError` enum contains variants for each of its fields, encoding that an error can occur when
setting any of them. In addition, it shall contain errors that can occur during verification or construction. We leave
the specification of these extra error variants to the implementation phase.

## `Bundle`

Bundles are immutable groups of immutable transactions. Once a bundle is created and validated, it shouldn't be changed.
An instantiated `Bundle` object represents a syntactically and semantically valid IOTA bundle, and can only be created
via the `IncomingBundleBuilder` and `OutgoingBundleBuilder` builder patterns. `IncomingBundleBuilder` is concerned with
absorbing a number of final `Transaction`s coming in over the wire, and verifying their correctness before combining
them into a final `Bundle`. `OutgoingBundleBuilder` is concerned with taking in a number of not yet ready transactions
represented by `TransactionBuilder`s, and then first inserting the bundle hash (and thus sealing the builder, not
allowing to push any more transaction builders), and finally calculating the nonce via proof of work on each transaction
before constructing the final `Bundle`.

### `Bundle` struct

As bundles are immutable, they shouldn't be modifiable outside of the scope of the bundle module.

There is a natural order to transactions in a bundle that can be represented in two ways:

+ each transaction has a `current_index` and a `last_index`. `current_index` goes from `0` to `last_index`. A bundle can
  then simply be represented by a data structure that contiguously keeps the order like `Vec`;
+ each transaction is chained to the next one through its `trunk` which means we can consider data structures like
  `HashMap` or `BTreeMap`;

For the purpose of this RFC, we opt for the simplest implementation in terms of `Vec` but hide the details behind
a newtype and leave the details of the underlying datastructure open to be changed in the future.

```rust
pub struct Transactions(Vec<Transaction>)
```

Then the `Bundle` type looks like:

```rust
pub struct Bundle {
    transactions: Transactions
}
```

And its implementation should only allow to retrieve transactions:

```rust
impl Bundle {
    pub fn transactions(&self) -> &Transactions {
        &self.transactions
    }
}
```

### `IncomingBundleBuilder`

The `IncomingBundleBuilder` offers a simple way to absorb transactions coming over the wire, and
construct a `Bundle` after verifying that the transactions are consistent.

```rust
pub struct IncomingBundleBuilder {
    transactions: Transactions,
}
```

```rust
impl IncomingBundleBuilder {
    /// Pushes a new transaction coming over the wire into the bundle builder.
    pub push(&mut self, transaction: Transaction) -> &mut Self {
        self.transactions.push(transaction);
        self
    }

    /// Calculate the bundle hash using some sponge.
    ///
    /// A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it
    /// clear that these transactions should be processed as a whole.
    ///
    /// The hash of a bundle is derived from the bundle essence of each of its transactions. A bundle essence is a `486` trits
    /// subset of the transaction fields.
    ///
    /// | Name          | Size      |
    /// | ------------- | --------- |
    /// | address       | 243 trits |
    /// | value         | 81 trits  |
    /// | obsolete_tag  | 81 trits  |
    /// | timestamp     | 27 trits  |
    /// | current_index | 27 trits  |
    /// | last_index    | 27 trits  |
    ///
    /// The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
    /// bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.
    ///
    /// NOTE: `iri v1.8.1` uses `Kerl` as a sponge.
    ///
    /// TODO: This function relies on some `Sponge` trait, which is not yet defined. The code below lays out
    ///       how the sponge is thought to be used.
    pub fn calculate_hash<S: Sponge<Output=BundleHash>>(&self, mut sponge: S) -> S::Output {
        for transaction in bundle_builder {
            sponge.absorb(transaction.essence());
        }
        sponge.squeeze()
    }

    /// Validates if the transactions inside the bundle builder are all consistent, and if they form a valid bundle.
    pub fn validate(&self) -> Result<(), IncomingBundleError> {
        unimplemented!()
    }

    /// Constructs a final, validated bundle from all contained transactions.
    pub fn build(&self) -> Bundle {
        unimplemented!()
    }
}
```

### `SignatureInputs`

Information that is required to sign those transactions in a bundle that withdraw funds from an address.

```rust
struct SignatureInputs {
    address: Address,
    security_level: u8,
    private_key_index: PrivateKeyIndex,
}
```

### `OutgoingBundleBuilder` and `SealedOutgoingBundleBuilder` struct

The `OutgoingBundleBuilder` and `SealedOutgoingBundleBuilder` are more involved compared to their incoming bundle
sibling. `OutgoingBundleBuilder` allows pushing not yet finished transactions into it. Once all `TransactionBuilder`s
are collected, the `OutgoingBundleBuilder` is sealed by calculating and setting the bundle hash on each of its contained
transaction drafts and setting their indices, turning it into a `SealedOutgoingBundleBuilder`.
`SealedOutgoingBundleBuilder` then sets transaction indices, signs transactions removing IOTA tokens from an address,
sets nonce fields, and finally verifies and builds a `Bundle`.

```rust
pub struct OutgoingBundleBuilder {
    transaction_builders: TransactionBuilders,
}

pub struct SealedOutgoingBundleBuilder {
    transaction_builders: TransactionBuilders,
}

struct TransactionBuilders(Vec<TransactionBuilder>);
```

As with `Transactions`, we make `TransactionBuilders` an opaque newtype that for the time being wraps a vector of
`TransactionBuilder`s.

```rust
impl OutgoingBundleBuilder {
    /// Push a new `TransactionBuilder` into the builder.
    ///
    /// Takes ownership of `Self` to allow for chaining with `seal`. For example:
    ///
    /// ```rust
    /// let sealed_outgoing_bundle_builder = outgoing_bundle_builder
    ///     .push(transaction_builder_1)
    ///     .push(transaction_builder_2)
    ///     .push(transaction_builder_3)
    ///     .seal();
    /// ```
    pub push(self, transaction_builder: TransactionBuilder) -> Self {
        self.transaction_builders.push(transaction_builder);
        self
    }

    /// Calculate the bundle hash using some sponge.
    ///
    /// A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it
    /// clear that these transactions should be processed as a whole.
    ///
    /// The hash of a bundle is derived from the bundle essence of each of its transactions. A bundle essence is a `486` trits
    /// subset of the transaction fields.
    ///
    /// | Name          | Size      |
    /// | ------------- | --------- |
    /// | address       | 243 trits |
    /// | value         | 81 trits  |
    /// | obsolete_tag  | 81 trits  |
    /// | timestamp     | 27 trits  |
    /// | current_index | 27 trits  |
    /// | last_index    | 27 trits  |
    ///
    /// The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
    /// bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.
    ///
    /// NOTE: `iri v1.8.1` uses `Kerl` as a sponge.
    ///
    /// NOTE: This function is defined on: `SealedOutgoingBundleBuilder`, `OutgoingBundleBuilder`, and `IncomingBundleBuilder`.
    ///
    /// TODO: This function relies on some `Sponge` trait, which is not yet defined. The code below lays out
    ///       how the sponge is thought to be used.
    pub calculate_hash<S: Sponge<Output=BundleHash>>(&self, mut sponge: S) -> S::Output {
        unimplemented!();

        // Pseudocode of how `calculate_hash` should look.

        for transaction_draft in bundle_builder {
            sponge.absorb(transaction_draft.essence());
        }
        sponge.squeeze()
    }

    /// Seals the bundle builder and sets the bundle hash on all transaction builders contained in it.
    ///
    /// This function calculates the bundle hash from its contained transactions and sets it for all of them, sets
    /// the transactions' indices, ensures that the M-Bug does not occur, and prevents any extra transactions from being
    /// pushed into it.
    ///
    /// Finalizing a bundle means computing the bundle hash, verifying that it matches the security requirement and setting it
    /// to all the transactions of the bundle. After finalization, transactions of a bundle are ready to be safely attached to
    /// the tangle. Finalization is also the place to set the transaction indexes since the bundle essence contains all
    /// `current_index` and `last_index` and we wouldn't be able to alter them after the bundle hash has been calculated.
    /// 
    /// # Normalization
    ///
    /// During signing scheme, the share of the private key being leaked is not uniform, introducing a risk
    /// to leak a critically large part of it. By normalizing the hash, we ensure that exactly half of the private key is
    /// leaked. The actual normalization algorithm is provided in the signing scheme RFC.
    /// Useful links: [Addresses and signatures](https://docs.iota.org/docs/dev-essentials/0.1/concepts/addresses-and-signatures),
    /// [Why is the bundle hash normalized?](https://iota.stackexchange.com/questions/1588/why-is-the-bundle-hash-normalized).
    /// 
    /// # M-Bug
    /// Due to the implementation of the signature scheme, the normalized bundle hash can't contain a `M` (or `13`)
    /// because it could expose a significant part of the private key, making it easier for attackers to forge signatures.
    /// The bundle hash is then repetitively generated with a slight modification until its normalization doesn't contain a `M`.
    /// This modification is usually operated by incrementing the `obsolete_tag` of the first transaction of the bundle since
    /// this field is not being used for any other reason. Useful links:
    /// [Why is the normalized hash considered insecure when containing the char 'M'](https://iota.stackexchange.com/questions/1241/why-is-the-normalized-hash-considered-insecure-when-containing-the-char-m).
    /// 
    /// *Since signature size depends on the security level, a single signature can spread out to up to 3 transactions.
    /// `inputs` is an object that contains all unused addresses of a seed with a sufficient balance.*
    /// 
    /// TODO: This function relies on some `Sponge` trait, which is not yet defined. The code below lays out
    ///       how the sponge is thought to be used.
    pub fn seal<S: Sponge<Output=BundleHash>>(self, sponge: S) -> Result<SealedOutgoingBundleBuilder, OutgoingBundleBuilderError> {
        unimplemented!()

        let mut current_index = 0;

        for transaction_draft in bundle_builder {
            transaction_draft.current_index = current_index++;
            transaction_draft.last_index = bundle_builder.size() - 1;
        }

        let final_hash = loop {
            // Use the `calculate_hash` function defined above.
            let hash = bundle_builder.calculate_hash();
            // See paragraphs on Normalization and M-Bug below pseudocode
            if hash.normalize().find('M') {
                bundle_builder
                    .first_transaction()
                    .increment_obsolete_tag();
            } else {
                break hash;
            }
        };

        for transaction_draft in &mut bundle_builder {
            transaction_draft.bundle = final_hash;
        }
    }
}

impl SealedOutgoingBundleBuilder {
    /// Calculate the bundle hash using some sponge.
    ///
    /// A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it
    /// clear that these transactions should be processed as a whole.
    ///
    /// The hash of a bundle is derived from the bundle essence of each of its transactions. A bundle essence is a `486` trits
    /// subset of the transaction fields.
    ///
    /// | Name          | Size      |
    /// | ------------- | --------- |
    /// | address       | 243 trits |
    /// | value         | 81 trits  |
    /// | obsolete_tag  | 81 trits  |
    /// | timestamp     | 27 trits  |
    /// | current_index | 27 trits  |
    /// | last_index    | 27 trits  |
    ///
    /// The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
    /// bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.
    ///
    /// NOTE: `iri v1.8.1` uses `Kerl` as a sponge.
    ///
    /// NOTE: This function is defined on: `SealedOutgoingBundleBuilder`, `OutgoingBundleBuilder`, and `IncomingBundleBuilder`.
    ///
    /// TODO: This function relies on some `Sponge` trait, which is not yet defined. The code below lays out
    ///       how the sponge is thought to be used.
    pub fn calculate_hash<S: Sponge<Output=BundleHash>>(&self, mut sponge: S) -> S::Output {
        unimplemented!();

        // Pseudocode of how `calculate_hash` should look.

        for transaction_draft in bundle_builder {
            sponge.absorb(transaction_draft.essence());
        }
        sponge.squeeze()
    }

    /// Perform the proof of work calculation, updating the `nonce` field in each of the contained
    /// `TransactionBuilder`s.
    ///
    /// Proof of Work (PoW) allows your transactions to be accepted by the network. On the IOTA network, PoW is only
    /// a rate control mechanism.  After PoW, a bundle is ready to be sent to the network. Given a `trunk` and
    /// a `branch` returned by [`getTransactionsToApprove`], the process iterates over each transaction of the bundle,
    ///  sets some attachment related fields and does individual PoW.
    ///
    /// [`getTransactionsToApprove`]: https://docs.iota.org/docs/node-software/0.1/iri/references/api-reference#gettransactionstoapprove
    ///
    /// TODO: This function relies on some `Sponge` and `ProofOfWork` traits, which are not yet defined. The code below lays out
    ///       how proof of work is thought to be used.
    pub fn perform_proof_of_work<S: Sponge, P: ProofOfWork>(
        mut self,
        trunk_tip: TransactionHash,
        branch_tip: TransactionHash,
        proof_of_work: P,
        sponge: S,
    ) -> Result<Self, OutgoingBundleError> {
        unimplemented!()

        // Iterates over all transactions starting from the head, setting trunks, branches, timestamps, tags, and nonce.
        //
        // NOTE: We do not provide an implementation here.
        let builder_iterator = self.iter_from_head();

        // Set the head transaction
        let head_transaction = builder_iterator.next().ok_or("builder does not contain any transaction builders")?;
        head_transaction.trunk(trunk_tip);
        head_transaction.branch(branch_tip);

        head_transaction.attachment_timestamp = timestamp();
        head_transaction.attachment_timestamp_lower = 0;
        head_transaction.attachment_timestamp_upper = 3812798742493;

        if head_transaction.tag.is_empty() {
            head_transaction.tag = transaction_draft.obsolete_tag;
        }
        head_transaction.set_nonce_from_proof_of_work(proof_of_work);

        // `next_local_trunk` is the hash that will be used as value of the current transaction pointing to the previous
        // one.
        let mut next_local_trunk = head_transaction.calculate_hash(sponge);

        // Set the rest of the transactions
        for transaction_draft in builder_iterator {
            transaction_draft.branch(trunk_tip);
            transaction_draft.trunk(branch_tip);

            transaction_draft.attachment_timestamp = timestamp();
            transaction_draft.attachment_timestamp_lower = 0;
            transaction_draft.attachment_timestamp_upper = 3812798742493;

            if transaction_draft.tag.is_empty() {
                transaction_draft.tag = transaction_draft.obsolete_tag;
            }

            transaction_draft.set_nonce_from_proof_of_work(proof_of_work);
            next_local_trunk = transaction_draft.calculate_hash(sponge);
        }
    }

    /// Signs those transaction drafts contained in the builder that withdraw funds from an address.
    ///
    /// Only the owner of the seed from which the source address is generated has the right to remove funds from it. The
    /// seed is thus used to generate the signature, which in turn is inserteed in those transaction drafts that withdraw
    /// funds. Bundles that contain withdrawing transactions that have missing or bad signatures will be considered invalid
    /// and thus rejected.
    ///
    /// TODO: This function relies on some `SignatureScheme` trait, which is not yet defined. The code below lays out
    ///       how the signing scheme is thought to be used.
    ///
    /// TODO: This function relies on some `Wallet` trait, which is not yet defined. It's supposed to provide some
    ///       `f: Wallet W -> Tx Address A -> Signature Scheme Inputs I`
    pub fn sign<I, S, W>(self, signature_scheme: S, seed: S::Seed, wallet: Wallet) -> Result<Self, OutgoingBundleError>
    where
        S: SignatureScheme<I>,
        W: Wallet<A, Output=S::I>,
    {
        unimplemented!();

        // Pseudocode how `sign` should be used.

        let mut current_index = 0

        // TODO: Explain what this loop is trying to achieve.
        for transaction_draft in self {
            if transaction_draft.value < 0 {
                if transaction_draft.current_index >= current_index {
                    let input = inputs[transaction_draft.address];

                    // NOTE: The specific arguments to the signature scheme depend on the implementation of `SignatureScheme`.
                    //
                    // NOTE: The specific arguments to wallet depend on the implementation of `Wallet`.
                    let fragments = transaction_draft.sign(signature_scheme, seed, wallet.get(transaction_draft.address));

                    for fragment in fragments {
                        bundle_builder[current_index].signature = fragment;
                        current_index = current_index + 1
                    }
                }
            } else {
                current_index = current_index + 1;
            }
        }
    }

    /// Validate a bundle, returning `Ok` if it's valid, and `Err` if not.
    ///
    /// Validating a bundle means checking the syntactic and semantic integrity of a bundle as a whole and of its constituent
    /// transactions. As bundles are atomic transfers, either all or none of the transactions will be accepted by the network.
    /// After validation, transactions of a bundle are candidates to be included to the ledger.
    /// 
    /// For a bundle to be considered valid, the following assertions must be true:
    /// 
    /// + bundle has the correct length;
    /// + transactions share the same bundle hash;
    /// + transactions absolute value doesn't exceed total IOTA supply;
    /// + bundle absolute sum never exceeds total IOTA supply;
    /// + order of transactions in the bundle is the same as announced by `current_index` and `last_index`;
    /// + input/output transactions have an address ending in `0` i.e. has been generated with Kerl;
    /// + bundle inputs and outputs are balanced i.e. the bundle sum equals `0`;
    /// + announced bundle hash matches the computed bundle hash;
    /// + for input transactions, the signature is valid;
    /// 
    /// TODO: `IOTA_SUPPLY` used below is some global constant which needs to be defined and set externally.
    pub fn validate<S: Sponge>(&mut self, sponge: S) -> Result<(), OutgoingBundleError> {
        unimplemented!()

        let mut value = 0;
        let mut current_index = 0;

        // NOTE: We are not defining `first_transaction` here but leave it to the implementation phase.
        if bundle_builder.len() != bundle_builder.first_transaction().last_index + 1 {
            Err(InvalidLength)?
        }

        let bundle_hash = bundle_builder.first_transaction().bundle_hash;
        let last_index = bundle_builder.first_transaction().last_index;

        for transaction_draft in bundle_builder {
            if transaction_draft.bundle_hash != bundle_hash {
                Err(InvalidHash)?
            }

            if abs(transaction_draft.value) > IOTA_SUPPLY {
                Err(InvalidTransactionValue)?
            }

            value += transaction_draft.value;
            if abs(value) > IOTA_SUPPLY {
                Err(InvalidValue)?
            }

            if transaction_draft.current_index != current_index++ {
                Err(InvalidIndex)?
            }

            if transaction_draft.last_index != last_index {
                Err(InvalidIndex)?
            }

            if transaction_draft.value != 0 && transaction_draft.address.last != 0 {
                Err(InvalidAddress)?
        }

        if value != 0 {
            Err(InvalidValue)?
        }

        if bundle_hash != self.calculate_hash() {
            Err(InvalidHash)?
        }

        // NOTE: We do not provide an example of `validate_withdrawal_transactions`. Its purpose is to walk
        //       through those transactions in the bundle builder and verify their signatures.
        bundle_builder.validate_withdrawal_transactions()?;

        Ok(())
    }

    pub fn build(self) -> Result<Bundle, OutgoingBundleError> {
        unimplemented!()
    }
}
```

## Workflow, how we use this

In this section, we describe the algorithms needed to build a `Bundle`. The work flow depends on whether one is
receiving a bundle (and its constituent transactions), or creating a new one to send it. `IncomingBundleBuilder` encodes
receipt of a bundle, while `OutgoingBundleBuilder` encodes sending.

The workflow for an incoming bundle would looke like this in rust pseudocode:

```
// receive: {add transaction}+ -> validate -> build`

for transaction in transactions {
    incoming_bundle_builder.push(transaction);
}

// The build step validates the contained transactions and builds the final `Bundle`. An invalid or incomplete
// collection of transactions will fail the build. A builder not containing any transactions is incomplete.
let bundle = incoming_bundle_builder.build()?;
```

The construction of an outgoing bundle would happen like this:

```rust
// send: {add transaction builders}+ -> insert_bundle_hash (-> sign)? -> pow -> validate -> build`
//
// + `sign` is optional because data transactions don't have to be signed
//
//
// for builder in transaction_builders {
//     outgoing_bundle_builder.push(builder);
// }
//
// let bundle = outgoing_bundle_builder
//      .insert_bundle_hash()?
//      // We expect the user of this builder to explicitly sign the bundle with their data, as we don't want to store
//      // this information inside the builder struct.
//      .sign(SignatureData)?
//      .pow()?
//      // Validation is part of the build step.
//      .build()?;
```


## (Missing) interfaces

This RFC is intended to be self contained and thus does not rely on the existence of any traits or types defined outside
of this proposal. There are however a few aspects of this proposal that would benefit from additional interface
definitions to make its types easier and more idiomatic to use, or in order to make it more typesafe. We list these here:

+ `Transaction` and its fields are heavily tied to the message format of `iri v1.8.1`, the fixed nature of its fields
  and the representation of each of its fields as `one byte per trit` is the obvious choice. However, equipping the
  field types with more semantics, for example implementing it in terms of a `BinaryCodedTernary` trait might be useful
  to enforce invariants.
+ Serialization (and deserialization) of transactions and bundles and its builders is done by concatenating the bytes of
  all its fields. It might be useful in the future to define traits to allow external code to take a transaction and
  serialize it.
+ `OutgoingBundleBuilder::proof_of_work` takes an `Fn(&mut [u8; 8019]) -> Result<(), Box<&dyn error::Error>>`. A future
  RFC should implement an interface instead, for example a `ProofOfWork` trait, and implement the method in terms of it
  to a) not tie it to strongly to the current transaction length, and b) to properly type its errors. 

# Drawbacks

+ There might be use cases that require the ability to directly create a `Transaction`, requiring exposing a builder
  pattern.
+ This proposal does not consider any kind of generics or abstraction over transactions and bundles. Future iterations
  of the IOTA network that have different transaction and/or bundle formats will probably require new types.

# Rationale and alternatives

+ Immutable `Transaction`s and `Bundle`s encode the fact that they should not be mutated after creation.
+ If there is a use case that requires direct creation of a `Transaction` it can be brought forward in a feature request
  or RFC. The interface can always be extended upon and be made public.
+ Forbidding the direct creation of a `Transaction` via a public API ensures that a complete message is only ever
  constructed via a `Bundle`.
+ This proposal is very straight forward and idiomatic Rust. It does not contain any overly complicated parts. As such,
  it should be easy to extend it in the future.
+ Hiding the fields of `Transaction` and `Bundle` behind opaque types allows us to flesh them out in the future.

# Open questions

+ The design of the builders currently take ownership and return `Self`. This is nice for chaining calls, but might
  suboptimal for performance. Is it more useful to take borrows, either `&Self` or `&mut Self`, instead?
+ `getTransactionsToApprove` function: the function executing proof of work on `SealedOutgoingBundleBuilder` is talking about
  getting tip and branch from the mentionede function. Should it be part of this RFC or assumed external?

# Blockers

+ `SignatureScheme` (includes normalization) interface (to sign withdrawal transactions)
+ `Wallet`: some interface that ties in with `SignatureScheme` to provide a map from a transaction address to the inputs
  of the signature scheme.
+ `ProofOfWork` interface (to calculate and fill the nonce)
+ `Sponge` interface (to calculate bundle and transaction hashes)
+ `Constants`: There are global constant which need to be defined and set externally; example: `IOTA_SUPPLY` 
