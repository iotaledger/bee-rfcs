+ Feature name: `iri_v1_8_1_transaction_and_bundle`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3),
  [iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18),
  [iotaledger/bee-rfcs#20](https://github.com/iotaledger/bee-rfcs/pull/20)
+ Bee issue: [iotaledger/bee#63](https://github.com/iotaledger/bee/issues/63)

# Summary

The fundamental communication unit in the IOTA protocol is the transaction. Messages can either represent payment
settlements or serve as plain data carriers. In IOTA terminology, messages are referred to as *bundles*, which ---
depending on the message size --- consist of one or more *transactions*. A transaction is the smallest unit of
communication, and has a fixed size.

This RFC proposes a `Transaction` type and a `Bundle` type to represent the transaction and bundle formats used by the
IOTA Reference Implementation as of release [`iri v1.8.1`]. To construct these types, this proposal also includes the
builder following patterns:

+ `TransactionBuilder`,
+ `IncomingBundleBuilder`,
+ `OutgoingBundleBuilder`,
+ `SealedBundleBuilder`,
+ `SignedBundleBuilder`,
+ `AttachedBundleBuilder`,
+ `ValidatedBundleBuilder`.

`IncomingBundleBuilder` is concerned with constructing and verifying complete messages coming in externally, while
`OutgoingBundleBuilder`, `SealedBundleBuilder`, and `SignedBundleBuilder` are together responsible for constructing
a bundle from scratch to be sent to the IOTA network. This includes signing its transactions, calculating the bundle
hash, and setting other relevant fields depending on context.

Useful links:

+ [Trinary](https://docs.iota.org/docs/dev-essentials/0.1/concepts/trinary)
+ [What is a transaction?](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-transaction)
+ [What is a bundle?](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-bundle)
+ [Bundles and transactions](https://docs.iota.org/docs/dev-essentials/0.1/concepts/bundles-and-transactions)
+ [Structure of a transaction](https://docs.iota.org/docs/dev-essentials/0.1/references/structure-of-a-transaction)
+ [Structure of a bundle](https://docs.iota.org/docs/dev-essentials/0.1/references/structure-of-a-bundle)
+ [Security levels](https://docs.iota.org/docs/dev-essentials/0.1/references/security-levels)

# Motivation

IOTA is a transaction settlement and data transfer layer for the Internet of Things. Messages on the network are
`Bundle`s of one or more `Transaction`s, which in turn are sent one at a time and are stored in a distributed ledger
called the *Tangle*. Each transaction encodes data such as sender and receiver addresses, referenced transactions in
the Tangle, bundle hash, timestamps, and other information required to verify and process each transaction. With
`Transaction` and `Bundle` representing the units of communication within the IOTA network, one needs to be able
to construct them and represent in memory.

The current design proposal intends to be as simple as possible, provide an idiomatic Rust interface, and use existing
IOTA terminology where applicable. The present types are not intended to be used by end users, but rather as building
blocks for higher level APIs. As such, the different kinds of transactions are not encoded as Rust types, and the bundle
builders do not contain any logic to implicitly push more transactions into their stack. For a bundle to be buildable,
all required transactions have to be present when validating and building. Otherwise the build will fail.

As an example, a simple `Bundle` that intends to transfer funds, withdrawing tokens from one address and depositing them
in another would be constructed like this:

```rust
let mut outgoing_bundle_builder = OutgoingBundleBuilder::new();

let mut withdrawing_tx_builder = TransactionBuilder::new();
let mut depositing_tx_builder = TransactionBuilder::new();

// The transactions here only need to set addresses, values, and tags. All other fields, such as the signature
// of the withdrawing transaction, will be filled in by the bundle builder.
withdrawing_tx_builder
    .address(address_providing_funds)
    .value(value_to_transfer)
    .tag(custom_tag);

depositing_tx_builder
    .address(address_receiving_funds)
    .value(value_to_transfer)
    .tag(custom_tag);

let outgoing_bundle = outgoing_bundle_builder
    .push(withdrawing_tx_builder)
    .push(depositing_tx_builder)
    // Seal the bundle, adding indices to its transactions and assigning a bundle hash. This also prevents
    // pushing more transactions into the bundle.
    .seal()?
    // Check if there are withdrawal transactions and sign those using some signature scheme, seed, and wallet.
    .sign(signature_scheme, seed, wallet)?
    // Attach to the tangle, chain transactions to each other by performing proof of work and calculating transaction hashes via the sponge.
    .chain_and_attach(trunk_tip, branch_tip, proof_of_work, sponge)?
    // Check if all transactions contained in the sealed builder are consistent.
    .validate(sponge)?
    // Construct the final `Bundle`.
    .build()?;
```

At the time of this RFC, the transaction format used by the IOTA network is defined by [`iri v1.8.1`]. The transaction
format might change in the future, but for the time being it is not yet understood how a common interface between
different versions transaction formats should be implemented, or if the network will support several transaction types
simultaneously.

This proposal thus does not consider generalizations over or interfaces for transactions, but only propose a basic
`Transaction` type. All mentions of *transactions* in general or of the `Transaction` type in particular will be
implicitly in reference to the format used by `iri v1.8.1`.

[`iri v1.8.1`]: https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE

# Detailed design

This RFC proposes the implementation of the following types:

+ `Transaction` to represent the transaction format as of `iri v1.8.1`;
+ `TransactionBuilder`, a builder pattern to create a `Transaction`;
+ `Bundle` to represent a set of `Transaction`s;
+ `IncomingBundleBuilder`, a builder pattern to create `Bundle` from a set of already created `Transaction`s, such as
  those coming in over the wire;
+ `OutgoingBundleBuilder`, `SealedBundleBuilder`, `SignedBundleBuilder`: builder patterns working in tandem to create
  a `Bundle` from `TransactionBuilder`s, i.e. transactions that are not yet fully constructed and will be finalized
  during the construction process of the entire `Bundle`.

This proposal keeps the representation of the various types flat. It is assumed that the user is responsible for pushing
the appropriate number of transaction drafts into a bundle builder. During construction, all transactions are
checked for consistency and for adherence to IOTA conventions. Among the higher level logic that this proposal
does not intend to provide are:

+ representation of different types of transactions at the type level;
+ allowing the bundle builders to push extra transactions into their collection automatically to encode signatures or
  messages exceeding the maximum capacity of one transaction.

The proposed types are meant as building blocks for higher level abstractions in client facing code to provide more
convenient and implicit ways of constructing transactions and bundles.

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
them before constructing a `Bundle` type. The `OutgoingBundleBuilder` takes in `TransactionBuilder`s,
and is primarily responsible for filling in all information that can only be set in the actual finalization
of the outgoing message into a `Bundle`, such as the bundle hash.

## Transaction

A transaction is a sequence of 15 fields of constant length. A transaction has a total length of 8019 trits. The fields'
name, description, and size are summarized in the table below, in their order of appearance in a transaction.

| Name                              | Description                                            | Size (in trits) |
| ---                               | ---                                                    | ---             |
| `payload`                         | contains a signature fragment of the transfer          |                 |
|                                   | or user-defined message fragment                       | 6561            |
| `address`                         | receiver (output) if value > 0,                        |                 |
|                                   | or sender (input) if value < 0                         | 243             |
| `value`                           | the transferred amount in IOTA                         | 81              |
| `obsolete_tag`                    | currently only used for the M-Bug (see later)          | 81              |
| `timestamp`                       | the time when the transaction was issued               | 27              |
| `index`                           | the index of the transaction in its bundle             | 27              |
| `last_index`                      | the index of the last transaction in the bundle        | 27              |
| `bundle`                          | the hash of the bundle essence                         | 243             |
| `trunk`                           | the hash of the first transaction referenced/approved  | 243             |
| `branch`                          | the hash of the second transaction referenced/approved | 243             |
| `tag`                             | arbitrary user-defined value                           | 81              |
| `attachment_timestamp`            | the timestamp for when Proof-of-Work is completed      | 27              |
| `attachment_timestamp_lowerbound` | *not specified*                                        | 27              |
| `attachment_timestamp_upperbound` | *not specified*                                        | 27              |
| `nonce`                           | the Proof-of-Work nonce of the transaction             | 81              |

Each transaction is uniquely identified by its *transaction hash*, which is calculated based on all fields of the
transaction. Note that the transaction hash is not part of the transaction.

Because modifying the fields of a transaction would invalidate its hash, `Transaction` is immutable after creation.

### `Transaction`

A transaction is represented by the `Transaction` struct and follows the structure presented in the table above:

```rust
pub struct Transaction {
    payload: Payload,
    address: Address,
    value: Value,
    obsolete_tag: Tag,
    timestamp: Timestamp,
    index: Index,
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

The `Transaction` type should be equipped with constructor methods to instantiate an object of its type from a type
implementing `std::io::Read`. We leave it to the implementation phase to define other appropriate methods. Other than
that it contains getter methods for read-only borrows of its fields.

```rust
impl Transaction {
    /// Create a `Transaction` from a reader object.
    pub fn from_reader<R: std::io::Read>(reader: R) -> Result<Self, TransactionError> {
        unimplemented!()
    }

    pub fn payload(&self) -> &Payload {
        &self.payload
    }

    pub fn address(&self) -> &Address {
        &self.address
    }

    pub fn value(&self) -> &Value {
        &self.value
    }
}
```

The fields in the `Transaction` struct are opaque newtypes so that they can be fleshed out during the implementation
phase or in future RFCs without requiring breaking changes. Because this RFC implements the transaction format as of
`iri v1.8.1`, the newtypes shall be constructed from byte slices of a certain length matching that of the reference
implementation. Conversion methods shall only verify that the passed byte slices (or fixed-size arrays or vectors of
bytes) are of appropriate length and that each contained byte correctly encodes a balanced trit (this is also referred
to as `T1B1` binary-coded ternary encoding).

```rust
pub struct Payload([u8; 6561]);
pub struct Address([u8; 243]);
pub struct Value([u8; 81]);
pub struct Tag([u8; 81]);
pub struct Timestamp([u8; 27]);
pub struct Index([u8; 27]);
pub struct BundleHash([u8; 243]);
pub struct TransactionHash([u8; 243]);
pub struct Nonce([u8; 81]);
```

Below is an example implementation for the `Tag`, with conversion from a `&[u8]`. An implementation in terms of `&[i8]`
would look similar. Checked conversions ensuring that each byte encodes `-1`, `0`, or `+1` are implemented in terms of
the `TryFrom` trait, while a `Tag::from_unchecked` function is provided as an unchecked faster constructor method.

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

### `TransactionBuilder`

The `TransactionBuilder` allows setting all the fields of a transaction, and verifies and builds a correct `Transaction`
type. The order of the fields in `TransactionBuilder` follows the table showing the different fields of a transaction,
the same as `Transaction`.

```rust
pub struct TransactionBuilder {
    payload: Option<Payload>,
    address: Option<Address>,
    value: Option<Value>,
    obsolete_tag: Option<Tag>,
    timestamp: Option<Timestamp>,
    index: Option<Index>,
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
some convenience when setting fields from byte slices. The `TryFrom` implementations for each field type ensure that the
byte slices only encode correct trits. If the data comes from a trusted source, the checks can be circumvented by
explicitly constructing the target object and that one in the setter. For example, when setting the `tag` field on
a `builder: TransactionBuilder` object with `trusted_bytes: &[u8]`, calling
`builder.tag(Tag::from_unchecked(trusted_bytes))` sets the `tag` field to a value that was not verified to only
contained valid bytes.

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
}
```

### `Error` types

The `TransactionBuilderError` enum contains variants for each of its fields, encoding that an error can occur when
setting any of them. In addition, it shall contain errors that can occur during verification or construction. The
specification of the concrete error types are left to be elaborated in the implementation phase.

## `Bundle`

Bundles are immutable groups of immutable transactions. Once a bundle is created and validated, it shouldn't be changed.
An instantiated `Bundle` object represents a syntactically and semantically valid IOTA bundle, and can only be created
via the `IncomingBundleBuilder` and `OutgoingBundleBuilder` builder patterns. `IncomingBundleBuilder` is concerned with
absorbing a number of final `Transaction`s coming in over the wire, and verifying their correctness before combining
them into a final `Bundle`. `OutgoingBundleBuilder` is concerned with taking in a number of not yet ready transactions
represented by `TransactionBuilder`s, and then first inserting the bundle hash (and thus sealing the builder, not
allowing to push any more transaction builders), and finally calculating the nonce via proof of work on each transaction
before constructing the final `Bundle`.

### `Bundle`

As bundles are immutable, they shouldn't be modifiable outside of the scope of the bundle module.

There is a natural order to transactions in a bundle that can be represented in two ways:

+ each transaction has a `index` and a `last_index`. `index` goes from `0` to `last_index`. A bundle can then simply be
  represented by a data structure that contiguously keeps the order like `Vec`;
+ each transaction is chained to the next one through its `trunk` hash, which means one can consider using
  a data-structure like `HashMap`.

This RFC opts for using the simplest implementation in terms of `Vec`, but hides the implementation details behind
a newtype. This way, the underlying data-structures can be changed in the future without breaking dependent code.

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

In this block we list all methods that we expect to be part of `IncomingBundleBuilder`. It is not exhaustive and we
expect the specifics to change during the implementation phase. Methods with more information are listed in their
sections below this one.

```rust
impl IncomingBundleBuilder {
    /// Pushes a new transaction coming over the wire into the bundle builder.
    pub push(&mut self, transaction: Transaction) -> &mut Self {
        self.transactions.push(transaction);
        self
    }
}
```

The functions below are all intended as methods on `IncomingBundleBuilder`.

#### `IncomingBundleBuilder::calculate_hash`

Calculates the bundle hash using [`Kerl`] as a sponge. A bundle hash ties different transactions together. By having
this common hash in their `bundle` field, it makes it clear that these transactions should be processed as a whole.

The hash of a bundle is derived from the bundle essence of each of its transactions. The bundle essence of each
transaction is a subset of its fields, with a total size of `486` trits, see the table below.

| Name            | Size (in trits) |
| ---             | ---             |
| `address`       | 243             |
| `value`         | 81              |
| `obsolete_tag`  | 81              |
| `timestamp`     | 27              |
| `index`         | 27              |
| `last_index`    | 27              |

The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.

+ **TODO:** `iri v1.8.1` uses [`Kerl`] as a sponge. A Rust implementation is the matter of a different RFC, and is not
  yet defined. The code below lays out how the sponge is thought to be used.

[`Kerl`]: https://github.com/iotaledger/kerl

```rust
pub fn calculate_hash(&self, mut sponge: Kerl) -> S::Output {
    for transaction in bundle_builder {
        // Transaction::essence is not specifically defined in this RFC.
        sponge.absorb(transaction.essence());
    }
    sponge.squeeze()
}
```

#### `IncomingBundleBuilder::validate`

Validates if the transactions inside the bundle builder are all consistent, and if they form a valid bundle.

**NOTE:** This shares logic with `SealedBundleBuilder::validate`, so one might be implementable in terms of the
other.

```rust
pub fn validate(&self) -> Result<(), IncomingBundleError> {
    unimplemented!()
}
```

### `OutgoingBundleBuilder`

Creating an outgoing message is more involved compared to reading an incoming one. `OutgoingBundleBuilder` is concerned
with collecting not-yet-finished transactions. Once all `TransactionBuilder`s are collected, the `OutgoingBundleBuilder`
is sealed by calculating and setting the bundle hash on each of its contained transaction drafts and setting their
indices, turning it into a `SealedBundleBuilder` (see its design below).

```rust
// NOTE: SealedBundleBuilder, `SignedBundleBuilder`, `AttachedBundleBuilder`, and `ValidatedBundleBuilder` all follow
//       the same pattern.
pub struct OutgoingBundleBuilder {
    transaction_builders: TransactionBuilders,
}

struct TransactionBuilders(Vec<TransactionBuilder>);
```

As with `Transactions`, `TransactionBuilders` are an opaque newtype that wraps a vector of `TransactionBuilder`s.

```rust
impl OutgoingBundleBuilder {
    /// Push a `TransactionBuilder` into the builder.
    pub push(self, transaction_builder: TransactionBuilder) -> Self {
        self.transaction_builders.push(transaction_builder);
        self
    }
```

#### `OutgoingBundleBuilder::calculate_hash`

Calculate the bundle hash using some sponge.

A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it
clear that these transactions should be processed as a whole.

The hash of a bundle is derived from the bundle essence of each of its transactions. A bundle essence is a `486` trits
subset of the transaction fields.

| Name          | Size (in trits) |
| ---           | ---             |
| address       | 243 trits       |
| value         | 81 trits        |
| obsolete_tag  | 81 trits        |
| timestamp     | 27 trits        |
| index         | 27 trits        |
| last_index    | 27 trits        |

The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.

+ **TODO:** `iri v1.8.1` uses [`Kerl`] as a sponge. A Rust implementation is the matter of a different RFC, and is not
  yet defined. The code below lays out how the sponge is thought to be used.

[`Kerl`]: https://github.com/iotaledger/kerl

```rust
pub calculate_hash(&self, mut sponge: Kerl) -> S::Output {
    unimplemented!();

    // Same as IncomingBundleBuilder::calculate_hash
}
```

#### `OutgoingBundleBuilder::seal`

Seals the bundle builder and sets the bundle hash on all transaction builders contained in it.

This function sets all transaction indices and calculates the bundle hash from the bundle essence (see the discussion of
in the section on `OutgoingBundleBuilder::calculate_hash`), setting it for all contained transactions. Because the bundle hash is a function of all transaction indices and the last index, pushing more transactions into the builder
would invalidate the hash. Hence the builder is sealed after calculating and setting all these fields.

**Normalization:** As a side effect of the WOTS signing scheme, the share of the private key being leaked is not
uniform, introducing a risk to leak a critically large part of it. By normalizing the hash, we ensure that exactly half
of the private key is leaked. The actual normalization algorithm is provided in the signing scheme RFC. Useful links:
[Addresses and signatures](https://docs.iota.org/docs/dev-essentials/0.1/concepts/addresses-and-signatures), [Why is the
bundle hash normalized?](https://iota.stackexchange.com/questions/1588/why-is-the-bundle-hash-normalized).

**M-Bug**: Due to the implementation of the signature scheme, the normalized bundle hash can't contain a `M` (or `13`)
because it could expose a significant part of the private key, making it easier for attackers to forge signatures. The
bundle hash is then repetitively generated with a slight modification until its normalization doesn't contain a `M`.
This modification is usually operated by incrementing the `obsolete_tag` of the first transaction of the bundle since
this field is not being used for any other reason. Useful links: [Why is the normalized hash considered insecure when
containing the char
'M'](https://iota.stackexchange.com/questions/1241/why-is-the-normalized-hash-considered-insecure-when-containing-the-char-m).

+ **TODO:** `iri v1.8.1` uses [`Kerl`] as a sponge. A Rust implementation is the matter of a different RFC, and is not
  yet defined. The code below lays out how the sponge is thought to be used.
+ **FIXME:** This pseudo code does not take into account that depending on the security level, the signature is
  calculated from one or more transactions.

[`Kerl`]: https://github.com/iotaledger/kerl

```rust
pub fn seal(self, sponge: Kerl) -> Result<SealedBundleBuilder, OutgoingBundleBuilderError> {
    unimplemented!()

    let mut index = 0;

    for transaction_builder in bundle_builder {
        transaction_builder.index = index++;
        transaction_builder.last_index = bundle_builder.size() - 1;
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

    for transaction_builder in &mut bundle_builder {
        transaction_builder.bundle = final_hash;
    }

    Ok(SealedBundleBuilder::from(self))
}
```

### `SealedBundleBuilder`

`SealedBundleBuilder` encodes on the type level that no more transaction drafts will be pushed into the bundle, and
signals that all the fields required to identify a valid bundle are set.

After `OutgoingBundleBuilder` has collected all required transactions, it is sealed to create a `SealedBundleBuilder`.
`SealedBundleBuilder` is concerned with signing transactions that remove IOTA tokens from an address, settings nonce
fields, and verifying that all transactions are valid and the bundle is complete, before building a final `Bundle`.

#### `SealedBundleBuilder::chain_and_attach`

Chains the contained transactions, filling the `nonce` fields via proof of work and then hashing transactions to
reference successive transactions.

Using `PearlDiver` as proof of work algorithm, this function performs proof of work and updates the `nonce` field in each
of the contained `TransactionBuilder`s. The transaction hash can then be calculated, and a chain is established between
transactions by putting the hash of a transaction into its downstream neighbor.

After proof of work, a bundle is ready to be sent to and accepted by the network. Given a `trunk` and a `branch`
returned by [`getTransactionsToApprove`], the process iterates over each transaction of the bundle, sets some attachment
related fields and performs proof of work on each individual transaction.

[`getTransactionsToApprove`]: https://docs.iota.org/docs/node-software/0.1/iri/references/api-reference#gettransactionstoapprove

+ **TODO:** This function relies on `Curl-P` for the calculation of transaction hashes, and `PearlDiver` as a proof of work algorithm, which are
both not defined yet. `chain_and_attach` uses them to show how they are thought to be used.

```rust
pub fn chain_and_attach<P, S>(
    mut self,
    trunk_tip: TransactionHash,
    branch_tip: TransactionHash,
    proof_of_work: PearlDiver,
    sponge: CurlP,
) -> Result<AttachedBundleBuilder, OutgoingBundleError>
{
    unimplemented!()

    // Iterates over all transactions starting from the head, setting trunks, branches, timestamps, tags, and nonce.
    //
    // NOTE: We do not provide an implementation here.
    let builder_iterator = self.iter_from_head();

    // Set the head transaction
    let head_transaction = builder_iterator.next().ok_or("builder does not contain any transaction builders")?;
    head_transaction.trunk(trunk_tip);
    head_transaction.branch(branch_tip);

    // FIXME: Explain these numbers.
    head_transaction.attachment_timestamp = timestamp();
    head_transaction.attachment_timestamp_lower = 0;
    head_transaction.attachment_timestamp_upper = 3812798742493;

    if head_transaction.tag.is_empty() {
        head_transaction.tag = transaction_builder.obsolete_tag;
    }
    head_transaction.set_nonce_from_proof_of_work(proof_of_work);

    // `next_local_trunk` is the hash that will be used as value of the current transaction pointing to the previous
    // one.
    let mut next_local_trunk = head_transaction.calculate_hash(sponge);

    // Set the rest of the transactions
    for transaction_builder in builder_iterator {
        transaction_builder.branch(trunk_tip);
        transaction_builder.trunk(next_local_trunk);

        transaction_builder.attachment_timestamp = timestamp();

        // FIXME: Explain these numbers.
        transaction_builder.attachment_timestamp_lower = 0;
        transaction_builder.attachment_timestamp_upper = 3812798742493;

        if transaction_builder.tag.is_empty() {
            transaction_builder.tag = transaction_builder.obsolete_tag;
        }

        transaction_builder.set_nonce_from_proof_of_work(proof_of_work);
        next_local_trunk = transaction_builder.calculate_hash(sponge);
    }

    Ok(AttachedBundleBuilder::from(self))
}
```

#### `AttachedBundleBuilder::sign`

Signs those transaction builders contained in the builder that withdraw funds from an address and transitions
`AttachedBundleBuilder` to `SignedBundleBuilder`.

The purpose of this function is not only to sign those transactions that require a signature, but also to signal that
all other remaining transactions can remain unsigned.

Only the owner of the seed from which the source address is generated has the right to remove funds from it. The seed is
thus used to generate the signature, which in turn is inserted in those transaction drafts that withdraw funds. Bundles
that contain withdrawing transactions that have missing or bad signatures will be considered invalid and thus rejected.

+ **TODO:** This function relies on IOTAs `WinternitzScheme` signature scheme and a `Wallet` that is supposed to provide the input
  necessary to sign a transaction together with the signature scheme. Both are not defined yet, and the code below is
  intended to demonstrate how both are to be used.
    + The wallet is supposed to provide some function `f: Wallet -> Tx Address -> Signature Scheme Inputs`
+ **FIXME:** The code below does not yet take into account that a signature can include one or more transactions.

```rust
pub fn sign<I, S, W>(self, signature_scheme: WinternitzScheme, seed: SignatureSeed, wallet: Wallet)
-> Result<SignedBundleBuilder, OutgoingBundleError>
{
    unimplemented!();

    // Pseudocode how `sign` should be used.

    let mut index = 0

    // TODO: Explain what this loop is trying to achieve.
    for transaction_builder in self {
        if transaction_builder.value < 0 {
            if transaction_builder.index >= index {
                let input = inputs[transaction_builder.address];

                // NOTE: The specific arguments to the signature scheme depend on the implementation of `SignatureScheme`.
                //
                // NOTE: The specific arguments to wallet depend on the implementation of `Wallet`.
                // FIXME: The sign function has to operate on several transactions, so this is not the right way of doing it.
                let fragments = transaction_builder.sign(signature_scheme, seed, wallet.get(transaction_builder.address));

                for fragment in fragments {
                    bundle_builder[index].signature = fragment;
                    index = index + 1
                }
            }
        } else {
            index = index + 1;
        }
    }

    Ok(SignedBundleBuilder::from(self))
}
```

### `SignedBundleBuilder`

`SignedBundleBuilder` encodes on the type level that all transactions within the bundle that need to be signed have been
signed, and that all other contained transactions do not require a signature.

#### `SealedBundleBuilder::validate`

Validates a bundle, returning an error if it's invalid.

Validating a bundle means checking the syntactic and semantic integrity of a bundle as a whole and of its constituent
transactions. As bundles are atomic transfers, either all or none of the transactions will be accepted by the network.
After validation, transactions of a bundle are candidates to be included to the ledger.

For a bundle to be considered valid, the following assertions must be true:

+ bundle has the correct length;
+ transactions share the same bundle hash;
+ transactions' absolute value doesn't exceed total IOTA supply;
+ bundle absolute sum never exceeds total IOTA supply;
+ order of transactions in the bundle is the same as announced by `index` and `last_index`;
+ input/output transactions have an address ending in `0` i.e. has been generated with Kerl;
+ bundle inputs and outputs are balanced i.e. the bundle sum equals `0`;
+ announced bundle hash matches the computed bundle hash.

+ **TODO:** `IOTA_SUPPLY` used below is some global constant which needs to be defined and set externally.

```rust
pub fn validate(&mut self) -> Result<ValidatedBundleBuilder, OutgoingBundleError>
{
    unimplemented!()

    let mut value = 0;
    let mut index = 0;

    // NOTE: We are not defining `first_transaction` here but leave it to the implementation phase.
    if bundle_builder.len() != bundle_builder.first_transaction().last_index + 1 {
        Err(InvalidLength)?
    }

    let bundle_hash = bundle_builder.first_transaction().bundle_hash;
    let last_index = bundle_builder.first_transaction().last_index;

    for transaction_builder in bundle_builder {
        if transaction_builder.bundle_hash != bundle_hash {
            Err(InvalidHash)?
        }

        if abs(transaction_builder.value) > IOTA_SUPPLY {
            Err(InvalidTransactionValue)?
        }

        value += transaction_builder.value;
        if abs(value) > IOTA_SUPPLY {
            Err(InvalidValue)?
        }

        if transaction_builder.index != index++ {
            Err(InvalidIndex)?
        }

        if transaction_builder.last_index != last_index {
            Err(InvalidIndex)?
        }

        if transaction_builder.value != 0 && transaction_builder.address.last != 0 {
            Err(InvalidAddress)?
    }

    if value != 0 {
        Err(InvalidValue)?
    }

    if bundle_hash != self.calculate_hash() {
        Err(InvalidHash)?
    }

    Ok(ValidatedBundleBuilder::from(self))
}
```

## How this is used

This section shows how to construct a `Bundle`. The work flow depends on whether one is receiving a bundle (and its
constituent transactions), or creating a new one to send it. `IncomingBundleBuilder` encodes receipt of a bundle, while
`OutgoingBundleBuilder` encodes sending.

### Workflow of `IncomingBundleBuilder`

The workflow for an incoming bundle would look like this:

```rust
// Push transactions into the the incoming bundle builder
for transaction in transactions {
    incoming_bundle_builder.push(transaction);
}

// Validate the bundle; this can be skipped if the transactions come from a trusted source.
let bundle = match incoming_bundle_builder.validate() {
    Ok(()) => incoming_bundle_builder.build()?,
    Err(e) => Err(e)?,
};
```

### Workflow of `OutgoingBundleBuilder`

The construction of an outgoing bundle would be encoded on the type level by moving through the following types:

```
OutgoingBundleBuilder -> SealedBundleBuilder -> SignedBundleBuilder -> AttachedBundleBuilder -> ValidatedBundleBuilder -> Bundle
```

A user of the library would not however need to know about the specifics of these types as they are never bound directly:

```rust

let mut bundle_builder = OutgoingBundleBuilder::new();

// Push transaction drafts into the builder
for builder in transaction_builders {
    outgoing_bundle_builder.push(builder);
}

let bundle = bundle_builder
    // Seals the bundle builder to prevent pushing any more transaction builders into it.
    .seal()?
    // skipped.
    // Check if there are withdrawal transactions and sign those using some signature scheme, seed, and wallet.
    .sign(signature_scheme, seed, wallet)?
    // Attach to the tangle, chain transactions to each other by performing proof of work and calculating transaction hashes via the sponge.
    .chain_and_attach(trunk_tip, branch_tip, proof_of_work, sponge)?
    // Checks if all transactions contained in the sealed builder are consistent.
    .validate(sponge)?
    // Constructs the final `Bundle`.
    .build()?;
```

# Drawbacks

+ There might be use cases that require the ability to directly create a `Transaction`, requiring exposing a builder
  pattern.
+ This proposal does not consider any kind of generics or abstraction over transactions and bundles. Future iterations
  of the IOTA network that have different transaction and/or bundle formats will probably require new types.

# Rationale and alternatives

+ Immutable `Transaction`s and `Bundle`s encode the fact that they should not be mutated after creation.
+ This proposal is very straight forward and idiomatic Rust. It does not contain any overly complicated parts. As such,
  it should be easy to extend it in the future.
+ Hiding the fields of `Transaction`, `Bundle`, and the builders behind opaque types allows us to flesh them out in the
  future.
+ The construction of an outgoing bundle is done in steps by walking through the types `OutgoingBundleBuilder`,
  `SealedBundleBuilder`, `SignedBundleBuilder`, `AttachedBundleBuilder`, and `ValidatedBundleBuilder` before finally
  constructing a `Bundle`. This encodes on the type level that invariants are upheld. By only allowing to go through the
  stages linearly, the final validation function does not need to validate the correctness of hashes, signatures, etc.

## Alternatives

+ **Smarter bundle builder:** An alternative to the present design would be to create a smarter bundle builder, that
  works by taking a message (be it a withdrawal of tokens or any other information) and automatically creating the
  required transaction drafts and pushing them onto its stack. However, at present it is not clear how this interface
  would look like and it would introduce a lot of complexity early on. The present design is intended to allow for
  building such an API on top of it.
+ **Generic builders:** The present transaction and bundle builders could be made generic over hashing
  functions/sponges, proof of work algorithms, and signing schemes. This would basically amount to having a fixed
  `Transaction` format following `iri v1.8.1`, but with different ways to fill its fields. This was decided to be not
  meaningful, because using, e.g., different sponges would constitute a breaking change with respect to the current IOTA
  main net. If such a step were to be taken, it would probably come with a change in the transaction structure, and thus
  require a completely new type, rendering the more general form of the present `Transaction` useless.

# Open questions

+ The design of the builders currently take ownership and return `Self`. This is nice for chaining calls, but might
  suboptimal for performance. Is it more useful to take borrows, either `&Self` or `&mut Self`, instead?
+ `getTransactionsToApprove` function: the function executing proof of work on `SealedBundleBuilder` is talking about
  getting tip and branch from the mentioned function. Should it be part of this RFC or assumed external?
+ Work out how to sign groups of transactions taken together (signature levels 1, 2, and 3 require the same number of
  transactions).
+ Explain constants used throughout the code.
+ Should the verification of the validity of fields in the outgoing builder be split up into the different transition
  phases?
+ Should a `TransactionBuilder` only expose those fields that are not set within a bundle builder?

# Blockers

+ `WinternitzScheme` as a signature scheme with normalization, to sign withdrawal transactions;
+ `Wallet`: a map from a transaction address to the inputs of the `WinternitzScheme` signature scheme;
+ `PearlDiver` proof of work algorithm to calculate and fill the nonce;
+ `Kerl` and `Curl-P` sponges to calculate various hashes;
+ `Constants`: There are global constant which need to be defined and set externally; example: `IOTA_SUPPLY`.

## Other, less specified blockers

This RFC is intended to be self contained and thus does not rely on the existence of any traits or types defined outside
of this proposal. There are however a few aspects of this proposal that would benefit from additional interface
definitions to make its types easier and more idiomatic to use, or in order to make it more type-safe. We list these here:
+ `Transaction` and its fields are heavily tied to the message format of `iri v1.8.1`, the fixed nature of its fields
  and the representation of each of its fields as `one byte per trit` is the obvious choice. However, equipping the
  field types with more semantics, for example implementing it in terms of a `BinaryCodedTernary` trait might be useful
  to enforce invariants.
+ Serialization (and deserialization) of transactions and bundles and its builders is done by concatenating the bytes of
  all its fields. It might be useful in the future to define traits to allow external code to take a transaction and
  serialize it.