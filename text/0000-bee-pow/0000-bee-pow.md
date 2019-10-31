+ Feature name: `bee-pow` 
+ Start date: 2019-10-03
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000) 
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

IOTA is a permissionless and feeless network which unavoidably makes it vulnerable to spam and sybil attacks. To make such
attacks more costly this RFC proposes a spam protection mechanism based on Proof-of-Work (PoW) where only those
messages are accepted and propagated by the honest nodes that include a valid nonce value. That nonce serves as a
proof to show that a predefined amount of computational work has been done upfront either by the publisher itself or
by some PoW service provider on its behalf.

# Motivation

Since IOTA is a permissionless and feeless network it needs methods of protecting itself against spam from
intentionally or even unintentionally misbehaving network participants (nodes or clients) that otherwise could
disrupt the network rather cheaply. Furthermore, this mechanism is currently (at the time of this writing) used in
the IOTA mainnet, and necessary to build compatible nodes with this framework.

### Job stories:

When I - as a client - want to send a message, for example a value transaction, I want to be able to find and include
a valid nonce, so that my transaction is accepted and propagated by all honest nodes in the network.

When I - as a node - receive a network message from the gossip pipeline, I want to be able to check if its contained
nonce value satisfies the pre-defined network difficulty, so that I can decide whether to discard or process that
message.

# Detailed design

## Hasher
A hash function always operates on some kind of input data, usually but not necessarily on some kind of `u8` array - and deterministically produces another piece of data - the hash - for example a `u64` that usually takes much less space, and therefore can be used as a label for the data it was derived from. In Rust such a hash function can be generalized using the following trait:

```Rust
pub trait Hasher {
    type Data;
    type Hash;

    fn hash(&mut self, data: &Self::Data) -> Self::Hash;
}
```

Notive that by making `Data` and `Hash` associated types this RFC doesn't make any commitment to the underlying representation which allows for a very flexible design.

## Sampler

The `Sampler` trait allows implementing different types that mutate the input data handed to the hasher on each try. The standard sampler used in IOTA simply starts with a zero nonce and increments it until it finds a valid one. There are, however, other samplers possible, like a random sampler.

```Rust
pub trait Sampler {
    type Data;

    fn next(&mut self, data: &mut Self::Data);
}
```

Like in the `Hasher` trait we now have to make `Data` an associated type as well, so it can always be compatible when used together in `PearlDiver`.

## Tester

The `Tester` trait allows implementing different types that place  plug different validation schemes into `PearlDiver`, like the `Hashcash` or the `Hamming` validation scheme.

```Rust
pub trait Tester {
    type Hash;

    fn is_valid(&self, hash: &Self::Hash) -> bool;
}
```

As with `Sampler` the `Tester` trait needs an associated type so it can always be implemented in a compatible way and operate on something served by a type implementing the `Hasher` trait. 

## PearlDiver

With the traits above a type can now be defined that is called `PearlDiver`. It encapsulates the following algorithm:

Starting with some serialized transaction data (which also encodes a zero nonce):
1. hash the data
2. check if the hash satisifies some pre-defined and adjustable constraint
   1. hash satisfies constraint => stop the algorithm 
   2. hash doesn't satisfy constraint 
      1. select another nonce 
      2. goto 1.

This RFC proposes the following `PearlDiver` type:

```Rust
pub struct PearlDiver<H, S, T, D, R>
where
    H: Hasher<Data = D, Hash = R>,
    S: Sampler<Data = D>,
    T: Tester<Hash = R>,
{
    hasher: H,
    sampler: S,
    tester: T,
}

impl<H, S, T, D, R> PearlDiver<H, S, T, D, R>
where
    H: Hasher<Data = D, Hash = R>,
    S: Sampler<Data = D>,
    T: Tester<Hash = R>,
{
    pub fn new(hasher: H, sampler: S, tester: T) -> Self {
        Self {
            hasher,
            sampler,
            tester,
        }
    }

    pub fn search(&mut self, mut data: &mut D) {
        loop {
            let hash = self.hasher.hash(data);
            if self.tester.is_valid(&hash) {
                break;
            } else {
                self.sampler.next(&mut data);
            }
        }
    }
}
```
## PoW

Proof-of-Work essentially is "just" about finding a piece of data (the Nonce) that is - in some way - associated with the data at hand. This proposal therefore introduces a special `PoW` trait for that.

```Rust
pub trait PoW {
    type Data;
    type Nonce;

    fn get_nonce(&mut self, data: Self::Data) -> PoWResult<Self::Nonce>);
}
```
whereby

```Rust
pub type PoWError = Box<dyn std::error::Error>;
pub type PoWResult<T> = Result<T, PoWError>;
```

Once implemented on a type, doing Proof-of-Work is a simple method call, and everything is hidden behind the scenes. The type could either perform the work locally, or for nodes on restricted devices, be something that queries a subscription-based PoW service.

## Code Example

### Implement the `Hasher` trait for `Curl`

`Curl` is a ternary hash function implementation used in the IOTA ecosystem. Its exact implementation is not of interest in this proposal, just how the `Hasher` trait can be implemented on it:

```Rust
struct Curl<'a> {
    num_rounds: usize,
    phantom: PhantomData<&'a i8>,
}

impl<'a> Curl<'a> {
    // implementation details (inner workings of Curl)
}

impl<'a> Hasher for Curl<'a> {
    type Data = &'a mut [i8; 8019];
    type Hash = [i8; CURL_HASH_LENGTH];

    fn hash(&self, trits: &Self::Data) -> Self::Hash {
        // code goes here that determines the Curl hash from the given trits
    }
}

```

The `PhantomData` field on `Curl` is necessary, because the lifetime is introduced by the associated type `Data`, not by any `Curl` field. There is no overhead in using `PhantomData`.

