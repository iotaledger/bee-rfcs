+ Feature name: `network`
+ Start date: 2020-04-29
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes a networking crate (`bee-network`) to be added to the **Bee** framework that provides means to exchange byte messages with *known* endpoints.

# Motivation

Bee nodes need a way to share messages with each other and other compatible node implementations as part of the IOTA gossip protocol, or they won't be able to form a network and synchronize with each other.

# Detailed design

The functionalities of this crate are relatively low-level from a node perspective in the sense, that they are independent from any specifics defined by the IOTA protocol. Consequently, it has no dependencies on other crates of the framework, and can easily be reused in different contexts.

The aim of this crate is to make it simple and straightforward for developers to build layers on top of it (e.g. a protocol layer) by abstracting away all the underlying networking logic. It is therefore a lot easier for them to focus on the modelling of other important aspects of a node software, like:

* `Peer`s: other nodes in the network (other Bees and Hornets),
* `Message`es: the information exchanged between peers, and its serialization/deserialization.

Given some endpoint identifier `epid`, sending a message to it becomes one line of asynchronous code:

```rust
network.send(SendMessage { epid, bytes: "hello".as_bytes() }).await?;
```

The purpose of this crate is to provide the following functionalities:
* maintain a list of endpoints (not peers!),
* establish and maintain connections with endpoints,
* allow to send, multicast or broadcast byte messages of variable size to any of the endpoints,
* allow to receive byte messages of variable size from any of the endpoints,
* reject connections from unknown endpoints (endpoints, that are not whitelisted),
* manage all connections asynchronously,
* provide an extensible, yet convenient to use API.

The key design decisions are now being discussed in the following sub-sections.

## async/await

This crate is by nature very much dependent on events happening outside of the control of the program, e.g. listening for incoming connections from peers, waiting for packets on a specific socket, etc. Hence, this crate under the hood makes heavy use of Rust's concurrency abstractions. Luckily, with the stabilization of the `async/await` syntax, writing asynchronous code has become almost as easy to read, write, and maintain as synchronous code. In a experimental implementation of this crate the [`async_std`](https://github.com/async-rs/async-std) library was used, which comes with asynchronous drop-in replacements for their synchronous equivalents. Additionally, asynchronous mpsc channels were taken from the [`futures`](https://github.com/rust-lang/futures-rs) crate.

## Message passing

This crate favors message passing via channels over globally shared state and locks. Instead of keeping the list of endpoints in a globally accessible hashmap this crate separates and moves such state into workers that run asynchronously, and are listening for commands and events to act upon, and also notify listeners by raising their own events.

## Initialization

In order for this crate to be used, it has to be initialized first by passing a `NetworkConfig` to it:

```rust
struct NetworkConfig {
    // The server or binding TCP port that remote endpoints can connect and send to.
    binding_port: u16,
    // The binding address that remote endpoints can connect and send to.
    binding_addr: IpAddr,
    // The interval between two connections attempts in case of connection losses.
    reconnect_interval: u64,
    // The maximum message length in terms of bytes.
    max_message_length: usize

    ...
}
```

This has the advantage of being easily extensible and keeping the signature of the static `init` function relatively small:

```rust
fn init(config: NetworkConfig) -> (Network, Events, Shutdown) { ... }
```

This function returns a tuple that allows the consumer of this crate to send commands (`Network`), listen to events (`Events`), and gracefully shutdown all running asynchronous tasks within the system (`Shutdown`).

## `Url`, `Address`, and `Port`

In order to send a message to a peer, a node needs to know the peer's address and the communication protocol it uses. This crate provides the following types to deal with that:

* `Address`: a 0-cost wrapper around a `SocketAddr`:
    ```rust
    struct Address(SocketAddr);
    ```

    Note, that this allows to change the internal representation of an address later on.

* `Url`: can be constructed from a `&str`. If successfull it resolves to an Ipv4 or Ipv6 address. Note, that this crate expects the Url string to always provide a scheme (e.g. `tcp://`) and a port (e.g. `15600`) when specifying a peer's address, or the implementation may either panic or return an error result.
    ```rust
    #[non_exhaustive]
    enum Url {
        // Represents the address of an endpoint using TCP to communicate.
        Tcp(Address),
    }
    ```

    Note, that future updates may support different protocols, which is why this enum is attributed as `non_exhaustive`, see `Unresolved Questions` section.

* `Port`: a 0-cost wrapper around a `u16`, which is introduced for type safety and code readability:
    ```rust
    struct Port(u16);
    ```


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

The `EndpointId` can be anything that uniquely identifies an `Endpoint`. In experiments it was implemented as a wrapper around the address.

```rust
#[derive(Clone, Copy, Eq, PartialEq, Hash)]
struct EndpointId(Address);
```

Note, that this `EndpointId` should be `Clone` and `Copy`, and requires to implement `Eq`, `PartialEq`, and `Hash`, because it is used as a hashmap key in several instances.

## `Command`

`Command`s are messages that are supposed to be issued by higher level business logic like the protocol layer. They are implemented as an `enum`, which in Rust is one way of expressing a polymorphic type:

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

The benefit of doing it this way is that the things you can do with this system is encoded on the type level, which makes it possible for the compiler to check, and therefore increases type safety. It is also very easy to add new commands which makes this crate easily extensible.

### `Event`

Similarly to `Command`s, the different kinds of `Event`s in the system are implemented as an `enum` allowing various types of concrete events being sent over an event channel:

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

In contrast to commands though, events are messages created by the system, and those that are considered relevant are published for the user to react upon.

## Workers

There are two essential asynchronous workers in the system that are running for the node's whole lifetime. That is the `EndpointWorker` and the `TcpWorker`.

### EndpointWorker

This worker manages the list of `Endpoint`s and processes the `Command`s issued by the consumer of this crate, and publishes `Event`s to be consumed by the user.


### TcpWorker

This worker is responsible for accepting incoming TCP connection requests from other peers. Once a connection is established two additional asynchronous tasks are spawned that respectively handle incoming and outgoing messages.

Note: connection attempts from unknown IP addresses (i.e. not part of the static peer list) will be rejected.

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

async fn execute(self) { ... }

```

This method will then send a shutdown signal to all registered workers and wait for those tasks to complete.

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