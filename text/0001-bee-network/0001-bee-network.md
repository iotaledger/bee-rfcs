+ Feature name: `bee-network`
+ Start date: 2019-02-04
+ RFC PR: [iotaledger/bee-rfcs#27](https://github.com/iotaledger/bee-rfcs/pull/27)
+ Bee issue: [iotaledger/bee#44](https://github.com/iotaledger/bee/issues/44)

# Summary

The IOTA Tangle is a distributed ledger. In contrast to the classic approach, where a general ledger is usually managed by only one instance, any number of basically equal copies of the ledger are maintained locally by different parties. 
A distributed ledger therefore has neither a central administrator nor centralized data storage. A peer-to-peer network is required to ensure replication across nodes.
Each peer, also called node, can have several neighboring nodes which it is connected to.
In the current state of the IOTA mainnet (at the time of writing: IRI v1.8.2), nodes only accept TCP connections. 
This RFC describes a networking interface which provides functionality to maintain neighboring nodes and exchange messages independent of the semantics of the underlying bytes.
The networking interface is designed to be convenient and relatively easy to maintain. In addition, the network interface is designed to be compatible with the IOTA mainnet .
This RFC proposes a asynchronous solution, that aims to be faster and use fewer resources than a corresponding threaded implementation.
 
# Motivation

[...]

# Detailed design

The network layer was designed to satisfy a clean flow of task interaction, where each task is decoupled as much as possible.
This facilitates the testing of individual tasks, the integration of extensions and generally contributes to the maintenance of the code base.

The networking layer makes use of message-passing channels. These channels allow to transfer data between tasks.
It's an increasingly popular approach that ensures safe concurrency [1].

### network_interface.rs

The `network_interface.rs` is the core of the networking layer. It provides the functionality to:

- listen for incoming peers
- add outgoing peers
- kick-out a individual neighbors while preserving overall architecture performance
- send messages through individual write tasks
- receive messages through individual read tasks

The signature of the following `bind` function shows different `Sender` and `Receiver` handles.
These handles belong to specific message-passing channels which enable interaction with the network interface.

```rust
pub async fn bind (

    server_config: TcpServerConfig,
    peers_to_add_receiver: Receiver<TcpClientConfig>,
    received_messages_sender: Sender<ReceivedMessage>,
    messages_to_send_receiver: Receiver<MessageToSend>,
    peers_to_remove_receiver: Receiver<SocketAddr>,
    graceful_shutdown_receiver: Receiver<()>,
    connected_peers_sender: Sender<SocketAddr>

    ) -> Result<(), Error> {

    // bind server
    let listener = TcpListener::bind(server_config.address).await?;
    let (bind_task_shutdown_sender, mut bind_task_shutdown_receiver) = mpsc::unbounded();
    let (mut tcp_stream_sender, tcp_stream_receiver) = mpsc::unbounded();

    // spawn add_peer_task
    let (add_peer_task_shutdown_sender, add_peer_task_shutdown_receiver) = mpsc::unbounded();
    let add_peer_task = task::spawn(add_peer::add_peer(add_peer_task_shutdown_receiver, peers_to_add_receiver, tcp_stream_sender.clone()));

    // process_stream_task
    let (read_task_sender, read_task_receiver) = mpsc::unbounded();
    let (write_task_sender, write_task_receiver) = mpsc::unbounded();
    let process_stream_task = task::spawn(process_stream::process_stream(tcp_stream_receiver, read_task_sender, write_task_sender));

    // start read_task broker
    let shutdown_handles_of_read_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>> = Arc::new(Mutex::new(HashMap::new()));
    let read_task_broker_task = task::spawn(read_task_broker::read_task_broker( read_task_receiver, received_messages_sender, Arc::clone(&shutdown_handles_of_read_tasks)));

    // start assign_message
    let (assign_message_task_shutdown_sender, assign_message_task_shutdown_receiver) = mpsc::unbounded();
    let senders_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<MessageToSend>>>> = Arc::new(Mutex::new(HashMap::new()));
    let assign_message_task = task::spawn(assign_message::assign_message(assign_message_task_shutdown_receiver, messages_to_send_receiver, Arc::clone(&senders_of_write_tasks)));

    // start write_task broker
    let shutdown_handles_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>> = Arc::new(Mutex::new(HashMap::new()));
    let write_task_broker_task = task::spawn(write_task_broker::write_task_broker( write_task_receiver, Arc::clone(&senders_of_write_tasks),Arc::clone(&shutdown_handles_of_write_tasks), connected_peers_sender));

    // remove_peer task
    let (remove_peer_shutdown_sender, remove_peer_shutdown_receiver) = mpsc::unbounded();
    let remove_peer_task = task::spawn(remove_peer::remove_peer(remove_peer_shutdown_receiver, peers_to_remove_receiver, Arc::clone(&shutdown_handles_of_read_tasks), Arc::clone(&shutdown_handles_of_write_tasks), Arc::clone(&senders_of_write_tasks)));

    // graceful shutdown
    task::spawn(graceful_shutdown::graceful_shutdown(graceful_shutdown_receiver, bind_task_shutdown_sender, add_peer_task_shutdown_sender, assign_message_task_shutdown_sender, remove_peer_shutdown_sender));

    // listen for incoming connections
    let mut incoming = listener.incoming();

    loop {

        let stream_result = select! {

            stream_option = incoming.next().fuse() => match stream_option {

               Some(stream_result) => stream_result,

               // The stream of connections is infinite, i.e awaiting the next connection will never result in None
               // https://docs.rs/async-std/0.99.9/async_std/net/struct.TcpListener.html#method.incoming
               None => {
                    unreachable!();
               }

            },

            void = bind_task_shutdown_receiver.next().fuse() => match void {
                Some(()) => break,
                None => break,
            }

        };

        match stream_result {

            Ok(stream) => {
                tcp_stream_sender.send(stream).await.unwrap();
            },

            Err(_error) => {
                eprintln!("can not accept client");
            }

        }

    }

    drop(tcp_stream_sender);

    add_peer_task.await;
    process_stream_task.await;
    read_task_broker_task.await;
    assign_message_task.await;
    write_task_broker_task.await;
    remove_peer_task.await;

    Ok(())

}
```

The `ServerTcpConfig` contains server relevant configuration details. It contains the address to which the server should be bound.
Once the server successfully bound to the provided socket address, multiple sub-tasks are started. Each of these sub-tasks will be described subsequently.
After all sub-tasks are started, `bind` will listen for incoming connections. It will match the result, and forward the new stream to the `process_stream.rs` on success.
The listening on new incoming connections happens within the select macro. This macro selects an event from a number of receivers.
It races the events and terminates with whatever was faster. In case of `graceful_shutdown_receiver` was triggered, `bind` breaks the loop and waits for completion of its sub-tasks.

### add_peer.rs

The following function allows to add peers. It receives `TcpClientConfig` which contains all relevant data to establish an outgoing connection to a peer. 
Similar to the `bind` function, streams will be sent to the `process_stream.rs`.

```rust
pub async fn add_peer(mut shutdown_receiver: Receiver<()>, mut client_config_receiver: Receiver<TcpClientConfig>, mut tcp_stream_sender: Sender<TcpStream>) {

    loop {

        let client_config = select! {

            client_config = client_config_receiver.next().fuse() => match client_config {
               Some(peer_config) => peer_config,
               None => break,
            },

            void = shutdown_receiver.next().fuse() => match void {
                Some(()) => break,
                None => break,
            }

        };

        match TcpStream::connect(client_config.address.clone()).await {

            Ok(stream) => {
                tcp_stream_sender.send(stream).await.unwrap();
            },

            Err(_error) => {
                eprintln!("can not accept client {}", client_config.address.clone());
            }

        }

    }

}
```

### process_stream.rs

The following function receives tcp streams sent by `bind` as well as `add_peer_task`. Once a stream is available, a reference will be sent to the `read_task_broker.rs` as well as the `write_task_broker` which process it further.

```rust
pub async fn process_stream(mut tcp_stream_receiver: Receiver<TcpStream>, mut read_task_sender: Sender<Arc<TcpStream>>, mut write_task_sender: Sender<Arc<TcpStream>>) {

    while let Some(stream) = tcp_stream_receiver.next().await {
        let stream = Arc::new(stream);
        read_task_sender.send(Arc::clone(&stream)).await.unwrap();
        write_task_sender.send(Arc::clone(&stream)).await.unwrap();
    }

}
```

### read_task_broker.rs

The `read_task_broker` receives and processes the reading-halfs of the client streams. For each received reading-half, a respective `read_task` is started.

```rust
pub async fn read_task_broker(mut read_task_receiver: Receiver<Arc<TcpStream>>, received_messages_sender: Sender<ReceivedMessage>, shutdown_handles_of_read_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>>) {

    while let Some(stream) = read_task_receiver.next().await {

        match stream.peer_addr() {

            Ok(address) => {
                let (read_task_shutdown_sender, read_task_shutdown_receiver) = mpsc::unbounded();
                let shutdown_handles_of_read_tasks: &mut HashMap<SocketAddr, Sender<()>> = &mut *shutdown_handles_of_read_tasks.lock().await;
                shutdown_handles_of_read_tasks.insert(address, read_task_shutdown_sender);
                spawn_and_log_error(read_task(read_task_shutdown_receiver, stream, received_messages_sender.clone()));
            },

            Err(e) => {
                eprintln!("{}", e);
            }

        }

    }

}

async fn read_task(mut shutdown_task: Receiver<()>, stream: Arc<TcpStream>, mut received_messages: Sender<ReceivedMessage>) -> Result<(), Error> {

    let mut reader = BufReader::new(&*stream);

    loop {

        // 1) Check message type
        let mut message_type_buf = [0u8; 1];
        select! {
            result = reader.read_exact(&mut message_type_buf).fuse() => result?,
            void = shutdown_task.next().fuse() => break
        }
        let message_type = u8::from_be_bytes(message_type_buf);

        // 2) Check message length
        let mut message_length_buf = [0u8; 2];
        select! {
            result = reader.read_exact(&mut message_length_buf).fuse() => result?,
            void = shutdown_task.next().fuse() => break
        }
        let message_length_as_usize = usize::try_from(u16::from_be_bytes(message_length_buf));

        if let Err(_) = message_length_as_usize {
            return Err(Error::new(ErrorKind::InvalidInput, "Can't convert message_length to usize"));
        }

        let message_length = message_length_as_usize.unwrap();

        // 3) Parse message based on type and length
        match message_type {

            1 => {

                let mut test_message_buf = vec![0u8; message_length];
                select! {
                    result = reader.read_exact(&mut test_message_buf).fuse() => result?,
                    void = shutdown_task.next().fuse() => break
                }

                if let Err(_) = received_messages.send(ReceivedMessage{from: stream.peer_addr()?, msg: MessageType::Test(Message::new(test_message_buf)? ) }).await {
                    break;
                }

            },

            _ => return Err(Error::new(ErrorKind::InvalidInput, "Invalid message type"))

        }

    }

    Ok(())

}

fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()> where F: Future<Output = Result<(), Error>> + Send + 'static {
    task::spawn(async move {
        if let Err(e) = fut.await {
            eprintln!("{}", e)
        }
    })
}
```

### assign_message.rs

```rust
pub async fn assign_message(mut shutdown_receiver: Receiver<()>, mut messages_to_send_receiver: Receiver<MessageToSend>, senders_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<MessageToSend>>>>) {

    loop {

        let message = select! {

            message_option = messages_to_send_receiver.next().fuse() => match message_option {
               Some(message) => message,
               None => break
            },

            void = shutdown_receiver.next().fuse() => match void {
                Some(()) => break,
                None => break,
            }

        };

        let message: MessageToSend = message;
        let map = &*senders_of_write_tasks.lock().await;

        if message.to.is_empty() {

            for (_key, mut value) in map {
                value.send(message.clone()).await.unwrap();
            }

        } else {

            for peer in &message.to {

                let message_sender = map.get(&peer);

                if let Some(mut sender) = message_sender {
                    sender.send(message.clone()).await.unwrap();
                } else {
                    eprintln!("peer with address {} not found", peer);
                }

            }

        }

    }

}
```

### write_task_broker.rs

```rust
pub async fn write_task_broker(mut write_task_receiver: Receiver<Arc<TcpStream>>, senders_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<MessageToSend>>>>, shutdown_handles_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>>, mut connected_peers_sender: Sender<SocketAddr>) {

    while let Some(stream) = write_task_receiver.next().await {

        match stream.peer_addr() {

            Ok(address) => {

                // register shutdown_sender of individual write_task
                let (write_task_shutdown_sender, write_task_shutdown_receiver) = mpsc::unbounded();
                let shutdown_handles_of_write_tasks: &mut HashMap<SocketAddr, Sender<()>> = &mut *shutdown_handles_of_write_tasks.lock().await;
                shutdown_handles_of_write_tasks.insert(address.clone(), write_task_shutdown_sender);

                // register message_sender of individual write_task
                let (write_task_message_sender, write_task_message_receiver) = mpsc::unbounded();
                let senders_of_write_tasks: &mut HashMap<SocketAddr, Sender<MessageToSend>> = &mut *senders_of_write_tasks.lock().await;
                senders_of_write_tasks.insert(address.clone(), write_task_message_sender);

                connected_peers_sender.send(address).await.unwrap();

                spawn_and_log_error(write_task(write_task_shutdown_receiver, stream, write_task_message_receiver));

            },

            Err(e) => {
                eprintln!("{}", e);
            }

        }

    }

}

async fn write_task(mut shutdown_task: Receiver<()>, stream: Arc<TcpStream>, mut message_receiver: Receiver<MessageToSend>) -> Result<(), Error> {

    let mut stream = &*stream;

    loop {

        let message = select! {

            message = message_receiver.next().fuse() => match message {
               Some(message) => message,
               None => break,
            },

            void = shutdown_task.next().fuse() => break

        };

        let message: MessageToSend = message;

        if !message.to.is_empty() && !message.to.contains(&stream.peer_addr()?) {
            continue;
        }

        match message.msg {

            MessageType::Test(msg) => {

                let message_type = [1u8;1];
                let message_length;

                let message_length_result: Result<u16, std::num::TryFromIntError> = msg.bytes().len().try_into();

                if let Err(_) = message_length_result {
                    return Err(Error::new(ErrorKind::InvalidInput, "Message is too big"));
                }

                message_length = message_length_result.unwrap();

                stream.write_all(&message_type).await?;
                stream.write_all(&message_length.to_be_bytes()).await?;
                stream.write_all(msg.bytes()).await?;

            }

        }

    }

    Ok(())

}

fn spawn_and_log_error<F>(fut: F) -> task::JoinHandle<()> where F: Future<Output = Result<(), Error>> + Send + 'static {
    task::spawn(async move {
        if let Err(e) = fut.await {
            eprintln!("{}", e)
        }
    })
}
```

### remove_peer.rs
```rust
pub async fn remove_peer(mut shutdown_receiver: Receiver<()>, mut peers_to_remove_receiver: Receiver<SocketAddr>, shutdown_handles_of_read_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>>, shutdown_handles_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<()>>>>, senders_of_write_tasks: Arc<Mutex<HashMap<SocketAddr, Sender<MessageToSend>>>>) {

    loop {

        let address = select! {

            address_option = peers_to_remove_receiver.next().fuse() => match address_option {
               Some(address) => address,
               None => break,
            },

            void = shutdown_receiver.next().fuse() => match void {
                Some(()) => break,
                None => break,
            }

        };

        let shutdown_handles_of_read_tasks: &mut HashMap<SocketAddr, Sender<()>> = &mut *shutdown_handles_of_read_tasks.lock().await;
        let shutdown_handles_of_write_tasks: &mut HashMap<SocketAddr, Sender<()>> = &mut *shutdown_handles_of_write_tasks.lock().await;

        match shutdown_handles_of_read_tasks.remove(&address) {

            Some(mut read_task_shutdown_sender) => {
                read_task_shutdown_sender.send(()).await.unwrap()
            },
            None => {
                eprintln!("can not shutdown read_task of {}", address);
            }

        }

        match shutdown_handles_of_write_tasks.remove(&address) {

            Some(mut write_task_shutdown_sender) => {
                write_task_shutdown_sender.send(()).await.unwrap()
            },
            None => {
                eprintln!("can not shutdown write_task of {}", address);
            }

        }

        // remove peer from message_assigner
        (&mut *senders_of_write_tasks.lock().await).remove(&address);

    }

}
```

### graceful_shutdown.rs
```rust
pub async fn graceful_shutdown(

    mut graceful_shutdown_receiver: Receiver<()>,
    mut bind_task_shutdown_sender: Sender<()>,
    mut add_peer_task_shutdown_sender: Sender<()>,
    mut assign_message_task_shutdown_sender: Sender<()>,
    mut remove_peer_task_shutdown_sender: Sender<()>,

    ) {

    while let Some(()) = graceful_shutdown_receiver.next().await {
        bind_task_shutdown_sender.send(()).await.unwrap();
        add_peer_task_shutdown_sender.send(()).await.unwrap();
        assign_message_task_shutdown_sender.send(()).await.unwrap();
        remove_peer_task_shutdown_sender.send(()).await.unwrap();
    }

}
```

### message.rs
```rust
pub trait Message {
    fn new(buf: Vec<u8>) -> Result<Self, Error> where Self: Sized;
    fn bytes(&self) -> &[u8];
}

#[derive(Clone)]
pub enum MessageType {
    Test(TestMessage),
}

#[derive(Clone)]
pub struct TestMessage {
    data: String
}

impl TestMessage {

    pub fn new(data: String) -> Self {
        Self {data}
    }

}

impl Message for TestMessage {

    fn new(buf: Vec<u8>) -> Result<Self, Error> {

        let data = String::from_utf8(buf);

        match data {
            Ok(x) => Ok(Self {data: x}),
            Err(_) => Err(Error::new(ErrorKind::InvalidData,"Invalid data"))
        }

    }

    fn bytes(&self) -> &[u8] {
        self.data.as_bytes()
    }

}

impl fmt::Display for TestMessage {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.data)
    }
}

#[derive(Clone)]
pub struct MessageToSend {
    pub to: HashSet<SocketAddr>,
    pub msg: MessageType
}

#[derive(Clone)]
pub struct ReceivedMessage {
    pub from: SocketAddr,
    pub msg: MessageType
}
```

[...]

### Error handling

The standard library of Rust will be used for error handling. It is straightforward and provides lots of tools to handle and propagate errors.
In the above code snippets `io::Error` are used and propagated through the code.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them?

This design seems relatively intuitive, no other designs were considered.

- What is the impact of not doing this?

Not doing this means no networking layer which implies nodes can not share information.

# Unresolved questions

# References
[1] https://doc.rust-lang.org/book/ch16-02-message-passing.html