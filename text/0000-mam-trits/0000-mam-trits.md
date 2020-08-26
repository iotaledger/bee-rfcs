+ Feature name: `mam-trits`
+ Start date: 2019-10-22
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: 

# Summary

How trits can be accessed easily and stored compactly? How can trits can be manipulated efficiently? How can trits API
be declared and implemented in rust? This PR answers these questions. There are three parts to this PR: Trit Encoding,
Storage and Accessors, and Utility Functions. MAM has distinguished properties that define the proposed design choice.

The prototype implementation is available [here](https://github.com/semenov-vladyslav/iota_mam/blob/master/trits.rs).

# Trit Encoding

## Motivation

Trit encodings are considered in [PR#10](https://github.com/iotaledger/bee-rfcs/pull/10) and a number of RFCs
describing the actual encodings: [PR#8](https://github.com/iotaledger/bee-rfcs/pull/8),
[PR#13](https://github.com/iotaledger/bee-rfcs/pull/13), [PR#14](https://github.com/iotaledger/bee-rfcs/pull/14),
[PR#16](https://github.com/iotaledger/bee-rfcs/pull/16). The trait `Encoding` is supposed to behave as a container for
trits where the actual representation (encoding) is implementation-specific. This however is not very much suitable for
purposes of MAM where such operations as conversions to and from integer types (trints) and `copy/swap_add/sub` must
be implemented efficiently. Also, not all Storage and Accessors designs allow for *any* external container.

## Detailed design

The idea is to declare a trait `TritWord` which should be thought of as ternary processor register. Logically, it is
considered as an atomic vector of (a fixed small number of) trits. Taking into account considerations of Storage and
Accessors section the following `TritWord` design is proposed:

```rust
pub trait TritWord {
    /// The number of trits per word.
    const SIZE: usize;

    /// All-zero trits word.
    fn zero() -> Self;
    /// Copy `n` trits from `(dx,x)` slice into `(dy,y)`.
    fn unsafe_copy(n: usize, dx: usize, x: *const Self, dy: usize, y: *mut Self);
    /// Set `n` trits in `(d,p)` slice to zero.
    fn unsafe_set_zero(n: usize, d: usize, p: *mut Self);
    /// Compare `n` trits from `(dx,x)` slice into `(dy,y)`.
    fn unsafe_eq(n: usize, dx: usize, x: *const Self, dy: usize, y: *const Self) -> bool;

    // Integer conversion utils

    fn put1(d: usize, p: *mut Self, t: Trint1);
    fn get1(d: usize, p: *const Self) -> Trint1;
    fn put3(d: usize, p: *mut Self, t: Trint3);
    fn get3(d: usize, p: *const Self) -> Trint3;
    ...

    // Spongos-related utils

    /// y:=x+s, s:=x, x:=y
    fn unsafe_swap_add(n: usize, dx: usize, x: *mut Self, ds: usize, s: *mut Self);
    /// x:=y-s, s:=x, y:=x
    fn unsafe_swap_sub(n: usize, dy: usize, y: *mut Self, ds: usize, s: *mut Self);
    /// y:=x+s, s:=x
    fn unsafe_copy_add(n: usize, dx: usize, x: *const Self, ds: usize, s: *mut Self, dy: usize, y: *mut Self);
    /// t:=y-s, s:=t, x:=t
    fn unsafe_copy_sub(n: usize, dy: usize, y: *const Self, ds: usize, s: *mut Self, dx: usize, x: *mut Self);
}
```

Contrary to the PRs mentioned above, `TritWord` does not own a buffer of trits, rather it operates on pointers. As a
`TritWord` contains `SIZE` trits, we need to specify the starting trit in a word. Thus the API functions take a pair
`(d,p)` encoding a slice of trits as input where `d` is the current trit offset, `p` is the raw pointer to the first
word in a slice. Such trait design allows for default implementation based only on `get1` and `put1` which can be used
for testing purposes.

## Drawbacks

- Unsafe, uses raw pointers without checked life-time.

## Rationale and alternatives

- Designed to be compatible with the Light-weight slice storage design.
- Can be implemented for any encoding.
- Simple, test-friendly design.
- Instead of raw pointers it's possible to use smart pointers at the cost of runtime checks.

# Storage and accessors

## Motivation

In MAM trits are used in iostream-like, usually sequential manner: during message wrapping the output trit stream is
sequentially constructed, and during message unwrapping the input trit stream is sequentially consumed. There are three
possible designs:

1. IOTritStream traits:

```rust
pub trait ITritStream {
    fn get(&mut self) -> Trit;
    fn avail(&self) -> usize;
    fn is_eof(&self) -> bool;
}
pub trait OTritStream {
    fn put(&mut self, t: Trit);
    fn flush(&self);
}
```

Pros:
- This is the most general design.
- Can support an actual IO.
Cons:
- This is overkill. In most real-world MAM scenarios a simpler API can be sufficient.
- The API will likely be much more complex in order to support different IO.
- Support for asynchronous IO may require some syntactic changes (`async fn`).

2. Dynamic trit buffer:

```rust
pub struct ITrits<'a, TE> {
    buf: &'a Vec<TE>,
    idx: usize,
}
pub struct OTrits<'a, TE> {
    buf: &'a mut Vec<TE>,
    idx: usize,
}
impl<'a, TE> ITrits<'a, TE> {
    pub fn get(&mut self) -> Trit;
    pub fn avail(&self) -> usize;
}
impl<'a, TE> OTrits<'a, TE> {
    pub fn put(&mut self, t: Trit);
}
```

Instead of reference `buf` can be passed by a smart pointer `Box`, `Rc`, `Arc`.
Pros:
- Simpler implementation compared to IOTritStream.
- Type-safety and runtime guarantees.
- `OTrits` can increase size of the `buf`.
Cons:
- Need to pass lifetime generic parameter with references.
- Run-time overhead of smart-pointers.

## Detailed design

3. Light-weight unsafe trit slice:

```rust
pub struct TritConstSlice<TW> {
    n: usize,
    d: usize,
    p: *TW,
}
pub struct TritMutSlice<TW> {
    n: usize,
    d: usize,
    p: *mut TW,
}
```

`n` is the total number of trits available in a slice, `d` is the current displacement/offset, `p` is the base pointer.
Trit words are actually stored in an accompanying type:

```rust
pub struct Trits<TW> {
    size: usize,
    buf: std::vec::Vec<TW>,
}
```

where `size` is the logical number of trits stored in the object, `buf` is the actual storage for `TritWord`s.

```rust
impl<TW> TritConstSlice<TW> {
    pub fn from_trits(t: &Trits<TW>) -> Self;
    pub fn get(self) -> Trit;
    pub fn take(self, n: usize) -> Self;
    pub fn drop(self, n: usize) -> Self;
}
impl<TW> TritMutSlice<TW> {
    pub fn from_mut_trits(t: &mut Trits<TW>) -> Self;
    pub fn put(self, t: Trit);
    pub fn take(self, n: usize) -> Self;
    pub fn drop(self, n: usize) -> Self;
}
```

The objects are very light-weight and can be passed by value into the interface functions. Thus slices themselves are
not mutable. Moving/changing a slice is done with the help of functions such as `take` and `drop` that return a new
(immutable) slice to a different range of trits. This idea is very much similar to Haskell's `ByteString`. The base
pointer is not usually modified (even in `take` and `drop`) and thus can be made into a smart pointer.

## Drawbacks

- Unsafe as it uses raw pointers.
- It takes some time and practice to use the API safely.
- Some functions are duplicated for const and mut slices.
- Memory for output slice must be allocated a-priory. Although this is not an issue with MAM as the size of all encoded
objects can be known before encoding.
- No life-time guarantees. Slice objects are designed to be short-lived and must not outlive the container.

## Rationale and alternatives

- Light-weight design.
- Immutable slice objects.
- Represents two ranges: used and available, which is convenient for Spongos implementation.
- Flexible subrange selection capabilities.

# Utility functions

## Motivation

Spongos and PB3 are the most trit utility demanding layers. Spongos adds and subtracts trits for encryption/decryption.
And PB3 needs convenient functions for message encoding/parsing. This functionality provides `TritWord` but over a trit
word/register, not a trit range. 

## Detailed design

The best place for such utility functions is as part of `TritConstSlice` and `TritMutSlice` implementation. Here's a
more specified API:

```rust
impl<TW> TritConstSlice<TW> where TW: TritWord + Copy {
    pub fn get1(self) -> Trint1;
    pub fn get3(self) -> Trint3;
    ...
    pub fn is_empty(self) -> bool;
    pub fn size(self) -> usize;
    pub fn take(self, n: usize) -> Self;
    pub fn drop(self, n: usize) -> Self;
    pub fn pickup(self, n: usize) -> Self;
    ...
    pub fn copy(self, to: TritMutSlice<TW>);
    pub fn copy_add(self, s: TritMutSlice<TW>, y: TritMutSlice<TW>);
    pub fn copy_sub(self, s: TritMutSlice<TW>, y: TritMutSlice<TW>);
    ...
}

impl<TW> TritMutSlice<TW> where TW: TritWord + Copy {
    pub fn put1(self, t: Trint1);
    pub fn put3(self, t: Trint3);
    pub fn set_zero(self);
    ...
    pub fn is_empty(self) -> bool;
    pub fn size(self) -> usize;
    pub fn take(self, n: usize) -> Self;
    pub fn drop(self, n: usize) -> Self;
    pub fn pickup(self, n: usize) -> Self;
    ...
    pub fn swap_add(self, s: Self);
    pub fn swap_sub(self, s: Self);
    ...
}
```

## Drawbacks

- Too verbose, need to constantly take care of the number trits processed previously.
- Need to pass `TW` type argument and associated contraint `TW: TritWord + Copy` in all modules using trits, including
Spongos and hence all the others.

## Rationale and alternatives

- The API can be refined to simplify it's usage. For example, functions returning `()` may return `Self` with processed
trits dropped.

## Unresolved questions

To rid of `TW` type argument we can turn trit slices impl into traits as `TW` is only used in the implementation. But
the implementations will still need to choose which encodings they support.

Another way to get rid of `TW` is to fix one global for MAM trit encoding. There are several candidates:
- t4b1, or 8 trits per 2 bytes, enables fast `add` and `sub` operations. LUTs can be used for integer conversions and
to/from other encodings.
- t9b2 is very compact and efficient for encoding/decoding *aligned* trytes. LUTs can be used for integer *unaligned*
conversions.
- t27b8 is used by Troika and with certain modifications can be used as a default trit encoding.
- Any encoding used internally for transactions/bundles. It requires no additional conversions when wrapping/unwrapping
a MAM message to/from a bundle, but utility functions may be not as efficient as with t4b1 or t9b2.
