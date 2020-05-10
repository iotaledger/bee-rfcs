+ Feature name: `configuration`
+ Start date: 2020-05-06
+ RFC PR: [iotaledger/bee-rfcs#00](https://github.com/iotaledger/bee-rfcs/pull/00)
+ Bee issue: [iotaledger/bee#00](https://github.com/iotaledger/bee/issues/00)

# Summary

This RFC proposes a configuration pattern for Bee binary and library crates.

# Motivation

This RFC contains a set of recommendations regarding configuration management in order to ensure consistency across the
different Bee crates.

# Detailed design

## Libraries

- [serde](https://github.com/serde-rs/serde): the go-to framework for **ser**ializing and **de**serializing Rust data
  structures efficiently and generically;
- [toml-rs](https://github.com/alexcrichton/toml-rs): a TOML encoding/decoding library for Rust with
  serialization/deserialization on top of `serde`;

## Recommendations

With the following recommendations, one can:
- read a configuration builder from a file;
- manually override a configuration builder;
- construct the configuration;

### Configuration Builder

The [Builder](https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder) pattern is very common in Rust when
it comes to constructing complex objects like configurations.

A configuration builder type should:
- be named `*ConfigBuilder`;
- derive the following traits;
    - `Default` to implement the `new` method;
    - `Deserialize`, from `serde`, to deserialize from a configuration file;
- provide a `new` method;
- have `Option` fields;
    - if there is a configuration file, the fields should have the same names as the keys;
- provide setters to set/override the fields;
- provide a `build` method constructing the actual configuration object;
- have default values defined as `const` and set with `Option::unwrap_or`;

Here is a small example fitting all these requirements.

`config.toml`
```toml
[snapshot]
meta_file_path  = "./data/snapshot/mainnet.snapshot.meta"
state_file_path = "./data/snapshot/mainnet.snapshot.state"
```

`config.rs`
```rust
const DEFAULT_META_FILE_PATH: &str = "./data/snapshot/mainnet.snapshot.meta";
const DEFAULT_STATE_FILE_PATH: &str = "./data/snapshot/mainnet.snapshot.state";

#[derive(Default, Deserialize)]
pub struct SnapshotConfigBuilder {
    meta_file_path: Option<String>,
    state_file_path: Option<String>,
}

impl SnapshotConfigBuilder {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn meta_file_path(mut self, meta_file_path: String) -> Self {
        self.meta_file_path.replace(meta_file_path);
        self
    }

    pub fn state_file_path(mut self, state_file_path: String) -> Self {
        self.state_file_path.replace(state_file_path);
        self
    }

    pub fn build(self) -> SnapshotConfig {
        SnapshotConfig {
            meta_file_path: self.meta_file_path.unwrap_or(DEFAULT_META_FILE_PATH.to_string()),
            state_file_path: self.state_file_path.unwrap_or(DEFAULT_STATE_FILE_PATH.to_string()),
        }
    }
}
```

### Configuration

A configuration type should:
- be named `*Config`;
- Derive `Clone` to be able to provide ownership to different components;
- have the same fields with the same names, without `Option`, as the builder;
- be immutable: no setters and no public fields;
- have getters or `pub(crate)` fields;

Here is a small example fitting all these requirements:

`config.toml`
```toml
[snapshot]
meta_file_path  = "./data/snapshot/mainnet.snapshot.meta"
state_file_path = "./data/snapshot/mainnet.snapshot.state"
```

`config.rs`
```rust
#[derive(Clone)]
pub struct SnapshotConfig {
    meta_file_path: String,
    state_file_path: String,
}

impl SnapshotConfig {
    pub fn meta_file_path(&self) -> &String {
        &self.meta_file_path
    }

    pub fn state_file_path(&self) -> &String {
        &self.state_file_path
    }
}

```

### Read a configuration builder

`toml.rs` makes it very easy to read a configuration builder from a file.

```rust
let config_builder = match fs::read_to_string("config.toml") {
    Ok(toml) => match toml::from_str::<SnapshotConfigBuilder>(&toml) {
        Ok(config_builder) => config_builder,
        Err(e) => {
            panic!("Error parsing config file: {:?}", e);
        }
    },
    Err(e) => {
        panic!("Error reading config file: {:?}", e);
    }
};

// Override fields if necessary e.g. with CLI arguments.

let config = config_builder.build();
```

### Sub-configuration

It is also very easy to create sub-configurations by nesting configuration builders and configurations.

`config.toml`
```toml
[snapshot]
[snapshot.local]
meta_file_path  = "./data/snapshot/local/mainnet.snapshot.meta"
state_file_path = "./data/snapshot/local/mainnet.snapshot.state"
[snapshot.global]
file_path       = "./data/snapshot/global/mainnet.txt"
```

`config.rs`
```rust
#[derive(Default, Deserialize)]
pub struct LocalSnapshotConfigBuilder {
    meta_file_path: Option<String>,
    state_file_path: Option<String>,
}

#[derive(Default, Deserialize)]
pub struct GlobalSnapshotConfigBuilder {
    file_path: Option<String>,
}

#[derive(Default, Deserialize)]
pub struct SnapshotConfigBuilder {
    local: LocalSnapshotConfigBuilder,
    global: GlobalSnapshotConfigBuilder,
}

impl SnapshotConfigBuilder {
    pub fn new() -> Self {
        Self::default()
    }

    // Setters

    pub fn build(self) -> SnapshotConfig {
        SnapshotConfig {
            local: LocalSnapshotConfig {
                meta_file_path: self
                    .local
                    .meta_file_path
                    .unwrap_or(DEFAULT_LOCAL_SNAPSHOT_META_FILE_PATH.to_string()),
                state_file_path: self
                    .local
                    .state_file_path
                    .unwrap_or(DEFAULT_LOCAL_SNAPSHOT_STATE_FILE_PATH.to_string()),
            },
            global: GlobalSnapshotConfig {
                file_path: self
                    .global
                    .file_path
                    .unwrap_or(DEFAULT_GLOBAL_SNAPSHOT_FILE_PATH.to_string()),
            },
        }
    }
}
#[derive(Clone)]
pub struct LocalSnapshotConfig {
    meta_file_path: String,
    state_file_path: String,
}

#[derive(Clone)]
pub struct GlobalSnapshotConfig {
    file_path: String,
}

#[derive(Clone)]
pub struct SnapshotConfig {
    local: LocalSnapshotConfig,
    global: GlobalSnapshotConfig,
}

// Impl

```

# Drawbacks

No specific drawback with this approach.

# Rationale and alternatives

Many configuration formats are usually considered:
- [TOML](https://github.com/toml-lang/toml)
- [JSON](https://www.json.org/json-en.html)
- [HOCON](https://github.com/lightbend/config/blob/master/HOCON.md)
- [YAML](https://yaml.org/)
- [XML](https://www.w3.org/XML/)

This RFC and its expected implementation are choosing `TOML` as configuration format because it is the preferred option
in the Rust ecosystem. However, it is not excluded by RFC that other formats may be provided in the future, `serde`
making it very easy to support other formats.

`Serde` itself has been chosen because it is the standard for serialization/deserialization in Rust.

# Unresolved questions

In case of binary crates, e.g.`bee-node`, configuration with CLI arguments is not described in this RFC but everything
is already set up to support it seamlessy. The builder setters allow setting fields or overriding fields that may have
already prefilled by the parsing of a configuration file. A CLI parser library like
[clap](https://github.com/clap-rs/clap) may be used on top of the builders.
