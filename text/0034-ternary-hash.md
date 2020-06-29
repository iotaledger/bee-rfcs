+ Feature name: `ternary-hash`
+ Start date: 2019-10-15
+ RFC PR: [iotaledger/bee-rfcs#34](https://github.com/iotaledger/bee-rfcs/pull/34)
+ Bee issue: [iotaledger/bee#77](https://github.com/iotaledger/bee/issues/77)

# Summary

This RFC proposes the ternary `Hash` type, the `Sponge` trait, and two cryptographic hash functions `CurlP` and `Kerl`
implementing it. The 3 cryptographic hash functions used in the current IOTA networks (i.e. as of IOTA Reference
Implementation `iri v1.8.1`) are `Kerl`, `CurlP27`, and `CurlP81`.

Useful links:

+ [Sponge function](https://en.wikipedia.org/wiki/Sponge_function)
+ [The sponge and duplex constructions](https://keccak.team/sponge_duplex.html)
+ [Curl-p](https://github.com/iotaledger/iri/blob/dev/src/main/java/com/iota/iri/crypto/Curl.java)
+ [Kerl specification](https://github.com/iotaledger/kerl/blob/master/IOTA-Kerl-spec.md)
+ [`iri v1.8.1`](https://github.com/iotaledger/iri/releases/tag/v1.8.1-RELEASE)
+ [IOTA transactions](https://docs.iota.org/docs/getting-started/1.0/understanding-iota/transactions)

# Motivation

In order to participate in the IOTA network, an application needs to be able to construct valid messages that can be
verified by other participants in the network. Among other operations, this is accomplished by validating transaction
signatures, transaction hashes and bundle hashes.

The two hash functions currently used are both sponge constructions: `CurlP`, which is specified entirely in balanced
ternary, and `Kerl`, which first converts ternary input to a binary representation, applies `keccak-384` to it, and then
converts its binary output back to ternary. For `CurlP` specifically, its variants `CurlP27` and `CurlP81` are used.

# Detailed design

## Hash

This RFC defines a ternary type `Hash` which is the base input and output of the `Sponge`.
The exact definition is an implementation detail but an example definition could
simply be the following:

```rust
struct Hash([i8; HASH_LENGTH]);
```

Where the length of a hash in units of binary-coded balanced trits would be defined as:
```rust
const HASH_LENGTH: usize = 243;
```

## Sponges

`CurlP` and `Kerl` are cryptographic sponge constructions. They are equipped with a memory state and a function that
replaces the state memory using some input string. A portion of the memory state is then the output. In the sponge
metaphor, the process of replacing the memory state by an input string is said to `absorb` the input, while the process
of producing an output is said to `squeeze` out an output.

The hash functions are expected to be used like this:

- `curlp`
  ```rust
  // Create a CurlP instance with 81 rounds.
  // This is equivalent to calling `CurlP::new(CurlPRounds::Rounds81)`.
  let mut curlp = CurlP81::new();

  // Assume we have some transaction trits, all zeroes for the sake of this example.
  let transaction = TritBuf::<T1B1Buf>::zeros(6561);
  let mut hash = TritBuf::<T1B1Buf>::zeros(243);

  // Absorb the transaction.
  curlp.absorb(&transaction);
  // Squeeze out a hash.
  curlp.squeeze_into(&mut hash);
  ```

- `kerl`
  ```rust
  // Create a Kerl instance.
  let mut kerl = Kerl::new();

  // `Kerl::digest` is a function that combines `Kerl::absorb` and `Kerl::squeeze`.
  // `Kerl::digest_into` combines `Kerl::absorb` with `Kerl::squeeze_into`.
  let hash = kerl.digest(&transaction);
  ```

The main proposal of this RFC are the `Sponge` trait and the `CurlP` and `Kerl` types that are implementing it.
This RFC relies on the presence of the types `TritBuf` and `Trits`, as defined by
[RFC36](https://github.com/iotaledger/bee-rfcs/blob/master/text/0036-ternary.md), which are assumed to be owning and
borrowing collections of binary-encoded ternary in the `T1B1` encoding (one trit per byte).

```rust
/// The common interface of cryptographic hash functions that follow the sponge construction and that act on ternary.
trait Sponge {
    /// An error indicating that a failure has occured during a sponge operation.
    type Error;

    /// Absorb `input` into the sponge.
    fn absorb(&mut self, input: &Trits) -> Result<(), Self::Error>;

    /// Reset the inner state of the sponge.
    fn reset(&mut self);

    /// Squeeze the sponge into a buffer
    fn squeeze_into(&mut self, buf: &mut Trits) -> Result<(), Self::Error>;

    /// Convenience function using `Sponge::squeeze_into` to return an owned version of the hash.
    fn squeeze(&mut self) -> Result<TritBuf, Self::Error> {
        let mut output = TritBuf::zeros(HASH_LENGTH);
        self.squeeze_into(&mut output)?;
        Ok(output)
    }

    /// Convenience function to absorb `input`, squeeze the sponge into a buffer, and reset the sponge in one go.
    fn digest_into(&mut self, input: &Trits, buf: &mut Trits) -> Result<(), Self::Error> {
        self.absorb(input)?;
        self.squeeze_into(buf)?;
        self.reset();
        Ok(())
    }

    /// Convenience function to absorb `input`, squeeze the sponge, and reset the sponge in one go.
    /// Returns an owned version of the hash.
    fn digest(&mut self, input: &Trits) -> Result<TritBuf, Self::Error> {
        self.absorb(input)?;
        let output = self.squeeze()?;
        self.reset();
        Ok(output)
    }
}
```

Following the sponge metaphor, an input provided by the user is `absorb`ed, and an output will be `squeeze`d from the
data structure. `digest` is a convenience method calling `absorb` and `squeeze` in one go. The `*_into` versions of
these methods are for passing a buffer into which the calculated hashes are written. The internal state will not be
cleared unless `reset` is called.

### Design of `CurlP`

`CurlP` is designed as a hash function that acts on a `T1B1` binary-encoded ternary buffer, with a hash length of `243`
trits and an inner state of `729`:

```rust
const STATE_LENGTH: usize = HASH_LENGTH * 3;
```

In addition, a lookup table is used as part of the absorption step.

`2` is used to pad the table since there are only 9 combinations should be taken:

```rust
const TRUTH_TABLE: [i8; 11] = [1, 0, -1, 2, 1, -1, 0, 2, -1, 1, 0];
```

The way `CurlP` is defined, it can not actually fail, because the input or outputs can be of arbitrary size; hence,
the associated type `Error = Infallible`.

`CurlP` has two common variants depending on the number of rounds of hashing to apply before a hash is squeezed.
```rust
#[derive(Copy, Clone)]
pub enum CurlPRounds {
    Rounds27 = 27,
    Rounds81 = 81,
}
```

Type definition:

```rust
struct CurlP {
    /// The number of rounds of hashing to apply before a hash is squeezed.
    rounds: CurlPRounds,

    /// The internal state.
    state: TritBuf,

    /// Workspace for performing transformations.
    work_state: TritBuf,
}
```

In addition, the two wrapper types for the very common `CurlP` variants with `27` and `81` rounds:

```rust
struct CurlP27(CurlP);

impl CurlP27 {
    pub fn new() -> Self {
        Self(CurlP::new(CurlPRounds::Rounds27))
    }
}

struct CurlP81(CurlP);

impl CurlP81 {
    pub fn new() -> Self {
        Self(CurlP::new(CurlPRounds::Rounds81))
    }
}
```

For convenience, they should both dereference to an actual `CurlP`:

```rust
impl Deref for CurlP27 {
    type Target = CurlP;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl DerefMut for CurlP27 {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

impl Deref for CurlP81 {
    type Target = CurlP;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl DerefMut for CurlP81 {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}
```

This allows using them as a `Sponge` as well if there is a blanket implementation like this:

```rust
impl<T: Sponge, U: DerefMut<Target = T>> Sponge for U {
    type Error = T::Error;

    fn absorb(&mut self, input: &Trits) -> Result<(), Self::Error> {
        T::absorb(self, input)
    }

    fn reset(&mut self) {
        T::reset(self)
    }

    fn squeeze_into(&mut self, buf: &mut Trits) -> Result<(), Self::Error> {
        T::squeeze_into(self, buf)
    }
}
```

### Design of `Kerl`

The actual cryptographic hash function underlying `Kerl` is `keccak-384`. The real task here is to transform an input
of 243 (balanced) trits to 384 bits in a correct and performant way. This is done by interpreting the 243 trits as a
signed integer `I` and converting it to a binary basis. In other words, the ternary encoded integer is expressed as the
series:

```
I = t_0 * 3^0 + t_1 * 3^1 + ... + t_241 * 3^241 + t_242 * 3^242,
```

where `t_i` is the trit at the `i`-th position in the ternary input array. The challenge is then to convert this integer
to base `2`, i.e. find a a series such that:

```
I = b_0 * 2^0 + b_1 * 2^1 + ... + b_382 * 2^382 + b_383 * 2^383,
```

with `b_i` the bit at the `i-th` position.

Assuming there exists an implementation of `keccak`, the main work in implementing `Kerl` is writing an efficient
converter between the ternary array interpreted as an integer, and its binary representation. For the binary
representation, one can either use an existing big-integer library, or write one from scratch with only a subset of
required methods to make the conversion work.

#### Important implementation details

##### Conversion is done via accumulation

Because a number like `3^242` does not fit into any existing primitive, it needs to be constructed from scratch by
taking a *big integer*, setting its least significant number to `1`, and multiplying it by `3` `242` times. This is the
core of the conversion mechanism. Essentially, the polynomial used to express the integer in ternary can be rewritten
like this:

```
I = t_0 * 3^0 + t_1 * 3^1 + ... + t_241 * 3^241 + t_242 * 3^242,
  = ((...((t_242 * 3 + t_241) * 3 + t_240) * 3 + ...) * 3 + t_1) * 3 + t_0
```

Thus, one iterates through the ternary buffer starting from the most significant trit, adds `t_242` onto the binary big
integer (initially filled with `0`s), and then keeps looping through it, multiplying it by 3 and adding the next `t_i`.


##### Conversion is done in unsigned space

First and foremost, IOTA is primarily written with balanced ternary in mind, meaning that each trit represents an
integer in the set `{-1, 0, +1}`. Because it is easier to do the conversion in positive space, the trits are shifted
into *unbalanced* space, by adding `+1` at each position, so that each unbalanced trit represents an integer in
`{0, 1, 2}`.

For example, the balanced ternary buffer (using only 9 trits to demonstrate the point) becomes after the shift (leaving
out signs in the unbalanced case):

```
[0, -1, +1, -1, -1, 0, 0, 0, +1] -> [1, 0, 2, 0, 0, 1, 1, 1, 2]
```

Remembering that each ternary array is actually some integer `I`, this is akin to adding another integer `H` to it, with
all trits in the ternary buffer representing it set to `1`, where `I'` is `I` shifted into unsigned space, and the
underscore `_t` means a ternary representation (either balanced or unbalanced):

```
I_t + H_t = [0, -1, +1, -1, -1, 0, 0, 0, +1] + [+1, +1, +1, +1, +1, +1, +1, +1] = I'_t
```

After changing the base to binary using some function which we call `to_bin` and which we require to distribute over
addition (because the base in which a number is represented should have no bearing over adding said numbers), `H` needs
to be subtracted again. We use `_b` to signify a binary representation:

```
I'_b = to_bin(I'_t) = to_bin(I_t + H_t) = to_bin(I_t) + to_bin(H_t) = I_b + H_b
=>
I_b = I'_b - H_b
```

In other words, the addition of the ternary buffer filled with `1`s that shifts all trits into unbalanced space is
reverted after conversion to binary, where the buffer of `1`s is also converted to binary and then subtracted from the
binary unsigned big integer. The result then is the integer `I` in binary.

##### 243 trits do not fit into 384 bits

Since 243 trits do not fit into 384 bits, a choice has to be made about how to treat the most significant trit.
For example, one could take the binary big integer, convert it to ternary, and then check if the 243 are smaller than
this converted maximum 384 bit big int. With `Kerl`, the choice was made to disregard the most significant trit, so that
one only ever converts 242 trits to 384 bits, which always fits.

For the direction `ternary -> binary` this does not pose challenges, other than making sure that one sets the most
significant trit to `0` after the shift by applying `+1` (if one chooses to reuse the array of 243 trits), and by making
sure to subtract `H_b` (see previous section) with the most significant trit set to `0` and all others set to `1`.

The direction `binary -> ternary` (the conversion of the 384-bit hash squeezed from keccak) is the challenging part: one
needs to ensure that the most significant trit is is set to 0 before the binary-encoded integer is converted to ternary.
However, this has to happen in binary space!

Take `J_b` as the 384 bit integer coming out of the sponge. Then after conversion to ternary, `to_ter(J_b) = J_t`, the
most significant trit (MST) of `J_t` might be `+1` or `-1`. However, since by convention the MST has to be 0, one needs
to check whether `J_b` would cause `J_t` to have its MST set after conversion. This is done the following way:

```
if J_b > to_bin([0, 1, 1, ..., 1]):
    J_b <- J_b - to_bin([1, 0, 0, ..., 0])

if J_b < (to_bin([0, 0, 0, ..., 0]) - to_bin([0, 1, 1, ..., 1])):
    J_b <- J_b + to_bin([1, 0, 0, ..., 0])
```

##### Kerl updates the inner state by applying logical not

The upstream keccak implementation uses a complicated permutation to update the inner state of the sponge construction
after a hash was squeezed from it. `Kerl` opts to apply logical not, `!`, to the bytes squeezed from `keccak`, and
updating `keccak`'s inner state with these.

#### Design and implementation

The main goal of the implementation was to ensure that representation of the integer was cast into types. To that end,
the following ternary types are defined as wrappers around `TritBuf` (read `T243` the same way you would think of `u32`
or `i64`):

```rust
struct T242<T: Trit> {
    inner: TritBuf<T1B1Buf<T>>,
}

struct T243<T: Trit> {
    inner: TritBuf<T1B1Buf<T>>,
}
```

where `TritBuf`, `T1B1Buf`, and `Trit` are types and traits defined in the `bee-ternary` crate. The bound `Trit` ensures
that the structs only contain ternary buffers with `Btrit` (for balancer ternary), and `Utrit` (for unbalanced ternary).
Methods on `T242` and `T243` assume that the most significant trit is always in the last position (think “little
endian”).

For the binary representation of the integer, the types `I384` (“signed” integer akin to `i64`), and `U384` (“unsigned”
akin to `u64`) are defined:

```rust
struct I384<E, T> {
    inner: T,
    _phantom: PhantomData<E>.
}

struct U384<E, T> {
    inner: T,
    _phantom: PhantomData<E>.
}
```

`inner: T` for encoding the inner fixed-size arrays used for storing either bytes, `u8`, or integers, `u32`:

```rust
type U8Repr = [u8; 48];
type U32Repr = [u32; 12];
```

The phantom type `E` is used to encode endianness, `BigEndian` and `LittleEndian`. These are just used as marker types
without any methods.

```rust
struct BigEndian {}
struct LittleEndian {}

trait EndianType {}

impl EndianType for BigEndian {}
impl EndianType for LittleEndian {}
```

The underlying arrays and endianness are important, because `keccak` expects bytes as input, and because `Kerl` made the
choice to revert the order of the integers in binary big int. When absorbing into the sponge the conversion thus flows
like this (little and big are short for little and big endian, respectively):

```
Balanced t243 -> unbalanced t242 -> u32 little u384 -> u32 little i384 -> u8 big i384
```

When squeezing and thus converting back to ternary the conversion flows like this:

```
u8 big i384 -> u32 little i384 -> u32 little u384 -> unbalanced t243 -> balanced t243 -> balanced t242
```

To understand the implementation in the prototype, the most important methods are:

```rust
T242<Btrit>::from_i384_ignoring_mst
T243<Utrit>::from_u384
I384<LittleEndian, U32Repr>::from_t242
I384<LittleEndian, U32Repr>::try_from_t242
U384<LittleEndian, U32Repr>::add_inplace
U384<LittleEndian, U32Repr>::add_digit_inplace
U384<LittleEndian, U32Repr>::sub_inplace
U384<LittleEndian, U32Repr>::from_t242
U384<LittleEndian, U32Repr>::try_from_t243
```

# Drawbacks

+ All hash functions, no matter if they can fail or not, have to implement `Error`;
+ There are a lot of types. Is it important to encode that the most significant trit is `0` by having a `T242`?;

# Rationale and alternatives

+ `CurlP` and `Kerl` are fundamental to the `iri v1.8.1` mainnet. They are thus essential for compatibility with it;
+ These types are idiomatic in Rust, and users are not required to know the implementation details of each hash
  algorithm;

# Unresolved questions

+ Parameters are slice reference in both input and output. Do we want to consume value or create a new instance as
  return values?;
+ Implementation of each hash functions and other utilities like HMAC should have separate RFCs for them;
+ Decision on implementation of `Troika` is still unknown;
+ Can (should?) the `CurlP` algorithm be explained in more detail?;
+ The truth table should be expressed in terms of `TritsBuf`;
