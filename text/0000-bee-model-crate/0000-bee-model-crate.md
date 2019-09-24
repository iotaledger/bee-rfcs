+ Feature Name: `bee-model-crate`
+ Start Date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#43](https://github.com/iotaledger/bee/issues/43)

# Summary

This feature is responsible for the creation and interpretation of transactions and bundles.

# Motivation

IOTA is a distributed ledger that was designed for payment settlement and data transfer between machines and devices in the Internet of Things (IoT) ecosystem.
The data packets that are sent through the network are called "transactions".
Settlements or data transfers can be done with the help of these transactions. Payment settlements require the help of multiple transactions.

# Detailed design

## General

A transaction consists of several fields (e.g. address, value, timestamp, tag). Each field is of static length. One transaction consists of 2673 trytes:

- **trunk hash** the hash of the first transaction referenced/approved = 81 trytes
- **branch hash** the hash of the second transaction referenced/approved = 81 trytes
- **signature_and_message_fragment** contains the signature of the transfer or user-defined message data = 2187 trytes
- **value** the transferred amount in IOTA  = 27 trytes
- **address** receipt (output) address if value > 0, or withdrawal (input) address if value < 0 = 81 trytes
- **timestamp** the time when the transaction was issued = 9 trytes
- **attachment_timestamp** the timestamp for when Proof-of-Work is completed = 9 trytes
- **attachment_timestamp_lowerbound** is a slot for future use = 9 trytes
- **attachment_timestamp_upperbound** is a slot for future use = 9 trytes
- **tag** arbitrary user-defined value = 27 trytes
- **obsolete_tag** another arbitrary user-defined tag = 27 trytes
- **nonce** is the Proof-of-Work nonce of the transaction = 27 trytes
- **bundle hash** is the hash of the entire bundle = 81 trytes
- **current_index** the position of the transaction in its bundle = 9 trytes
- **last_index** the total number of transactions in the bundle = 9 trytes

A bundle is a collection of specific transactions. Bundles are required to bundle related information.
For example, if not all data fits into one transaction, it has to be split across multiple transactions. An example would be payment settlements - they require the help of multiple transactions.
A bundle then represents the collection of these transactions. It should be noted, that each transaction of a bundle will be sent separately through the network.

Each transaction is identified by its hash, the transaction hash. Transactions should be final and therefore not modifiable after construction.
The same applies to bundles. Bundles are final, and therefore not modifiable after construction.
Having bundles as well as transactions final helps achieve correctness among the data objects.

Transactions can be built from Transaction Builders.
Bundles can be built from Bundle Builders. Both builder objects can be manipulated as desired.
It should be noted, Bundle Builders don't process Transactions (as transactions are final), instead, they process Transaction Builders.

## Exposed Interface

### Transaction

 A transaction consists of following fields:

```rust
struct Transaction {

    transaction_hash: String,
    signature_fragments: String,
    address: String,
    value: i64,
    obsolete_tag: String,
    timestamp: i64,
    current_index: usize,
    last_index: usize,
    bundle_hash: String,
    trunk: String,
    branch: String,
    nonce: String,
    tag: String,
    attachment_timestamp: i64,
    attachment_timestamp_lower_bound: i64,
    attachment_timestamp_upper_bound: i64,

}
```

As mentioned, Transactions are final. Transaction fields therefore should be only accessible by getter functions.
Besides that, Transactions can be only built by Transaction Builders, or by the bytes of a received transaction.

