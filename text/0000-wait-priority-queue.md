+ Feature name: `wait-priority-queue`
+ Start date: 2020-08-05
+ RFC PR: [iotaledger/bee-rfcs#0000](https://github.com/iotaledger/bee-rfcs/pull/0000)
+ Bee issue: [iotaledger/bee#0000](https://github.com/iotaledger/bee/issues/0000)

# Summary

This RFC introduces a data structure known as `WaitPriorityQueue` that
implements an `async`-compatible priority queue.

# Motivation

A priority queue with asynchronous blocking semantics is a useful component for
prioritising tasks in various parts of Bee. The requirements of the data
structure are as follows:

- Non-blocking insertion.
- Asynchronously await new items from the queue, in priority order.
- Implement `Stream` to allow waiting for many queue items.

# Detailed design

The following snippets document the API of the data structure.

```rust
pub struct WaitPriorityQueue<T> { ... }
```

```rust
impl<T: Ord + Eq> WaitPriorityQueue<T> {
    /// Push an item into the queue.
    pub fn push(&self, item: T) { ... }

    /// Attempt to pop the item with highest priority from the queue, should
    /// one be available.
    pub fn try_pop(&self) -> Option<T> { ... }

    /// Return an iterator over items that are immediately available from the
    /// queue, in priority order.
    pub fn pending(&self) -> impl Iterator<Item = T> + '_ { ... }

    /// Asynchronously pop the item with histed priority from the queue.
    pub async fn pop(&self) -> T { ... }

    /// Return a stream that produces items, in priority order, from the queue.
    pub fn incoming(&self) -> impl Stream<Item = T> + FusedStream + '_ { ... }
}
```

# Drawbacks

No significant drawbacks have been found.

# Rationale and alternatives

No such data structure appears to be available on [crates.io](https://crates.io)
so there is no alternative other than to implement this for Bee.

# Unresolved questions

No unresolved questions exist
