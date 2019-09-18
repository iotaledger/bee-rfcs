+ Feature Name: `bee-model-crate`
+ Start Date: 2019-09-06
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#43](https://github.com/iotaledger/bee/issues/43)

# Summary

This feature is responsible for the creation and interpretation of transactions and bundles.

# Motivation

IOTA is a distributed ledger that was designed for payment settlement and data transfer between machines and devices in the Internet of Things (IoT) ecosystem.
The data packets that are sent through the network are called "transactions".
Settlements or data transfers can be done with the help of these transactions.

# Detailed design

## General

A transaction consists of several fields (e.g. address, value, tag). Each field has a static length.
A bundle is a collection of specific transactions. They are required to bundle related information.
For example, if not all data fits into one transaction, it has to be split across multiple transactions.

It is important to understand that transactions should be final and therefore not modifiable after construction.
This implies that there is the need for a "Transaction Builder" that can be manipulated as desired.

The same applies to bundles. Bundles are final, and therefore not modifiable after construction.
Also this implies that there is the need for a "Bundle Builder" that can be manipulated as desired.
A "Bundle Builder" does process "Transaction Builders".

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
Besides that, Transactions can be only built by "Transaction Builders", or by the byte sequence of received transactions.

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

# Drawbacks

Without any bee-model crate, nodes can not exchange transactions. Therefore this crate seems necessary.

# Rationale and alternatives

- The distinction between Transaction and Transaction Builder as well as Bundle and Bundle Builder makes the code cleaner 
and helps achieve correctness among the data objects. Properties are clearly assigned to specific data objects and not mixed up.

- The proposed crate interface is intuitive. Completely different alternatives did not naturally come to mind.

- The validation logic could be done in a separate validator class, which then will be called in the build() function.

# Unresolved questions

- How is Proof-Of-Work acceleration handled?