```rust
impl Transaction {

    pub fn from_transaction_builder(transaction_builder: &TransactionBuilder) -> Result<Transaction, TransactionBuilderValidationError> {
                
        match TransactionBuilderValidator::validate(&transaction_builder) {
            Ok(()) => { ; }
            Err(e) => { return Err(e) }
        }

        let (transaction_hash, nonce, timestamp) = Pow::compute(&transaction_builder);

        Ok(Transaction {
            transaction_hash,
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
        })
                
    }
    
    pub fn from_bytes(bytes: &Vec<u8>) -> Result<Transaction, TransactionBuilderValidationError> {
        unimplemented!()
    }

    pub fn transaction_hash(&self) -> &String {
        &self.transaction_hash
    }

    pub fn signature_fragments(&self) -> &String {
        &self.signature_fragments
    }

    pub fn address(&self) -> &String {
        &self.address
    }

    pub fn value(&self) -> &i64 {
        &self.value
    }

    pub fn obsolete_tag(&self) -> &String {
        &self.obsolete_tag
    }

    pub fn timestamp(&self) -> &i64 {
        &self.timestamp
    }

    pub fn current_index(&self) -> &usize {
        &self.current_index
    }

    pub fn last_index(&self) -> &usize {
        &self.last_index
    }

    pub fn bundle_hash(&self) -> &String {
        &self.bundle_hash
    }

    pub fn trunk(&self) -> &String {
        &self.trunk
    }

    pub fn branch(&self) -> &String {
        &self.branch
    }

    pub fn nonce(&self) -> &String {
        &self.nonce
    }

    pub fn tag(&self) -> &String {
        &self.tag
    }

    pub fn attachment_timestamp(&self) -> &i64 {
        &self.attachment_timestamp
    }

    pub fn attachment_timestamp_lower_bound(&self) -> &i64 {
        &self.attachment_timestamp_lower_bound
    }

    pub fn attachment_timestamp_upper_bound(&self) -> &i64 {
        &self.attachment_timestamp_upper_bound
    }

}
```

The Transcation_Builder struct contains public accessible fields. 
The set values can be validated in the build() function which then returns the constructed transaction.
Moreover, also the Proof-Of-Work will be part of the build() function.

```rust
#[derive(Default)]
pub struct TransactionBuilder {
    pub signature_fragments: String,
    pub address: String,
    pub value: i64,
    pub obsolete_tag: String,
    pub current_index: usize,
    pub last_index: usize,
    pub bundle_hash: String,
    pub trunk: String,
    pub branch: String,
    pub tag: String,
    pub attachment_timestamp: i64,
    pub attachment_timestamp_lower_bound: i64,
    pub attachment_timestamp_upper_bound: i64,
}

impl TransactionBuilder {

    pub fn build(&self) -> Result<Transaction, TransactionBuilderValidationError> {
        Transaction::from_transaction_builder(self)
    }

}
```

### Bundle

All transactions in the same bundle have the same value in the bundle field. This field contains the bundle hash, which is derived from a hash of the values of each transaction's address, value, obsoleteTag, currentIndex, lastIndex and timestamp fields.
- **address**
- **value**
- **obsoleteTag**
- **currentIndex**
- **lastIndex**
- **timestamp**

#### Bundle Builder

Similar to Transaction Builder, BundleBuilder makes it possible to create a Bundle. As mentioned, a bundle is a special collection of transactions.
Bundles are read from head to tail but created from tail to head. This is why it makes sense to have a dedicated class for this purpose.
The transactions inside a bundle are connected through the trunk. The trunk field of the head transaction (index 0) references the transaction with index 1, and so on.

```rust

struct BundleBuilder<'a> {

    pub tailToHead: Vec<&'a TransactionBuilder>
    
}

impl<'a> BundleBuilder<'a> {

    pub fn append(&mut self, transaction_builder: &'a TransactionBuilder) {
        self.tailToHead.push(transaction_builder);
    }
    
    pub fn build(&self) -> Result<Bundle, BundleBuildError> {
        
        if self.tailToHead.size() == 0 {
            return Err(BundleBuilder("Cannot build: bundle is empty (0 transactions)."))
        }
        
        setFlags();
        buildTrunkLinkedChainAndReturnHead();
        
        Ok(Bundle::new(tailToHead))
    }

}

```


# Drawbacks

Without any bee-model crate, nodes can not exchange transactions. Therefore this crate seems necessary.

# Rationale and alternatives

- The distinction between Transaction and Transaction Builder as well as Bundle and Bundle Builder makes the code cleaner 
and helps achieve correctness among the data objects. Properties are clearly assigned to specific data objects and not mixed up.

- The proposed crate interface is intuitive. Completely different alternatives did not naturally come to mind.

- The validation logic could be done in a separate validator class, which then will be called in the build() function.

# Unresolved questions

- How is Proof-Of-Work acceleration handled?