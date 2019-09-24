+ Feature name: trits
+ Start date: 2019-09-20
+ RFC PR: [iotaledger/bee-rfcs#10](https://github.com/iotaledger/bee-rfcs/pull/10)
+ Bee issue: [iotaledger/bee#45](https://github.com/iotaledger/bee/issues/45)

# Summary

WIP

# Motivation

A `trit` is a digit in base `3` and is usually represented by either `-1`, `0` or `1`.

Even though the most convenient way to encode trits is to map one `trit` to one `byte`, this is obviously not the most memory efficient way nor the most readable way to do it.

In the IOTA ecosystem, different trits encodings are used in different situations like:

- hashing is done on raw trits because IOTA uses ternary hash functions like `Curl` or `Troika`;
- sending trits on the network is done by encoding `5` trits to a single byte for throughput efficiency;
- for UX purposes, we usually encode `3` trits to a single byte because of the nice readable mapping it offers (`9ABCDEFGHIJKLMNOPQRSTUVWXYZ`);

At the time this RFC is written, the following encodings are considered. This list is not exhaustive, some may be dropped, some may be added.

| Name      | Encoding              | Usage                 | RFC |
| --------- | --------------------- | --------------------- | - |
| `1t/1b`   | 1 trit per byte       | Hashing, tests        |   |
| `3t/1b`   | 3 trits per byte      | API, UX, tests        |   |
| `27t/8b`  | 27 trits per 8 bytes  | Troika optimization   |   |
| `9t/2b`   | 9 trits per 2 bytes   | Bee internal encoding |   |
| `5t/1b`   | 5 trits per byte      | Network communication |   |
| `ptrit`   | Hardware dependant    | PCurl batch hashing   |   |

Because there are many different trits encodings and switching from one to another is often needed, having a trits interface to have a uniform way to interact with trits, whatever the underlying encoding, could be useful.

This interface should offer methods like allocations, conversions and manipulations.

# Detailed design

Any encoding should be convertible to any encoding. If `N` is the number of encodings available, then the number of possible conversions is `N^2` which could grow quickly. While some of these conversions are heavily used (e.g. `5t/1b` to/from `9t/2b`) and need to be as efficient as possible, some other are rarely or never expected to be used (e.g. `5t/1b` to/from `3t/1b`).

To allow implementing all conversions without having to focus too much on unnecessary work, we propose the following requirements:
- a conversion from any encoding to the same encoding is done by copy;
- a conversion from `1t/1b` to any other encoding is efficiently implemented;
- a conversion from any encoding to `1t/1b` is efficiently implemented;
- a conversion, proven to be heavily used, from any encoding to any other encoding is efficiently implemented;
- any other conversion falls back to a trivial conversion with an intermediary `1t/1b` step;

```rust
pub trait Trinary {
    fn with_capacity(n: usize) -> Self;
    fn double_in_place(&mut self) -> bool;
    fn reserve(&mut self, used: usize, needed: usize);
    fn to_trits(ref: &[u8]) -> Self;
    ...
}

pub struct 1T1B {
    raw: ...,
}

impl Trinary for 1T1B {

}

pub struct Trits<T: Trinary>{
    inner: T,
}

impl<T: Trinary> Trits<T> {
    pub fn new() -> Self {
        ...
    }

    pub fn with_capacity(n: usize) -> Self {
        ...
    }

    pub fn push(&mut self, x: Trits) {
        ...
    }

    pub fn pop(&mut self) -> Option<Trits> {
        ...
    }
}

impl<T: Trinary> From<[u8]> for Trits<T> {
    fn from(s: [u8]) -> Self {
        ...
    }
}
```

# Drawbacks

WIP

# Rationale and alternatives

WIP

# Unresolved questions

WIP
