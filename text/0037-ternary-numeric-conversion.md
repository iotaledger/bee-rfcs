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
// Binary to ternary
impl<I: AsPrimitive<i64>, T: RawEncodingBuf> From<I> for TritBuf<T> {}

// Ternary to binary (these cannot be implemented as above due to Rust's orphan rules)
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i64 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i32 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i16 {}
impl<'a, T: RawEncoding<Trit = Btrit> + ?Sized> TryFrom<&'a Trits<T>> for i8 {}
```

The aforementioned trait implementations allow conversion to and from a variety
of numeric binary types for ternary types. The `AsPrimitive` trait comes from
the [`num_traits`](https://docs.rs/num-traits/0.2.12/num_traits/cast/trait.AsPrimitive.html)
crate, common throughout the Rust ecosystem.

# Drawbacks

It doesn't allow *arbitrary* binary conversion, only for fixed-size integers.

# Rationale and alternatives

No alternatives exist.

# Unresolved questions

None so far.
