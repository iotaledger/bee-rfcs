- Feature name: `chronicle-module`
- Start date: 2019-10-14
- RFC PR: [iotaledger/bee-rfcs#22](https://github.com/iotaledger/bee-rfcs/pull/22)
- Bee issue: [iotaledger/bee#64](https://github.com/iotaledger/bee/issues/64)

# Summary

This RFC proposes a `chronicle` module to provide a flexible and powerful framework for storing/accessing
historical/incoming transactions for a long period of time.

# Motivation

In current IOTA nodes, old transactions are removed after snapshots, in order to reduce the storage cost. It does not
mean, however, the old transactions are valueless in real applications. For data analytics point of view, it is
essential to keep all of the historical data (if the cost is affordable) to ensure no information is missed. It is
impossible to ensure that the historical data are useless for the target application before the corresponding
research/analysis on the historical data is done. Also, to apply machine learning (including deep learning) for model
building on the data, a huge amount of data is crucial for training and testing. Hence the chronicle module is
important and should be developed. In the following we list some application examples: 1) From the aspect of
financial analysis, one may want to build a model to predict the future prices and/or transaction volumes, which
needs a large amount of historical data for training and testing. 2) Developers can further improve/enhance the
framework design and structure by identifying the weakness of them. 3) A company/organization can collect transaction
data and provide customized services.

This module provides a framework to implement chronicle in an efficient and robust way, as well as
transaction/bundle-related metrics, dashboard, and analytics examples.

# Detailed design

The chronicle design should leverage the fundamental crates used in IOTA Bee, including transaction/bundle crates, so
as to be consistent with other Bee projects, efficient to perform operations, and easy to maintain. The chronicle
module subscribes the unconfirmed transactions received by ledger node(s) (achieved by [gossip
crates](https://To-DO)), filter out unnecessary transactions, and then store the transactions into cloud databases
(ScyllaDB is adopted in our first version). The user must define the _time to live_ (TTL) for different transaction
categories, which is defined as how many seconds the transaction should be reserved, after that the transactions will
be deleted automatically. If TTL is defined as 0, then the transactions of the category will be reserved until
deletion operation is issued by the user. The TTL ranges from 0 to 630,720,000 seconds (20 years).

The filter/TTL behavior should be based on the transaction categories, which are classified by

- Confirmed/unconfirmed transactions
- Valid/invalid transactions after verification
- Zero/non-zero value transactions

This crate mainly focuses on the `runtime` implementation, which makes the transaction
storing/retrieving/adding/deleting in the database efficient.
In chronicle runtime, **scheduler** represents the one runs inside a single thread and handles the execution flow of
(channel -> executor -> reactor), and **ring** represents [scylla-visualized-ring](https://docs.scylladb.com/architecture/ringarchitecture/).

The high-level of the chronicle event (has `future` trait) we have two options for the flow loop:

Please note: both options are based on shared-nothing-architecture

## Runtime Design Option 1: channel -> executor -> reactor

**Channel**

- Unbounded channel per thread.
- Channel enables the thread to receive events in FIFO fashion.

**Executor**

- Runs one operation to collect the events from the channel.
- Has a loop through the collected events.
- The event’s data-structure enables the executor to fetch the right task which is ready to do progress from the
  tasks_map. An event contains a tuple (`actor_id`, `msg_function`, `msg`).
  - `actor_id` is the task_key in the tasks_map, therefore we will use the actor_id to fetch the task(actor) from the map.
  - `msg_function` indicates which function in the actor should be executed with the params (`msg`, `actor_state`).
    A toy example of adding msg and actor_state:
    ```rust
    fn add(msg ,actor_state) -> updated_actor_state{
        msg + actor_state // 1 + 9
    }
    ```
  - the complexity of the executor in Option 1 is ~O(n) where n is the number of actors that are ready to do progress.

**Reactor**

- It should receive the io_events from the tasks somehow, we can have another channel per thread where the tasks send
  the io_events to it or if possible we return the io_events (i.e., socket.write/read, if any) from the poll function.
  For instance:
  ```rust
  Async::ready(io_events) -> io_events_list.push(io_events)
  ```
- The reactor should have access to all the sockets in the thread

## Runtime Design Option 2: channel -> executor -> reactor

Note: each actor has its own mpsc channel

**Executor**

- Owns a `vector<Actor>` with all the actors that belong to the executor's thread.
- Has a loop through all the actors using SIMD/AVX(if possible) to check on which actors are ready to do progress, we
  can determine if the actor is ready to do progress by checking if there are any events in its channel, so this
  eliminate the need for notifications to wake up given actors, because we are already looping through all the actors.
- The possibility of leveraging SIMD/AVX on mutual independent actors. - the complexity of the executor in Option 2
  is ~O(n/w) where n is the total number of all the actors that belong to a given thread. and w is the instruction
  weight, in general w=1, but w=width in case we want to leverage SIMD/AVX.

**Reactor**

- As above.

The chronicle framework also provides useful metrics for ease of data analytics. Good examples of these metrics are
in [tanglescope](https://github.com/iotaledger/entangled/tree/develop/tanglescope) in IOTA entangled project.

# Drawbacks

- Using this chronicle framework needs cloud to store the historical/new-coming transactions, consuming more power
  and storage than to maintain a node w/ periodic data deletion in snapshots.

# Rationale and alternatives

- ScyllaDB is adopted as the Chronicle database currently
- The use can define the deletion period of the transactions with specific characteristics. The characteristics contain
  - **To Do**

# Unresolved questions

- Is TTL range needed to be modified?
- Should the runtime details be separated to different RFCs?
- How do we guarantee/verify that all txs are collected in Chronicle nodes? (Subscribe redundant IOTA nodes?)
- Do we need extra mechanism to verify that no txs are lost?
- How do we detect that some txs are lost?
- How do we recollect the lost txs when exceptions occur (e.g., zmq disconnected)
- [Options] Which option do you prefer, notifications-based with channel per thread or iteration-like with channel per actor?
- [Executor] Should we implement the actor (top level of a future) using async block, or implementing a custom futures?
- [Reactor] What syscalls we should apply?
  - Adopt epoll? (notification based)
  - Adopt io_submit/io_getevents? (Batching io_events, where io_submit is blocking because it does the most work)
  - Adopt Io_uring?
  - Adopt epoll-like syscalls on top of io_uring?
