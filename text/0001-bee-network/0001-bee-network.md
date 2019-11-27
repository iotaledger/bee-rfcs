+ Feature name: `bee-network`
+ Start date: 2019-11-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/)
+ Bee issue: [iotaledger/bee#44](https://github.com/iotaledger/bee/issues/44)

# Summary

The IOTA Tangle is a distributed ledger. Participating nodes use a gossip protocol to exchange messages between them.
Each node can therefore have several neighboring nodes with which it communicates.
In an Internet-of-Things environment, a node might need to understand different protocols  (TCP, UDP, Bluetooth, ...) in order to interact with its surroundings.
This RFC proposes a generic, asynchronous networking interface which abstracts specific communication protocols. Furthermore it provides an easy and convenient way to exchange messages between neighboring nodes.

# Motivation

Each node can have several neighboring nodes. To send or receive messages from one node to another node, a networking interface is required.
In the current state of the IOTA mainnet, nodes only accept TCP connections. This however could change, especially to suite the needs of Internet of Things (IoT).
In an IoT environment for example, there could exist devices which listen to TCP, whereas others only have UDP or Bluetooth ports to connect.
The goal of this RFC is to provide a generic networking interface. It should provide an relatively convenient way for nodes to peer and exchange information with its neighbors.
Furthermore, this networking interface should be able to handle different types of connections at the same time, independent of their nature.

# Detailed design

It is assumed that all types of connection have certain similarities:

- a connection needs to be opened / initialized
- there must exist a functionality to send data over it
- there must exist a functionality to read data from it
- after a connection is no longer needed, it will get dissolved

As a result, the following trait is defined:

```rust
pub trait Connection: Sized {

    type InitConfig;
    type Error;

    fn connect(config: Self::InitConfig) -> Result<Self, Self::Error>;
    fn send(&mut self, msg: Message) -> Result<(), Self::Error>;
    fn recv(&mut self, size: u64) -> Result<Message, Self::Error>;
    fn try_recv(&mut self) -> Option<Result<Message, Self::Error>>;
    fn disconnect(self);

}
```

As shown in the above `Connection` trait, each connection will be initialized by its `InitConfig`.
This `InitConfig` contains all parameters that are needed to initialize/open up the connection.
Furthermore, each connection contains its own `Error` type.

**Example**: a TCP connection could be defined as follows:

```rust
pub struct TcpConnection {

    stream: TcpStream,

}

impl Connection for TcpConnection {

    type InitConfig = (IpAdress, Port);
    type Error = ();

    fn connect(config: Self::InitConfig) -> Result<Self, Self::Error> {
                
        let mut stream = TcpStream::connect(ip_address, port)?;
        Ok(TcpConnection{ stream })
        
    }

    fn send(&mut self, msg: Message) -> Result<(), Self::Error> {
       self.stream.write(msg.bytes())?;
    }

    fn recv(&mut self, size: u64) -> Result<Message, Self::Error> {
       self.stream.read(&mut [0; size])?;
    }

    fn try_recv(&mut self) -> Option<Result<Message, Self::Error>> {
        unimplemented!()
    }
    
    fn disconnect(self) {
        self.stream.shutdown(std::net::Shutdown::Both)?;      
    }

}
```

All network connections are created and handled by the `Router`.
The `Router` represents the generic networking interface which offers functionality to:

- connect to a peer by providing a `PeerInitConfig` only
- send data to a certain peer, by passing the relevant `PeerId` and the desired `Message`
- receive data from a certain peer
- disconnect from a certain peer

```rust
pub struct Router {
    connections: DashMap<PeerId, PeerConnection>,
    id_counter: usize, //AtomicCounter,
}
```

`PeerConnection` abstracts different types of connections and will be used as value for the **PeerId -> Connection** mapping.

```rust
#[derive(Copy, Clone, Hash, PartialEq, Eq)]
pub struct PeerId(usize);

pub enum PeerConnection {
    Tcp(TcpConnection),
    //Udp(UdpConnection),
    //Bluetooth(BluetoothConnection),
}
```

Implementation of the `Router` would look as follows:

```rust
impl Router {

    fn gen_peer_id(&mut self) -> PeerId {
        self.id_counter += 1;
        PeerId(self.id_counter)
    }

    pub fn connect_peer(&mut self, config: impl Into<PeerInitConfig>) -> Result<PeerId, Error> {

        let id = self.gen_peer_id();

        let conn = match config.into() {
            PeerInitConfig::Tcp(config) => PeerConnection::Tcp(TcpConnection::connect(config)?),
        };

        self.connections.insert(id, conn);

        Ok(id)
    }

    pub fn send_to(&self, id: PeerId, msg: Message) -> Result<(), Error> {
     
        match self.connections.get_mut(&id).ok_or(Error::NoSuchPeer)?.deref_mut() {
            PeerConnection::Tcp(tcp) => Ok(tcp.send(msg)?),
            //PeerConnection::Udp(udp) => Ok(udp.send(msg)?),
            //PeerConnection::Bluetooth(bt) => Ok(bt.send(msg)?),
        }

    }

}
```

Each `Connection` has its on `InitConfig` type. The `PeerInitConfig` abstracts all different `InitConfigs`.

```rust
pub enum PeerInitConfig {
    Tcp(<TcpConnection as Connection>::InitConfig),
    //Udp(<UdpConnection as Connection>::InitConfig),
    //Bluetooth(<BluetoothConnection as Connection>::InitConfig),
}
```

### Error handling

As already mentioned, each connection has its own error type. The error enum abstracts all different error types which can be propagated trough the data structures.
```rust
pub enum Error {
    NoSuchPeer,
    Tcp(<TcpConnection as Connection>::Error),
}

impl From<<TcpConnection as Connection>::Error> for Error {
    fn from(e: <TcpConnection as Connection>::Error) -> Self {
        Error::Tcp(e)
    }
}
```

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

- How modular do we want this crate to be?
- Which parts should be async and which not?
- Should we use different error types for different functions in the connection?
- Do we need the PeerConnection enum, or could we go directly with Connection trait or maybe an trait object which can be stored in the map of the router?
- Should we use [u8] for message or is it indeed better to use the `Message` type?