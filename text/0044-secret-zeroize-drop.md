+ Feature name: `secret-zeroize-drop`
+ Start date: 2020-07-08
+ RFC PR: [iotaledger/bee-rfcs#44](https://github.com/iotaledger/bee-rfcs/pull/44)
+ Bee issue: [iotaledger/bee#104](https://github.com/iotaledger/bee/issues/104)

# Summary

This RFC does not really introduce a feature but rather a recommendation when dealing with sensitive data; their memory
should be securely erased when not used anymore.

# Motivation

When dealing with sensitive data like key materials (e.g. seeds, private keys, ...), it is wise to explicitly clear
their memory to avoid having them continue existing and potentially exposing them to an attack.

Since such an attack already assumes unauthorized access to the data, this technique is merely an exploit mitigation by
ensuring sensitive data is no longer available.

Useful links:
- [How to zero a buffer](https://www.daemonology.net/blog/2014-09-04-how-to-zero-a-buffer.html)
- [Erasing Sensitive Data](https://www.gnu.org/software/libc/manual/html_node/Erasing-Sensitive-Data.html)
- [Ensure that sensitive data is not written out to disk](https://wiki.sei.cmu.edu/confluence/display/c/MEM06-C.+Ensure+that+sensitive+data+is+not+written+out+to+disk)
- [Would it be good secure programming practice to overwrite a “sensitive” variable before deleting it?](https://security.stackexchange.com/questions/74280/would-it-be-good-secure-programming-practice-to-overwrite-a-sensitive-variable)

# Detailed design

This task is made a bit more complex by compilers that have a tendency of removing zeroing of memory they deem pointless
when doing optimizations. It is then not as trivial as resetting the memory to 0.

These optimizations can however be bypassed by using [volatile memory](https://en.wikipedia.org/wiki/Volatile_(computer_programming)).

## `Zeroize` and `Drop`

This RFC recommends:
- implementing the following traits on sensitive data types:
    - [Zeroize](https://docs.rs/zeroize/1.1.0/zeroize/trait.Zeroize.html) to `Securely zero memory with a simple trait
    (Zeroize) built on stable Rust primitives which guarantee the operation will not be "optimized away"`;
    - [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) `Used to run some code when a value goes out of scope.
    This is sometimes called a 'destructor'` and making it call `self.zeroize()`;
- having `Zeroize` and `Drop` as requirements of sensitive data traits;

## Examples

Traits requirements:
```rust
trait PrivateKey: Zeroize + Drop {
    ...
}
```

If the type is trivial, `Zeroize` and `Drop` can be  derived:
```rust
#[derive(Zeroize)]
#[zeroize(drop)]
struct ExamplePrivateKey([u8; 32]);
```

Otherwise, `Zeroize` and `Drop` need to be manually implemented.

- `Zeroize` implementation:

    ```rust
    impl Zeroize for ExamplePrivateKey {
        fn zeroize(&mut self) {
            ...
        }
    }
    ```

- `Drop` implementation:

    ```rust
    impl Drop for ExamplePrivateKey {
        fn drop(&mut self) {
            self.zeroize()
        }
    }
    ```
    A [derive macro](https://doc.rust-lang.org/reference/procedural-macros.html) `SecretDrop` could be written to derive
    the `Drop` implementation that is always expected to be the same.

Useful links:
- [Handling sensitive data in memory](https://users.rust-lang.org/t/handling-sensitive-data-in-memory/14388)
- [Erasing secrets from memory (zero on drop)](https://github.com/dalek-cryptography/curve25519-dalek/issues/11)

# Drawbacks

There are no major drawbacks in doing that, except the little overhead it comes with.

# Rationale and alternatives

The alternative is to not recommend or enforce anything and risk users exposing their sensitive data to attacks.

# Unresolved questions

Even though this technique uses volatile memory and is probably effective most of the time the compiler is still free to
move all of the inner data to another part of memory without zeroing the original data since it doesn't consider the
value to be dropped yet. A solution to prevent the compiler moving things whenever possible would be to
[box](https://doc.rust-lang.org/std/boxed/struct.Box.html) secrets and access them through [Pin](https://doc.rust-lang.org/std/pin/index.html).
