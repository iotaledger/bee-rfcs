+ Feature name: `trits-t3b1`
+ Start date: 2019-09-27)
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC proposes a trit encoding that stores sequences of trytes (representing the values `-13,...,0,...13`) in a `Vec` of signed bytes `i8`. It also introduces simply handling of trytes using their character equivalents `N,..Z,9,A,...M` or `9A-Z` for short. It is one of the trit encodings mentioned in RFC #10 and basically useful for displaying trinary data due to its convenient mapping to the alphabet. Instead of `0` the `9` represents the zero tryte. That is to not confuse it with the letter `O`.

# Motivation

WIP



# Detailed design

WIP

# Drawbacks

This encoding is very inefficient in terms of memory as it only uses 27 values out of 256 available in a (signed) byte and therefore should not be used for storing or transmitting trinary data. 

# Rationale and alternatives

There is no reason to not have this encoding. Although not relevant for storage or transport it is a very helpful encoding when it comes to debugging, unit testing and logging.

# Unresolved questions

It is a well-known and long-time used encoding in the IOTA ecosystem, so there are no open questions around it known to the author.