---
title: storage
tags: RFC, draft
---

# bee-storage
+ Feature name: `storage`
+ Start date: 2020-07-15
+ RFC PR: [iotaledger/bee-rfcs#46](https://github.com/iotaledger/bee-rfcs/pull/46)
+ Bee issue:  [iotaledger/bee#106](https://github.com/iotaledger/bee/issues/106)

# Summary

An abstraction for the persistent module which will be exclusively used by the Bee crates

# Motivation

The Tangle is a DAG where each vertex's data is a transaction,
this data should of course be persisted, it is better to have this logic
and functionality grouped into one separate crate that exposes a clear API, this will also allow for implementing different database backends and ease of operation usage.

The storage operation can be performed by using the methods associated with the structure, which are implemented through macros and compiled statically.

This crate supports different databases, which can be indicated by using the [features](https://doc.rust-lang.org/cargo/reference/features.html#the-features-section).

Note that different databases have different basic operations as well as data model definitions, which make it hard to define a general operation trait (like insertion, deletion, update, etc.) for a general structure. The target of this crate, however, is mainly for IOTA protocol, so defining operation macros for different structures used in IOTA protocal (like transaction, milestone, state_delta, etc.) is a proper approach. In this way, in addition to defining the start and shutdown methods in the `Backend` trait (which has `start` and `shutdown` methods), different databases can be used as long as the operations macros associtated to the database and a given object are implemented in the object space. Based on this design, a user can switch to a different database backend through cargo flag feature.


use cases:

1. Storing and accessing IOTA objects with databases in an efficent way
2. Increase felixablity of adding/deleting/updating IOTA-object-related operations

# Detailed Design

- Supporting of each database should be switched by using the [features](https://doc.rust-lang.org/cargo/reference/features.html#the-features-section) section for ease of usage
- Consider only RocksDB in the begining for ease of implementation and compatible with the current [IRI](https://github.com/iotaledger/iri) design
- File hierarchy

```
    +-- access
    |   +-- transaction.rs
    |   +-- milestone.rs
    |   +-- state_delta.rs
    |   +-- tangle.rs
    |   +-- snapshot.rs
    |   +-- address.rs
    |   +-- ..
    +-- storage
    |   +-- config.rs
    |   +-- mod.rs
    |   +-- rocksdb.rs
    |   +-- ..
```

- Current Structures

```rust
/// The Storage struct will be created automatically by the assignment in feature
/// Here we use rocks_db as an example for the storage structure

#[cfg(feature = "rocks_db")]
pub struct Storage {
    inner: ::rocksdb::DB  // The storage backend
}

// From https://github.com/iotaledger/bee-rfcs/pull/30
#[derive(Copy, Clone)]
pub struct Milestone {
    pub(crate) hash: Hash,
    pub(crate) index: MilestoneIndex,
}

/// From https://github.com/iotaledger/bee-rfcs/pull/20
#[derive(Clone, Debug, PartialEq, Hash)]
pub struct Address(pub(crate) TritBuf<T1B1Buf>);

/// From https://github.com/iotaledger/bee-rfcs/pull/20
#[derive(PartialEq, Clone, Debug)]
pub struct BundledTransaction {
    pub(crate) payload: Payload,
    pub(crate) address: Address,
    pub(crate) value: Value,
    pub(crate) obsolete_tag: Tag,
    pub(crate) timestamp: Timestamp,
    pub(crate) index: Index,
    pub(crate) last_index: Index,
    pub(crate) bundle: Hash,
    pub(crate) trunk: Hash,
    pub(crate) branch: Hash,
    pub(crate) tag: Tag,
    pub(crate) attachment_ts: Timestamp,
    pub(crate) attachment_lbts: Timestamp,
    pub(crate) attachment_ubts: Timestamp,
    pub(crate) nonce: Nonce,
}
```

- Backend trait and the associated methods
    - start method
    - shutdown method

```rust
#[async_trait]
/// Trait to be implemented on storage backend,
/// which determine how to start and shutdown the storage
pub trait Backend {
    /// start method should impl how to start and initialize the corrsponding database
    /// It takes config_path which define the database options, and returns Result<Self, Box<dyn Error>>
    async fn start(config_path: String) -> Result<Self, Box<dyn Error>> where Self: Sized;
    /// shutdown method should impl how to shutdown the corrsponding database
    /// It takes the ownership of self, and returns ()
    async fn shutdown(self) -> Result<(), Box<dyn Error>>;
}
#[cfg(feature = "rocks_db")]
#[async_trait]
impl Backend for Storage {
    /// It starts RocksDB instance and then initialize the required column familes
    async fn start(config_path: String) -> Result<Self, Box<dyn Error>> {
        let config_as_string = fs::read_to_string(config_path)?;
        let config: config::Config = toml::from_str(&config_as_string)?;
        let db = rocksdb::RocksdbBackend::new(config)?;
        Ok(Storage { inner: db })
    }
    /// It shutdown RocksDB instance,
    /// Note: the shutdown is done through flush method and then droping the storage object
    async fn shutdown(self) -> Result<(), Box<dyn Error>> {
        self.inner.flush()?
    }
}
```

- Operations macros which should be imported in the associated object mod in other crates
    - impl_transaction_ops!(BundleTransaction);
    - impl_milestone_ops!(Milestone);
    - impl_tangle_ops!(Tangle);
    - impl_.._ops!(..);

```rust
// Using macros is how bee_storage expose database operations
#[macro_export]
macro_rules! impl_transaction_ops {
    ($object:ty) => {
        use bee_storage::storage::rocksdb::TRANSACTION_CF_HASH_TO_TRANSACTION;
        use bee_transaction::bundled::{BundledTransaction,BundledTransactionField};
        use bee_ternary::{T1B1Buf, T5B1Buf, TritBuf, Trits, T5B1};
        use bytemuck::cast_slice;
        use bee_crypto::ternary::Hash;
        use bee_storage::access::OpError;
        #[cfg(feature = "rocks_db")]
        impl $object {
            async fn insert(&self, hash: Hash, storage: &Storage) -> Result<(), OpError> {
                // get column family handle to hash_to_tx table in order presist the transaction;
                let hash_to_tx = storage.inner.cf_handle(TRANSACTION_CF_HASH_TO_TRANSACTION).unwrap();
                let mut tx_trit_buf = TritBuf::<T1B1Buf>::zeros(Self::trit_len());
                self.into_trits_allocated(tx_trit_buf.as_slice_mut());
                let hash_buf = hash.to_inner().encode::<T5B1Buf>();
                storage.inner.put_cf(
                    &hash_to_tx,
                    cast_slice(hash_buf.as_i8_slice()),
                    cast_slice(tx_trit_buf.encode::<T5B1Buf>().as_i8_slice()),
                )?;
                Ok(())
            }
            async fn find_by_hash(hash: Hash, storage: &Storage) -> Result<Option<Self>, OpError> {
                let hash_to_tx = storage.inner.cf_handle(TRANSACTION_CF_HASH_TO_TRANSACTION).unwrap();
                if let Some(res) = storage.inner.get_cf(
                    &hash_to_tx,
                    cast_slice(hash.to_inner().encode::<T5B1Buf>().as_i8_slice()),
                )?
                {
                    let trits = unsafe {
                        Trits::<T5B1>::from_raw_unchecked(&cast_slice(&res), Self::trit_len())
                    }.encode::<T1B1Buf>();
                    let transaction = Self::from_trits(&trits).unwrap();
                    Ok(Some(transaction))
                } else {
                    Ok(None)
                }
            }
            async fn todo!();
        }
    };
}

#[macro_export]
macro_rules! impl_milestone_ops {..}

#[macro_export]
macro_rules! impl_tangle_ops {..}

```

- Implementations in other crates
    - Transaction object

    ```rust
    // in bee-transaction/src/bundled/transaction/transaction.rs
    use bee_storage::impl_transaction_ops;
    impl_transaction_ops!(BundleTransaction);
    ```
    - ..

# Drawbacks

The need to impl the storage operations for a given object in the same module for that object to prevent cyclic dependency, still I don't see as a drawback because all what it takes is just importing and using the macro

# Rationale and alternatives

- Having storage as a trait allows for multiple backends implementations

# Unresolved questions

- Need to pass the `Storage` structure for all the database operation methods

- StateDeltaMap should be maintained by other crates instead of storage crate

- The proper way to propagate the feature settings to inner crates?
