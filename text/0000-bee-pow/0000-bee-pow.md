+ Feature name: `bee-pow` 
+ Start date: 2019-10-03
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000) 
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

IOTA is a permissionless and feeless network which unavoidably makes it vulnerable to spam attacks. To make such
attacks more costly this RFC proposes a spam protection mechanism based on Proof-of-Work (PoW) where only those
messages are accepted and propagated by the honest nodes that include a valid nonce value. That nonce serves as a
proof to show that a predefined amount of computational work has been done upfront either by the publisher itself or
by some PoW service provider on its behalf.

# Motivation

Since IOTA is a permissionless and feeless network it needs methods of protecting itself against spam from
intentionally or even unintentionally misbehaving network participants (nodes or clients) that otherwise could
disrupt the network rather cheaply. Furthermore, this mechanism is currently (at the time of this writing) used in
the IOTA mainnet, and necessary to build compatible nodes with this framework.

### Job stories:

When I - as a client - want to send a message, for example a value transaction, I want to be able to find and include
a valid nonce, so that my transaction is accepted and propagated by all honest nodes in the network.

When I - as a node - receive a network message from the gossip pipeline, I want to be able to check if its contained
nonce value satisfies the pre-defined network difficulty, so that I can decide whether to discard or process that
message.

# Detailed design

WIP (various designs are currently evaluated in small prototype implementations)


For more information on PoW in general and the relation to the proposed `Transaction` struct please read:
* [Minimum Weight Magnitude](https://docs.iota.org/docs/dev-essentials/0.1/concepts/minimum-weight-magnitude),
* [Transaction/Bundle RFC](https://github.com/iotaledger/bee-rfcs/pull/20)

# Drawbacks

A spam protection mechanism based on PoW 
* consumes energy for doing throw-away calculations,
* slows down all participants even if there's no attack (always-on defense),
* is not favorable for battery-driven devices, many of which form the IoT space,
* amplifies problems that arise in networks where nodes have diverse computation capabilities like in the IoT space

# Rationale and alternatives

- Why is this design the best in the space of possible designs? 

The main purpose of this proposal is to support building nodes that can interoperate with the current IOTA mainnet.

- What other designs have been considered and what is the rationale for not choosing them? 

No other designs have been considered. Other anti-spam mechanisms are still being researched.

- What is the impact of not doing this?

Not doing this means that nodes built with this framework are incompatible with the current IOTA mainnet.


# Unresolved questions

- How modular do we want this crate to be?
- Should we focus on Curl or make the hashfunction plug and play?
- Should we design this a kind of service component employing its own runtime that performs PoW jobs asynchronously?
- Should support for external PoW be part of the implementation following the proposal?