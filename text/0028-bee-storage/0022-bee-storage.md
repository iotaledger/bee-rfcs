+ Feature name: `bee-storage`
+ Start date: 2019-13-11
+ RFC PR: [iotaledger/bee-rfcs#28](https://github.com/iotaledger/bee-rfcs/pull/28)
+ Bee issue:  [iotaledger/bee#65](https://github.com/iotaledger/bee/issues/65)

# Summary

An abstraction for the persistent module which will be exclusively used by the Tangle module
for CRUD operations.

# Motivation

The Tangle is a DAG where each vertex's data is a transaction,
this data should of course be persisted, it is better to have this logic
and functionality grouped into one separate module that exposes a clear API
rather than have it in several places, this will also allow for implementing different database backends
that will implement the storage API

Whenever a bee module requires updating a transaction, it will do so by interacting with the Tangle module
which will internally use this storage module

use cases:

1.When a new transaction arrives to the node and is not in the database, we would want to insert it as a new record
2.When a transaction is required we would want to retrieve it from the database
3.When we want to update a transaction as being solid
4.When we want to delete a transaction during the local snapshot process


# Detailed design

```rust

use std::{collections::HashMap, collections::HashSet, rc::Rc};

/// A transaction hash. To be replaced later with whatever implementation is required.
#[derive(Copy, Clone, Debug, Eq, PartialEq, Hash)]
pub struct TxHash(u64);
impl TxHash {
    pub fn is_genesis(&self) -> bool {
        unimplemented!()
    }
}

/// A transaction address. To be replaced later with whatever implementation is required.
#[derive(Copy, Clone, Debug, Eq, PartialEq, Hash)]
pub struct TxAddress(u64);

/// A transaction. Cannot be mutated once created.
pub struct Tx {
    hash: TxHash,
    trunk: TxHash,
    branch: TxHash,
    body: (),
}

/// A milestone hash. To be replaced later with whatever implementation is required.
#[derive(Copy, Clone, Debug, Eq, PartialEq, Hash)]
pub struct MilestoneHash(u64);
impl MilestoneHash {
    //Returns true if the milestone is the genesis of the Tangle
    pub fn is_genesis(&self) -> bool {
        unimplemented!()
    }
}

/// A transaction. Cannot be mutated once created.
pub struct Milestone {
    hash: MilestoneHash,
    index: u32,
}

pub enum StorageError {
    TransactionAlreadyExist,
    TransactionNotFound,
    UnknownError,
    //...
}

pub enum ConnectionError {
    InvalidUrl,
    UnknownError,
    //...
}

type HashesToApprovers = HashMap<TxHash, Vec<TxHash>>;
type MissingHashesToRCApprovers = HashMap<TxHash, Vec<Rc<TxHash>>>;
//This is a mapping between an iota address and it's balance change
//practically, a map for total balance change over an addresses will be collected
//per milestone (snapshot_index), when we no longer have milestones, we will have to find
//another way to decide on a check point where to store an address's delta if we want to snapshot
type StateDeltaMap = HashMap<TxAddress, i64>;

pub trait Connection<Conn> {
    fn establish_connection(url: &str) -> Result<Conn, ConnectionError>;
    fn destroy_connection(connection: Conn) -> Result<(), ConnectionError>;
}

pub trait Storable {
    //**Operations over transaction's schema**//

    async fn insert_transaction(&self, tx: &Tx) -> Result<(), StorageError>;
    async fn find_transaction(&self, tx_hash: TxHash) -> Result<Tx, StorageError>;
    async fn update_transactions_set_solid(
        &self,
        transaction_hashes: HashSet<TxHash>,
    ) -> Result<(), StorageError>;
    async fn update_transactions_set_snapshot_index(
        &self,
        transaction_hashes: HashSet<TxHash>,
        snapshot_index: u32,
    ) -> Result<(), StorageError>;
    async fn delete_transactions(&self, transaction_hashes: HashSet<TxHash>) -> Result<(), StorageError>;

    //This method is heavy weighted and will be used to populate Tangle struct on initialization
    fn map_existing_transaction_hashes_to_approvers(
        &self,
    ) -> Result<HashesToApprovers, StorageError>;
    //This method is heavy weighted and will be used to populate Tangle struct on initialization
    fn map_missing_transaction_hashes_to_approvers(
        &self,
    ) -> Result<MissingHashesToRCApprovers, StorageError>;

    //**Operations over milestone's schema**//

    async fn insert_milestone(&self, milestone: &Milestone) -> Result<(), StorageError>;
    async fn find_milestone(&self, milestone_hash: MilestoneHash) -> Result<Milestone, StorageError>;
    async fn delete_milestones(
        &self,
        milestone_hashes: HashSet<MilestoneHash>,
    ) -> Result<(), StorageError>;

    //**Operations over state_delta's schema**//

    async fn insert_state_delta(
        &self,
        state_delta: StateDeltaMap,
        index: u32,
    ) -> Result<(), StorageError>;

    async fn load_state_delta(&self, index: u32) -> Result<StateDeltaMap, StorageError>;
}

pub struct Storage<Conn: Connection<Conn>> {
    connection:   Conn,
}

impl Storage<DummyConnection> {
    fn establish_connection(&mut self, url: &str) -> Result<(), ConnectionError> {
        self.connection = DummyConnection::establish_connection(url)?;
        Ok(())
    }
    fn destroy_connection(connection: DummyConnection) -> Result<(), ConnectionError> {
        DummyConnection::destroy_connection();
        Ok(())
    }
}

pub struct DummyConnection {}

impl DummyConnection {
    fn new() -> Self {
        Self {}
    }
}

impl Connection<DummyConnection> for DummyConnection {
    fn establish_connection(url: &str) -> Result<DummyConnection, ConnectionError> {
        Ok(DummyConnection::new())
    }
    fn destroy_connection(connection: DummyConnection) -> Result<(), ConnectionError> {
        Ok(())
    }
}

type DummyStorage = Storage<DummyConnection>;

impl Storable for DummyStorage {
    //Implement all methods here
}

#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}

```

# Drawbacks

There is no reason not to have a storage module

# Rationale and alternatives

- Having storage as a trait allows for multiple backends implementations
- API for the moment is minimal, but more methods will be added when needed

# Unresolved questions

- Which backend should we implement this trait for first?

- If we implement coordicide version first, milestone and snapshot_index does not exist
  
- This is highly dependent on whether or not we are first following IRI too, but if we don't, 
  should we record state_delta? this has to be answered in the context of how we then do snapshot? (research)

- How should we implement `map_xxx_transaction_hashes_to_approvers`? both methods exist due to how we chose to
  impl Tangle  [iotaledger/bee-rfcs#23](https://github.com/iotaledger/bee-rfcs/pull/) 
  and will require a traversal over the entire transaction's database, 
  should we maybe have one method that return both maps as a tuple? (should probably DFS/BFS from the genesis)
  
