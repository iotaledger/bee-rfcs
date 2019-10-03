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
| for transaction in bundle
| | sponge.absorb(transaction.essence())
| return sponge.squeeze()
```

*In the current mainnet, the hash function of the sponge used to generate bundle hashes is Kerl.*

## Bundle finalisation

<!-- TODO -->

## Bundle validation

<!-- TODO -->

# Drawbacks

<!-- TODO -->

# Rationale and alternatives

<!-- TODO -->

# Unresolved questions

<!-- TODO -->
