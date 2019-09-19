+ Feature Name: `trinary-trytes`
+ Start Date: 2019-09-09
+ RFC PR: [iotaledger/bee-rfcs#4](https://github.com/iotaledger/bee-rfcs/pull/4)
+ Bee issue: [iotaledger/bee#50](https://github.com/iotaledger/bee/issues/50)

# Summary

Define type `Trytes` for higher level usage. `Trytes` is going to be similar to
`String` which is a vector of `u8` under the structure but with the coverage of
`BytesMut` from tokio's bytes crate. Bytes from tokio has many zero-copy
features we can utilize, So we can initialize a `Trytes` from a slice without
heap allocation. `Trytes` provides almost same but restricted methods with
different signatures compares to `String`, because we are going to make sure
every inputs to `Trytes` are valid **tryte alphabete**. **Tryte alphabete** is a
table to make trytes easy to read. Since it serves this purpose and would
interact with underlying ternary operations in most cases, `Trytes` will behave
like string literal and each charaters are actually UTF-8 encoded character. So
`9` has value of 57 and `A` to `Z` have values of `65` to `90` in its vector.

# Motivation

The reason to implement trytes only at the moment is not only to be loosly
coupled from **trits**, but also layout what a ternary type could looks like. We
will eventually have various encoding techniques. It may become a mess if we
implement all in one structure, or seperate files but without a well-defined
layout. For the reason to be `BytesMut` is because we want to take advantage of
its zero-copy characteristic. We can also transfer into the `Buf` or `BufMut`
defined by `bytes` for the later usage of `trit` encoding. I also would like to
make it be able to represent as string literals and slice of values at the same
time. As for choice of tryte alphabete being UTF-8 encoded is because we expect
most users would like to conver strings and characters to `Trytes`. It makes
more scene to stay same encoding as others in Rust. Maybe we will need to
convert byte literals in the end, but still this can be done afterwards.

# Detailed design

## Trytes

Type `Trytes` is a type `BytesMut` and moreover a smart pointer.
The structure simply looks like this:

```rust
pub type Tryte = u8;

pub struct Trytes(BytesMut);
```

Note that `BytesMut` is not public, we provides plenty of methods and even implment
traits needed in most scenarios. Many of them are similar to `String`'s but have
different signatures mainly for checking if inputs are actaully valid. So
essentailly, these would be literally same methods but with `Result` returned.
The methods and traits are going to br implemented are roughly list below:

```rust
impl Trytes {
    pub fn new() -> Trytes {
        Trytes(BytesMut::new())
    }

    pub fn with_capacity(capacity: usize) -> Trytes {
        Trytes(BytesMut::with_capacity(capacity))
    }

    pub fn as_bytes(&self) -> &[u8] {
        &self.0.as_ref()
    }

    ...

    pub fn encode(&mut self, plain: &str) {
        for byte in plain.bytes() {
            let first = byte % 27;
            let second = (byte - first) / 27;
            self.0.put(TRYTE_ALPHABET[first as usize]);
            self.0.put(TRYTE_ALPHABET[second as usize]);
        }
    }

    pub fn decode(&self) -> String {
        let mut s = String::with_capacity(self.len() / 2);
        for slice in self.chunks(2) {
            let first: u8 = TRYTE_ALPHABET.iter().position(|&x| x == slice[0]).unwrap() as u8;
            let second: u8 = TRYTE_ALPHABET.iter().position(|&x| x == slice[1]).unwrap() as u8;
            let decimal = first + second * 27;

            s.push(decimal as char);
        }
        s
    }

    ...

    fn all_tryte_alphabete<I>(vals: I) -> Result<()>
    where
        I: Iterator<Item = u8>,
    {
    }

    fn is_tryte_alphabete(t: u8) -> bool {
    }
}

impl From<...> for Trytes {}
impl Index ...
...
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
