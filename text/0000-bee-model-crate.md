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

    pub fn from_transaction_builder(transaction_builder: &TransactionBuilder) -> Transaction {
        unimplemented!()
    }
    
    pub fn from_bytes(bytes: &Vec<u8>) -> Transaction {
        unimplemented!()
    }

    pub fn get_transaction_hash(&self) -> &String {
        &self.transaction_hash
    }

    pub fn get_signature_fragments(&self) -> &String {
        &self.signature_fragments
    }

    pub fn get_address(&self) -> &String {
        &self.address
    }

    pub fn get_value(&self) -> &i64 {
        &self.value
    }

    pub fn get_obsolete_tag(&self) -> &String {
        &self.obsolete_tag
    }

    pub fn get_timestamp(&self) -> &i64 {
        &self.timestamp
    }

    pub fn get_current_index(&self) -> &usize {
        &self.current_index
    }

    pub fn get_last_index(&self) -> &usize {
        &self.last_index
    }

    pub fn get_bundle_hash(&self) -> &String {
        &self.bundle_hash
    }

    pub fn get_trunk(&self) -> &String {
        &self.trunk
    }

    pub fn get_branch(&self) -> &String {
        &self.branch
    }

    pub fn get_nonce(&self) -> &String {
        &self.nonce
    }

    pub fn get_tag(&self) -> &String {
        &self.tag
    }

    pub fn get_attachment_timestamp(&self) -> &i64 {
        &self.attachment_timestamp
    }

    pub fn get_attachment_timestamp_lower_bound(&self) -> &i64 {
        &self.attachment_timestamp_lower_bound
    }

    pub fn get_attachment_timestamp_upper_bound(&self) -> &i64 {
        &self.attachment_timestamp_upper_bound
    }

}
```

The Transcation_Builder struct contains public accessible fields. 
The set values can be validated in the build() function which then returns the constructed transaction.
Moreover, also the Proof-Of-Work will be part of the build() function.

```rust
impl Transaction_Builder {

    pub fn new() -> TransactionBuilder {

        !unimplemented()

    }

    pub fn build(&self) -> Transaction {
        Transaction::from_transaction_builder(self)
    }
    
    ...
    
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