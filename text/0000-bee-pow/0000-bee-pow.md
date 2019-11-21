+ Feature name: `bee-pow` 
+ Start date: 2019-10-03
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000) 
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

IOTA is a distributed ledger technology (DLT), in which the transactions that are happening on the network are disseminated employing a gossip protocol. Each node can serve as an entry point for new transactions (issued by its clients), which will then ideally be gossiped to all other nodes in the network (or shard) and become part of the ledger state. Gossipping a received transaction is a service granted to each individual node. But since there are no fees in IOTA, and no restrictions on who can join the network, this opens up opportunities for exploitation.

To mainly address accidental spam (e.g. from a malfunctioning node), this RFC proposes a CPU-bound Proof-of-Work (PoW) mechanism based on the ternary hash function Curl-P-81 and [Hashcash](https://en.wikipedia.org/wiki/Hashcash) with a globally fixed (but adjustable) difficulty. 

# Motivation

It must be stressed that this proposal is by no means a definitive answer to an attacker that can make use of hardware accelerated PoW. However, this mechanism - which is called `PearlDiver` - is currently used in the IOTA mainnet (at the time of writing: IRI v1.8.2) to prevent simple forms of spam. For compatibility reasons it is therefore required to be implemented in the Bee node framework as well.

# Detailed design

The Rust implementation of `PearlDiver` as proposed here is supposed to be as type-safe, efficient, and Rust idiomatic as possible.

## `PearlDiver` overview 
For some transaction data `PearlDiver` finds a piece of data - called a `nonce` - that, when appended and hashed together with the transaction data results in a hash with a certain property. Finding that `nonce` is intended to be a CPU intensive task that requires brute-force, i.e. sampling the search space of possible nonces and rehashing many times. Validating a "powed" transaction however can happen with a single function call, and is therefore orders of magnitude faster. This allows to quickly identify invalid transactions, whereby "invalid" in terms of Hashcash based PoW means a transaction with a nonce, that results in a transaction hash without that property.

## `PearlDiver` algorithm
The basic `PearlDiver` algorithm looks like this:

Given some pre-defined hash constraint `C`, a transaction `T` with a nonce set to 0, and its serialized representation called `data`:
1. hash `data` yielding `H`
2. check if `H` satisfies `C` (e.g. at least last 14 trits are 0). There are two cases:
   1. `C` is satisfied -> stop the algorithm, 
   2. otherwise -> select the next nonce from the search space, modify `data` with it, and return to 1.

## `PearlDiver` assumptions

Instead of making `PearlDiver` extremely generic from the get-go, this proposal makes the following assumptions, and leaves everything else to future RFCs or improvement proposals.

`PearlDiver`
* receives transactions in serialized form using the `t1b1` ternary encoding, and has to convert that into `ptrits`/`t8b2`/ binary-coded ternary (BCT) prior to its execution,
* employs a Curl-P-81 implementation that operates on `ptrits`,
* allows to employ all available CPU cores on the system,
* supports async/non-blocking operation

## The `HashValidator` trait

In IOTA two hash validation schemes are of importance:
* *Hashcash* (the number of zero trits at the end of a transaction hash)
* *Hamming* (TODO)

To allow implementing various of those schemes the following trait is introduced:

```Rust
trait HashValidator {
    fn is_valid(&self, hash: &BCTHash) -> bool;
}
```

This trait requires to implement a method `is_valid` which contains the logic for the hash validation. It expects a reference to a bct-encoded hash, which is also the output of the BCT-Curl implementation. 

`PearlDiver` is bound to a type implementing that trait to determine when a search has successfully completed, that is, when a valid nonce has been found.

**Code Example:**

```Rust
struct Hashcash {
    min_weight_magnitude: usize,
};

impl HashValidator for Hashcash {
    fn is_valid(&self, hash: &BCTHash) -> bool {
        // TODO
    }
}
```

## The `BCTNonceSampler` struct

This RFC proposes a `BCTNonceSampler` struct, that can be configured to start walking the search space at a pre-defined index, and in pre-defined steps. This is used to create disjoint subsets of the whole search space, that concurrent threads can process without doing double work. Since the underlying ...

```Rust

struct BCTNonceSampler {
    start: usize,
    step: usize,
}

impl BCTNonceSampler {
    pub fn new(start: usize, step: usize) -> Self {
        Self {
            start,
            step,
        }
    }
}

```

## The `PearlDiver` struct

This RFC proposes the following `PearlDiver` type:

```Rust
struct PearlDiver<V>
where
    V: HashValidator,
{
    validator: V,
    hasher: BCTCurl,
    sampler: 
}

impl<V> PearlDiver<V>
where
    V: HashValidator,
{
    pub fn new(validator: V) -> Self {
        Self { validator }
    }

    pub fn search(&mut self, mut data: &mut BCTTransaction) {
        loop {
            let hash = self.hasher.hash(data);
            if self.validator.is_valid(&hash) {
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