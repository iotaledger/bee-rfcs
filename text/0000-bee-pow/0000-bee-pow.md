+ Feature name: `bee-pow`
+ Start date: 2019-10-03
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes a dedicated crate to perform proof-of-work (PoW) for a single gossip message, that achieves to be accepted and propagated by the network. 

# Motivation

In order to protect the network from spam each device that wishes to get a message (e.g. a value transaction) propagated by all the nodes needs a way to proove to the network, that it has invested the necessary amount of computational work. 

### Background info on PoW
Cryptographic hash functions are one-way-functions. That means in simple terms that given the output you cannot calculate the input from it. Proof-of-Work works by setting some constraint on the output like a certain number of zeros at the end or the beginning of the hash. It's completely arbitrary. It could be ten `1`s or the sequence `1234567890` because any particular sequence is equally likely. What matters is that your only chance to find an input that satisfies the constraint is guessing/brute-forcing it. You can also easily set the difficulty by making the constraint harder to satisfy (extend the sequence). So to check if PoW was correctly done all a validating node has to do is hash the given message which includes the nonce, and see if the constraint is satisfied. On the other hand the message publisher has to repeat the following cycle many times until he has found a valid nonce:
* Pick a (random) nonce value
* Hash the message together with selected nonce
* Compare if the hash satisfies the constraint given by the majority of the nodes

This process is called `Mining` and in this analogy the nonce is the nugget.

TODO

Why are we doing this? What use cases does it support? What is the expected
outcome?

1. Write a summary of the motivation.
2. List all the specific use cases that your proposal is trying to address. 
3. Where applicable, write from the perspective of the person who will be using
   the software, for example using the "Job story" format:

When ＿＿＿ , I want to ＿＿＿, so I can ＿＿＿.

+ **Example 1:** When I query a node for a list of transactions, I want to be
  able to sort them by date, so I can work with the most relevant ones.
+ **Example 2:** When I configure a node, I want to be able to control how much
  transaction history the node stores, so I can make sure I only store the data
  I need without incurring additional operational costs.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the IOTA and to understand, and for somebody familiar with Rust
to implement. This should get into specifics and corner-cases, and include
examples of how the feature is used.

# Drawbacks

Why should we *not* do this?

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Unresolved questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
