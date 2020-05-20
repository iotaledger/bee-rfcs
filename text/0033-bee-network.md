+ Feature name: `network`
+ Start date: 2020-04-29
+ RFC PR: [iotaledger/bee-rfcs#33](https://github.com/iotaledger/bee-rfcs/pull/33)
+ Bee issue: [iotaledger/bee#76](https://github.com/iotaledger/bee/issues/76)

# Summary

This RFC proposes a networking crate (`bee-network`) to be added to the **Bee** framework that provides means to exchange byte messages with *known* endpoints.

# Motivation

Bee nodes need a way to share messages with each other and other compatible node implementations as part of the IOTA gossip protocol or they won't be able to form a network and synchronize with each other.

# Detailed design

The functionalities of this crate are relatively low-level from a node perspective in the sense, that they are independent from any specifics defined by the IOTA protocol. Consequently, it has no dependencies on other crates of the framework, and can easily be reused in different contexts.

The aim of this crate is to make it simple and straightforward for developers to build layers on top of it (e.g. a protocol layer) by abstracting away all the underlying networking logic. It is therefore a lot easier for them to focus on the modelling of other important aspects of a node software, like:

* `Peer`s: other nodes in the network,
* `Message`es: the information exchanged between peers, and its serialization/deserialization.

Given some identifier `epid`, sending a message to its corresponding endpoint becomes a single line of asynchronous code:

```rust
network.send(SendMessage { epid, bytes: "hello".as_bytes() }).await?;
```

The purpose of this crate is to provide the following functionalities:
* maintain a list of endpoints (not peers!),
* establish and maintain connections with endpoints,
* allow to send, multicast or broadcast byte encoded messages of variable size to any of the endpoints,
* allow to receive byte encoded messages of variable size from any of the endpoints,
* reject connections from unknown, i.e. not whitelisted, endpoints,
* manage all connections and message transfers asynchronously,
* provide an extensible, yet convenient-to-use, and documented API.

The key design decisions are now being discussed in the following sub-sections.

## async/await

This crate is by nature very much dependent on events happening outside of the control of the program, e.g. listening for incoming connections from peers, waiting for packets on a specific socket, etc. Hence, - under the hood - this crate makes heavy use of Rust's concurrency abstractions. Luckily, with the stabilization of the `async/await` syntax, writing asynchronous code has become almost as easy to read, write, and maintain as synchronous code. In an experimental implementation of this crate the [`async_std`](https://github.com/async-rs/async-std) library was used, which comes with asynchronous drop-in replacements for their synchronous equivalents found in Rust's standard library `std`. Additionally, asynchronous mpsc channels were taken from the [`futures`](https://github.com/rust-lang/futures-rs) crate.

## Message passing

This crate favors message passing via channels over globally shared state and locks. Instead of keeping the list of endpoints in a globally accessible hashmap this crate separates and moves such state into workers that run asynchronously, and are listening for commands and events to act upon, and also notify listeners by sending events.

## Initialization

In order for this crate to be used, it has to be initialized first by providing a `NetworkConfig`:

```rust
struct NetworkConfig {
    // The server or binding TCP port that remote endpoints can connect and send to.
    binding_port: u16,
    // The binding address that remote endpoints can connect and send to.
    binding_addr: IpAddr,
    // The interval between two connection attempts in case of connection failures.
    reconnect_interval: u64,
    // The maximum message length in terms of bytes.
    max_message_length: usize

    ...
}
```

This has the advantage of being easily extensible and keeping the signature of the static `init` function small:

```rust
fn init(config: NetworkConfig) -> (Network, Events, Shutdown) { ... }
```

This function returns a tuple struct that allows the consumer of this crate to send commands (`Network`), listen to events (`Events`), and gracefully shutdown all running asynchronous tasks that were spawned within this system (`Shutdown`).

See also  [RFC 31](https://github.com/iotaledger/bee-rfcs/blob/master/text/0031-configuration.md), which describes a configuration pattern for Bee binary and library crates.

## `Port`, `Address`, `Protocol`, and `Url`

In order to send a message to an endpoint, a node needs to know the endpoint's address and the communication protocol it uses. This crate provides the following abstractions to deal with that:

* `Port`: a 0-cost wrapper around a `u16`, which is introduced for type safety and better readability:

    ```rust
    struct Port(pub u16);
    ```

    For convenience, `Port` dereferences to `u16`.

* `Address`: a 0-cost wrapper around a `SocketAddr` which provides an adjusted, but overall simpler API than its inner type:

    ```rust
    #[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)
    struct Address(SocketAddr);
    ```

    An `Address` can be constructed from Ipv4 and Ipv6 addresses and a port:

    ```rust
    // Infallible construction of an `Address` from Ipv4.
    fn from_v4_addr_and_port(address: Ipv4Addr, port: Port) -> Self { ... }

    // Infallible construction of an `Address` from Ipv6.
    fn from_v6_addr_and_port(address: Ipv6Addr, port: Port) -> Self { ... }

    // Fallible construction of an `Address` from `&str`.
    async fn from_addr_str(address: &str) -> Result<Self> { ... }
    ```

    Note, that the last function is `async`. That is because of possible delays when performing domain name resolution.

* `Protocol`: an `enum`eration of supported communication protocols:

    ```rust
    #[non_exhaustive]
    enum Protocol {
        Tcp,
    }
    ```

    Note, that future updates may support different protocols, which is why this enum is declared as `non_exhaustive`, see `Unresolved Questions` section.

* `Url`: an `enum`eration that can always be constructed from an `Address` and a `Protocol` (infallible), or from a `&str`, which can fail if parsing or domain name resolution fails. If successfull however, it resolves to an Ipv4 or Ipv6 address stored in a variant of the `enum` depending on the url scheme part. Note, that this crate therefore expects the `Url` string to always provide a scheme (e.g. `tcp://`) and a port (e.g. `15600`) when specifying an endpoint's address.

    ```rust
    #[non_exhaustive]
    enum Url {
        // Represents the address of an endpoint communicating over TCP.
        Tcp(Address),
    }
    ```

    Note, that future updates may support different protocols, which is why this enum is declared as `non_exhaustive`, see `Unresolved Questions` section.

    ```rust
    // Infallible construction of a `Url` from an `Address` and a `Protocol`.
    fn new(addr: Address, proto: Protocol) -> Self { ... }

    // Fallible construction of a `Url` from `&str`.
    async fn from_url_str(url: &str) -> Result<Self> { ... }
    ```

    Note, that the second function is `async`. That is because of possible delays when performing domain name resolution.



## `Endpoint` and `EndpointId`

To model the remote part of a connection, this crate introduces the `Endpoint` type:

```rust
struct Endpoint {
    // The id of the endpoint.
    id: EndpointId,

    // The address of the endpoint.
    address: Address,

    // The protocol used to communicate with that endpoint.
    protocol: Protocol,
}
```

Note: In a peer-to-peer (p2p) network peers usually have two endpoints initially as they are actively trying to connect, but also need to accept connections from their peers. This crate is agnostic about how to handle duplicate connections as this is usually resolved by a handshaking protocol defined at a higher layer.

To uniquely identify an `Endpoint`, this crate proposes the `EndpointId` type, which can be implemented as a wrapper around an `Address`.

```rust
#[derive(Clone, Copy, Eq, PartialEq, Hash)]
struct EndpointId(Address);
```

Note, that this `EndpointId` should be `Clone` and `Copy`, and requires to implement `Eq`, `PartialEq`, and `Hash`, because it is used as a hashmap key in several instances.

## `Command`

`Command`s are messages that are supposed to be issued by higher level business logic like a protocol layer. They are implemented as an `enum`, which in Rust is one way of expressing a polymorphic type:

```rust
enum Command {
    // Adds an endpoint to the list.
    AddEndpoint { url: Url, ... },

    // Removes an endpoint from the list.
    RemoveEndpoint { epid: EndpointId, ... },

    // Attempts to connect to an endpoint.
    Connect { epid: EndpointId, ... },

    // Disconnects from an endpoint.
    Disconnect { epid: EndpointId, ... },

    // Sends a message to a particular endpoint.
    SendMessage { epid: EndpointId, bytes: Vec<u8>, ... },

    // Sends a message to multiple endpoints.
    MulticastMessage { epids: Vec<EndpointId>, bytes: Vec<u8>, ...},

    // Sends a message to all endpoints in its list.
    BroadcastMessage { bytes: Vec<u8>, ... },
}
```

This `enum` makes the things the consumer can do with the crate very explicit and descriptive, and also easily extensible.

### `Event`

Similarly to `Command`s, the different kinds of `Event`s in the system are implemented as an `enum` allowing various types of concrete events being sent over event channels:

```rust
enum Event {
    // An endpoint was added to the list.
    EndpointAdded { epid: EndpointId, ... },

    // An endpoint was removed from the list.
    EndpointRemoved { epid: EndpointId, ... },

    // A new connection is being set up.
    NewConnection { ep: Endpoint, ... },

    // A broken connection.
    LostConnection { epid: EndpointId },

    // An endpoint was successfully connected.
    EndpointConnected { epid: EndpointId, address: Address, ... },

    // An endpoint was disconnected.
    EndpointDisconnected { epid: EndpointId, ... },

    // A message was sent.
    MessageSent { epid: EndpointId, num_bytes: usize },

    // A message was received.
    MessageReceived { epid: EndpointId, bytes: Vec<u8> },

    // A connection attempt should occur.
    TryConnect { epid: EndpointId, ... },
}

```

In contrast to commands though, events are messages created by the system, and those that are considered relevant are published for the consumer to execute custom logic.

## Workers

There are two essential asynchronous workers in the system that are running for the node's whole lifetime. That is the `EndpointWorker` and the `TcpWorker`.

### EndpointWorker

This worker manages the list of `Endpoint`s and processes the `Command`s issued by the consumer of this crate, and publishes `Event`s to be consumed by the user.


### TcpWorker

This worker is responsible for accepting incoming TCP connection requests from other endpoints. Once a connection is established two additional asynchronous tasks are spawned that respectively handle incoming and outgoing messages.

Note: connection attempts from unknown IP addresses (i.e. not part of the static whitelist) will be rejected.

## Shutdown

`Shutdown` is a `struct` that facilitates a graceful shutdown. It stores the sender halfs of `oneshot` channels (`ShutdownNotifier`) and the task handles (`WorkerTask`).

```rust
struct Shutdown {
    notifiers: Vec<ShutdownNotifier>,
    tasks: Vec<WorkerTask>,
}
```

Apart from providing the necessary API to register channels and tasks, executing the shutdown happens by calling:

```rust

async fn execute(self) -> Result<()> { ... }

```

This method will then try to send a shutdown signal to all registered workers and wait for those tasks to complete.

# Drawbacks

* This design might not be the most performant solution as it requires each byte message to be wrapped into a `SendMessage` command, that is then sent through a channel to the `EndpointWorker`. Proper benchmarking is necessary to determine if this design can be a performance bottleneck;
* Currently UDP is not a requirement, so the proposal focuses on TCP only. Nonetheless, the crate is designed in way that allows for simple addition of UDP support;

# Rationale and alternatives

* The reason for not choosing an already available crate, is that there aren't many crates yet, that are implemented in the proposed way, and are also widely used in the Rust ecosystem. Additionally we reduce dependencies, and have the option to tailor the crate exactly to our needs even on short notice;
* There are many ways to implement such a crate, many of which are probably more performant than the proposed one. However, the main focus of this proposal is ease-of-use and extensibility. It leaves performance optimization to later versions and/or RFCs, when more data has been collected through stress tests and benchmarks;
* In particular, the `Command` abstraction could be removed entirely, and replaced by an additional static function for each command. This might be a more efficient thing to do, but it is still unclear how much there is to be gained by doing this, and if this would be necessary taking into account the various other CPU expensive things like hashing, signature verification, etc, that a node needs to do;
* Since mpsc channels are still unstable in `async_std` at the time of this writing, this RFC suggests using those from the `futures` crate. Once this has been stabilized, a swith to those channels might improve performance, since to the authors knowledge those are based on the more efficient `crossbeam` implementation;
* To implement this RFC, any async library or mix can be used. Other options to consider are [`tokio`](https://github.com/tokio-rs/tokio), and for asynchronous channels [`flume`](https://github.com/zesterer/flume). Using those libraries, might result in a much more efficient implementation;
* `EndpointId` may be something different than a wrapper around an `Address`, but instead be derived from some sort of cryptographic hash;

# Unresolved questions

* The design has been tested in a prototype, and it seems to work reliably, so there are no unresolved questions in terms of the validity of this proposal;
* It is still unclear - due to the lack of benchmark results - how efficient this design is; however, due to its asynchronous design it should be able to make use from multicore systems;
* Handling of endpoints with dynamic IPs;
