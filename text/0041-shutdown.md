+ Feature name: `shutdown`
+ Start date: 2020-07-03
+ RFC PR: [iotaledger/bee-rfcs#41](https://github.com/iotaledger/bee-rfcs/pull/41)
+ Bee issue: [iotaledger/bee#96](https://github.com/iotaledger/bee/issues/96)

# Summary

This RFC proposes a unified way of achieving a graceful shutdown of arbitrarily many async workers and their utilized resources in the Bee framework.

# Motivation

Rust provides one of the most powerful, performant, and still safest abstractions to deal with *concurrency*. You can read about Rust's concurrency story [here](https://doc.rust-lang.org/book/ch16-00-concurrency.html).

One problem that arises when dealing with many asynchronous tasks within a system is to ensure that all of them terminate gracefully once an agreed upon shutdown signal was issued. An example for this is when a software intercepts the press of `^C` in the terminal, and then executes specialized instructions to prevent data corruption before finally exiting the application.

This feature solves this problem by introducing a type called `Shutdown`, that allows to register ansynchronous workers, and `execute` a shutdown in a 3-step process:

1. Send a shutdown signal to all registered workers;
2. Wait for all the workers to terminate;
3. Perform final actions (like flushing buffers to databases, saving files, etc.)

# Detailed design

As the shutdown signal itself contains no data, the zero-sized unit type `()` is used to model it. Furthermore, as the synchronizing mechanism, this RFC proposes the use of `oneshot` channels from the `futures` crate. Those channels are consumed, i.e. destroyed, when calling `send` once; this way Rust ensures at compile-time, that no further redundant shutdown signals can be sent to the same listener.

To make the code more readable and less verbose, this RFC proposes the following type aliases:

```rust
// The "notifier" is the origin of the shutdown signal. For each spawned worker a
// dedicated notifier will be created and kept in a list.
type ShutdownNotifier = oneshot::Sender<()>;

// The "listener" is the receiver of the shutdown signal. It awaits the shutdown
// signal as part of the event loop of the corresponding worker.
type ShutdownListener = oneshot::Receiver<()>;

// For each async worker the async runtime creates a `Future`, that completes when the
// worker terminates, i.e. shuts down. To make sure, the shutdown is complete, one has
// to store those `Future`s in a list, and await all of them iteratively before exiting
// the program.
type WorkerShutdown = Box<dyn Future<Output = Result<(), WorkerError>> + Unpin>;

// Before shutting down a system, workers oftentimes need to perform one or more final
// actions to prevent data loss and/or data corruption. Therefore each worker can register
// an arbitrary amount of actions, that will be executed as the final step of the shutdown
// process.
type Action = Box<dyn Fn()>;
```

**NOTE:** `WorkerShutdown` and `Action` are boxed trait objects; hence have a single owner, and are stored on the heap. The `Unpin` marker trait requirement ensures that the given trait object can be moved into the internal datastructure. If the worker terminates with an error, it will be wrapped by a variant of the `WorkerError` enum. If the worker terminates without any error, `()` is returned - this time indicating that the worker doesn't produce a return value.

## `Shutdown`

The shutdown functionality can be implemented using only one central type, that essentially is an append-only registry with a single `execute` operation. There is no need for removal, or identifying single items as the operation is applied to all of them equally, and once done, the whole program can be expected to terminate.

This `Shutdown` type looks like this:

```rust
struct Shutdown {
    // A list of all registered shutdown signal senders, briefly labeled as "notifiers".
    notifiers: Vec<ShutdownNotifier>,

    // A list of all registered worker termination `Future`s.
    worker_shutdowns: Vec<WorkerShutdown>,

    // A list of all registered finalizing actions performed once all async workers
    // have terminated.
    actions: Vec<Action>,
}
```

The API of `Shutdown` is very small. It only needs to allow filling those internal lists, and provide an `execute` method to facilitate the actual shutdown:

The implementation of `Shutdown` looks like this:
```rust
impl Shutdown {

    // Creates a new instance
    fn new() -> Self {
        Self::default()
    }

    // Registers a shutdown notification channel by providing its sender half.
    fn add_notifier(&mut self, notifier: ShutdownNotifier) {
        self.notifiers.push(notifier);
    }

    // Registers a worker shutdown future, which completes once the worker terminates in one
    // way or another.
    fn add_worker_shutdown(&mut self, worker: impl Future<Output = Result<(), WorkerError>> + Unpin + 'static) {
        self.worker_shutdowns.push(Box::new(worker));
    }

    // Registers an action to perform when the shutdown is executed.
    fn add_action(&mut self, action: impl Fn() + 'static) {
        self.actions.push(Box::new(action));
    }

    // Executes the shutdown.
    async fn execute(self) -> Result<(), Error> {
        // Step 1: notify all registrees
        for notifier in self.notifiers {
            notifier.send(()).map_err(|_| Error::SendingShutdownSignalFailed)?
        }

        // Step 2: await workers to terminate their event loop
        for worker in self.worker_shutdowns {
            if let Err(e) = worker.await {
                error!("Awaiting worker failed: {:?}.", e);
            }
        }

        // Step 3: perform finalizing actions to prevent data/resource corruption
        for action in self.actions {
            action();
        }

        Ok(())
    }
}
```

About the `execute` method there are two things worth mentioning:

1. takes `self` - in other words - claims ownership over the `Shutdown` instance, which means that it will be deallocated at the end of the method body, and can no longer be used.
2. is decorated with the `async` keyword, which means that under the hood it returns a `Future` that needs to be polled by an async runtime in order to make progress. In this specific case of a shutdown it almost always makes sense to `block_on` this method.


# Drawbacks
* No known drawbacks; the proposal uses trait objects, that have a certain performance overhead, but this shutdown is a one-time operation, performance is not a major issue;

# Rationale and Alternatives
* No viable alternatives known to the author;

# Unresolved questions
* Should we have a forced shutdown after a certain time interval, in case some worker fails to terminate? When would such a thing actually happen?
