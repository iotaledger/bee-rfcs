+ Feature name: `trits-t3b1`
+ Start date: 2019-09-27)
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes a byte encoding for trit sequences of arbitrary length, which sequentially maps groups of 3 trits (trytes) to corresponding bytes starting from the left. If the last group consists of less than 3 trits (incomplete tryte), then those missing trits are assumed to be 0 and then mapped accordingly. This encoding is 3 times more memory efficient than [`t1b1`](), but only 60% as efficient as [`t5b1`](). It is a human-readable encoding that can be used anywhere a user needs to interact with trinary data, like reading IOTA addresses or entering seeds to access an account.


# Motivation

We would like to have a trinary representation that is as compact as possible and still human-readable. We could use [`t1b1`](https://github.com/Alex6323/bee-rfcs/blob/rfc-t3b1/text/0000-trits-t3b1/0000-trits-t3b1.md) which can easily be mapped onto the characters `{'-', '0', '1'}` for display. But because it uses only so few characters that are available on any keyboard those sequences would be too long for any practical purposes. And we shouldn't force users to type in more characters than absolutely necessary.

As an example consider the following trit sequence: `-10--1--1`. Let's imagine that it represents a seed (a very bad one though) owned by some user. Rather than the whole trit sequence with the proposed `t3b1` encoding all the user would have to enter to access this account would be `BEE` (I am oversimplifying here on purpose).

### List of  use-cases:
* During development for debugging, testing, and logging
* In production for displaying and inputting trinary data in wallets such as addresses or seeds

### Job stories:
When I design a UI element to enter a seed, I want it to be as convenient as possible to use, so instead of raw trits one can pick from the frequently used characters `9,A-Z`. Whether the input is valid or not can be checked easily by using a regular expressions.

When I have to display the trinary representation of some arbitrary data, I want it to be represented only by characters that I am familiar with, yet it should be as short as possible, so I can convert that data into the `t3b1` encoding and display it in an alphabet-like format.

# Detailed design

Any trit sequence will be interpreted as a sequence of trit triplets (trytes) like so: `-10|--1|--1` and then mapped to a corresponding integer from the set `{-13,...,0,...,13}` which can be stored in a signed byte `i8`. If the length of the sequence is not a multiple of 3, then the remaining trits are assumed to be `0`. The following table shows the mapping between all possible `27` trit triplets, their corresponding decimal value, and their corresponding character representation:

| trit triplet (tryte) | decimal value | character symbol |
| -------------------- | ------------- | ---------------- |
| 000                  | 0             | '9'              |
| 100                  | 1             | 'A'              |
| -10                  | 2             | 'B'              |
| 010                  | 3             | 'C'              |
| 110                  | 4             | 'D'              |
| --1                  | 5             | 'E'              |
| 0-1                  | 6             | 'F'              |
| 1-1                  | 7             | 'G'              |
| -01                  | 8             | 'H'              |
| 001                  | 9             | 'I'              |
| 101                  | 10            | 'J'              |
| -11                  | 11            | 'K'              |
| 011                  | 12            | 'L'              |
| 111                  | 13            | 'M'              |
| ---                  | -13           | 'N'              |
| 0--                  | -12           | 'O'              |
| 1--                  | -11           | 'P'              |
| -0-                  | -10           | 'Q'              |
| 00-                  | -9            | 'R'              |
| 10-                  | -8            | 'S'              |
| -1-                  | -7            | 'T'              |
| 01-                  | -6            | 'U'              |
| 11-                  | -5            | 'V'              |
| --0                  | -4            | 'W'              |
| 0-0                  | -3            | 'X'              |
| 1-0                  | -2            | 'Y'              |
| -00                  | -1            | 'Z'              |

That table can optionally be implemented with up to 3 lookup table (LUT).

```Rust
const TRYTEINDEX_TO_TRYTE: [[i8; 3]; 27] = [[0,0,0], [1,0,0] ...];
const TRYTEINDEX_TO_VALUE: [i8; 27] = [0, 1, ...];
const TRYTEINDEX_TO_CHAR: [char; 27] = ['9', 'A', ...];
```

Because an `i8` can represent any number in the set `{-128,...,0,...127}` not every such number is valid. To provide type safety and perform bounds checks this proposal employs the newtype pattern to define a `Tryte` struct. This way we can profit from compile-time checks and make our code much less error prone.

Note: For readability sake visibility modifiers are omitted. Also, since this RFC is a proposal for `T3B1` and not for the `Tryte` implementation itself it is kept simple and internal.

```Rust
struct Tryte(i8);

impl Tryte {
    fn value(&self) -> i8 {
        self.0
    }
    ...
}
```

From that the actual encoding can be implemented using `std::vec::Vec` from Rust's standard library (optimizations are possible but excluded on purpose in this RFC). 

```Rust
struct T3B1(Vec<Tryte>);
```

```Rust
impl Encoding for T3B1 {
    ...
}
```

For more details checkout [this prototype](https://github.com/Alex6323/trits-module-preview) of a trits-module, that implements and abstracts over several different trit encodings.

# Drawbacks

This encoding is very inefficient in terms of memory as it only uses 27 values out of 256 available in a (signed) byte and therefore should not be used for storing or transmitting trinary data. 

# Rationale and alternatives

There is no reason to not have this encoding. Although not relevant for storage or transport it is indispensible for everything UI related, and also very helpful when it comes to debugging, testing and logging.

# Unresolved questions

* Should the interface only allow the list to grow? When does it actually make sense to remove trits or trytes from the encoding?