+ Feature name: (fill me in with a unique ident, `my_awesome_feature`)
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3),
[iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18),
[iotaledger/bee-rfcs#20](https://github.com/iotaledger/bee-rfcs/pull/20)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

## Transaction

In the context of IOTA, a `Bundle` is a message that is shared over the network, and a `Transaction` is the fundamental
unit that a `Bundle` is constructed from. To use TCP as a well known analogy, a `Bundle` corresponds to a message, and a `Transaction` corresponds to a packet.

This RFC proposes a `Transaction` type to represent the transaction format used by the IOTA Reference Implementation as
of version [`iri v1.8.1`], commit `e1776fbad5d90df86a26402f9025e4b0b2ef7d3e`. The `Transaction` type is implemented as a
struct and is intended to be a public interface. It thus comes with a number of opaque types for its fields that we
leave to be fleshed out at a later time.

We also propose a `TransactionDraft` type that follows the builder pattern. It is intended to be an implementation
detail and only be used by a `Bundle` builder type and not exposed publicly.

## Bundle

The smallest communication unit in the IOTA protocol is the transaction. Everything, including payment settlements
and/or plain data, is propagated through the IOTA network in transactions.

A transaction is `8019` trits and the payload (i.e. `sig_or_msg`) is `6561` trits. This payload holds a signature in
case of a payment settlement and plain data otherwise. Since it has a limited size, a user often needs more than one
transaction to fulfil their operation, for example signatures with security level `2` or `3` don't fit in a single
transaction and user-provided data may exceed the allowance so they need to be fragmented across multiple transactions.
Moreover, a value transaction doesn't make sense on its own because it would change the total amount of the ledger so
it has to be paired with other complementary transactions that together will balance the total value to zero.

For these reasons, transactions have to be processed as a whole, in groups called bundles. A bundle is an atomic
operation in the sense that either all or none of its transactions are accepted by the network. Even single
transactions are propagated through the network within a bundle making it the only confirmable communication unit of
the IOTA protocol.

This RFC proposes ways to create and manipulate a bundle and describe the associated algorithms.

Useful links:
+ [Trinary](https://docs.iota.org/docs/dev-essentials/0.1/concepts/trinary)
+ [What is a bundle?](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-bundle)
+ [Bundles and transactions](https://docs.iota.org/docs/dev-essentials/0.1/concepts/bundles-and-transactions)
+ [Structure of a bundle](https://docs.iota.org/docs/dev-essentials/0.1/references/structure-of-a-bundle)

# Motivation

## Transaction

IOTA is a transaction settlement and data transfer layer for IoT (the Internet of Things). Messages on the network are
`Bundle`s of individual `Transaction`s, which in turn are sent one at a time and are stored in a distributed ledger
called the *Tangle*. Each `Transaction` encodes data such as sender and receiver addresses, reference nodes in the
Tangle, `Bundle` hash, timestamps, and other information required to verify and process each transaction. With the
transaction being the fundamental unit that is sent over the network, we need to represent it in memory.

At the time of this RFC, the transaction format used by the IOTA network is defined by [release v1.8.1 of the IOTA
Reference Implementation, IRI](`iri v1.8.1`), commit `e1776fbad5d90df86a26402f9025e4b0b2ef7d3e`. The transaction format
might change in the future, but for the time being it is not yet understood how a common interface between different
versions transaction formats should be implemented, or if the network will support several transaction types
simultaneously.

We thus do not consider generalizations over or interfaces for transactions, but only propose a basic `Transaction`
type. All mentions of *transactions* in general or the `Transaction` type in particular will be implicitly in reference
to that format used by `iri v1.8.1`.

The `Transaction` type is intended to be *final* and immutable. This contract is enforced by not expose any methods that
allow manipulating its fields directly. The only public constructor method that allows creating a `Transaction` directly
are the `from_reader` and `from_slice` method to construct it from a type implementing `std::io::Read`, or from byte
slice. Otherwise, `Transaction`s should only be built through `Bundle` constructors, and only from that context direct
access to fields is permitted. To this purpose, this RFC contains the `TransactionDraft` builder pattern type.

[`iri v1.8.1`]: https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE

## Bundle

<!-- TODO -->

# Detailed design

## Transaction

A transaction is a sequence of 15 fields of constant length. A transaction has a total length of 8019 trits. The fields'
name, description, and size (in trits) is summarized in the table below, in their order of appearance in a transaction.

| Name                              | Description                                            | Size (in trits) |
| ---                               | ---                                                    | ---             |
| `signature_or_message_fragment`   | contains the signature of the transfer                 |                 |
|                                   | or user-defined message data                           | 6561            |
| `address`                         | receiver (output) if value > 0,
|                                   | or sender (input) if value < 0                         | 243             |
| `value`                           | the transferred amount in IOTA                         | 81              |
| `obsolete_tag`                    | another arbitrary user-defined tag                     | 81              |
| `timestamp`                       | the time when the transaction was issued               | 27              |
| `current_index`                   | the position of the transaction in its bundle          | 27              |
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
    bundle_hash: BundleHash,
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
the implementation phase or in future RFCs.

```rust
pub struct Trit(u8);
pub struct SignatureOrMessageFragment([Trit; 6561]);
pub struct Address([Trit; 243]);
pub struct Tag([Trit; 81]);
pub struct Timestamp(u64);
pub struct Index(u64);
pub struct Value(i64);
pub struct BundleHash([Trit; 243]);
pub struct TransactionHash([Trit; 243]);
pub struct Nonce([Trit; 81]);
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

The `from_slice` and `from_reader` methods will use a `TransactionDraft` internally, but these are implementation
details.

The `TransactionError` is not fully fleshed out yet. For the time being, it definitely contains `io::Error`:

```rust
pub enum TransactionError {
    Io(io::Error),
}
```

#### `TransactionDraft`

`TransactionDraft` is a basic builder pattern for creating and setting the individual fields of a new transaction.

```rust
#[derive(Default)]
struct TransactionDraft {
    signature_or_message_fragment: Option<SignatureOrMessageFragment>,
    address: Option<Address>,
    value: Option<Value>,
    obsolete_tag: Option<ObsoleteTag>,
    current_index: Option<CurrentIndex>,
    last_index: Option<LastIndex>,
    bundle_hash: Option<BundleHash>,
    trunk: Option<Trunk>,
    branch: Option<Branch>,
    tag: Option<Tag>,
    attachment_timestamp: Option<AttachmentTimestamp>,
    attachment_timestamp_lower_bound: Option<AttachmentTimestampLowerbound>,
    attachment_timestamp_upper_bound: Option<AttachmentTimestampUpperbound>,
    nonce: Option<Nonce>
}

impl TransactionDraft {
    /// Create a `TransactionDraft` from a reader.
    pub fn from_reader<R: std::io::Read>(reader: R) -> Result<Self, TransactionDraftError> {
        unimplemented!()
    }

    /// Create a `TransactionDraft` from a slice of bytes
    pub fn from_slice(stream: &[u8]) -> Result<Self, TransactionDraftError> {
        unimplemented!()
    }

    /// Verify that all fieleds are set and build the `Transaction`.
    pub fn build(self) -> Result<Transaction, TransactionDraftError> {
        unimplemented!()
    }

    pub fn address(&mut self, address: Address) -> &mut Self {
        self.address.replace(address);
        self
    }

    // Rest of the setter methods similarly
}
```

`TransactionDraft` is a very basic builder pattern. For the sake of this proposal, it does not contain any complex
logic. It should simply allow to set the fields piecemeal, and then give back a `Transaction` object.

The `from_slice` and `from_reader` methods mimick those of the `Transaction` type and are primarily intended as
convenience methods to be used from within `Bundle` (and to implement `Transaction::{from_slice,from_reader}`).

#### `TransactionMetadata`

This needs to be fleshed out.

## Bundle

<!-- TODO -->

## Bundle and BundleBuilder

Transactions are final and bundles, essentially being arrays of transactions, are also final. Once a bundle is created
and validated, it shouldn't be tempered. For this reason we have a `Bundle` type and a `BundleBuilder` type.
An instantiated `Bundle` object represents a syntactically and semantically valid IOTA bundle and a `BundleBuilder` is
the only gateway to a `Bundle` object.

### Bundle

As bundles are final, they shouldn't be modifiable outside of the scope of the bundle module.

There is a natural order to transactions in a bundle that can be represented in two ways:
+ each transaction has a `current_index` and a `last_index` and `current_index` goes from `0` to `last_index`, a bundle
can then simply be represented by a data structure that contiguously keeps the order like `Vec`;
+ each transaction is chained to the next one through its `trunk` which means we can consider data structures like
`HashMap` or `BTreeMap`;

For this reason, we hide this as an implementation detail and instead provide a newtype:

```rust
struct Transactions(Vec<Transaction>)
//struct Transactions(HashMap<Transaction>)
//struct Transactions(BTreeMap<Transaction>)
```

Then the `Bundle` type looks like:

```rust
struct Bundle {
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

### BundleBuilder

The `BundleBuilder` offers a simple and convenient way to build bundles:

```rust
pub struct BundleBuilder {
    transactions: Vec<TransactionBuilder>
}
```

```rust
impl BundleBuilder {    
    pub fn calculate_hash(&self) {
      unimplemented!()
    }

    pub fn finalise(&self) {
      unimplemented!()
    }

    pub fn sign(&self) {
      unimplemented!()
    }

    pub fn calculate_proof_of_work(&self) {
      unimplemented!()
    }

    pub fn validate(&self) {
      unimplemented!()
    }

    pub fn build(&self) {
      unimplemented!()
    }
}
```

*We do not list parameters and/or return values as they are implementation details.*

## Algorithms

In this section, we describe the algorithms needed to build a `Bundle`. The lifecycle of a `BundleBuilder` depends on
if it's being used in client side or server side:
+ client side: `finalise` -> [`sign` ->] [`pow` ->] `validate` -> `build`
+ server side: `add_transaction`/`add_transaction_builder` -> `validate` -> `build`

*`sign` is optional because data transactions don't have to be signed. `pow` is optional because one can use remote
pow instead.*

### Hash

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

Pseudocode:

```
calculate_hash(bundle)
| sponge = Sponge(Kerl)
|
| for transaction in bundle
| | sponge.absorb(transaction.essence())
|
| return sponge.squeeze()
```

### Finalise

*Client side operation.*

Finalising a bundle means computing the bundle hash, verifying that it matches the security requirement and setting it
to all the transactions of the bundle. After finalisation, transactions of a bundle are ready to be safely attached to
the tangle.

Pseudocode:

```
finalise(bundle)
| hash = bundleHash(bundle)
|
| while hash.normalise().find('M')
| | bundle.at(0).obsolete_tag++
| | hash = bundleHash(bundle)
|
| for transaction in bundle
| | transaction.setBundleHash(hash)
```

*Security requirement: due to the implementation of the signature process, the normalised bundle hash can't contain a
`M` or `13` because it could expose a significant part of the private key, weakening the signature. The bundle hash is
then repetitively generated with a slight modification until its normalisation doesn't contain a `M`.*

### Sign

*Client side operation.*

Signing a bundle allow you to prove that you are the owner of the address you are trying to move funds from. With no
signature or a bad signature, the bundle won't be considered valid and the funds won't be moved. Only the owner of the
right seed is able to generate the right signature for this address.

Pseudocode:

```
sign(bundle, seed, inputs)
| current_index = 0
|
| for transaction in bundle
| | if transaction.value < 0
| | | if transaction.current_index >= current_index
| | | | input = inputs[transaction.address]
| | | | fragments = sign(seed, input.index, input.security, transaction.bundle)
| | | | for fragment in fragments
| | | | | bundle.at(current_index).signature = fragment
| | | | | current_index = current_index + 1
| | else
| | | current_index = current_index + 1
```

*Since signature size depends on the security level, a single signature can spread out to up to 3 transactions.
`inputs` is an object that contain all unused addresses of a seed with a sufficient balance.*

### Proof of Work

*Client side operation.*

Proof of Work (PoW) allows your transactions to be accepted by the network. On the IOTA network, PoW is only a rate
control mechanism. Doing PoW on a bundle means doing PoW on each of its transactions and setting trunks and branch
accordingly. After PoW, a bundle is ready to be sent to the network.

Pseudocode:

```
calculate_proof_of_work(bundle, trunk, branch, mwm)
| for transaction in rev(bundle)
| | transaction.trunk = trunk
| | transaction.branch = branch
| | if transaction.current_index == transaction.last_index
| | | branch = trunk
| | transaction.attachment_timestamp = timestamp()
| | transaction.attachment_timestamp_lower = 0
| | transaction.attachment_timestamp_upper = 3812798742493
| | if transaction.tag.empty()
| | | transaction.tag = transaction.obsolete_tag
| | trunk = transaction.pow(mwm)

```

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
+ value transactions have an address ending in `0` i.e. has been generated with Kerl;
+ bundle inputs and outputs are balanced i.e. the bundle sum equals `0`;
+ announced bundle hash matches the computed bundle hash;
+ for spending transactions, the signature is valid;

Pseudocode:

```
validate(bundle):
| value = 0
| current_index = 0
|
| if bundle.length() != bundle.at(0).last_index + 1
| | return BUNDLE_INVALID_LENGTH
|
| bundle_hash = bundle.at(0).bundle_hash
| last_index = bundle.at(0).last_index
|
| for transaction in bundle
| | if transaction.bundle_hash != bundle_hash
| | | return BUNDLE_INVALID_HASH
| | if abs(transaction.value) > IOTA_SUPPLY
| | | return BUNDLE_INVALID_TRANSACTION_VALUE
| | value = value + transaction.value
| | if abs(value) > IOTA_SUPPLY
| | | return BUNDLE_INVALID_VALUE
| | if transaction.current_index != current_index++
| | | return BUNDLE_INVALID_INDEX
| | if transaction.last_index != last_index
| | | return BUNDLE_INVALID_INDEX
| | if transaction.value != 0 && transaction.address.last != 0
| | | return BUNDLE_INVALID_ADDRESS
|
| if value != 0
| | return BUNDLE_INVALID_VALUE
| if bundle_hash != bundleHash(bundle)
| | return BUNDLE_INVALID_HASH
| if !bundleSignaturesValidate(bundle)
| | return BUNDLE_INVALID_SIGNATURE
|
| return BUNDLE_VALID
```

# Drawbacks

## Transaction

+ There might be use cases that require the ability to directly create a `Transaction`, requiring exposing
  `TransactionDraft` or a similar builder pattern.
+ This proposal does not consider any kind of generics or abstraction over transactions. Future iterations of the IOTA
  network that have different transaction formats will probably require new types.

## Bundle

<!-- TODO -->

# Rationale and alternatives

## Transaction

+ Immutable `Transaction`s encode the fact that a `Transaction` should not be mutated after creation.
+ If there is a use case that requires direct creation of a `Transaction` it can be brought forward in a feature request
  or RFC. The interface can always be extended upon and be made public.
+ Forbidding the direct creation of a `Transaction` via a public API ensures that a complete message is only ever
  constructed via a `Bundle`.
+ This proposal is very straight forward and idiomatic Rust. It does not contain any overly complicated parts. As such,
  it should be easy to extend it in the future.
+ Hiding the fields of `Transaction` behind opaque types allows us to flesh them out in the future.

## Bundle

+ A `Bundle` is a fundamental component of the IOTA protocol and must be implemented;
+ There is no more intuitive and simple way to implement a `Bundle` than the one proposed;
+ Since bundles are final, `BundleBuilder` is mandatory;

# Unresolved questions

+ What assumption do we have to make about the incoming bytes and how to parse them into a `Transaction`? Is the
  structure of the incoming packet static? Or do we need to write a byteparser?
+ How much logic should the setter methods on `TransactionDraft` contain? Do we introduce traits for parsing into the
  opaque types we have provided? What would a natural interaction with the setters look like?
+ Should there be an intermediate type to go `TransactionDraft -> Transaction`. Given that the `TransactionDraft` will
  probably need to be serialized to some byte slice so that the proof of work hasher can work on it, it might be
  convenient to provide an intermediate type.
+ What should go into a `TransactionError`?
+ What should go into a `TransactionDraftError`?
+ Should we use some error libraries?
+ What `TransactionMetadata` is there? Is this part of the `Bundle` or part of the `Transaction`?
+ Should this RFC expands a bit more on the M-Bug ? Or give a link ?
+ Should `Bundle` provide a `.transactions` or a `.at` method ?
