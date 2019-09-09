+ Feature Name: `trinary-trytes`
+ Start Date: 2019-09-09
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/4)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/50)

# Summary

Define type `Trytes` for higher level usage. `Trytes` is going to be similar to
`String` which is a vector of `u8` under the structure. It provides almost same
methods but different signatures compares to `String`, because we are going to
make sure every inputs to `Trytes` are valid **tryte alphabete**. **Tryte alphabete**
is a table to make trytes easy to read. Since it serves this purpose and would
interact with underlying ternary operations in most cases, `Trytes` will behave
like string literal and each charaters are actually UTF-8 encoded character. So
`9` has value of 57 and `A` to `Z` have values of `65` to `90` in its vector.

# Motivation

The reason to implement trytes only at the moment is not only to be loosly
coupled from **trits**, but also layout what a ternary type could looks like. We
will eventually have various encoding techniques. It may become a mess if we
implement all in one structure, or seperate files but without a well-defined layout.
For the reason to use vector of `u8` is because it's sufficent in most cases,
implementations of `trit` encoding would need `bytes` from tokio or maybe
`bitflags` instead. I would like to make it be able to represent as string
literals and slice of values at the same time. As for choice of tryte alphabete
being UTF-8 encoded is because we expect most users would like to conver strings
and characters to `Trytes`. It makes more scene to stay same encoding as others
in Rust. Maybe we will need to convert byte literals in the end, but still this
can be done afterwards.

# Detailed design

## Trytes

Type `Trytes` is a vector of valus `Vec<u8>` and moreover a smart pointer.
The structure simply looks like this:

```rust
pub type Tryte = u8;

pub struct Trytes {
    vec: Vec<Tryte>,
}
```

Note that `vec` is not public, we provides plenty of methods and even implment
traits needed in most scenarios. Many of them are similar to `String`'s but have
different signatures mainly for checking if inputs are actaully valid. So
essentailly, these would be literally same methods but with `Result` returned.
The methods and traits are going to br implemented are roughly list below:

```rust
impl Trytes {
    pub fn new() -> Trytes {
        Trytes { vec: Vec::new() }
    }

    pub fn with_capacity(capacity: usize) -> Trytes {
        Trytes {
            vec: Vec::with_capacity(capacity),
        }
    }

    pub fn from_utf8(vec: Vec<u8>) -> Result<Trytes> {
        Self::all_tryte_alphabete(vec.iter())?;
        Ok(Trytes { vec })
    }

    pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> Trytes {
        Trytes { vec: bytes }
    }

    pub unsafe fn from_raw_parts(buf: *mut u8, length: usize, capacity: usize) -> Trytes {
    }

    pub fn into_bytes(self) -> Vec<u8> {
        self.vec
    }

    pub fn as_bytes(&self) -> &[u8] {
        &self.vec
    }

    pub fn as_str(&self) -> &str {
        str::from_utf8(&self.vec).unwrap()
    }

    pub fn as_mut_str(&mut self) -> &mut str {
        str::from_utf8_mut(&mut self.vec).unwrap()
    }

    pub fn push_str(&mut self, string: &str) -> Result<()> {
    }

    pub fn capacity(&self) -> usize {
        self.vec.capacity()
    }

    pub fn reserve(&mut self, additional: usize) {
        self.vec.reserve(additional)
    }

    pub fn reserve_exact(&mut self, additional: usize) {
        self.vec.reserve_exact(additional)
    }

    pub fn shrink_to_fit(&mut self) {
        self.vec.shrink_to_fit()
    }

    pub fn push(&mut self, ch: char) -> Result<()> {
    }

    pub fn truncate(&mut self, new_len: usize) {
    }

    pub fn pop(&mut self) -> Option<char> {
    }

    pub fn remove(&mut self, idx: usize) -> char {
    }

    pub fn insert(&mut self, idx: usize, ch: char) -> Result<()> {
    }

    pub fn insert_str(&mut self, idx: usize, string: &str) -> Result<()> {
    }

    pub unsafe fn as_mut_vec(&mut self) -> &mut Vec<u8> {
        &mut self.vec
    }

    pub fn len(&self) -> usize {
        self.vec.len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    pub fn split_off(&mut self, at: usize) -> Trytes {
    }

    pub fn clear(&mut self) {
    }

    unsafe fn insert_bytes(&mut self, idx: usize, bytes: &[u8]) {
    }

    fn all_tryte_alphabete<I>(vals: I) -> Result<()>
    where
        I: Iterator<Item = u8>,
    {
    }

    fn is_tryte_alphabete(t: u8) -> bool {
    }
}
```


# Drawbacks

- Methods of `Trytes` is a little bit overkill since it only serves as high level readness.
- Lack of convertnions with other traits and smart pointer like `FromIterator`, `Cow`, `Box`

# Rationale and alternatives

There are other ways to do this like a type alias for example, but we couldn't
restrict methods or traits we wish to have in this way. Some methods may even
contain results we don't expect. Re-implement a smart pointer can make sure the
scope we want. Another alternative is contruct with trits but this will become
tightly coupled and I'm fear it will be hard to refactor afterwards. 

# Unresolved questions

- How is `Tryte` going to convert with any trit encoded types?
- Would `Trit`-related methods implement in `Trytes` structure?
- What will `Trit` be implemented?
