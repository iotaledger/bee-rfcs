+ Feature name: `bee-network`
+ Start date: 2019-11-06
+ RFC PR: [iotaledger/bee-rfcs#3](https://github.com/iotaledger/bee-rfcs/pull/)
+ Bee issue: [iotaledger/bee#44](https://github.com/iotaledger/bee/issues/44)

# Summary

[...]

# Motivation

Ihe IOTA Tangle is a distributed ledger. It uses a gossip protocol to share messages among the participating nodes.
Each node can have several neighboring nodes. To send messages from one node to another node, a networking interface is required.
In the current state of the IOTA mainnet, nodes only accept TCP connections. This however could change, especially to suite the needs of Internet of Things (IoT).
In an IoT environment for example, there could exist devices which listen to TCP, whereas others only have UDP or Bluetooth ports to connect.
The goal of this RFC is to provide a generic networking interface. It should provide an relatively convenient way for nodes to peer and exchange information with its neighbors.
Furthermore, this networking interface should be able to handle different types of connections at the same time, independent of their nature.

# Detailed design

```rust

use dashmap::DashMap;
use std::ops::DerefMut;

pub enum Message {
    Transaction {},
    CloseConnection {},
}

pub trait Connection: Sized {

    type InitConfig;
    type Error;

    fn connect(config: Self::InitConfig) -> Result<Self, Self::Error>;
    fn send(&mut self, msg: Message) -> Result<(), Self::Error>;
    fn recv(&mut self) -> Result<Message, Self::Error>;
    fn try_recv(&mut self) -> Option<Result<Message, Self::Error>>;

}

pub struct TcpConnection;

impl Connection for TcpConnection {

    type InitConfig = (String, u16);
    type Error = ();

    fn connect(config: Self::InitConfig) -> Result<Self, Self::Error> {
        unimplemented!()
    }

    fn send(&mut self, msg: Message) -> Result<(), Self::Error> {
        let mut buf = [0; 4096];
        let len = buf.len();
        //self.stream.write_all(&buf[..len]);
        Ok(())
    }

    fn recv(&mut self) -> Result<Message, Self::Error> {
        unimplemented!()
    }

    fn try_recv(&mut self) -> Option<Result<Message, Self::Error>> {
        unimplemented!()
    }

}

pub enum PeerInitConfig {
    Tcp(<TcpConnection as Connection>::InitConfig),
}

pub enum PeerConnection {
    Tcp(TcpConnection),
}

#[derive(Copy, Clone, Hash, PartialEq, Eq)]
pub struct PeerId(usize);

pub enum Error {
    NoSuchPeer,
    Tcp(<TcpConnection as Connection>::Error),
}

impl From<<TcpConnection as Connection>::Error> for Error {
    fn from(e: <TcpConnection as Connection>::Error) -> Self {
        Error::Tcp(e)
    }
}

pub struct Router {
    connections: DashMap<PeerId, PeerConnection>,
    id_counter: usize, //AtomicCounter,
}

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
        //self.connections[id].send(msg);

        match self.connections.get_mut(&id).ok_or(Error::NoSuchPeer)?.deref_mut() {
            PeerConnection::Tcp(tcp) => Ok(tcp.send(msg)?),
            //PeerConnection::Udp(udp) => Ok(udp.send(msg)?),
            ///PeerConnection::Bluetooth(bt) => Ok(bt.send(msg)?),
        }

    }

}


```

# Drawbacks

[...]

# Rationale and alternatives

- Why is this design the best in the space of possible designs?

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them?

This design seems relatively intuitive, no other designs were considered.

- What is the impact of not doing this?

Not doing this means no networking layer which implies nodes can not share information.

# Unresolved questions

- How modular do we want this crate to be?
- Which parts should be async?