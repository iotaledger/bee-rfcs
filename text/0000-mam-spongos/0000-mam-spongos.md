+ Feature name: `mam-spongos`
+ Start date: 2019-10-24
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: 

# Summary

This PR proposes Spongos API. A prototype implementation is available
[here](https://github.com/semenov-vladyslav/iota_mam/blob/master/spongos.rs).

# Motivation

`Sponge` trait is considered in [PR#21](https://github.com/iotaledger/bee-rfcs/pull/21). It has minimalist API and only
supports `reset`, `absorb` and `squeeze`. Obviously it doesn't cover the other API functions provided by Spongos:
`init`, `encr`, `decr`, `commit`, `fork`, `join`. Also, different constructions will have different features, such as
rate, capacity, security level and so on. Also, `Sponge` deals with trits which may be not very efficient. So it
doesn't make much sense to (ab)use `Sponge` trait.

From the other hand it might make sense to fix a sponge construction (ie. make it `struct` with a fixed `impl` rather
than a `trait`), but parametrize it by pseudorandom permutation. The most efficient design would look like this:

```rust
trait PRP<TW> {
    fn outer(&mut self) -> TritMutSlice<TW>;
    fn transform(&mut self);
}
```

where `outer` returns a mutable slice to the outer state allocated within PRP itself. But not all PRPs have state
layout compatible with trit encoding `TW`. And Troika is an example of such PRP, as the fast Troika implementation (aka
`ftroika`) uses a rather peculiar scheme for packing 27 trits of Troika slice into a pair of 32-bit words. But the
consecutive trits of state (or rather outer state) are not mapped to consecutive trits of Troika slice and have to be
loaded trit by trit. Here we have two options: make t27b8 default trit encoding and change the way trits are loaded
into the state (and thus lose compatibility with other implementations using Troika in a standard way), or introduce a
less efficient `PRP` trait.

# Detailed design

This PR proposes the following `PRP` trait and its implementation for `Troika`:

```rust
trait PRP<TW> {
    fn transform(&mut self, outer: &mut TritMutSlice<TW>);
}
```

where buffer for the `state` slice is allocated within `Spongos` object.

Also, this PR proposes the following `Spongos` implementation.

```rust
pub struct Spongos<TW> {
    /// Spongos transform is Troika.
    s: Troika,
    /// Current position in the outer state.
    pos: usize,
    /// Outer state is stored externally due to Troika implementation.
    /// It is injected into Troika state before transform and extracted after.
    outer: Trits<TW>,
}
impl<TW> Spongos<TW> where TW: TritWord + Copy {
    /// Create a Spongos object, initialize state with zero trits.
    pub fn init() -> Self;
    /// Absorb a trit slice into Spongos object.
    pub fn absorb(&mut self, mut x: TritConstSlice<TW>);
    /// Squeeze a trit slice from Spongos object.
    pub fn squeeze(&mut self, mut y: TritMutSlice<TW>);
    /// Encrypt a trit slice with Spongos object.
    /// Input and output slices must either be the same (point to the same memory/trit location) or be non-overlapping.
    pub fn encr(&mut self, mut x: TritConstSlice<TW>, mut y: TritMutSlice<TW>);
    /// Decrypt a trit slice with Spongos object.
    /// Input and output slices must either be the same (point to the same memory/trit location) or be non-overlapping.
    pub fn decr(&mut self, mut x: TritConstSlice<TW>, mut y: TritMutSlice<TW>);
    /// Fork Spongos object into another.
    /// Essentially this just creates a copy of self.
    pub fn fork(&self, fork: &mut Self);
    /// Force transform even if for incomplete (but non-empty!) outer state.
    /// Commit with empty outer state has no effect.
    pub fn commit(&mut self);
    /// Join two Spongos objects.
    /// Joiner -- self -- object absorbs data squeezed from joinee.
    pub fn join(&mut self, joinee: &mut Self);
}
```

This way it'll be possible to create local `Spongos` instances. For example, `hash_data` can be implemented as a free
function as follows:

```rust
/// Hash data with Spongos.
pub fn hash_data<TW>(data: TritConstSlice<TW>, hash_value: TritMutSlice<TW>) where TW: TritWord + Copy {
    let mut s = Spongos::<TW>::init();
    s.absorb(data);
    s.commit();
    s.squeeze(hash_value);
}
```

# Drawbacks

# Rationale and alternatives

- Light-weight trit slices are a current design choice of Trits module; `TW` type argument is inherent to it.
- Spongos implementation can be turned into a trait. Although there seem to be no reason for it.
- Rate, capacity and security level are fixed.
