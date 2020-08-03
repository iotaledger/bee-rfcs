+ Feature name: `trits-t1b1`
+ Start date: 2019-09-25
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes an encoding that stores sequences of balanced trits (`-1`, `0`, `1`) in a `Vec` of signed bytes `i8`. It is one of the encodings used in the IOTA ecosystem, which are mentioned in RFC #10. It is the most memory inefficient encoding possible to store or transfer trits, because only 3 out of 256 possible states in a byte are actually used. Nevertheless this sometimes referred to as "raw trits" encoding is the representation expected as input by the trinary `Curl` hash function, and as well the representation of its output. 

# Motivation

This encoding of sequences of balanced trits is necessary for the `Curl` hash function and therefore still necessary to allow for the Bee framework to interoperate with the current IOTA mainnet.

# Detailed design

The design of this encoding consists of two steps:
* create a type-safe trit representation `Trit` on top of `i8`
* create the `t1b1` encoding on top of `Trit`
  
For step 1 this proposal employs the `newtype` pattern. It allows to add semantics to the wrapped datatype, which in this case is a signed byte `i8`. 

Note: For the sake of simplicity and increased code readability access modifiers are omitted. Just be aware that the wrapped `i8` can be accessed only from inside the crate which is trusted to do things right.

```Rust
struct Trit(i8);

impl Trit {
    fn value(&self) -> i8 {
        self.0
    }
    fn index(&self) -> usize {
        match self.0 {
            0 | 1 => self.0 as usize,
            -1 => 2 as usize,
            _ => unreachable!(),
        }
    }
}
```
As you can see this implementation provides two accessor methods `value` and `index` which are self-explanatory. The `index` method is currently only used internally to retreive the trit `char` representation from a lookup table.

To make working with `Trit`s convenient this proposal suggests implementing the following traits, which allow to safely create single `Trit`s from their `char` or their `i8` representation as well as display them for easier debugging or logging:

```Rust
impl TryInto<Trit> for char {...}
impl TryInto<Trit> for i8 {...}
impl Display for Trit {...}
```

For step 2 this proposal suggests modeling `T1B1` as a compound type of `Vec` and `Trit` using the newtype pattern again rather than a type alias. This allows to have full control over the API of this type, and is more secure.

```Rust
struct T1B1(Vec<Trit>);
```

This proposal suggests two basic operations for now for adding and removing `Trit`s.
```Rust
impl T1B1 {
    fn push(&mut self, trit: Trit) {
        self.0.push(trit);
    }
    fn pop(&mut self) {
        self.0.pop();
    }
}
```

Furthermore most of the functionality of this type is realized in trait implementations:
```Rust
// implements all functionality required for a (trit) encoding
impl Encoding for T1B1 {...}

// conversion logic to move to 'T3B1' encoding
impl ToT3B1 for T1B1 {...}

// conversion logic to move to 'T5B1' encoding
impl ToT5B1 for T1B1 {...}

// conversion logic to move to 'T9B2' encoding
impl ToT9B2 for T1B1 {...}

// conveniently converts string slices into 'T1B1'
impl<'a> TryInto<T1B1> for &'a str {...}

// displays 'T1B1' as a trit sequence, e.g. 1--100-11
impl Display for T1B1 {...}
```
# Drawbacks

This type is only necessary for compatibility with the current IOTA mainnet. It is inefficient and should ideally not be used for anything other than hashing and debugging. 

# Rationale and alternatives

There are no alternatives as long as standard `Curl` hashing requires and produces data in this encoding.

# Unresolved questions

It is a well-known and long-time used encoding in the IOTA ecosystem, so there are no open questions around it known to the author.