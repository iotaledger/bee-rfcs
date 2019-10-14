+ Feature name: `bee-transaction`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3)
+ Bee issue: [iotaledger/bee#43](https://github.com/iotaledger/bee/issues/43)

# Summary

In the context of IOTA, a `Bundle` is a message that is shared over the
network, and a `Transaction` is the fundamental unit that a `Bundle` is
constructed from. To use TCP as a well known analogy, a `Bundle` corresponds
to a message, and a `Transaction` corresponds to a packet.

This RFC proposes a `Transaction` type to represent the transaction format used
by the IOTA Reference Implementation as of version [`iri v1.8.1`], commit
`e1776fbad5d90df86a26402f9025e4b0b2ef7d3e`. The `Transaction` type is implemented
as a struct and is intended to be a public interface. It thus comes with a number
of opaque types for its fields that we leave to be fleshed out at a later time.

We also propose a `TransactionDraft` type that follows the builder pattern. It
is intended to be an implementation detail and only be used by a `Bundle` builder
type and not exposed publicly.

# Motivation

IOTA is a transaction settlement and data transfer layer for IoT (the Internet
of Things). Messages on the network are `Bundle`s of individual `Transaction`s,
which in turn are sent one at a time and are stored in a distributed ledger
called the *Tangle*. Each `Transaction` encodes data such as sender and
receiver addresses, reference nodes in the Tangle, `Bundle` hash, timestamps,
and other information required to verify and process each transaction. With the
transaction being the fundamental unit that is sent over the network, we need
to represent it in memory.

At the time of this RFC, the transaction format used by the IOTA network is
defined by [release v1.8.1 of the IOTA Reference Implementation, IRI](`iri
v1.8.1`), commit `e1776fbad5d90df86a26402f9025e4b0b2ef7d3e`. The transaction
format might change in the future, but for the time being it is not yet
understood how a common interface between different versions transaction
formats should be implemented, or if the network will support several
transaction types simultaneously.

We thus do not consider generalizations over or interfaces for transactions,
but only propose a basic `Transaction` type. All mentions of *transactions* in
general or the `Transaction` type in particular will be implicitly in reference
to that format used by `iri v1.8.1`.

The `Transaction` type is intended to be *final* and immutable. This contract
is enforced by not expose any methods that allow manipulating its fields
directly. The only public constructor method that allows creating
a `Transaction` directly are the `from_reader` and `from_slice` method to
construct it from a type implementing `std::io::Read`, or from byte slice.
Otherwise, `Transaction`s should only be built through `Bundle` constructors,
and only from that context direct access to fields is permitted. To this
purpose, this RFC contains the `TransactionDraft` builder pattern type.

[`iri v1.8.1`]: https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE

# Detailed design

A transaction is a sequence of 15 fields of constant length. A transaction has
a total length of 8019 trits. The fields' name, description, and size (in
trits) is summarized in the table below, in their order of appearance in
a transaction.

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

Each transaction is uniquely identified by its *transaction hash*, which is
calculated based on all fields of the transaction. Note that the transaction
hash is not part of the transaction.

Because modifying the fields of a transaction would invalidate its hash,
`Transaction` is immutable after creation.

## Transaction struct

A transaction is represented by the `Transaction` struct and follows the structure presented
in the table above:

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

We treat the types of the `Transaction` struct's fields as opaque newtypes for
now so that we can flesh them out during the implementation phase or in future
RFCs.

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

The `Transaction` type only contains two constructor methods getter methods for
read-only borrows of its fields.

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

The `from_slice` and `from_reader` methods will use a `TransactionDraft` internally,
but these are implementation details.

The `TransactionError` is not fully fleshed out yet. For the time being, it
definitely contains `io::Error`:

```rust
pub enum TransactionError {
    Io(io::Error),
}
```

### `TransactionDraft`

`TransactionDraft` is a basic builder pattern for creating and setting the individual
fields of a new transaction.

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

`TransactionDraft` is a very basic builder pattern. For the sake of this
proposal, it does not contain any complex logic. It should simply allow to
set the fields piecemeal, and then give back a `Transaction` object.

The `from_slice` and `from_reader` methods mimick those of the `Transaction` type
and are primarily intended as convenience methods to be used from within `Bundle`
(and to implement `Transaction::{from_slice,from_reader}`).

### `TransactionMetadata`

This needs to be fleshed out.

# Drawbacks

+ There might be use cases that require the ability to directly create
  a `Transaction`, requiring exposing `TransactionDraft` or a similar builder pattern.
+ This proposal does not consider any kind of generics or abstraction over
  transactions. Future iterations of the IOTA network that have different
  transaction formats will probably require new types.

# Rationale and alternatives

+ Immutable `Transaction`s encode the fact that a `Transaction` should not be
  mutated after creation. 
+ If there is a use case that requires direct creation of a `Transaction` it
  can be brought forward in a feature request or RFC. The interface can always
  be extended upon and be made public.
+ Forbidding the direct creation of a `Transaction` via a public API ensures
  that a complete message is only ever constructed via a `Bundle`.
+ This proposal is very straight forward and idiomatic Rust. It does not
  contain any overly complicated parts. As such, it should be easy to extend it
  in the future.
+ Hiding the fields of `Transaction` behind opaque types allows us to flesh
  them out in the future.

# Unresolved questions

+ What assumption do we have to make about the incoming bytes and how to parse
  them into a `Transaction`? Is the structure of the incoming packet static? Or
  do we need to write a byteparser?
+ How much logic should the setter methods on `TransactionDraft` contain? Do we
  introduce traits for parsing into the opaque types we have provided? What
  would a natural interaction with the setters look like?
+ Should there be an intermediate type to go `TransactionDraft -> Transaction`.
  Given that the `TransactionDraft` will probably need to be serialized to some byte
  slice so that the proof of work hasher can work on it, it might be convenient
  to provide an intermediate type.
+ What should go into a `TransactionError`?
+ What should go into a `TransactionDraftError`?
+ Should we use some error libraries?
+ What `TransactionMetadata` is there? Is this part of the `Bundle` or part of
  the `Transaction`?
