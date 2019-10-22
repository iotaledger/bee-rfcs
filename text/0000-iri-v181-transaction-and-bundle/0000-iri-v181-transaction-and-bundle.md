+ Feature name: `iri_v181_transaction_and_bundle`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3),
  [iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18),
  [iotaledger/bee-rfcs#20](https://github.com/iotaledger/bee-rfcs/pull/20)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

The fundamental communication unit in the IOTA protocol is the transaction. Messages, including payment settlements and
plain data, are propagated through the IOTA network in transactions. Message that are too large are split up into
several transactions.

In total, a transaction is 8019 trits large, and its payload --- placed in its `sig_or_msg` field --- 6561 trits. The
payload is defined to either hold a signature fragment or a message fragment. Since it has a limited size, a user often
needs more than one transaction to execute an operation. For example, signatures with [security level 2 or 3](Security
levels) don't fit in a single transaction, and user-provided messages may exceed the maximum payload capacity so they
need to be fragmented across multiple transactions. Moreover, because the total amount of tokens stored in the ledger
has to stay constant, *input transactions* (which are called thus because an address is *put into* a transfer as
a source of tokens, thus removing them from the address) have to always be matched with *output transactions* such that
their total value is zero. For these reasons, transactions have to be processed as a whole in groups called bundles.
A bundle is an atomic operation in the sense that either all or none of its transactions are accepted by the network.
Even single transactions are propagated through the network within a bundle.

By analogy with IP fragmentation, a bundle corresponds to a packet, while transactions correspond to fragments.

This RFC proposes a `Transaction` type and a `Bundle` type to represent the transaction and bundle formats used by the
IOTA Reference Implementation as of release [`iri v1.8.1`]. We also propose the `TransactionBuilder`,
`IncomingBundleBuilder`, and `OutgoingBundleBuilder` types following the builder pattern to construct both message
types, respectively. We distinguish between incoming and outgoing bundles because one is concerned with verifying the
veracity of an existing message being received, while the other one is concerned with constructing a new message
intended to be sent, which includes setting a number of fields like the bundle hash.

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

The `Transaction` and `Bundle` types are intended to be final and immutable. This contract is enforced by not exposing
any methods that allow manipulating their fields directly. `Transaction`s are intended to be constructed through
the `TransactionBuilder` builder pattern, while `Bundle`s can be created either via the `IncomingBundleBuilder` or
`OutgoingBundleBuilder`. The `IncomingBundleBuilder` takes in existing `Transaction`s, compiles them, and verifies
their veracity before constructing a `Bundle` type. The `OutgoingBundleBuilder` takes in `TransactionBuilder`s,
and is primarily responsible for filling in all information that can only be set in the actual finalization
of the outgoing message into a `Bundle`, such as the bundle hash.

[`iri v1.8.1`]: https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE

# Detailed design

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

### Transaction struct

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

We treat the types of the `Transaction` struct's fields as opaque newtypes for now so that we can flesh them out during
the implementation phase or in future RFCs. While IOTA is operating on balanced binary-coded ternary, we choose to not
introduce specific types for ternary here, and instead just treat each field as a bunch of bytes.

```rust
pub struct SignatureOrMessageFragment([u8; 6561]);
pub struct Address([u8; 243]);
pub struct Value(i64);
pub struct Tag([u8; 81]);
pub struct Timestamp(u64);
pub struct Index(u64);
pub struct BundleHash([u8; 243]);
pub struct TransactionHash([u8; 243]);
pub struct Nonce([u8; 81]);
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

    ...
}
```

### `TransactionBuilder` struct

The `TransactionBuilder` allows setting all the fields of a transaction.

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

Since the fields are all newtypes and in order to make the setter methods convenient to use, we implement them using the
`TryInto` trait. This implies that we implement `TryFrom` for all the different newtypes in `TransactionBuilder` to
convert from those types where appropriate. We leave determining which types to use to the implementation phase. Types
that come to mind already now are `&str` or `String` to allow setting a field from a combination of
`"9ABCDEFGHIJKLMNOPQRSTUVWXYZ"`, the permitted characters representing an IOTA tryte (this is where the requirement for
`TryFrom` comes from, rather than `From`, because not all utf-8 characters encoded in a Rust `String` are allowed, and
conversion can thus fail).

```rust
enum Error {
    InvalidTryteChar,
    WrongLength { expected: usize, given: usize },
}

impl fmt::Display for Error {
    fn display(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        use Error::*;

        let msg: Cow<'_, str> = match self {
            InvalidTryteChar =>
                "Tried converting invalid tryte character from string. Only one of '9ABCDEFGHIJKLMNOPQRSTUVWXYZ' may represent a tryte.".into(),

            WrongLength { expected, given } =>
                format!("Expected {} characters, found {} in String").into(),
        };

        fmt!(f, &msg)
    }
}

impl error::Error for Error {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}

impl TryFrom<String> for Tag {
    type Error = Error;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        if value.len() != 27 {
            Err(Error::WrongLength { expected: 27, given: value.len()})?;
        }

        let mut tryte_array = [0u8; 27];
        for (tryte, byte) in tryte_array.iter_mut().zip(value.bytes()) {
            *tryte = match byte {
                b'9' => 0,
                b'A' ..= b'M' => byte - b'A' + 1,
                b'N' ..= b'Z' => 255 - 'Z' + byte,
                _ => Err(Error::InvalidTryteChar)?,
            }
        }
    }
}

impl TransactionBuilder {
    fn tag<T: TryInto<Tag>>(&mut self, tag: T) -> Result<&mut Self, TransactionBuilderError> {
        self.tag.replace(tag.try_into()?);
        self
    }
```

### Error types

<!-- Fill this in -->


## Bundle

Bundles are immutable groups of immutable transactions. Once a bundle is created and validated, it shouldn't be changed.
An instantiated `Bundle` object represents a syntactically and semantically valid IOTA bundle, and can only be created
via the `IncomingBundleBuilder` and `OutgoingBundleBuilder` builder patterns. We distinguish between these builders,
because the `IncomingBundleBuilder` is concerned with absorbing a number of final `Transaction`s coming in over the
wire, and verifying their correctness before combining them into a final `Bundle`. The `OutgoingBundleBuilder` is
concerned with taking in a number of not yet ready transactions represented by `TransactionBuilder`s, and then first
inserting the bundle hash, and finally calculating the nonce via proof of work on each transaction before constructing
the final `Bundle`.

### Bundle struct

As bundles are immutable, they shouldn't be modifiable outside of the scope of the bundle module.

There is a natural order to transactions in a bundle that can be represented in two ways:
+ each transaction has a `current_index` and a `last_index` and `current_index` goes from `0` to `last_index`, a bundle
  can then simply be represented by a data structure that contiguously keeps the order like `Vec`;
+ each transaction is chained to the next one through its `trunk` which means we can consider data structures like
  `HashMap` or `BTreeMap`;

For this reason, we hide this as an implementation detail and instead provide a newtype:

```rust
pub struct Transactions(Vec<Transaction>)
// pub struct Transactions(HashMap<Transaction>)
// pub struct Transactions(BTreeMap<Transaction>)
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

### `IncomingBundleBuilder` struct

The `InocmBundleBuilder` offers a simple and convenient way to build bundles:

```rust
pub struct BundleBuilder {
    transaction_drafts: TransactionDrafts,
}
```

*`TransactionDrafts` is a constructor for `Transaction`s, but since we don't expose the `transaction_drafts`
field publicly, it remains an implementation detail for the end user.*

```rust
impl BundleBuilder {    
    pub fn add_message(&self, address: Address, tag: Tag, message: &[i8]) {
        unimplemented!()
    }

    pub fn add_input(&self, address: Address, tag: Tag, value: Value, security: u8) {
        unimplemented!()
    }

    pub fn add_output(&self, address: Address, tag: Tag, value: Value) {
        unimplemented!()
    }

    pub fn calculate_hash(&self) -> BundleHash {
        unimplemented!()
    }

    pub fn finalize(&self) {
        unimplemented!()
    }

    pub fn sign(&self, seed: Seed, inputs: Inputs) {
        unimplemented!()
    }

    pub fn calculate_proof_of_work(&self, mut trunk: TransactionHash, mut branch: TransactionHash, mwm: MinimumWeightMagnitude) {
        unimplemented!()
    }

    pub fn validate(&self) -> Result<(), BundleValidationError> {
        unimplemented!()
    }

    pub fn build(&self) {
        unimplemented!()
    }
}
```

## Algorithms

In this section, we describe the algorithms needed to build a `Bundle`. The lifecycle of a `BundleBuilder` depends on
if it's being used in client side or server side:

+ client side: `[add_message/add_input/add_output ->] finalize -> [sign ->] [pow ->] validate -> build`
+ server side: `add_transaction_draft -> validate -> build`

*`sign` is optional because data transactions don't have to be signed. `pow` is optional because one can use remote
pow instead.*

### Add message

*Client side operation.*

Given an address, a tag and a message, this function adds the message to the bundle builder by splitting it into chunks
of 6561 trits (size of an individual `signature_or_message_fragment` field) and spreading it across newly created
transactions.

Rust pseudocode:

```rust
fn add_message(bundle_builder: BundleBuilder, address: Address, tag: Tag, message: &[i8]) {
    for chunk in message.chunks(6561) {
        let mut draft: TransactionDraft = TransactionDraft::new();

        draft.address = address;
        draft.tag = tag;
        draft.message = chunk;
        bundle_builder.push_back(draft);
    }
}
```

### Add input/output

Given an address, a tag and a value, these functions creates new input and/or output transactions and add them to the
bundle builder. In case of input transaction, and depending on the security level, enough transactions will be created
to hold a full signature.

```rust
fn add_input(bundle_builder: BundleBuilder, address: Address, tag: Tag, value: Value, security: u8) {
    for i in 0..security {
        let mut draft: TransactionDraft = TransactionDraft::new();

        draft.address = address;
        draft.tag = tag;
        if i == 0 {
            draft.value = value;
        } else {
            draft.value = 0;
        }
        bundle_builder.push_back(draft);
    }
}
```

```rust
fn add_output(bundle_builder: BundleBuilder, address: Address, tag: Tag, value: Value) {
    let mut draft: TransactionDraft = TransactionDraft::new();

    draft.address = address;
    draft.tag = tag;
    draft.value = value;
    bundle_builder.push_back(draft);
}
```

### Calculate hash

*Client side and server side operation.*

A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it
clear that these transactions should be processed as a whole.

The hash of a bundle is derived from the bundle essence of each of its transactions. A bundle essence is a `486` trits
subset of the transaction fields.

| Name          | Size      |
| ------------- | --------- |
| address       | 243 trits |
| value         | 81 trits  |
| obsolete_tag  | 81 trits  |
| timestamp     | 27 trits  |
| current_index | 27 trits  |
| last_index    | 27 trits  |

The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the
bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.

Rust pseudocode:

```rust
fn calculate_hash(bundle_builder: BundleBuilder) -> BundleHash {
    let mut sponge: Sponge<Kerl> = Sponge::new();

    for transaction_draft in bundle_builder {
        sponge.absorb(transaction_draft.essence());
    }

    sponge.squeeze()
}
```

### Finalize

*Client side operation.*

Finalizing a bundle means computing the bundle hash, verifying that it matches the security requirement and setting it
to all the transactions of the bundle. After finalization, transactions of a bundle are ready to be safely attached to
the tangle. Finalization is also the place to set the transaction indexes since the bundle essence contains all
`current_index` and `last_index` and we wouldn't be able to alter them after the bundle hash has been calculated.

Rust pseudocode:

```rust
fn finalize(bundle_builder: BundleBuilder)
    let mut current_index = 0;

    for transaction_draft in bundle_builder {
        transaction_draft.current_index = current_index++;
        transaction_draft.last_index = bundle_builder.size() - 1;
    }

    let final_hash = loop {
        // Use the `calculate_hash` function defined above.
        let hash = calculate_hash(&bundle_builder);
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
```
**Normalization**: during signing scheme, the share of the private key being leaked is not uniform, introducing a risk
to leak a critically large part of it. By normalizing the hash, we ensure that exactly half of the private key is
leaked. The actual normalization algorithm is provided in the signing scheme RFC.
Useful links: [Addresses and signatures](https://docs.iota.org/docs/dev-essentials/0.1/concepts/addresses-and-signatures),
[Why is the bundle hash normalized?](https://iota.stackexchange.com/questions/1588/why-is-the-bundle-hash-normalized).

**M-Bug**: due to the implementation of the signature scheme, the normalized bundle hash can't contain a `M` (or `13`)
because it could expose a significant part of the private key, making it easier for attackers to forge signatures.
The bundle hash is then repetitively generated with a slight modification until its normalization doesn't contain a `M`.
This modification is usually operated by incrementing the `obsolete_tag` of the first transaction of the bundle since
this field is not being used for any other reason. Useful links:
[Why is the normalized hash considered insecure when containing the char 'M'](https://iota.stackexchange.com/questions/1241/why-is-the-normalized-hash-considered-insecure-when-containing-the-char-m).

### Sign

*Client side operation.*

Signing a bundle allow you to prove that you are the owner of the address you are trying to move funds from. With no
signature or a bad signature, the bundle won't be considered valid and the funds won't be moved. Only the owner of the
right seed is able to generate the right signature for this address.

Rust pseudocode:

```rust
fn sign(bundle_builder: BundleBuilder, seed: Seed, inputs: Inputs) {
    let mut current_index = 0

    // TODO: Explain what this loop is trying to achieve.
    for transaction_draft in bundle_builder {
        if transaction_draft.value < 0 {
            if transaction_draft.current_index >= current_index {
                let input = inputs[transaction_draft.address];
                // FIXME: This `sign` function below seems to be a different sign function compares to the one above?
                //        It has a different signature.
                let fragments = sign(seed, input.index, input.security, transaction_draft.bundle);
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
```

*Since signature size depends on the security level, a single signature can spread out to up to 3 transactions.
`inputs` is an object that contains all unused addresses of a seed with a sufficient balance.*

### Proof of Work

*Client side operation.*

Proof of Work (PoW) allows your transactions to be accepted by the network. On the IOTA network, PoW is only a rate
control mechanism.  After PoW, a bundle is ready to be sent to the network. Given a `trunk` and a `branch` returned by
[getTransactionsToApprove](https://docs.iota.org/docs/node-software/0.1/iri/references/api-reference#gettransactionstoapprove)
and a minimum weight magnitude `mwm` which represents the difficulty of the PoW, the process iterates over each
transaction of the bundle, sets some attachment related fields and does individual PoW.

Rust pseudocode:

```rust
fn calculate_proof_of_work(bundle_builder: BundleBuilder, mut trunk: TransactionHash, mut branch: TransactionHash, mwm: MinimumWeightMagnitude, pow: Fn(...)->...) {
    for transaction_draft in rev(&mut bundle_builder) {
        transaction_draft.trunk = trunk;
        transaction_draft.branch = branch;
        if transaction_draft.current_index == transaction_draft.last_index {
            branch = trunk;
        }
        transaction_draft.attachment_timestamp = timestamp();
        transaction_draft.attachment_timestamp_lower = 0;
        transaction_draft.attachment_timestamp_upper = 3812798742493;
        if transaction_draft.tag.is_empty() {
            transaction_draft.tag = transaction_draft.obsolete_tag;
        }
        trunk = transaction_draft.pow(mwm);
    }
}
```

Useful links: [Minimum weight magnitude](https://docs.iota.org/docs/dev-essentials/0.1/concepts/minimum-weight-magnitude).

### Validate

*Client side and server side operation.*

Validating a bundle means checking the syntactic and semantic integrity of a bundle as a whole and of its constituent
transactions. As bundles are atomic transfers, either all or none of the transactions will be accepted by the network.
After validation, transactions of a bundle are candidates to be included to the ledger.

For a bundle to be considered valid, the following assertions must be true:

+ bundle has announced size;
+ transactions share the same bundle hash;
+ transactions absolute value doesn't exceed total IOTA supply;
+ bundle absolute sum never exceeds total IOTA supply;
+ order of transactions in the bundle is the same as announced by `current_index` and `last_index`;
+ input/output transactions have an address ending in `0` i.e. has been generated with Kerl;
+ bundle inputs and outputs are balanced i.e. the bundle sum equals `0`;
+ announced bundle hash matches the computed bundle hash;
+ for input transactions, the signature is valid;

Rust pseudocode:

```rust
fn validate(bundle_builder: BundleBuilder) -> Result<(), BundleValidationError> {
    use BundleValidationError::*;
    let mut value = 0;
    let mut current_index = 0;

    if bundle_builder.length() != bundle_builder.first_transaction().last_index + 1 {
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

    // Use the `calculate_hash` function defined above.
    if bundle_hash != calculate_hash(bundle_builder) {
        Err(InvalidHash)?
    }

    if !validate_bundle_signatures(bundle_builder) {
        Err(InvalidSignature)?
    }

    Ok(())
```

**Note:** `validate_bundle_signatures` is a function that actually checks the validity of signatures. It is defined in
the signing scheme RFC.

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

# Unresolved questions

+ How does remote PoW works ?
