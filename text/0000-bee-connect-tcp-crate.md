+ Feature Name: `bee-connect-tcp-crate`
+ Start Date: 2019-09-05
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

Build `bee-connect-tcp`, a crate responsible for byte transmission from and to neighbored bees via TCP.

# Motivation

The IOTA Tangle is a distributed ledger. That requires multiple bee nodes to form a network and exchange gossip. To
build such a gossip protocol, one must allow bee nodes to exchange information by sending/receiving bytes to/from
neighbored bees.

# Detailed design

## General

This crate does not consider any IOTA concepts such as transactions. Instead, it handles the transmission of bytes
without knowing the underlying semantics of these bytes (whether they represent an IOTA transaction or a transaction request).
This crate works closely together with `bee-gossip`, which is responsible for translating between IOTA concepts and their
pure byte representations as well as handling incoming and outgoing traffic.

Further, every `bee-connect` implementation uses a specific protocol (e.g. TCP). Multiple `bee-connect` crate implementations
with different protocols can be installed simultaneously. EEE makes this possible. Each `bee-connect` crate simply ignores
actions related to other protocols.

## Exposed Interface

### NodeID

`NodeID` instances describe how to reach a single external node. They specify the protocol and address of such a node so
that a connection can be established based on those information. They must include a protocol identifier as well as an
unique address.

This struct should actually be independent of the `bee-connect-tcp` crate since it covers every `bee-connect crate`.
For simplicity, it will be put it into this crate for now anyways and factored out later.

### Node

Each `Node` instance exposes a TCP socket to allow other nodes to connect to it. Further, each `Node` instance can connect
to the TCP sockets of other `Node` instances. This specific crate only supports TCP. However, the interface of the `Node`
struct is general enough to allow for other protocol implementations.

```rust
impl Node {
    /// Creates a new node with a socket listening on the protocol and address specified by the NodeID.
    /// Only TCP is supported as protocol in this crate implementation.
    pub fn new(id : &node::NodeID) -> Self { }

    /// Returns the NodeID of this node instance. That NodeID specifies the protocol used and
    /// the address of the exposed ServerSocket.
    pub fn get_id(&self) -> &node::NodeID { }
    
    /// Connects this node to the node addresses by the `peer_id`.
    /// Both nodes need to connect to each-other to peer successfully.
    /// This crate only supports TCP connections.
    pub fn connect_to_peer(&mut self, peer_id : &node::NodeID) -> Result<(), String> { }

    /// Disconnects from a peer. Returns `Err` if not connected to that peer.
    pub fn disconnect_from_peer(&mut self, peer_id : &node::NodeID) -> Result<(), String> { }

    /// Returns a vector containing the [NodeID](crate::node::NodeID) instance of each connected peer.
    pub fn get_peers(&self) -> Vec<node::NodeID> { }

    /// Returns whether the node is connected to a specific peer.
    pub fn is_connected_to_peer(&self, peer_id : &node::NodeID) -> bool { }

    /// Sends a message to a connected peer.
    pub fn send_message_to_peer(&mut self, peer_id : &node::NodeID, message : &message::Message) -> Result<(), String> { }

    /// Sends a message to all connected peers.
    pub fn broadcast_message_to_all_peers(&mut self, message : &message::Message) {}
}
```

# Drawbacks

Without any bee-connect crate, nodes could not exchange information. Therefore this crate seems necessary.

# Rationale and alternatives

- The proposed crate interface abstracts from the implemented protocol. TCP might not be the best protocol but it is simple
to implement and will therefore be used for the beginning. Replacing it with a different protocol later can be done without
breaking other crates.
- The proposed crate interface is intuitive. Completely different alternatives did not naturally come to mind.
- One might consider whether a new connection should be established when `send_message_to_peer()` is called with a `node`
argument for which no connection exists. However, returning an error instead seems more accurate as it highlights an
existing issue rather than trying to solve it behind the curtain.

# Unresolved questions

- Which overhead/meta messages should be exchanged to establish, maintain and close a connection?