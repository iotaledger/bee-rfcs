+ Feature name: `logging`
+ Start date: 2020-05-23
+ RFC PR: [iotaledger/bee-rfcs#35](https://github.com/iotaledger/bee-rfcs/pull/35)
+ Bee issue: [iotaledger/bee#78](https://github.com/iotaledger/bee/issues/78)

# Summary

This RFC introduces a logging library - `log` - and some recommendations on how to log in the Bee project.

# Motivation

Logging is done across almost all binary and library crates in the Bee project, so consistency is needed to provide a
better user experience.

# Detailed design

Logging in Bee should be done through the [log](https://crates.io/crates/log) crate (`A Rust library providing a
lightweight logging facade`) which is the de-facto standard library for logging in the Rust ecosystem.

The `log` crate itself is just a frontend and supports the implementation of many different backends.

> It provides a single logging API that abstracts over the actual logging implementation. Libraries can use the logging
API provided by this crate, and the consumer of those libraries can choose the logging implementation that is most
suitable for its use case.

## Frontend

> Libraries should link only to the log crate, and use the provided macros to log whatever information will be useful
to downstream consumers.

The `log` crate provides the following macros, from lowest priority to highest priority:

- `trace!`
- `debug!`
- `info!`
- `warn!`
- `error!`

These macros have a usage similar to the `println!` macro.

A log request consists of a `target`, a `level`, and a `message`.

By default, the target represents the location of the log request, but may be overridden.

Example with default target:
```rust
info!("Connected to port {} at {} Mb/s", conn_info.port, conn_info.speed);
```

Example with overridden target:
```rust
info!(target: "connection_events", "Successful connection, port: {}, speed: {}", conn_info.port, conn_info.speed);
```

## Backend

> Executables should choose a logging implementation and initialize it early in the runtime of the program. Logging
implementations will typically include a function to do this. Any log messages generated before the implementation is
initialized will be ignored.

There are a lot of [available backends](https://docs.rs/log/0.4.8/log/#available-logging-implementations) for the `log`
frontend.

This RFC opts for the [fern](https://docs.rs/fern) backend which is a very complete one with an advanced configuration.

### Initialization

The backend API is limited to one function call to initialize the logger. It is designed to be updatable without
breaking changes by taking a `LoggerConfig` configuration object and returning a `Result`.

```rust
fn logger_init(config: LoggerConfig) -> Result<(), LoggerError>;
```

### Configuration

The following configuration - compliant with [RFC 31](https://github.com/iotaledger/bee-rfcs/blob/master/text/0031-configuration.md) -
primarily allows configuring different outputs with different log levels.

`config.rs`
```rust
#[derive(Clone)]
struct LoggerOutputConfig {
    name: String,
    level: LevelFilter,
    ...
}

#[derive(Clone)]
struct LoggerConfig {
    color: bool,
    outputs: Vec<LoggerOutputConfig>,
    ...
}
```

`config.toml`
```toml
[logger]
color = true
[[logger.outputs]]
name  = "stdout"
level = "info"
[[logger.outputs]]
name  = "errors.log"
level = "error"
```

**Note**: `stdout` is a reserved name for the standard output of the console.

### Errors

Since different backends may have different errors, we need to abstract them to provide a consistent `logger_init`
function.

```rust
#[derive(Debug)]
#[non_exhaustive]
enum LoggerError {
    ...
}
```

**Note**: The `LoggerError` enum has to be `non_exhaustive` to allow further improving / extending the logger without
risking a breaking change.

## Format

The following elements should appear in this order:
- Date in the `%Y-%m-%d` format e.g. `2020-05-25`;
- Time in the `%H:%M:%S` format e.g. `10:23:03`;
- Target e.g. `bee_node::node`;
    - The default target is preferred but can be overridden if needed;
- Level e.g. `INFO`;
- Message e.g. `Initializing...`;

All elements should be enclosed in brackets `[...]` with only a space between the level and the message.

Example: `[2020-05-25][10:23:03][bee_node::node][INFO] Initializing...`

**Note**: All messages should end either with `...` (3 dots) to indicate something potentially long-lasting is
happening, or with `.` (1 dot) to indicate events that just happened.

# Drawbacks

No specific drawbacks using this library.

# Rationale and alternatives

- The `log` crate is maintained by the rust team, so it is a widely used and trusted dependency in the Bee framework;
- Most of the logging crates in the Rust ecosystem are actually backends for the `log` crate with different features;
- It should be very easy to switch to a different backend in the future by only changing the initialization function;
- There are a lot of [available backends](https://docs.rs/log/0.4.8/log/#available-logging-implementations) but `fern`
  offers a very fine-grained logging configuration;

# Unresolved questions

There are no open questions as this RFC is pretty straightforward and the topic of logging is not a complex one.