### Implement the `Sampler` trait for `IncrementalNonceSampler`

The following Rust code shows an example implementation of a ternary based nonce sampler, that simply increments the tested nonce using a ternary increment operation:

```Rust
struct IncrementalNonceSampler<'a> {
    current: [i8; NONCE_TRIT_LENGTH],
    step; usize,
    phantom: PhantomData<&'a i8>,
}

impl<'a> IncrementalNonceSampler<'a> {
    pub fn new(step: usize) -> Self {
        Self {
            current_nonce: [0_i8; NONCE_TRIT_LENGTH],
            step,
            phantom: PhantomData,
        }
    }
}

impl<'a> Sampler for IncrementalNonceSampler<'a> {
    type Data = &'a mut [i8; 8019];

    fn next(&mut self, data: &mut Self::Data) {
        for _ in 0..self.step {
            // this function calls into a ternary crate that provides the ability to increment trit sequences
            increment(&mut self.current_nonce, NONCE_TRIT_LENGTH)
        }
        data[7938..8019].copy_from_slice(&self.curent_nonce[0..NONCE_TRIT_LENGTH]);
    }

}
```

### Implement the `Tester` trait for `Hashcash`

`Hashcash` is a label for something that checks if a certain number of subsequent digits are equal (for example equally 0). The number of digits included is called the difficulty, or in IOTA's case the `minimum weight magnitude`. An example Rust implementation implementing the proposed `Tester` trait would look like this:

```Rust
struct Hashcash {
    pub fn new(minimum_weight_magnitude: usize) -> Self {
        Self {
            minimum_weight_magnitude,
        }
    }
}

impl Tester for Hashcash {
    type Hash = [i8; CURL_HASH_LENGTH];

    fn is_valid(&self, hash: &Self::Hash) -> bool {
        let start_index = CURL_HASH_LENGH - self.minimum_weight_magnitude;
        for i in start_index..CURL_HASH_LENGTH {
            if hash[i] != 0 {
                return false;
            }
        }
        true
    }
}
```

### Implement the `PoW` trait for `CurlHashcashPoW`

To abstract the details of the hashing function, the sampler and the tester away it makes sense to introduce a wrapper type `CurlHashcashPoW`, which operates on 8019 trits, uses Curl as hasher, and implements the `PoW` trait:

```Rust
struct CurlHashcashPoW<'a> {
    pearldiver: PearlDiver<Curl<'a>, IncrementalNonceSampler<'a>, Hashcash, &'a mut [i8; 8019], [i8; 243]>,
}

impl<'a> CurlHashcashPoW<'a> {
    pub fn new(num_rounds: usize, minimum_weight_magnitude: usize) -> Self {

        let curl = Curl::new(num_rounds);
        let sampler = IncrementalNonceSampler::new(1);
        let hashcash = Hashcash::new(minimum_weight_magnitude);

        Self { pearldiver: PearlDiver::new(curl, sampler, hashcash) }
    }
}

impl<'a> PoW for CurlHashcashPow<'a> {
    type Input = &'a mut [i8; 8019];
    type Nonce = [i8; NONCE_TRIT_LENGTH];

    fn get_nonce(&mut self, mut input: Self::input) -> PoWResult<Self::Nonce> {
        self.pearldiver.search(&mut input);

        let mut nonce = [0_i8; NONCE_TRIT_LENGTH];
        nonce[0..NONCE_TRIT_LENGTH].copy_from_slice(&input[7938..8019]);

        Ok(nonce)
    }
}
```

### Use `CurlHashcashPoW` type to perform Proof-of-Work on a transaction and update its nonce

Having all that in place doing Proof-of-Work for a single transaction (provided by some other crate) can be done with a few simple instructions: 

```Rust
// Create a Curl and Hashcash based PoW instance with:
//  number of Curl rounds: 81
//  minimum_weight_magnitude: 9
let mut pow = CurlHashcashPoW::new(81, 9);

// get the current trit represantation of a transaction (the last 81 trits should be all 0)
let tx_trits = transaction.get_trits(); 

// 
let nonce = match pow.get_nonce(&mut tx_trits) {
    Ok(nonce) => nonce,
    Err(e) => panic!("{:?}", e),
}

transaction.set_nonce(nonce);

```
Note that this proposal doesn't make suggestions about how to parallelize the Proof-Of-Work algorith, how which async library to use. The implementor can have a look at `iota.rs` for an inspiration how to do this with the `crossbeam` crate.


For more information on PoW, and how it can be impemented in Rust the following links might be helpful:
* [Minimum Weight Magnitude](https://docs.iota.org/docs/dev-essentials/0.1/concepts/minimum-weight-magnitude),
* [Transaction/Bundle RFC](https://github.com/iotaledger/bee-rfcs/pull/20)
* [iota.rs](https://github.com/iotaledger/iota.rs)

# Drawbacks

A spam protection mechanism based on PoW 
* consumes energy for doing throw-away calculations,
* slows down all participants even if there's no attack (always-on defense),
* is not favorable for battery-driven devices, many of which form the IoT space,
* amplifies problems that arise in networks where nodes have diverse computation capabilities like in the IoT space

# Rationale and alternatives

- Why is this design the best in the space of possible designs? 

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them? 

No other designs have been considered. Other anti-spam mechanisms are still being researched.

- What is the impact of not doing this?

Not doing this means that nodes built with this framework are incompatible with the current IOTA mainnet.


# Unresolved questions

- How modular do we want this crate to be?
- Should we focus on Curl or make the hashfunction plug and play?
- Should we design this a kind of service component employing its own runtime that performs PoW jobs asynchronously?
- Should support for external PoW be part of the implementation following the proposal?