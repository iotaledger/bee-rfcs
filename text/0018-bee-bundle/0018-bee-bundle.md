+ Feature name: `bee-bundle`
+ Start date: 2019-10-02
+ RFC PR: [iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18)
+ Bee issue: [iotaledger/bee#63](https://github.com/iotaledger/bee/issues/63)

# Summary

This RFC is based on [`transaction-module`]().
<!-- TODO FIx link to RFC#3 -->

The smallest communication unit in the IOTA protocol is the transaction. Everything, including payment settlements and/or plain data, is propagated through the IOTA network in transactions.

A transaction is `2673` trytes and the part available to the user is `2187` trytes. This part holds a signature in case of a payment settlement and plain data otherwise. Since it has a limited size, a user often needs more than one transaction to fulfil their operation, for example signatures with security level `2` or `3` don't fit in a single transaction and user-provided data may exceed the allowance so they need to be fragmented across multiple transactions. Moreover, a value transaction doesn't make sense on its own because it would change the total amount of the ledger so it has to be paired with other complementary transactions that together will balance the total value to zero.

For these reasons, transactions have to be processed as a whole, in groups called bundles. A bundle is an atomic operation in the sense that either all or none of its transactions are accepted by the network. Even single transactions are propagated through the network within a bundle making it the only confirmable communication unit of the IOTA protocol.

This RFC proposes ways to create and manipulate a bundle and describe the associated algorithms.

Useful links:
- [What is a bundle?
](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-bundle)
- [Bundles and transactions
](https://docs.iota.org/docs/dev-essentials/0.1/concepts/bundles-and-transactions)

# Motivation

<!-- TODO -->

# Detailed design

<!-- TODO -->

## Bundle and BundleBuilder

Transactions are final and bundles, essentially being arrays of transactions, are also final. Once a bundle is created and validated, it shouldn't be tempered. For this reason we have a `Bundle` type and a `BundleBuilder` type. An instantiated `Bundle` object represents a syntactically and semantically valid IOTA bundle and a `BundleBuilder` is the only gateway to a `Bundle` object.

### Bundle

A bundle can simply be represented as an array of transactions. As bundles are final, they shouldn't be modifiable outside of the scope of the bundle module.

```rust
struct Bundle {
    transactions: Vec<Transaction>
}
```

```rust
impl Bundle {    
    pub fn transactions(&self) -> &Vec<Transaction> {
        &self.transactions
    }
}
```

<!-- TODO -->

### BundleBuilder

<!-- TODO -->

## Algorithms

<!-- TODO -->

### Bundle hash generation

*Client side and server side operation.*

A bundle hash ties different transactions together. By having this common hash in their `bundle` field, it makes it clear that these transactions should be processed as a whole.

The hash of a bundle is derived from the bundle essence of each of its transactions. The bundle essence of a transaction is composed of the following fields.

| Name          | Size      |
| ------------- | --------- |
| address       | 243 trits |
| value         | 81 trits  |
| obsolete_tag  | 81 trits  |
| timestamp     | 27 trits  |
| current_index | 27 trits  |
| last_index    | 27 trits  |

The bundle hash is generated with a sponge by iterating through the bundle, from `0` to `last_index`, absorbing the bundle essence of each transaction and eventually squeezing the bundle hash from the sponge.

Pseudocode:

```
bundleHash(bundle)
| sponge = Sponge(HASH_FUNCTION)
|
| for transaction in bundle
| | sponge.absorb(transaction.essence())
|
| return sponge.squeeze()
```

*In the current mainnet, the hash function of the sponge used to generate bundle hashes is Kerl.*

### Bundle finalisation

*Client side operation.*

Finalising a bundle means computing the bundle hash, verifying that it matches the security requirement and setting it to all the transactions of the bundle. After finalisation, transactions of a bundle are ready to be safely attached to the tangle.

Pseudocode:

```
bundleFinalise(bundle)
| hash = bundleHash(bundle)
|
| while hash.normalise().find('M')
| | bundle.at(0).obsolete_tag++
| | hash = bundleHash(bundle)
|
| for transaction in bundle
| | transaction.setBundleHash(hash)
```

*Security requirement: due to the implementation of the signature process, the normalised bundle hash can't contain a `M` or `13` because it could expose a significant part of the private key, weakening the signature. The bundle hash is then repetitively generated with a slight modification until its normalisation doesn't contain a `M`.*

### Bundle validation

*Server side operation.*

Validating a bundle means checking the syntactic and semantic integrity of a bundle as a whole and of its constituent transactions. As bundles are atomic transfers, either all or none of the transactions will be accepted by the network. After validation, transactions of a bundle are candidates to be included to the ledger.

For a bundle to be considered valid, the following assertions must be true:

- bundle has announced size;
- transactions share the same bundle hash;
- transactions absolute value doesn't exceed total IOTA supply;
- bundle absolute sum never exceeds total IOTA supply;
- order of transactions in the bundle is the same as announced by `current_index` and `last_index`;
- value transactions have an address ending in `0` i.e. has been generated with Kerl;
- bundle inputs and outputs are balanced i.e. the bundle sum equals `0`;
- announced bundle hash matches the computed bundle hash;
- for spending transactions, the signature is valid;

Pseudocode:

```
bundleValidate(bundle):
| value = 0
| current_index = 0
|
| if bundle.length() != bundle.at(0).last_index + 1
| | return BUNDLE_INVALID_LENGTH
|
| bundle_hash = bundle.at(0).bundle_hash
| last_index = bundle.at(0).last_index
|
| for transaction in bundle
| | if transaction.bundle_hash != bundle_hash
| | | return BUNDLE_INVALID_HASH
| | if abs(transaction.value) > IOTA_SUPPLY
| | | return BUNDLE_INVALID_TRANSACTION_VALUE
| | value = value + transaction.value
| | if abs(value) > IOTA_SUPPLY
| | | return BUNDLE_INVALID_VALUE
| | if transaction.current_index != current_index++
| | | return BUNDLE_INVALID_INDEX
| | if transaction.last_index != last_index
| | | return BUNDLE_INVALID_INDEX
| | if transaction.value != 0 && transaction.address.last != 0
| | | return BUNDLE_INVALID_ADDRESS
|
| if value != 0
| | return BUNDLE_INVALID_VALUE
| if bundle_hash != bundleHash(bundle)
| | return BUNDLE_INVALID_HASH
| if !bundleSignaturesValidate(bundle)
| | return BUNDLE_INVALID_SIGNATURE
|
| return BUNDLE_VALID
```

# Drawbacks

<!-- TODO -->

# Rationale and alternatives

<!-- TODO -->

# Unresolved questions

<!-- TODO -->

- Should this RFC expands a bit more on the M-Bug ? Or give a link ?
