+ Feature name: `bee-network`
+ Start date: 2019-11-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/)
+ Bee issue: [iotaledger/bee#44](https://github.com/iotaledger/bee/issues/44)

# Summary

The IOTA Tangle is a distributed ledger. Participating nodes use a gossip protocol to exchange messages between them.
Each node can therefore have several neighboring nodes, also called peers, with which it communicates.
In an Internet-of-Things (IoT) environment, a node might also need to understand different protocols (TCP, UDP, NFC, Bluetooth ...) in order to interact with its surroundings.
Interoperability on the Internet of Things is therefore crucial for communication between objects.
This RFC proposes a generic networking interface which abstracts underlying protocols, independent of their nature. It enables protocol additions without losing any functionality and provides an relatively simple and convenient way to exchange messages between neighboring nodes. It should be noted, this RFC proposes a synchronous solution and represents the initial step towards an asynchronous solution. The asynchronous solution will have its own elaboration in a subsequent RFC.

# Motivation

Each node can have several neighboring nodes. To send or receive messages from one node to another node, a networking interface is required.
In the current state of the IOTA mainnet (at the time of writing: IRI v1.8.2), nodes only accept TCP connections. This however could change, especially to suite the needs of Internet-of-Things.
In an IoT environment for example, there could exist devices which listen to TCP, whereas others only have UDP or Bluetooth ports to connect.
The goal of this RFC is to provide a generic networking interface. It should provide an relatively convenient way for nodes to peer and exchange information with its neighbors.
Furthermore, this networking interface should be able to handle different types of connections at the same time, independent of their nature.

# Detailed design

It is assumed that all types of protocols have following similarities:

- there must exist a functionality to send data over it
- there must exist a functionality to get data from it

Furthermore, there might also exist:

- a setup phase, which happens before the actual communication e.g. initialization of parameters
- a teardown phase, which happens after the communication e.g. dissolution / deletion

As a result, the following trait is defined:

*protocol.rs*
```rust
pub trait Protocol {

    // allows to send data to the peer
    fn send(&mut self, msg: &[u8]) -> Result<(), Error>;

    // allows to receive data from the peer
    fn recv(&mut self) -> Result<Vec<u8>, Error>;

    // serves for protocol initialization
    fn setup(config: ProtocolConfig) -> Result<Self, Error> where Self: Sized;

    // serves for dissolution
    fn teardown(self: Box<Self>);

}

pub enum ProtocolConfig {
    Tcp(TcpConfig),
    Udp(UdpConfig)
}
```

As shown in the above `Protocol` trait, each protocol will be initialized by its `ProtocolConfig`.
This `ProtocolConfig` contains all parameters that are needed to initialize/open up a connection to a peer.
The networking interface can be simply extended by adding desired protocols to the defined enums.

**Example**: the TCP protocol could be defined as follows:

*tcp.rs*
```rust
pub struct TcpProtocol {
    stream: TcpStream,
}

impl Protocol for TcpProtocol {

    fn send(&mut self, msg: &[u8]) -> Result<(), Error> {

        self.stream.write(msg)?;
        Ok(())

    }

    fn recv(&mut self) -> Result<Vec<u8>, Error> {
        let mut buf = Vec::new();
        self.stream.read_to_end(&mut buf)?;
        Ok(buf)
    }

    fn setup(config: ProtocolConfig) -> Result<Self, Error> {

        if let ProtocolConfig::Tcp(config) = config {

            let stream = TcpStream::connect(config.address)?;
            Ok(Self{ stream })

        } else {

            Err(Error::new(ErrorKind::InvalidInput, "Invalid config file provided"))

        }

    }

    fn teardown(self: Box<Self>) {
        (*self).stream.shutdown(Shutdown::Both);
    }

}

pub struct TcpConfig {
    pub address: String,
}
```

All peer connections are created and handled by the `Router`.
The `Router` represents the generic networking interface which offers functionality to:

- initialize a peer by providing a `ProtocolConfig` only
- send data to a certain peer, by passing the relevant `PeerId` and the desired data vector.
- receive data from a certain peer
- disconnect from a certain peer

*router.rs*
```rust
#[derive(Copy, Clone, Hash, PartialEq, Eq)]
pub struct PeerId(pub usize);

pub struct Router {
    connections: DashMap<PeerId, Box<Protocol>>,
    id_counter: AtomicUsize,
}
```

Implementation of the `Router` would look as follows:

```rust
impl Router {

    pub fn new () -> Self {
        Self {
            connections: DashMap::default(),
            id_counter: AtomicUsize::new(0)
        }
    }

    fn gen_peer_id(&mut self) -> PeerId {
        let prev = self.id_counter.fetch_add(1, Ordering::SeqCst);
        PeerId(prev)
    }

    pub fn add_peer(&mut self, config: ProtocolConfig) -> Result<PeerId, Error> {

        let id = self.gen_peer_id();

        match config {
            ProtocolConfig::Tcp(x) => {
                let protocol = TcpProtocol::setup(protocol::ProtocolConfig::Tcp(x))?;
                self.connections.insert(id, Box::new(protocol));
            }
            ProtocolConfig::Udp(x) => {
                let protocol = UdpProtocol::setup(protocol::ProtocolConfig::Udp(x))?;
                self.connections.insert(id, Box::new(protocol));
            }
        }

        Ok(id.clone())

    }

    pub fn send_to(&self, id: PeerId, msg: &[u8]) -> Result<(), Error> {

        let mut map_ref_mut= self.connections.get_mut(&id).ok_or(ErrorKind::NotFound)?;
        let mut box_ref_mut = map_ref_mut.deref_mut();
        let protocol_ref_mut = box_ref_mut.deref_mut();

        Ok((protocol_ref_mut.send(msg)?))

    }

    pub fn recv(&self, id: PeerId) -> Result<Vec<u8>, Error> {

        let mut map_ref_mut= self.connections.get_mut(&id).ok_or(ErrorKind::NotFound)?;
        let mut box_ref_mut = map_ref_mut.deref_mut();
        let protocol_ref_mut = box_ref_mut.deref_mut();

        Ok(protocol_ref_mut.recv()?)

    }

    pub fn remove_peer(&mut self, id: PeerId) -> Result<(), Error> {

        let protocol_box = self.connections.remove(&id).ok_or(ErrorKind::NotFound)?.1;
        Protocol::teardown(protocol_box);
        Ok(())

    }

}
```

### Error handling

The standard library of Rust will be used for error handling. It is straightforward and provides lots of tools to handle and propagate errors.
In the above code snippets `ErrorKinds` are used and propagated through the code.


# Advantages

- Being able to open a connection to a peer by simply providing an `InitConfig` to the `Router` seems relatively convenient.
- In addition, this design makes it easy to support new protocols.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them?

This design seems relatively intuitive, no other designs were considered.

- What is the impact of not doing this?

Not doing this means no networking layer which implies nodes can not share information.

# Unresolved questions

- Should we use &[u8] as parameter for the send function and Vec<u8> as parameter for the receive function, or instead use a `Message`(enum) type?