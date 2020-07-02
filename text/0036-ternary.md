+ Feature name: `ternary`
+ Start date: 2020-05-28
+ RFC PR: [iotaledger/bee-rfcs#36](https://github.com/iotaledger/bee-rfcs/pull/36)
+ Bee issue: [iotaledger/bee#80](https://github.com/iotaledger/bee/issues/80)

# Summary

This RFC introduces both the API and implementation of the `bee-ternary`
crate, a general-purpose ternary manipulation crate written in Rust that may be
used to handle ternary data in the IOTA ecosystem.

# Motivation

Ternary has been a fundamental part of the IOTA technology and is used
throughout, including for fundamental features like wallet addresses and transaction
identification.

More information about ternary in IOTA can be found
[here](https://docs.iota.org/docs/getting-started/0.1/introduction/ternary).

*Note that the IOTA foundation is seeking to reduce the prevalence of ternary
in the IOTA protocol to better accomodate modern binary architectures. See
[Chrysalis](https://blog.iota.org/chrysalis-b9906ec9d2de) for more information.*

Manipulating ternary in an ergonomic manner on top of existing binary-driven
code can require complexity and, up until now, comes either with a lot of
boilerplate code, implementations that are difficult to verify as correct, or a
lack of features.

`bee-ternary` seeks to become the canonical ternary crate in the Rust IOTA
ecosystem by providing an API that is efficient, ergonomic, and featureful all
at once.

`bee-ternary` will allow IOTA developers to more easily write code that works
with ternary and allow the simplification of existing codebases through use of
the API.

Broadly, there are 3 main benefits that `bee-ternary` aims to provide
over use-specific implementations of ternary manipulation code.

- **Ergonomics**: The API should be trivial to use correctly and should
naturally guide developers towards correct and efficient solutions.

- **Features**: The API should provide a fundamental set of core features that
can be used together to cover most developer requirements. Examples of such
features include multiple encoding schemes, binary/ternary conversion,
(de)serialization, and tryte string de/encoding.

- **Performance**: The API should allow 'acceptably' efficient manipulation of
ternary data. This is difficult to ruggedly define, and implementation details
may change later, but broadly this RFC intends to introduce an API that does not
inherently inhibit high performance through poor design choices.

# Detailed design

`bee-ternary` is designed to be extensible and is built on top of a handful of
fundamental abstraction types. Below are listed the features of the API and a
summary of the core API features.

## Features

- Efficient manipulation of ternary buffers (trits and trytes).
- Multiple encoding schemes.
- Extensible design that allows it to sit on top of existing data structures,
avoiding unnecessary allocation and copying.
- An array of utility functions to allow for easy manipulation of ternary data.
- Zero-cost conversion between trit and tryte formats (i.e: no slower than
the equivalent code would be if hand-written).

## Key Types & Traits

The crate supports both balanced and unbalanced trit representations, supported
through two types, `Btrit` and `Utrit`. `Btrit` is the more common
representation and so is the implicit default for most operations in the crate.

### `Btrit` and `Utrit`

```rust
enum Btrit { NegOne, Zero, PlusOne }
enum Utrit { Zero, One, Two }
```

### `Trits`

```rust
struct Trits<T: RawEncoding> { ... }
```

- Generic across different trit encoding schemes (see below).
- Analogous to `str` or `[T]`.
- Unsized type, represents a buffer of trits of a specified encoding.
- Most commonly used from behind a reference (as with `&str` and `&[T]`),
representing a 'trit slice'.
- Can be created from a variety of types such as`TritBuf` and `[i8]` with
minimal overhead.
- Sub-slices can be created from trit slices at a per-trit level.

### `TritBuf`

```rust
struct TritBuf<T: RawEncodingBuf> { ... }
```

- Generic across different trit encoding schemes (see below).
- Analogous to `String` or `Vec<T>`.
- Most common way to manipulate trits.
- Allows pushing, popping, and collecting trits.
- Implements `Deref<Target=Trits>` (in the same way that `Vec<T>` implements
`Deref<[T]>`).

### `RawEncoding`

```rust
trait RawEncoding { ... }
```

- Represents a raw trit buffer of some encoding.
- Common interface implemented by all trit encodings (`T1B1`, `T3B1`, `T5B1`,
etc.).
- Largely an implementation detail, not something you need to care about unless
implementing your own encodings.
- Minimal implementation requirements: safety and most utility functionality is
provided by `Trits` instead.

### `RawEncodingBuf`

```rust
trait RawEncodingBuf { ... }
```

- Buffer counterpart of `RawEncoding`, always associated with a specific
`RawEncoding`.
- Distinct from `RawEncoding` to permit different buffer-like data structures
in the future (linked lists, stack-allocated arrays, etc.).

### `T1B1`/`T2B1`/`T3B1`/`T4B1`/`T5B1`

```rust
struct TXB1 { ... }
```

- Types that implement `RawEncoding`.
- Allow different encodings in `Trits` and `TritBuf` types.

### `T1B1Buf`/`T2B1Buf`/`T3B1Buf`/`T4B1Buf`/`T5B1Buf`

```rust
struct TXB1Buf { ... }
```

- Types that implement `RawEncodingBuf`.
- Allow different encodings in `TritBuf` types.
- Each type is associated with a `RawEncoding` type.

### `Tryte`

```rust
enum Tryte {
    N = -13,
    O = -12,
    P = -11,
    Q = -10,
    R = -9,
    S = -8,
    T = -7,
    U = -6,
    V = -5,
    W = -4,
    X = -3,
    Y = -2,
    Z = -1,
    Nine = 0,
    A = 1,
    B = 2,
    C = 3,
    D = 4,
    E = 5,
    F = 6,
    G = 7,
    H = 8,
    I = 9,
    J = 10,
    K = 11,
    L = 12,
    M = 13,
}
```

- Type that represents a ternary tryte.
- Has the same representation as 3 `T3B1` byte-aligned trits.

### `TryteBuf`

```rust
struct TryteBuf { ... }
```

- A growable linear buffer of `Tryte`s.
- Roughly analogous to `Vec`.
- Has utility methods for converting to/from tryte strings.

## API

The API makes up the body of this RFC. Due to its considerable length, this RFC
simply refers to the documentation in question. You can find those docs in the
`bee` repository.

## Encodings

`bee-ternary` supports many different trit encodings. Notable encodings are
explained in the crate documentation.

## Common Patterns

When using the API, the most common types interacted with are `Trits` and
`TritBuf`. These types are designed to play well with the rest of the Rust
ecosystem. Here follows some examples of common patterns that you may wish to
make use of.

### Turning some `i8`s into a trit buffer

```rust
[-1, 0, 1, 1, -1, 0]
	.iter()
	.map(|x| Btrit::try_from(*x).unwrap())
	.collect::<TritBuf>()
```

Alternatively, for the `T1B1` encoding only, you may directly reinterpret the
`i8` slice.

```rust
let xs = [-1, 0, 1, 1, -1, 0];

Trits::try_from_raw(&xs, xs.len())
	.unwrap()
```

If you are *certain* that the slice contains *only* valid trits then it is
possible to unsafely reinterpret with amortised O(1) cost (i.e: it's basically
free). However, scenarios in which this is necessary and sound are exceedingly
uncommon. If you find yourself doing this, ask yourself first whether it is
necessary.

```rust
let xs = [-1, 0, 1, 1, -1, 0];

unsafe { Trits::from_raw_unchecked(&xs, xs.len()) }
```

### Turning a trit slice into a tryte string

```rust
trits
	.iter_trytes()
	.map(|trit| char::from(trit))
	.collect::<String>()
```

This becomes even more efficient easier with a `T3B1` trit slice since it has
the same underlying representation as a tryte slice.

```rust
trits
	.as_trytes()
	.iter()
	.map(|trit| char::from(*trit))
	.collect::<String>()
```

### Turning a tryte string into a tryte buffer

```rust
tryte_str
	.chars()
	.map(Tryte::try_from)
	.collect::<Result<TryteBuf, _>>()
	.unwrap()
```

Since this is a common operation, there exists a shorthand.

```rust
TryteBuf::try_from_str(tryte_str).unwrap()
```

### Turning a trit slice into a trit buffer

```rust
trits.to_buf()
```

### Turning a trit slice into a trit buffer of a different encoding

```rust
trits.encode::<T5B1>()
```

### Overwriting a sub-slice of a trit with copies of a trit

```rust
trits[start..end].fill(Btrit::Zero)
```

### Copying trits from a source slice to a destination slice

```rust
tgt.copy_from(&src[start..end])
```

# Drawbacks

This RFC does not have any particular drawbacks over an alternative approach.
Ternary is currently essential to the function of much of the IOTA ecosystem
and Bee requires a ternary API of some sort in order to effectively operate
within it.

# Rationale and alternatives

No suitable alternatives exist to this RFC. Rust does not have a mature ternary
manipulation crate immediately available that suits our needs.

# Unresolved questions

The API has now been in use in Bee for several months. Questions about
additional API features exist, but the current API seems to have proved its
suitability.
