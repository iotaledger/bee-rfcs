+ Feature name: `ternary-numeric-conversion`
+ Start date: 2020-06-24
+ RFC PR: [iotaledger/bee-rfcs#37](https://github.com/iotaledger/bee-rfcs/pull/37)
+ Bee issue: [iotaledger/bee#93](https://github.com/iotaledger/bee/issues/93)

# Summary

Add a way to convert to and from numeric binary types for ternary types.

# Motivation

Conversion between binary and ternary is often useful and so a good solution to
this is required.

For example, this feature permits the following conversions.

```rust
let x = 42;
let x_trits = TritBuf::from(x);
let y = i64::try_from(x_trits.as_slice()).unwrap();
assert_eq!(x, y);
```

# Detailed design

This RFC introduces the following trait implementations:

```rust
// Signed binary to balanced ternary
impl<T: RawEncodingBuf> From<i64> for TritBuf<T> where T::Slice: RawEncoding<Trit = Btrit> {}
impl<T: RawEncodingBuf> From<i32> for TritBuf<T> where T::Slice: RawEncoding<Trit = Btrit> {}
impl<T: RawEncodingBuf> From<i16> for TritBuf<T> where T::Slice: RawEncoding<Trit = Btrit> {}
impl<T: RawEncodingBuf> From<i8> for TritBuf<T> where T::Slice: RawEncoding<Trit = Btrit> {}

// Balanced ternary to signed binary
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i64 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i32 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i16 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i8 {}

// Unsigned binary to unbalanced ternary
impl<T: RawEncodingBuf> From<u64> for TritBuf<T> where T::Slice: RawEncoding<Trit = Utrit> {}
impl<T: RawEncodingBuf> From<u32> for TritBuf<T> where T::Slice: RawEncoding<Trit = Utrit> {}
impl<T: RawEncodingBuf> From<u16> for TritBuf<T> where T::Slice: RawEncoding<Trit = Utrit> {}
impl<T: RawEncodingBuf> From<u8> for TritBuf<T> where T::Slice: RawEncoding<Trit = Utrit> {}

// Unbalanced ternary to unsigned binary
impl<'a, T: RawEncoding<Trit = Utrit> + ?Sized> TryFrom<&'a Trits<T>> for u64 {}
impl<'a, T: RawEncoding<Trit = Utrit> + ?Sized> TryFrom<&'a Trits<T>> for u32 {}
impl<'a, T: RawEncoding<Trit = Utrit> + ?Sized> TryFrom<&'a Trits<T>> for u16 {}
impl<'a, T: RawEncoding<Trit = Utrit> + ?Sized> TryFrom<&'a Trits<T>> for u8 {}
```

The aforementioned trait implementations allow conversion to and from a variety
of numeric binary types for ternary types.

In addition, there exist utility functions that allow implementing this
behaviour for arbitrary numeric values that implement relevant traits from
`num_traits`.

```rust
pub fn trits_to_int<
    I: Clone + num_traits::CheckedAdd + num_traits::CheckedSub + PartialOrd + num_traits::Num,
    T: RawEncoding + ?Sized,
>(trits: &Trits<T>) -> Result<I, Error> { ... }

pub fn signed_int_trits<I: Clone
    + num_trits::AsPrimitive<i8>
    + num_trits::FromPrimitive
    + num_traits::Signed>(x: I) -> impl Iterator<Item=Btrit> + Clone { ... }

pub fn unsigned_int_trits<I: Clone
    + num_trits::AsPrimitive<u8>
    + num_trits::FromPrimitive
    + num_traits::Num>(x: I) -> impl Iterator<Item=Utrit> + Clone { ... }
```

The `AsPrimitive` trait comes from the
[`num_traits`](https://docs.rs/num-traits/0.2.12/num_traits/cast/trait.AsPrimitive.html)
crate, common throughout the Rust ecosystem.

All of the aforementioned numeric conversion functions and traits operate
correctly for all possible values including numeric limits.

# Drawbacks

- No support for fractional ternary floating-point conversion (although this is
probably not useful for IOTA's use case).

- Rust's orphan rules do not allow implementing foreign traits for foreign types
so automatically implementing `TryFrom` for all numeric types is not possible.

- [Rust's type system](https://github.com/rust-lang/rust/issues/20400) cannot
yet reason about type equality through associated type bindings and so it is
not possible to implement `Btrit` trit slice to/from unsigned integers yet
(although it's possible to use the utility functions to achieve this).

# Rationale and alternatives

No competing alternatives exist at this time.

# Unresolved questions

None so far.
