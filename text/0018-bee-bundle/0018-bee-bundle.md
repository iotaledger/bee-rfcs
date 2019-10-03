+ Feature name: `bee-bundle`
+ Start date: 2019-10-02
+ RFC PR: [iotaledger/bee-rfcs#18](https://github.com/iotaledger/bee-rfcs/pull/18)
+ Bee issue: [iotaledger/bee#63](https://github.com/iotaledger/bee/issues/63)

# Summary

Useful links:
- [Bundles and transactions
](https://docs.iota.org/docs/dev-essentials/0.1/concepts/bundles-and-transactions)
- [What is a bundle?
](https://docs.iota.org/docs/getting-started/0.1/introduction/what-is-a-bundle)

<!-- TODO -->

# Motivation

<!-- TODO -->

# Detailed design

## Bundle hash generation

The hash of a bundle is based on the bundle essence of each of its transactions. The bundle essence of a transaction is composed of the following fields.

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

## Bundle finalisation

Finalising a bundle means computing the bundle hash, verifying that it matches the security requirement and setting it to all the transactions.

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

## Bundle validation

Validating a bundle means checking the syntactic and semantic integrity of a bundle as a whole and of its constituent transactions. As bundles are atomic transfers, either all or none of the transactions will be accepted by the network.

The following assertions must be true:

- bundle has announced size;
- transactions share the same bundle hash;
- transactions absolute value doesn't exceed total IOTA supply;
- bundle absolute sum never exceeds total IOTA supply;
- transactions appear in the announced order;
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

- Should this RFC include explanation and/or pseudocode for bundle hash normalisation ?
- Should this RFC expands a bit more on the M-Bug ? Or give a link ?
