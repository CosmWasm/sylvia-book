# Good practices

All the relevant basics are covered. Now let's talk about some good practices.

## JSON renaming

Due to Rust style, all our message variants are spelled in a camel case. It is standard practice,
but it has a drawback - all messages are serialized and deserialized by the serde using those
variants names. The problem is that it is more common to use snake cases for field names in the
JSON world. Luckily there is an effortless way to tell the serde, to change the names casing for
serialization purposes. I mentioned it earlier when talking about query messages -
`#[serde(rename_all = "snake_case")]`. Sylvia will automatically generate it for you in case of
messages. Unfortunately, in case of responses to your messages, you will have to do it by yourself.
Let's update our response with this attribute:

```rust,noplayground
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone, PartialEq, Eq, schemars::JsonSchema, Debug, Default)]
#[serde(rename_all = "snake_case")]
pub struct AdminListResp {
    pub admins: Vec<String>,
}
```

Looking at our `AdminListResp`, you might argue that all these derive look too clunky, and I agree.
Luckily the [`cosmwasm-schema`](https://docs.rs/cosmwasm-schema/latest/cosmwasm_schema/index.html)
crate delivers `cw_serde` macro, which we can use to reduce a boilerplate:

```rust,noplayground
use cosmwasm_schema::cw_serde;

#[cw_serde]
pub struct AdminListResp {
    pub admins: Vec<String>,
}
```

## JSON schema

Talking about JSON API, it is worth mentioning JSON Schema. It is a way of defining the shape of
JSON messages. It is a good practice to provide a way to generate schemas for contract API.
The problem is that writing JSON schemas by hand is a pain. The good news is that there is a crate
that would help us with that. We have already used it before, and it is called
[`schemars`](https://docs.rs/schemars/latest/schemars/). Sylvia will force you to add this
`derive` to your responses and will generate messages with it.

The only thing missing is new a `crate-type` in our `Cargo.toml`:

```toml,noplayground
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
library = []

[dependencies]
cosmwasm-std = { version = "2.0.4", features = ["staking"] }
cosmwasm-schema = "2.0.4"
serde = { version = "1.0.147", features = ["derive"] }
sylvia = "1.1.0"
schemars = "0.8.16"
thiserror = "1.0.37"
cw-storage-plus = "2.0.0"
cw-utils = "2.0.0"

[dev-dependencies]
anyhow = "1"
cw-multi-test = "2.1.0"
```

I added `rlib`. `cdylib` crates cannot be used as typical Rust dependencies. As a consequence, it is
impossible to create examples for such crates.

The next step is to create a tool generating actual schemas. We will do it by creating a binary in
our crate. Create a new `bin/schema.rs` file:

```rust,noplayground
use contract::contract::sv::{ContractExecMsg, ContractQueryMsg, InstantiateMsg};
use cosmwasm_schema::write_api;

fn main() {
    write_api! {
        instantiate: InstantiateMsg,
        execute: ContractExecMsg,
        query: ContractQueryMsg,
    }
}
```

Notice that I used here `ContractExecMsg` and `ContractQueryMsg` instead of `ExecMsg` and `QueryMsg`.
It is important as the latter will not expose the `interface` messages.

Cargo is smart enough to recognize files in the `src/bin` directory as utility binaries for the crate.
Now we can generate our schemas:

```bash
cargo run schema
   Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/schema schema`
Exported the full API as /home/janw/workspace/confio/sylvia-book-contract/schema/contract.json
```

I encourage you to go to generated file to see what the schema looks like.

The problem is that unfortunately creating this binary makes our project fail to compile on the Wasm
target - which is the most important in the end. Hopefully, we don't need to build the schema
binary for the Wasm target - let's align the `.cargo/config` file:

```toml,noplayground
[alias]
wasm = "build --target wasm32-unknown-unknown --release --lib"
wasm-debug = "build --target wasm32-unknown-unknown --lib"
schema = "run schema"
```

The `--lib` flag added to wasm cargo aliases tells the toolchain to build only the library target - it
would skip building any binaries. Additionally, I added the convenience schema alias to
generate schema calling simply cargo schema.

## Disabling entry points for libraries

Since we added the `rlib` target for the contract, it is, as mentioned before, usable as a
dependency. The problem is that the contract depending on ours, would have Wasm entry points
generated twice - once in the dependency and once in the final contract. We can work this around
by disabling generating Wasm entry points for the contract if the crate is used as a dependency.
We would use [`feature flags`](https://doc.rust-lang.org/cargo/reference/features.html) for that.

Start with updating `Cargo.toml`:

```toml,noplayground
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
library = []

[dependencies]
cosmwasm-std = { version = "2.0.4", features = ["staking"] }
sylvia = "1.1.0"
schemars = "0.8.16"
cosmwasm-schema = "2.0.4"
serde = "1.0.180"
cw-storage-plus = "2.0.0"
thiserror = "1.0.44"

[dev-dependencies]
sylvia = { version = "1.1.0", features = ["mt"] }
cw-multi-test = "2.1.0"
```

This way, we created a new feature flag for our crate. Now we want to disable the `entry_points`
attribute if our contract would be used as a dependency. We will do it by a slight update of our
`src/contract.rs`:

```rust,noplayground
#[cfg_attr(not(feature = "library"), entry_points)]
#[contract]
impl CounterContract {
}
```

The [`cfg_attr`](https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute)
is a conditional compilation attribute, similar to the `cfg` we used before for
the test. It expands to the given attribute if the condition expands to true. In our case - it would
expand to nothing if the feature "library" is enabled, or it would expand just to `#[entry_point]`
in another case.

Since now to add this contract as a dependency, don't forget to enable the feature like this:

```toml,noplayground
[dependencies]
my_contract = { version = "0.1", features = ["library"] }
```
