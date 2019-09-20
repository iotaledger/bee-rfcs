+ Feature Name: `9/2 encoding`
+ Start Date: 2019-09-19)
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This feature introduces a special trits encoding to Bee that might be referred to as `9trits/2bytes`, `9t/2b`, or simply `9/2`, or maybe even `4.5trits/byte` encoding. In comparison there is the `10trits/2bytes`, `10t/2b`, or `5trits/byte`, or just `10/2` alternative encoding which is currently used in IOTAs reference implementation IRI. In contrast to that the `9/2` encoding is less memory/bandwidth efficient, but on the other hand is more optimized for byte-wise data processing. This encoding was successfully used already in the Ict node software which was intended to be able to run on weaker devices like Raspberry Pis which is also a primary objective for the Bee node framework. It can be expected that this encoding will positively affect a node's performance by:
* reducing (or maybe even eliminating) conversions between different trinary encodings in the gossip processing pipeline,
* allowing for efficient internal transaction representation, that also allows very fast field-wise deserialization,
* zero-copying (if received in this encoding),
* allowing for efficient byte-wise trimming of redundant data.

# Motivation

**A little bit of background information**: The most efficient way memory-wise to store trits in a byte is the `10t/2b` (10 trits per 2 bytes) encoding. While 2 bytes or 16 bits can represent `2^16 = 65536` different numbers, 11 trits can represent `3^11 = 177147` different numbers (don't fit), 10 trits can represent `3^10 = 59049` (do fit, cover ~90%), and `3^9 = 19683` (do fit, cover ~30%) different numbers. So no matter what trinary encoding you use you will have some significant inefficiencies in a world that is based on byte-wise data processing.

**Differences between `10/2` and `9/2`**: Consider the very common trit length `243` in the IOTA protocol: This trit length for example appears as the output of IOTA's trinary hash functions `Curl` and `Troika`). How many bytes would be required in each encoding to store that many trits?
* **10/2**: `(243/10)*2 = 243/5 = 48.6 bytes`
* **9/2**: `(243/ 9)*2 = 243/4.5 = 54 bytes`

Additionally let's consider an IOTA transaction which usually consists of 8019 trits:
* **10/2**: `(8019/10)*2 = 8019/5 = 1603,8 bytes`
* **9/2**: `(8019/ 9)*2 = 8019/4.5 = 1782 bytes`

There are three things noticeable here:
* the `9/2` unsurprisingly requires more bytes to represent the same amount of information (is less memory efficient),
* the `9/2` encoding maps onto integers for commonly used trit lengths, like the ones appearing as transaction field lengths: 27, 81, 243, 6561, 8019,
* the more data to encode the bigger the gap between both encodings gets in terms of memory efficiency.

Using the `9/2` encoding means additional - although redundant - `178,2` sent and stored bytes per uncompressed transaction. That might seem like a small overhead, but actually is quite significant considering billions of transactions going over the wire on a global scale.

There are, however, some reasons that might mitigate the downsides quite a bit:
* **trimming**: IOTA transactions, especially data transactions which do not fully utilize the available payload size, can be trimmed. The more a transaction can be trimmed the less overhead there is for the `9/2` encoding.
* **steganography**: Since `3^9` is even smaller than `2^15 = 32768`, we could make use of an unused bit every two bytes to store some additional metadata. In case of an uncompressed transaction there would be `1782/2/8 = ~111` bytes available. This would reduce the memory overhead - in the best case scenario where those residual bits are fully used - to mere `66.825` bytes, which is almost a 3 times improvement.

**Why are we doing this?**
- **compression**: efficient trimming of zero bytes in IOTA transactions on the order of microseconds (tested on a Raspberry 3B+), ~10x faster than general purpose lz4 compresssion, which is therefore possibly insignificant compared to other more intense node operations in the processing pipeline, 
- **(almost) no conversions**: since this encoding plays nicely with byte-wise data processing, bytes as received over the wire can be stored as-is in memory without any prior conversion involving memory allocations and copy operations to receive the converted representation 
- **partial deserialization**: enables fast deserialization of single transaction fields by simply pointing into a single array with integer offsets and field lengths, thereby preventing costly memory allocations for each transaction field.

# Detailed design

TODO: FINISH THIS SECTION

**Example transaction implementation based on 9/2 encoding**: This example assumes that a trivial trimming algorithm is used that only trims the 0 bytes from the signature message fragment.

```Rust
const FULL_TRANSACTON_BYTE_SIZE: usize = 1782;
const ADDRESS_BYTE_OFFSET: usize = 1512;
const ADDRESS_BYTE_LENGTH: usize = 54;

struct Transaction {
    bytes: Trits_9enc2, // a type that represents 9trits/2bytes encoded trits
    offset: usize,
}

impl Transaction {
    fn new(bytes: Trits_9enc2) -> Self {
        Self {
            bytes,
            offset: FULL_TRANSACTION_BYTE_SIZE - bytes.len(),
        }
    }
    fn address(&self) -> [Tryte; 81] {
        //now access and deserialize the address from self.bytes
        let offset = ADDRESS_BYTE_OFFSET - self.offset;
        let address = convert::trytes::from_trits_9enc2(&self.bytes[offset..offset+ADDRESS_BYTE_LENGTH]);

        address
    }
    fn value(&self) -> u64 {}
    fn trunk(&self) -> [Tryte; 81] {}
    fn branch(&self) -> [Tryte; 81] {}
    fn tag(&self) -> [Tryte; 27] {}
    ...
}
```
# Drawbacks

Bee needs to be able to integrate with the current IOTA mainnet which employs the `10/2` encoding for the transport, which - as explained earlier - prioritizes memory efficiency over CPU load reduction. If the internal representation of the Bees uses `9/2` then IRI neighbors will become a higher burden for a Bee node, since it has to convert the IRI encoding into `9/2`.

# Rationale and alternatives
    
There are basically only 2 options that make sense. Fully use `10/2` and be most memory efficient or fully use `9/2` and be most CPU efficient. During the transition phase  towards the new node software Bee will have to support both encodings at the same time. `10/2` is not optimal in a world of byte-wise data processing. It is more memory efficient, but while bandwidth available to a device can increase externally, the processing capabilities of a single device is predefined and unchangeable. That's why the encoding which optimizes for CPU load reduction should be preferred.


# Unresolved questions

TODO