+ Feature name: `bee-storage`
+ Start date: 2019-13-11
+ RFC PR: [iotaledger/bee-rfcs#28](https://github.com/iotaledger/bee-rfcs/pull/)
+ Bee issue:  [iotaledger/bee#65](https://github.com/iotaledger/bee/issues/65)

# Summary

An abstraction for the persistent module which will be exclusively used by the Tangle module
for CRUD operations 

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

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the IOTA and to understand, and for somebody familiar with Rust
to implement. This should get into specifics and corner-cases, and include
examples of how the feature is used.

# Drawbacks

There is no reason not to have a storage module

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Unresolved questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
