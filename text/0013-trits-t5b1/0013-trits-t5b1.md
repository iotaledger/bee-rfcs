+ Feature name: `trits-t5b1`
+ Start date: 2019-09-25
+ RFC PR: [iotaledger/bee-rfcs#13](https://github.com/iotaledger/bee-rfcs/pull/13)
+ Bee issue: [iotaledger/bee#61](https://github.com/iotaledger/bee/issues/61)

# Summary

This RFC proposes an encoding that packs `5` `balanced trits` to a single `signed byte`. It may be referred to as `5t/1b`, `5t1b`, `t5/b1`, `t5b1`, `5/1`, `10/2`, ... To comply with most languages identifier rules, we'll prefer using `t5b1`.

<!-- TODO Update with link to RFC#10 -->
This encoding implements the interface presented by RFC#10.

# Motivation

A `byte` is composed of `8` `bits` that can hold `2^8 = 256` different values. On the other hand, `6` `trits` can hold `3^6 = 729` values while `5` `trits` can hold `3^5 = 243` values. The maximum number of `trits` a `byte` can hold is then `5`.

As this is the most memory-efficient way to encode `trits` in `bytes`, `t5b1` is often used for network communications to increase throughput and for storage layers to reduce memory consumption.

# Detailed design

This RFC assumes the reader has minimum knowledge on base conversions.

For efficient conversions, we can make use of LookUp Tables (`LUTs`).

Because the memory footprint (2 times ~`1.2kb`) of these `LUTs` can be too high for some very limited systems, we also present algorithms that do not use it.

## Decoding 1 signed byte to 5 balanced trits

### With LUT

This LUT contains the `243` possible values ordered like this: `0`, `1`, ... `120`, `121`, `-121`, `-120`, ... `-2`, `-1`.

```rust
[
    [ 0, 0, 0, 0, 0 ],     [ 1, 0, 0, 0, 0 ],     [ -1, 1, 0, 0, 0 ],     [ 0, 1, 0, 0, 0 ],     [ 1, 1, 0, 0, 0 ],
    [ -1, -1, 1, 0, 0 ],   [ 0, -1, 1, 0, 0 ],    [ 1, -1, 1, 0, 0 ],     [ -1, 0, 1, 0, 0 ],    [ 0, 0, 1, 0, 0 ],
    [ 1, 0, 1, 0, 0 ],     [ -1, 1, 1, 0, 0 ],    [ 0, 1, 1, 0, 0 ],      [ 1, 1, 1, 0, 0 ],     [ -1, -1, -1, 1, 0 ],
    [ 0, -1, -1, 1, 0 ],   [ 1, -1, -1, 1, 0 ],   [ -1, 0, -1, 1, 0 ],    [ 0, 0, -1, 1, 0 ],    [ 1, 0, -1, 1, 0 ],
    [ -1, 1, -1, 1, 0 ],   [ 0, 1, -1, 1, 0 ],    [ 1, 1, -1, 1, 0 ],     [ -1, -1, 0, 1, 0 ],   [ 0, -1, 0, 1, 0 ],
    [ 1, -1, 0, 1, 0 ],    [ -1, 0, 0, 1, 0 ],    [ 0, 0, 0, 1, 0 ],      [ 1, 0, 0, 1, 0 ],     [ -1, 1, 0, 1, 0 ],
    [ 0, 1, 0, 1, 0 ],     [ 1, 1, 0, 1, 0 ],     [ -1, -1, 1, 1, 0 ],    [ 0, -1, 1, 1, 0 ],    [ 1, -1, 1, 1, 0 ],
    [ -1, 0, 1, 1, 0 ],    [ 0, 0, 1, 1, 0 ],     [ 1, 0, 1, 1, 0 ],      [ -1, 1, 1, 1, 0 ],    [ 0, 1, 1, 1, 0 ],
    [ 1, 1, 1, 1, 0 ],     [ -1, -1, -1, -1, 1 ], [ 0, -1, -1, -1, 1 ],   [ 1, -1, -1, -1, 1 ],  [ -1, 0, -1, -1, 1 ],
    [ 0, 0, -1, -1, 1 ],   [ 1, 0, -1, -1, 1 ],   [ -1, 1, -1, -1, 1 ],   [ 0, 1, -1, -1, 1 ],   [ 1, 1, -1, -1, 1 ],
    [ -1, -1, 0, -1, 1 ],  [ 0, -1, 0, -1, 1 ],   [ 1, -1, 0, -1, 1 ],    [ -1, 0, 0, -1, 1 ],   [ 0, 0, 0, -1, 1 ],
    [ 1, 0, 0, -1, 1 ],    [ -1, 1, 0, -1, 1 ],   [ 0, 1, 0, -1, 1 ],     [ 1, 1, 0, -1, 1 ],    [ -1, -1, 1, -1, 1 ],
    [ 0, -1, 1, -1, 1 ],   [ 1, -1, 1, -1, 1 ],   [ -1, 0, 1, -1, 1 ],    [ 0, 0, 1, -1, 1 ],    [ 1, 0, 1, -1, 1 ],
    [ -1, 1, 1, -1, 1 ],   [ 0, 1, 1, -1, 1 ],    [ 1, 1, 1, -1, 1 ],     [ -1, -1, -1, 0, 1 ],  [ 0, -1, -1, 0, 1 ],
    [ 1, -1, -1, 0, 1 ],   [ -1, 0, -1, 0, 1 ],   [ 0, 0, -1, 0, 1 ],     [ 1, 0, -1, 0, 1 ],    [ -1, 1, -1, 0, 1 ],
    [ 0, 1, -1, 0, 1 ],    [ 1, 1, -1, 0, 1 ],    [ -1, -1, 0, 0, 1 ],    [ 0, -1, 0, 0, 1 ],    [ 1, -1, 0, 0, 1 ],
    [ -1, 0, 0, 0, 1 ],    [ 0, 0, 0, 0, 1 ],     [ 1, 0, 0, 0, 1 ],      [ -1, 1, 0, 0, 1 ],    [ 0, 1, 0, 0, 1 ],
    [ 1, 1, 0, 0, 1 ],     [ -1, -1, 1, 0, 1 ],   [ 0, -1, 1, 0, 1 ],     [ 1, -1, 1, 0, 1 ],    [ -1, 0, 1, 0, 1 ],
    [ 0, 0, 1, 0, 1 ],     [ 1, 0, 1, 0, 1 ],     [ -1, 1, 1, 0, 1 ],     [ 0, 1, 1, 0, 1 ],     [ 1, 1, 1, 0, 1 ],
    [ -1, -1, -1, 1, 1 ],  [ 0, -1, -1, 1, 1 ],   [ 1, -1, -1, 1, 1 ],    [ -1, 0, -1, 1, 1 ],   [ 0, 0, -1, 1, 1 ],
    [ 1, 0, -1, 1, 1 ],    [ -1, 1, -1, 1, 1 ],   [ 0, 1, -1, 1, 1 ],     [ 1, 1, -1, 1, 1 ],    [ -1, -1, 0, 1, 1 ],
    [ 0, -1, 0, 1, 1 ],    [ 1, -1, 0, 1, 1 ],    [ -1, 0, 0, 1, 1 ],     [ 0, 0, 0, 1, 1 ],     [ 1, 0, 0, 1, 1 ],
    [ -1, 1, 0, 1, 1 ],    [ 0, 1, 0, 1, 1 ],     [ 1, 1, 0, 1, 1 ],      [ -1, -1, 1, 1, 1 ],   [ 0, -1, 1, 1, 1 ],
    [ 1, -1, 1, 1, 1 ],    [ -1, 0, 1, 1, 1 ],    [ 0, 0, 1, 1, 1 ],      [ 1, 0, 1, 1, 1 ],     [ -1, 1, 1, 1, 1 ],
    [ 0, 1, 1, 1, 1 ],     [ 1, 1, 1, 1, 1 ],     [ -1, -1, -1, -1, -1 ], [ 0, -1, -1, -1, -1 ], [ 1, -1, -1, -1, -1 ],
    [ -1, 0, -1, -1, -1 ], [ 0, 0, -1, -1, -1 ],  [ 1, 0, -1, -1, -1 ],   [ -1, 1, -1, -1, -1 ], [ 0, 1, -1, -1, -1 ],
    [ 1, 1, -1, -1, -1 ],  [ -1, -1, 0, -1, -1 ], [ 0, -1, 0, -1, -1 ],   [ 1, -1, 0, -1, -1 ],  [ -1, 0, 0, -1, -1 ],
    [ 0, 0, 0, -1, -1 ],   [ 1, 0, 0, -1, -1 ],   [ -1, 1, 0, -1, -1 ],   [ 0, 1, 0, -1, -1 ],   [ 1, 1, 0, -1, -1 ],
    [ -1, -1, 1, -1, -1 ], [ 0, -1, 1, -1, -1 ],  [ 1, -1, 1, -1, -1 ],   [ -1, 0, 1, -1, -1 ],  [ 0, 0, 1, -1, -1 ],
    [ 1, 0, 1, -1, -1 ],   [ -1, 1, 1, -1, -1 ],  [ 0, 1, 1, -1, -1 ],    [ 1, 1, 1, -1, -1 ],   [ -1, -1, -1, 0, -1 ],
    [ 0, -1, -1, 0, -1 ],  [ 1, -1, -1, 0, -1 ],  [ -1, 0, -1, 0, -1 ],   [ 0, 0, -1, 0, -1 ],   [ 1, 0, -1, 0, -1 ],
    [ -1, 1, -1, 0, -1 ],  [ 0, 1, -1, 0, -1 ],   [ 1, 1, -1, 0, -1 ],    [ -1, -1, 0, 0, -1 ],  [ 0, -1, 0, 0, -1 ],
    [ 1, -1, 0, 0, -1 ],   [ -1, 0, 0, 0, -1 ],   [ 0, 0, 0, 0, -1 ],     [ 1, 0, 0, 0, -1 ],    [ -1, 1, 0, 0, -1 ],
    [ 0, 1, 0, 0, -1 ],    [ 1, 1, 0, 0, -1 ],    [ -1, -1, 1, 0, -1 ],   [ 0, -1, 1, 0, -1 ],   [ 1, -1, 1, 0, -1 ],
    [ -1, 0, 1, 0, -1 ],   [ 0, 0, 1, 0, -1 ],    [ 1, 0, 1, 0, -1 ],     [ -1, 1, 1, 0, -1 ],   [ 0, 1, 1, 0, -1 ],
    [ 1, 1, 1, 0, -1 ],    [ -1, -1, -1, 1, -1 ], [ 0, -1, -1, 1, -1 ],   [ 1, -1, -1, 1, -1 ],  [ -1, 0, -1, 1, -1 ],
    [ 0, 0, -1, 1, -1 ],   [ 1, 0, -1, 1, -1 ],   [ -1, 1, -1, 1, -1 ],   [ 0, 1, -1, 1, -1 ],   [ 1, 1, -1, 1, -1 ],
    [ -1, -1, 0, 1, -1 ],  [ 0, -1, 0, 1, -1 ],   [ 1, -1, 0, 1, -1 ],    [ -1, 0, 0, 1, -1 ],   [ 0, 0, 0, 1, -1 ],
    [ 1, 0, 0, 1, -1 ],    [ -1, 1, 0, 1, -1 ],   [ 0, 1, 0, 1, -1 ],     [ 1, 1, 0, 1, -1 ],    [ -1, -1, 1, 1, -1 ],
    [ 0, -1, 1, 1, -1 ],   [ 1, -1, 1, 1, -1 ],   [ -1, 0, 1, 1, -1 ],    [ 0, 0, 1, 1, -1 ],    [ 1, 0, 1, 1, -1 ],
    [ -1, 1, 1, 1, -1 ],   [ 0, 1, 1, 1, -1 ],    [ 1, 1, 1, 1, -1 ],     [ -1, -1, -1, -1, 0 ], [ 0, -1, -1, -1, 0 ],
    [ 1, -1, -1, -1, 0 ],  [ -1, 0, -1, -1, 0 ],  [ 0, 0, -1, -1, 0 ],    [ 1, 0, -1, -1, 0 ],   [ -1, 1, -1, -1, 0 ],
    [ 0, 1, -1, -1, 0 ],   [ 1, 1, -1, -1, 0 ],   [ -1, -1, 0, -1, 0 ],   [ 0, -1, 0, -1, 0 ],   [ 1, -1, 0, -1, 0 ],
    [ -1, 0, 0, -1, 0 ],   [ 0, 0, 0, -1, 0 ],    [ 1, 0, 0, -1, 0 ],     [ -1, 1, 0, -1, 0 ],   [ 0, 1, 0, -1, 0 ],
    [ 1, 1, 0, -1, 0 ],    [ -1, -1, 1, -1, 0 ],  [ 0, -1, 1, -1, 0 ],    [ 1, -1, 1, -1, 0 ],   [ -1, 0, 1, -1, 0 ],
    [ 0, 0, 1, -1, 0 ],    [ 1, 0, 1, -1, 0 ],    [ -1, 1, 1, -1, 0 ],    [ 0, 1, 1, -1, 0 ],    [ 1, 1, 1, -1, 0 ],
    [ -1, -1, -1, 0, 0 ],  [ 0, -1, -1, 0, 0 ],   [ 1, -1, -1, 0, 0 ],    [ -1, 0, -1, 0, 0 ],   [ 0, 0, -1, 0, 0 ],
    [ 1, 0, -1, 0, 0 ],    [ -1, 1, -1, 0, 0 ],   [ 0, 1, -1, 0, 0 ],     [ 1, 1, -1, 0, 0 ],    [ -1, -1, 0, 0, 0 ],
    [ 0, -1, 0, 0, 0 ],    [ 1, -1, 0, 0, 0 ],    [ -1, 0, 0, 0, 0 ]
]
```

Decoding a `signed byte` into `5` `balanced` `trits` with the help of the `LUT` is fairly simple:
- `byte >= 0`: use `byte` as an index for the `LUT` e.g. if the `byte = 42` then `LUT[42] = [ 0, -1, -1, -1, 1 ]` and `0 * 3^0 + (-1) * 3^1 + (-1) * 3^2 + (-1) * 3^3 + 1 * 3^4 = 42`;
- `byte < 0`: use `byte + 243` as an index for the `LUT` e.g. if the `byte = -42` then `LUT[201] = [ 0, 1, 1, 1, -1 ]` and `0 * 3^0 + 1 * 3^1 + 1 * 3^2 + 1 * 3^3 + (-1) * 3^4 = -42`;

Even though this RFC won't cover every little implementation details, it can be noted that the `LUT` could cover all `256` possible values of a `byte` to avoid invalid accesses. To the missing `byte` values we can simply assign some default mapping to `5` `trits`, eg. `[ 0, 0, 0, 0, 0 ]`. This way runtime checks can be avoided as branching on each iteration of a loop can degrade performance in case of false branch prediction.

Additionally, there could be two variants of the decoding function:
- checked & slow;
- unchecked & fast;

The overhead of the `13` additional values is only +~5% to the size of `LUT`.

### Without LUT

For this method, we use the fact that `3^n+1 = 2 * (3^0 + 3^1 + ... 3^n-2 * 3^n-1) + 1` (e.g. `81 = 2 * (1 + 3 + 9 + 27) + 1`) meaning that half a power of `3` can be balanced by the sum of the lower powers of `3`.

For every rank from `4` to `0`, we try to fit a power of `3` taking into account that half of it can be balanced by the sum of lower ranks. If the byte can't be balanced, a `-1` or `1` trit is produced depending on its sign and it is updated. If the byte can be balanced, a `0` trit is produced and it is not updated.

For example:
- Rank  4: `|46| > 81 / 2` can't be balanced so `1` is produced and `46 - 81 = -35` is updated;
- Rank  3: `|-35| > 27 / 2` can't be balanced so `-1` is produced and `-35 + 27 = -8` is updated;
- Rank  2: `|-8| > 9 / 2` can't be balanced so `-1` is produced and `-8 + 9 = 1` is updated;
- Rank  1: `|1| < 3 / 2` can be balanced so `0` is produced and `1` is not updated;
- Rank  0: `|1| > 1 / 2` can't be balanced so `1` is produced and `1 - 1 = 0` is updated;

The resulting trits are then `[1, 0, -1,  -1, 1]` and we can verify that `1 * 3^0 + 0 * 3^1 + (-1) * 3^2 + (-1) * 3^3 + 1 * 3^4 = 46`.

## Encoding 5 balanced trits to 1 signed byte

### With LUT

This LUT contains the `243` possible values ordered by all combinations of `trits` `-1`, `0` and `1` like `LUT[-1][-1][-1][-1][-1] = -121`, `LUT[-1][-1][-1][-1][0] = -40`, `LUT[-1][-1][-1][-1][1] = 41`, ..., `LUT[1][1][1][1][-1] = -41`, `LUT[1][1][1][1][0] = 40`, `LUT[1][1][1][1][1] = 121`

```rust
[
    [
        [
            [
                [ -121, -40, 41 ],
                [ -94, -13, 68 ],
                [ -67, 14, 95 ],
            ],
            [
                [ -112, -31, 50 ],
                [ -85, -4, 77 ],
                [ -58, 23, 104 ],
            ],
            [
                [ -103, -22, 59 ],
                [ -76, 5, 86 ],
                [ -49, 32, 113 ],
            ],
        ],
        [
            [
                [ -118, -37, 44 ],
                [ -91, -10, 71 ],
                [ -64, 17, 98 ],
            ],
            [
                [ -109, -28, 53 ],
                [ -82, -1, 80 ],
                [ -55, 26, 107 ],
            ],
            [
                [ -100, -19, 62 ],
                [ -73, 8, 89 ],
                [ -46, 35, 116 ],
            ],
        ],
        [
            [
                [ -115, -34, 47 ],
                [ -88, -7, 74 ],
                [ -61, 20, 101 ],
            ],
            [
                [ -106, -25, 56 ],
                [ -79, 2, 83 ],
                [ -52, 29, 110 ],
            ],
            [
                [ -97, -16, 65 ],
                [ -70, 11, 92 ],
                [ -43, 38, 119 ],
            ],
        ],
    ],
    [
        [
            [
                [ -120, -39, 42 ],
                [ -93, -12, 69 ],
                [ -66, 15, 96 ],
            ],
            [
                [ -111, -30, 51 ],
                [ -84, -3, 78 ],
                [ -57, 24, 105 ],
            ],
            [
                [ -102, -21, 60 ],
                [ -75, 6, 87 ],
                [ -48, 33, 114 ],
            ],
        ],
        [
            [
                [ -117, -36, 45 ],
                [ -90, -9, 72 ],
                [ -63, 18, 99 ],
            ],
            [
                [ -108, -27, 54 ],
                [ -81, 0, 81 ],
                [ -54, 27, 108 ],
            ],
            [
                [ -99, -18, 63 ],
                [ -72, 9, 90 ],
                [ -45, 36, 117 ],
            ],
        ],
        [
            [
                [ -114, -33, 48 ],
                [ -87, -6, 75 ],
                [ -60, 21, 102 ],
            ],
            [
                [ -105, -24, 57 ],
                [ -78, 3, 84 ],
                [ -51, 30, 111 ],
            ],
            [
                [ -96, -15, 66 ],
                [ -69, 12, 93 ],
                [ -42, 39, 120 ],
            ],
        ],
    ],
    [
        [
            [
                [ -119, -38, 43 ],
                [ -92, -11, 70 ],
                [ -65, 16, 97 ],
            ],
            [
                [ -110, -29, 52 ],
                [ -83, -2, 79 ],
                [ -56, 25, 106 ],
            ],
            [
                [ -101, -20, 61 ],
                [ -74, 7, 88 ],
                [ -47, 34, 115 ],
            ],
        ],
        [
            [
                [ -116, -35, 46 ],
                [ -89, -8, 73 ],
                [ -62, 19, 100 ],
            ],
            [
                [ -107, -26, 55 ],
                [ -80, 1, 82 ],
                [ -53, 28, 109 ],
            ],
            [
                [ -98, -17, 64 ],
                [ -71, 10, 91 ],
                [ -44, 37, 118 ],
            ],
        ],
        [
            [
                [ -113, -32, 49 ],
                [ -86, -5, 76 ],
                [ -59, 22, 103 ],
            ],
            [
                [ -104, -23, 58 ],
                [ -77, 4, 85 ],
                [ -50, 31, 112 ],
            ],
            [
                [ -95, -14, 67 ],
                [ -68, 13, 94 ],
                [ -41, 40, 121 ],
            ],
        ],
    ],
]
```

Encoding up to `5` `balanced` `trits` to a `signed byte` with the help of the `LUT` is fairly simple: add `1` to the individual `trits` to map them to array indexes (`-1` -> `0`, `0` -> `1` and `1` -> `2`) and use them to access the `LUT`. If you're encoding less than `5` `trits`, just fill with `0`s.

For example:
- encoding `[-1, 1, 1]`;
- filling with `0`s `[-1, 1, 1, 0, 0]`;
- adding `1` `[0, 2, 2, 1, 1]`;
- accessing `LUT[0][2][2][1][1] = 11`;
- checking `-1 * 3^0 + 1 * 3^1 + 1 * 3^2 = 11`;

### Without LUT

Encoding up to `5` `balanced` `trits` to a `signed byte` is a simple conversion from balanced ternary to decimal: the resulting `byte` is the sum of all the digits being multiplied by `3` to the rank.

For example:
- encoding `[ 0, -1, -1, -1, 1 ]`;
- computing `0 * 3^0 + (-1) * 3^1 + (-1) * 3^2 + (-1) * 3^3 + 1 * 3^4 = 42`;

## Usual sizes

For convenience purposes, here are some usual sizes and their encoded equivalents:

- `27` `trits` fit in `6` `bytes`
- `81` `trits` fit in `17` `bytes`
- `243` `trits` fit in `49` `bytes`
- `6561` `trits` fit in `1313` `bytes`
- `8019` `trits` fit in `1604` `bytes`

# Drawbacks

In the current state of the IOTA ecosystem, specifically the network, this encoding is absolutely necessary. There are no intrinsic drawbacks with this encoding, a user should just be careful when choosing it for his own application because it can't be used or is not recommended for every situation.

# Rationale and alternatives

If the intended purpose of using this encoding is saving memory or increasing communication throughput, then there is no better alternative since it is proven to be the most memory-efficient.

# Unresolved questions

There are no unresolved questions because:
- algorithms are well defined and vetted as they have been used by [IRI](https://github.com/iotaledger/iri)/[cIRI](https://github.com/iotaledger/entangled/tree/develop/ciri) for both network communications and storage layers for a long time;
<!-- TODO Edit link -->
- it will follow a well-defined interface provided by RFC#10;

It should only be noted, as previously stated, that different implementations are possible and some may raise security concerns.
