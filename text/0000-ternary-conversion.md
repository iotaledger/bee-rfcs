+ Feature name: `ternary-conversion`
+ Start date: 2020-06-24
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#93](https://github.com/iotaledger/bee/issues/93)

# Summary

Add a way to convert to and from binary types for ternary types.

# Motivation

Conversion between binary and ternary is often useful and a good solution to this is required.

# Detailed design

This RFC introduces the following trait implementations:

```rust
// Binary to ternary
impl<I: AsPrimitive<i64>, T: RawEncodingBuf> From<I> for TritBuf<T> {}

// Ternary to binary (these cannot be implemented as above due to Rust's orphan rules)
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i64 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i32 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i16 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i8 {}
```

The aforementioned trait implementations allow conversion to and from a variety of binary types for ternary types.
The `AsPrimitive` trait comes from the `num_traits` crate, common throughout the Rust ecosystem.

# Drawbacks

It doesn't allow *arbitrary* binary conversion, only for fixed-size integers.

# Rationale and alternatives

The previous implementation used only `i64`. This implementation works for a wider range of types.

# Unresolved questions

None so far.
