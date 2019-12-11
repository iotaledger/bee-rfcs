+ Feature name: `bee-pow` 
+ Start date: 2019-10-03
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000) 
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC intends to add a Rust implementation of the local Proof-of-Work algorithm *PearlDiver* as used in many places throughout the IOTA codebase. The performance should be on par with the (non-SIMD) C version, but offer more type-safety, better usability and integration into an asynchronous context.


# Motivation

At the time of this writing IOTA utilizes a form of computational Proof-of-Work (PoW) to detect a certain category of invalid messages at the networking layer. For that, a node has to calculate and check the hash of each uniquely received message for a specific property. In IOTA this usually means that a pre-defined number of trailing trits in the hash (Curl) all have to be zero for it to be considered valid in that regard. Because of the one-wayness of the underlying hashing function, finding a hash with such a property involves brute-force on the issuing side. This attaches a certain real-world cost to each network message, which in theory should discourage sending useless messages. However, the proposed implementation cannot prevent more sophisticated spam making use of hardware accelerated PoW, and its main purpose is to achieve compatibility with current IOTA networks. 


# Detailed design

In the current transaction model the following constants are relevant for a `PearlDiver` implementation.

## Proposed constants

```Rust
/// The size of a transaction in trits (at the time of this writing).
const TRANSACTION_LENGTH: usize = 8019;

/// The size of a nonce in trits.
const NONCE_LENGTH: usize = 81;

/// The full state size of the Curl hash function.
const CURL_STATE_LENGTH: usize = 729;
```

## Proposed new-types

To increase type-safety and readibility this RFC proposes the following new-types (wrapper types):

```Rust
/// A new-type to represent the input for PearlDiver, which are balanced trits {-1, 0, 1} (T1B1 encoding).
struct Input([i8; TRANSACTION_LENGTH]);

/// A new-type to represent the output of PearlDiver in balanced trits {-1, 0, 1} (T1B1 encoding).
struct Nonce([i8; NONCE_LENGTH]);

/// A new-type to represent a valid number of logical cores to use for `PearlDiver`. Its purpose is to guarantee only valid `usize`s.
#[derive(Clone)]
struct CoreCount(usize);

/// A new-type to represent a valid minimum weight magnitude value, that is, the PoW difficulty. Its purpose is to guarantee only valid `usize`s.
#[derive(Clone)]
struct Difficulty(usize);
```

## Proposed models

```Rust
/// The different states `PearlDiver` can be in.
#[derive(Debug, Eq, PartialEq)]
enum PearlDiverState {
    /// The initial state after instantiation.
    Initialized,
    /// The state while `PearlDiver` is searching for a nonce.
    Searching,
    /// The state when `PearlDiver` is aborted. 
    Cancelled,
    /// The state when `PearlDiver` completed either with a nonce as a result, or no result at all. 
    Completed,
}

/// The `PearlDiver` abstraction itself.
#[derive(Clone)]
struct PearlDiver {
    // The number of cores to use.
    num_cores: CoreCount,
    // The number of trailing zero trits to search for.
    difficulty: Difficulty,
    // The shared state of `PearlDiver`. But reads will happen much more often then writes.
    state: Arc<RwLock<PearlDiverState>>,
}
```

Since doing something like Proof-of-Work is a time-consuming operation (doens't matter if done locally or remotely) it matches the very definition of a `Future` (a computation that may yield a result at some point in the future). It allows us to write high-performant non-blocking/asynchronous code. For that this RFC proposes to model the `PearlDiver` search as a `std::future::Future`.

```Rust
/// A `std::future::Future` that completes once `PearlDiver` finds Some(nonce), None or gets cancelled.
struct PearlDiverSearch {
    // A cloned instance of `PearlDiver`
    pearldiver: PearlDiver,
}

impl std::future::Future for PearlDiverSearch {
    type Ouput = Option<Nonce>;

    fn poll(self: std::pin::Pin<&mut Self>, _: &mut std::task::Context) -> Poll<Self::Output> {
        // this future continuesly polls `PearlDiver`s state, which can be updated by any of the threads
        // it completes, once a poll yields either:
        // * PearlDiverState::Cancelled
        // * PearlDiverState::Completed(None)
        // * PearlDiverState::Completed(Some(nonce))
    }
}
```

## Proposed `PearlDiver` API:

```Rust
impl PearlDiver {
    /// Creates a new `PearlDiver`, and tells it to use a certain number of cores, and a pre-defined difficulty.
    pub fn new(num_cores: CoreCount, difficulty: Difficulty) -> Self {}
    /// Initiates the search for a nonce by immediately return a future that can be spawned onto some runtime.
    pub fn search(&mut self, input: &Input) -> PearlDiverSearch {}
    /// Cancels `PearlDiver` which causes the future to complete immediatedly without a result.
    pub fn cancel(&mut self) {}
}

```

A prototype implementation can be found here:

[pow-preview](https://github.com/Alex6323/pow-preview.git)

# Drawbacks

* There are only drawbacks around Proof-of-Work in general, as it can be quite a burden for weakish IoT devices. Solutions to combat that disadvantage are being researched. The proposed design is rather minimalistic, and already implemented similarly in `iotaledger/iota.rs`.

# Rationale and alternatives

- Why is this design the best in the space of possible designs? 

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them? 

No other designs have been considered. Other anti-spam mechanisms are still being researched.

- What is the impact of not doing this?

Not doing this means that nodes built with this framework are incompatible with the current IOTA mainnet.


# Unresolved questions

- Should this RFC be restricted to local PoW?