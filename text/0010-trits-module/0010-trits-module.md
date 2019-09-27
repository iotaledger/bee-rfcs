+ Feature name: `trits-module`
+ Start date: 2019-09-20
+ RFC PR: [iotaledger/bee-rfcs#10](https://github.com/iotaledger/bee-rfcs/pull/10)
+ Bee issue: [iotaledger/bee#45](https://github.com/iotaledger/bee/issues/45)

# Summary

This RFC proposes a `trits` interface to standardise the usage of different underlying encodings and their conversions from one to another.

# Motivation

A `trit` is a digit in base `3` and is usually represented, in balanced ternary, by either `-1`, `0` or `1`.

Even though the most convenient way to encode trits is to map one balanced `trit` to one `signed byte`, this is obviously not the most memory efficient way nor the most readable way to do it.

In the IOTA ecosystem, different trits encodings are used in different situations like:

- hashing is done on raw trits because IOTA uses ternary hash functions like `Curl` or `Troika`;
- sending trits on the network is done by encoding `5` trits to a single byte for throughput efficiency;
- for UX purposes, we usually encode `3` trits to a single byte because of the nice readable mapping it offers (`9ABCDEFGHIJKLMNOPQRSTUVWXYZ`);

At the time this RFC is written, the following encodings are considered. This list is not exhaustive, some may be dropped, some may be added.

| Name    | Encoding              | Usage                           | RFC |
| ------- | --------------------- | ------------------------------- | --- |
| `t1b1`  | 1 trit per byte       | Hashing, tests                  |     |
| `t3b1`  | 3 trits per byte      | API, UX, tests                  |     |
| `t27b8` | 27 trits per 8 bytes  | Troika optimization             |     |
| `t9b2`  | 9 trits per 2 bytes   | Bee internal encoding           |     |
| `t5b1`  | 5 trits per byte      | Network communication, storage  |     |
| `ptrit` | Hardware dependant    | PCurl batch hashing             |     |

Because there are many different trits encodings and switching from one to another is often needed, having a trits interface to have a uniform way to interact with trits, whatever the underlying encoding, could be useful.

This interface should offer methods like allocations, conversions and manipulations.

# Detailed design

Any encoding should be convertible to any encoding. If `N` is the number of encodings available, then the number of possible conversions is `N^2` which could grow quickly. While some of these conversions are heavily used (e.g. `t5b1` to/from `t9b2`) and need to be as efficient as possible, some other are rarely or never expected to be used (e.g. `t5b1` to/from `t3b1`).

To allow implementing all conversions without having to focus too much on unnecessary work, we propose the following requirements:
- a conversion from any encoding to the same encoding is done by copy;
- a conversion from `t1b1` to any other encoding is efficiently implemented by `From` trait;
- a conversion from any encoding to `t1b1` is efficiently implemented by `From` trait;
- a conversion, proven to be heavily used, from any encoding to any other encoding is efficiently implemented by `From` trait with specialization support;
- any other conversion falls back to a trivial conversion with an intermediary `t1b1` step;

Note that inner structure should not provide any method to public. It should be
implemented by methods of `Trits<T>` to that specific type. This interface can
dereference to inner structure and provide more functionality other than general
methods.

```rust
pub trait Encoding {
    fn with_capacity(n: usize) -> Self;
    fn len(&self) -> usize;
    ...
}

pub struct T1B1 {
    raw: ...,
}

impl Encoding for T1B1 {
    ...
}

pub struct Trits<T: Encoding>{
    inner: T,
}

impl<T: Encoding> Trits<T> {
    pub fn new() -> Self {
        Self { inner: T::with_capacity(0) }
    }

    pub fn with_capacity(n: usize) -> Self {
        Self { inner: T::with_capacity(n) }
    }

    pub fn len(&self) -> usize {
        self.inner.len()
    }

    ...
}

impl<T: Encoding> Deref for Trits<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.inner
    }
}

impl From<Trits<T3B1>> for Trits<T1B1> {
    ...
}
```

# Drawbacks

We define a general interface with different encoded types within to make
consistency of public usage. But there are some method may needed for one
encoded type but not others. Take hashing for example, types like `ptrits` would
need some ways to do permutations and transformations, while types like 5T1B or
9T2B are not required because they are mainly used for transfer.

We can implement more methods but it still depends on what trait `Encoding` can
provide. Or we give more freedom to have method like `into_inner` to manipulate
the inner structure directly, but this might contradict to the purpose of this
interface.

# Rationale and alternatives

- Just define type alias and some convert functions. This give way more freedom to the users.
- Each encoded trit is a type of its own instead of interface like above.

# Unresolved questions

- Do we want to type usual sizes for more safety ?
- In case of missing conversion, how are fallbacks to slower implementation handled ?
- Should we do all conversion once and for all, or just provide minimum ways to do it first ?
