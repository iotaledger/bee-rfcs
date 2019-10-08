+ Feature name: `bee-transaction`
+ Start date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/3)
+ Bee issue: [iotaledger/bee#43](https://github.com/iotaledger/bee/issues/43)

# Summary

IOTA is a distributed ledger that was designed for payment settlement and data transfer between machines and devices in the Internet of Things (IoT) ecosystem.
The messages that are sent through the network are called "transactions".
Payment settlements or data transfers can be done with the help of these transactions. 

This feature is responsible for the creation and interpretation of transactions.

# Motivation
[...]

# Detailed design

## General

A transaction can be understood as a **tuple** of 15 elements. Essentially, it's an **ordered list (sequence)** of following elements/fields: 

| Name  | Description | Size in trytes |
| ------------- | ------------- | ------------- |
| `signature_or_message_fragment` | contains the signature of the transfer or user-defined message data | 2187 trytes |
| `address`  | receiver (output) if value > 0, or sender (input) if value < 0  | 81 trytes |
| `value`  | the transferred amount in IOTA  | 27 trytes |
| `obsolete_tag`  | another arbitrary user-defined tag  | 27 trytes |
| `timestamp`  | the time when the transaction was issued  | 9 trytes |
| `current_index` | the position of the transaction in its bundle  | 9 trytes |
| `last_index`  | the index of the last transaction in the bundle  | 9 trytes |
| `bundle`  | the hash of the bundle essence | 81 trytes |
| `trunk`  | the hash of the first transaction referenced/approved | 81 trytes |
| `branch`  | the hash of the second transaction referenced/approved | 81 trytes |
| `tag`  | arbitrary user-defined value | 27 trytes |
| `attachment_timestamp`  |  the timestamp for when Proof-of-Work is completed | 9 trytes |
| `attachment_timestamp_lowerbound`  |  *not specified* | 9 trytes |
| `attachment_timestamp_upperbound`  |  *not specified* | 9 trytes |
| `nonce`  | is the Proof-of-Work nonce of the transaction | 27 trytes |

As reflected in the table, each field is of constant length. Totally, one transaction consists of **2673 trytes** (**8019 trits**).

Transactions are classified as unique. Each transaction is identified by its hash, the transaction hash, which includes all fields of a transaction.
Note, the transaction hash is not part of the transaction.

Once a transaction is built, it can be identified by its transaction hash. Manipulating a transaction afterwards would result in the hash no longer matching the transaction, which essentially is like creating a new transaction.
Therefore, transactions should **not be modifiable** after construction. Transaction fields should only be accessible by getter functions.

## Exposed Interface

### Transaction

The structure of a transaction is defined as follows:

```rust
struct Transaction {
    signature_or_message_fragment: SignatureOrMessageFragment,
    address: Address,
    value: Value,
    obsolete_tag: ObsoleteTag,
    timestamp: Timestamp,
    current_index: CurrentIndex,
    last_index: LastIndex,
    bundle_hash: BundleHash,
    trunk: Trunk,
    branch: Branch,
    tag: Tag,
    attachment_timestamp: AttachmentTimestamp,
    attachment_timestamp_lower_bound: AttachmentTimestampLowerbound,
    attachment_timestamp_upper_bound: AttachmentTimestampUpperbound,
    nonce: Nonce
}
```

As reflected above, **each field gets its own type**. This makes the code more readable and clean. Furthermore it makes sure allocations are valid. The validation happens in each type separately.

The type implementation is defined as follows:

```rust
impl Transaction {
    
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

### TransactionBuilder

The TransactionBuilder offers a **simple and convenient** way to build transactions. It's defined as follows:

```rust
#[derive(Default)]
pub struct TransactionBuilder {
    signature_or_message_fragment: SignatureOrMessageFragment,
    address: Address,
    value: Value,
    obsolete_tag: ObsoleteTag,
    current_index: CurrentIndex,
    last_index: LastIndex,
    bundle_hash: BundleHash,
    trunk: Trunk,
    branch: Branch,
    tag: Tag,
    attachment_timestamp: AttachmentTimestamp,
    attachment_timestamp_lower_bound: AttachmentTimestampLowerbound,
    attachment_timestamp_upper_bound: AttachmentTimestampUpperbound,
    nonce: Nonce
}

impl TransactionBuilder {

    pub fn from_bytes(bytes: &Vec<u8>) -> Result<Transaction, TransactionBuilderValidationError> {
        unimplemented!()
    }

    pub fn build(self) -> (Transaction, TranscationMetadata) {

        Pow::compute(&self)

    }
    
    pub fn set_signature_or_message_fragment(&mut self, signature_or_message_fragment: SignatureOrMessageFragment) {
        self.signature_or_message_fragment = signature_or_message_fragment;
    }
    
    pub fn set_address(&mut self, address: Address) {
        self.address = address;
    }
    
    pub fn set_value(&mut self, value: Value) {
        self.value = value;
    }
    
    ...
    
    pub fn signature_or_message_fragment(&self) {
        self.signature_or_message_fragment
    }
    
    pub fn address(&self) {
        self.signature_or_message_fragment
    }
    
    pub fn value(&self) {
        self.value
    }
    
    ...

}
```
In comparison to Transaction, TransactionBuilder can be manipulated as desired with the help of setter methods.

The **from_bytes(bytes: &Vec)** is striking here. It serves as constructor and allows to build transactions from received bytes, e.g. from the network socket.
It's one way to build transactions. Another way would be from ground up via setters and use **build()** from **TransactionBuilder**.

Also **Pow::compute(&self)** in **build()** striking here. This function calls the needed Proof-of-Work (PoW) for the transaction.
As reflected in the code, the **PoW will return the tuple (Transaction, TranscationMetadata).**

### TransactionMetadata

Each transaction contains a mutable metadata which is defined as following:

```rust
pub struct TransactionMetadata {
    transaction_hash: TransactionHash,
    ...
}

impl TransactionMetadata {

    pub fn set_transaction_hash(&mut self, transaction_hash: TransactionHash) {
        self.transaction_hash = transaction_hash;
    }
    
    pub fn transaction_hash(&self) -> TransactionHash {
        self.transaction_hash
    }
    
    ...
    
}

```

The TransactionMetadata contains important information about a specific transaction.

### Proof-of-Work (PoW)

Even though PoW should be handled in its own RFC, it interacts with TransactionBuilder (in the current moment of time) and therefore deserves some review:

*pub fn compute(transaction_builder: &TransactionBuilder)*

The following example shows how PoW interacts with the **TransactionBuilder**  and how Transactions finally are built.
This example assumes that the *actual Transaction building* (initialization of the Transaction struct that is to be returned) takes place in PoW.

```rust
pub struct Pow;

impl Pow {

    // operates on String encoding, should be replaced with trits interface once ready
    pub fn compute(transaction_builder: &TransactionBuilder) -> (Transaction, TranscationMetadata) {

        const CHARSET: &[u8] = b"ABCDEFGHIJKLMNOPQRSTUVWXYZ9";

        let iv: String = (0..27)
            .map(|_| {
                let mut rng = rand::thread_rng();
                let idx = rng.gen_range(0, CHARSET.len());
                char::from(unsafe { *CHARSET.get_unchecked(idx) })
            })
            .collect();

        let trytes_backup = transaction_builder.signature_fragments.clone(); // + value + all other fields
        let mut hash;
        let mut nonce = iv.clone();
        let mut timestamp = 0;
        let mut transaction_metadata: TranscationMetadata = TranscationMetadata::default();

        loop {

            let mut trytes = trytes_backup.clone();
            trytes.push_str(&nonce);

            hash = HashFunction::hash(&trytes);
            nonce = String::from(&hash[..27]);
            // timestamp = ... (update timestamp)

            if hash.ends_with(MIN_WEIGHT_MAGNITUDE_AS_STRING){
                transaction_metadata.hash = hash.clone();
                break
            }

        }

        (
            Transaction {
                signature_fragments: transaction_builder.signature_fragments.clone(),
                address: transaction_builder.address.clone(),
                value: transaction_builder.value.clone(),
                obsolete_tag: transaction_builder.obsolete_tag.clone(),
                timestamp,
                current_index: transaction_builder.current_index.clone(),
                last_index: transaction_builder.last_index.clone(),
                bundle_hash: transaction_builder.bundle_hash.clone(),
                trunk: transaction_builder.trunk.clone(),
                branch: transaction_builder.branch.clone(),
                nonce,
                tag: transaction_builder.tag.clone(),
                attachment_timestamp: transaction_builder.attachment_timestamp.clone(),
                attachment_timestamp_lower_bound: transaction_builder.attachment_timestamp_lower_bound.clone(),
                attachment_timestamp_upper_bound: transaction_builder.attachment_timestamp_upper_bound.clone(),
            },

            transaction_metadata

        )

    }

}

```

As reflected by the code above, the Transaction struct (that is to be returned) gets initialized directly in PoW.
This has some advantages, like the *timestamp* and the *nonce* can directly be passed to the struct.
A disadvantage is, the building of transactions takes place in the PoW model.

# Drawbacks

Without any bee-model crate, nodes can not exchange transactions. Therefore this crate seems necessary.

# Rationale and alternatives

- The distinction between Transaction and TransactionBuilder makes the code cleaner. Properties are clearly assigned to specific data objects and not mixed up.

- The proposed crate interface seems relatively intuitive. Completely different alternatives did not naturally come to mind.

- This kind of interface is relatively minimal and easily extended upon in a future iteration.

# Unresolved questions

- How to handle deserialization of the incoming, encoded transaction?

- Where should the actual Transaction struct be initialized/returned? Current ideas are to put it in the build() of the TransactionBuilder or as it currently is in the compute() of Pow.

- Should we introduce a fluent API to the TransactionBuilder for more easy manipulation?

- Should we use Option<T> for the fields in TransactionBuilder? 

- Should we use a intermediary building type like **TransactionBuilder -> UnpowedTransaction -> Transaction**
